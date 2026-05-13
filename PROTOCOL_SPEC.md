# MASSO Link UDP Protocol Specification

**⚠️ WORK IN PROGRESS - INCOMPLETE DOCUMENTATION**

**Note**: This protocol specification is based on reverse engineering and packet analysis. Many packet fields and status bytes are not fully understood or documented. This document represents the current state of knowledge and may contain errors or omissions.

---

This document describes the MASSO controller UDP protocol as implemented in `masso_udp_client.py`.

## Overview

- **Transport**: UDP
- **Controller IP**: User-specified (required via --host argument)
- **Send Ports**: 11000-11050 (client binds to one of these)
- **Receive Port**: 65535 (controller broadcasts from this port)
- **Packet Structure**: `[CRC16-CCITT 2 bytes][Magic 0x03 0x00 2 bytes][Type 1 byte][Payload...]`

## Packet Types

### Discovery/Version (Type 0x02)
- **Request**: 8 bytes payload
  - Magic: `0x03 0x00`
  - Type: `0x02`
  - Payload: `0xf8 0x2a 0x00 0x00 0x0b`
- **Response**: 46 bytes
  - Contains version string starting at byte 12

### Configuration Request (Type 0x03)
- **Request**: 14 bytes
  - Magic: `0x03 0x00`
  - Type: `0x03`
  - Unknown: 9 bytes (unused by controller - can send zeros)
- **Response**: 10 bytes
  - Magic: `0x03 0x00`
  - Type: `0x03`
  - Serial Number: 2 bytes (little-endian)
### Keepalive/Status Request (Type 0x01)
- **Request**: 8 bytes payload
  - Magic: `0x03 0x00`
  - Type: `0x01`
  - Unknown: 5 bytes (possibly timestamp, unused by controller - can send zeros)
- **Response**: 270 bytes (status packet)

### Tool Data Request (Type 0x08)
- **Request**: 8 bytes payload
  - Magic: `0x03 0x00`
  - Type: `0x08`
  - Tool Index: 1 byte (1-255)
  - Payload: `0x22 0x2c 0x1c 0x0b`
- **Response**: 38 bytes
  - Tool index at byte 5
  - Tool name starts at byte 6, null-terminated

### File Upload - Start Upload (Type 0x0A)
- **Request**: 30 bytes total (28 bytes payload + 2 bytes checksum)
  - Magic: `0x03 0x00`
  - Type: `0x0A`
  - File Size: 4 bytes (little-endian)
  - Unknown: 3 bytes `0x00 0x00 0x01`
  - Backslash + Null: `0x5c 0x00`
  - Filename: ASCII string, null-terminated (max 15 characters)
  - Padding: Zeros to 28 bytes payload
- **Response**: 10 bytes

### File Upload - Data Chunk (Type 0x0B)
- **Request**: Variable length
  - Magic: `0x03 0x00`
  - Type: `0x0B`
  - Chunk Index: 4 bytes (little-endian, starts at 0)
  - Chunk Length: 4 bytes (little-endian, typically 1422)
  - Data: 1422 bytes (padded with zeros if needed)
  - Padding: 3 bytes `0x00 0x00 0x00`
- **Response**: 10 bytes

## Controller Identification

The MASSO controller provides identification information through specific packet responses:

### Serial Number Discovery

**Location**: Configuration Response (Type 0x03, 10 bytes)
- **Bytes 5-6**: Serial number stored in little-endian format
- **Range**: 0-65535 (16-bit unsigned integer)
- **Example**: Serial "G3-12345" → numeric part "12345" stored as `0x30 0x39`

**Packet Structure**:
```
Bytes 0-1: Unknown (checksum-related or header)
Bytes 2-3: Packet type confirmation (0x03)
Byte 4:    Unknown field (often 0x03)
Bytes 5-6: Serial number (little-endian)
Bytes 7-9: Reserved (typically 0x00)
```

**Implementation Notes**:
- Serial number is only available in the configuration response
- Must be extracted during initial connection sequence
- Numeric serial range suggests 16-bit unsigned integer format
- The "G3-" prefix (or similar) is not stored in the packet - only numeric portion

**Usage Example**:
```python
# Extract serial from configuration response
serial = int.from_bytes(data[5:7], 'little')  # 12345
print(f"Controller Serial: {serial}")
```

### Version Information

**Location**: Discovery/Version Response (Type 0x02, 46 bytes)
- **Bytes 12+**: Version string (ASCII, null-terminated)
- **Example**: `@Lathe v5.09`

## Status Packet Structure (270 bytes)

> [!NOTE]
> The following field mappings have been reverse-engineered. Recent tests on newer firmware (v5.10) reveal the true purpose of several bytes.

Key fields:
- Byte 5: Job Progress Percentage
  - Decimal value from `0` to `100` (e.g., `0x64` = `100%`)
  - Increments steadily during execution
  - Remains at the last value if execution is interrupted
- Byte 6: Execution Active Flag
  - `0x00`: Not Running (Idle, Feed Hold, or E-Stop)
  - `0x02`: Actively Running
  - **Note**: Internal operations like Homing or Probing often appear as "Running" and may execute internal macros.
- Byte 7: Always `0xFF` — fixed delimiter/padding, does not change
- Bytes 8-11: Job count (little-endian)
- Byte 12: User Prompt Waiting Flag (Tool Change, M0, M1, etc.)
  - `0x01`: Normal operation
  - `0x00`: Machine paused, waiting for user input (e.g., manual tool change) and Cycle Start
- Byte 13: Line number (0–255, single byte)
  - **Note**: During Homing, this typically increments as MASSO runs its internal homing macro.
- Bytes 14–16: Always `0x00` — reserved/unused in observed captures
- Bytes 17-80: Filename (null-terminated, up to 63 bytes)
- Bytes 81-269: Unused/Padded with `0x00` during normal operation

## Feed Hold Detection
Because Feed Hold and E-Stop states simply set Byte 6 to `0x00` while Byte 5 freezes at its current progress percentage, these bytes alone cannot distinguish the exact stop reason. The client infers a feed hold state when:
1. The Execution Active Flag is Running (`0x02`)
2. The Line Number hasn't changed for 1.5 seconds or more
3. The Line Number is greater than 0

## Checksum Calculation

CRC16-CCITT algorithm:
- Polynomial: 0x1021
- Initial value: 0x0000
- Input data: Magic + Type + Payload (all bytes after checksum)
- Output: Little-endian 2 bytes

## File Upload Process

1. Send Start Upload packet with filename and file size
2. Wait for ACK (Type 0x0A, 10 bytes)
3. Send file in chunks of 1422 bytes each
4. Each chunk includes chunk index (not byte offset)
5. Wait for ACK after each chunk (Type 0x0B, 10 bytes)
6. Last chunk is padded with zeros to 1422 bytes

## Filename Restrictions

- Maximum length: 15 characters
- ASCII encoding only
- Subdirectories: Use backslash `\` separator
  - Example: `MASSO\file.nc` or `\MASSO\file.nc`
  - Forward slash `/` is not supported
- Directories must exist on MASSO (not created automatically)

## Error Handling

- Upload packets are retried up to 3 times
- Timeout for ACK response: 2.0 seconds
- Filename validation prevents uploads of files with names > 15 characters

## Implementation Notes

- Client binds to first available port in range 11000-11050
- Listen thread runs in background to receive broadcasts
- Keepalive packets sent every 1.0 second when connected
- All packets use little-endian byte order for multi-byte fields

## Unused Protocol Fields

### Unknown 9-Byte Fields (Configuration Request)

**Location**: Configuration Request (Type 0x03) - bytes 5-13

**Finding**: These 9 bytes are **completely ignored** by the MASSO controller.

**Test Results**:
- **Original values** (`0x0d143a1c0b19000000`): Controller responds normally
- **Modified values** (`0x0000000000FFFFFFFF`): Controller responds normally  
- **All zeros** (`0x000000000000000000`): Controller responds normally
- **Random values**: Controller responds normally

**Conclusion**: The controller does not read, validate, or use these 9 bytes for any purpose.

**Implementation Recommendation**:
- Send zeros for maximum simplicity and clarity
- No impact on functionality or reliability
- Reduces code complexity significantly

**Packet Structure**:
```
CLIENT: [checksum][0300][03][000000000][0000]  # Config request (all zeros)
CLIENT: [checksum][0300][01][0000000000]       # Keepalive (5 zeros)
CONTROLLER: [checksum][0300][03][1682][000000] # Response with serial
```

The controller always responds with its own data structure regardless of these field values.

**Note**: The keepalive packet still uses 5 unknown bytes (not 9) as it's a shorter packet structure.

---

