# MASSO Link UDP Protocol Specification

**⚠️ WORK IN PROGRESS - INCOMPLETE DOCUMENTATION**

**Note**: This protocol specification is based on reverse engineering, packet captures, and controller testing. Many packet fields and status bytes are still not fully understood. Treat this as a working document, not official MASSO documentation.

---

This document describes the observed MASSO controller UDP protocol used by MASSO Link and compatible clients.

## Overview

- **Transport**: UDP
- **Controller IP**: User-specified
- **Controller Port**: UDP `65535`
- **Client Ports**: UDP `11000-11050`
  - MASSO documentation identifies these as client bind/send ports.
  - Some MASSO Link captures show commands/uploads sent from an ephemeral Windows source port while status/ACK traffic returns to client port `11000`.
  - For third-party clients, binding to the documented `11000-11050` range is recommended unless further testing proves otherwise.
- **Packet Structure**:

```text
[CRC16-CCITT 2 bytes][Magic 0x03 0x00 2 bytes][Type 1 byte][Payload...]
```

- **Checksum Input**: all bytes after the checksum field, starting with `0x03 0x00`.

## Packet Types

### Discovery / Version Request (Type `0x02`)

#### Request

Total packet length: 10 bytes including checksum.

```text
[CRC16][03 00][02][f8 2a 00 00 ??]
```

Known/observed payload:

```text
f8 2a 00 00 0b
```

In later MASSO Link captures, the final byte matched the current month, for example:

```text
f8 2a 00 00 06
```

where `0x06` was observed during June testing. This final byte may be a date/month field rather than a fixed constant.

#### Response

- Length: 46 bytes
- Version string starts at byte 12.
- Example strings observed/expected:
  - `@Lathe v5.09`
  - MASSO Touch / Standalone version strings may vary by firmware/core version.

---

### Configuration Request (Type `0x03`)

#### Request

Total packet length: 14 bytes including checksum.

```text
[CRC16][03 00][03][hour minute second day month year 00 00 00]
```

Earlier testing showed the controller will still respond if these fields are zeroed. However, MASSO Link appears to use these fields to set/sync the MASSO clock. Sending zeros can result in the controller clock being set to `12:00 AM`.

Observed MASSO Link example:

```text
03 00 03 0b 01 15 04 06 1a 00 00 00
```

Decoded:

```text
hour   = 0x0b = 11
minute = 0x01 = 1
second = 0x15 = 21
day    = 0x04 = 4
month  = 0x06 = 6
year   = 0x1a = 26
```

Meaning:

```text
11:01:21 on 04/06/26
```

#### Response

- Length: 10 bytes
- Type: `0x03`
- Bytes 5-6: controller serial number, little-endian.

Example extraction:

```python
serial = int.from_bytes(data[5:7], "little")
```

---

### Keepalive / Status Request (Type `0x01`)

#### Request

Total packet length: 10 bytes including checksum.

```text
[CRC16][03 00][01][hour minute second day month]
```

MASSO Link sends this roughly once per second after connection.

Observed examples:

```text
03 00 01 0b 01 15 04 06
```

Decoded:

```text
11:01:21 on day 04, month 06
```

The keepalive does not include the year byte. The year is present in the configuration request.

#### Response

- The controller sends 270-byte status packets.

---

### Tool Data Request (Type `0x08`)

#### Request

Total packet length: 10 bytes including checksum.

```text
[CRC16][03 00][08][tool_index][22 2c 1c 0b]
```

- `tool_index`: 1 byte, usually `1-255`.

#### Response

- Length: 38 bytes
- Type: `0x08`
- Byte 5: tool index
- Byte 6 onward: ASCII tool name, null-terminated.

---

## Status Packet Structure

Status packets are 270 bytes.

Known fields:

| Byte(s) | Meaning | Notes |
|---:|---|---|
| 0-1 | CRC/checksum | Little-endian CRC16-CCITT |
| 2-3 | Magic | `03 00` |
| 4 | Packet type | Often `0x01` for status |
| 5 | Job progress percentage | `0-100`, e.g. `0x64 = 100%` |
| 6 | Execution active flag | `0x00 = not running`, `0x02 = running` |
| 7 | Fault/status code | `0xFF = normal/no fault observed`, `0x15 = torch breakaway observed` |
| 8-11 | Job count | Little-endian integer |
| 12 | User prompt / tool-change waiting flag | `0x01 = normal`, `0x00 = waiting for user input` |
| 13 | Line number | Single byte, wraps at 255 |
| 14-16 | Reserved/unknown | Usually `00 00 00` in observed captures |
| 17-80 | Current/last file path/name | ASCII, null-terminated |
| 81-269 | Unused/padding/unknown | Usually zero in normal operation |

### Byte 5 - Job Progress

- Decimal `0-100`.
- During idle/stopped, it may remain at the last completed value, often `100`.
- During running, MASSO Link displays this as the running percentage.

### Byte 6 - Execution Active Flag

Observed values:

```text
0x00 = not running / stopped / idle
0x02 = actively running
```

A brief `0x00` blip was observed during a running capture, so applications should debounce before re-enabling upload.

Recommended upload-enable logic:

```text
Upload allowed only when:
- connected
- byte 6 == 0x00
- byte 7 == 0xFF
- stopped state has been stable for ~1.5 seconds
- no upload is already active
```

### Byte 7 - Fault / Status Code

Earlier notes listed byte 7 as fixed `0xFF`, but breakaway testing disproved that.

Observed values:

```text
0xFF = normal / no fault observed
0x15 = torch breakaway active
```

This byte should be treated as a status/fault/alarm code field. Additional codes are unknown.

### Byte 12 - User Prompt / Tool Change

Observed/working values:

```text
0x01 = normal operation
0x00 = machine paused, waiting for user input, such as M6/manual tool change
```

### Byte 13 - Line Number

- Single-byte line/index value.
- Can wrap at 255.
- Useful for activity detection, but not a full G-code line number.

---

## Feed Hold Detection

Feed hold is not fully decoded yet.

Existing client logic infers feed hold when:

1. Execution active flag is running (`byte 6 == 0x02`)
2. Line number has not changed for ~1.5 seconds
3. Line number is greater than 0

MASSO Link appears able to display “Feed Hold,” but it is not yet confirmed whether it uses this same inference or a separate status byte.

---

## File Upload - Start Upload (Type `0x0A`)

There appear to be at least two start-upload formats.

### Simple / Short Start Upload Format

Earlier observed/documented format:

- Total packet length: 30 bytes including checksum.
- Payload length after checksum: 28 bytes.

```text
[CRC16][03 00][0A][file_size 4 LE][00 00 01][5c 00][filename NUL][padding]
```

Notes:

- This form appears to be limited by the fixed packet size.
- Earlier client implementations assumed a 15-character filename limit.
- This may be a simplified/root-upload format.

### Folder-Aware Start Upload Format

MASSO Link captures showed a longer folder-aware start-upload packet.

Observed total packet lengths include:

```text
38 bytes
50 bytes
```

This format carries folder path and filename separately and supports paths such as:

```text
\Test\
```

and filenames longer than the original assumed 15-character limit.

Known-good observed filename:

```text
18_Inch__CLAD.nc
```

This is 16 characters including `.nc`, so the 15-character limit is not universal.

### Start Upload Response

Response length: 10 bytes.

Type: `0x0A`.

The response type alone is not enough to determine success. Bytes 5-6 appear to carry the status/result code.

Observed:

```text
bytes 5-6 = 00 00  => start upload accepted
bytes 5-6 = f7 00  => start upload rejected / failed
```

Example accepted ACK:

```text
?? ?? 03 00 0a 00 00 ff ?? ??
```

Applications should check both the ACK type and status bytes.

Recommended logic:

```python
start_ok = (ack[4] == 0x0A and ack[5:7] == b"\x00\x00")
```

---

## File Upload - Data Chunk (Type `0x0B`)

### Full Chunk Packet

Normal full chunks use 1422 bytes of data.

Total UDP payload length:

```text
1438 bytes
```

Structure:

```text
[CRC16 2]
[03 00 2]
[0B 1]
[chunk_index 4 LE]
[chunk_length 4 LE]
[data 1422]
[pad 3]
```

For full chunks:

```text
chunk_length = 1422
trailing pad = 3 bytes
total length = 2 + 2 + 1 + 4 + 4 + 1422 + 3 = 1438
```

### Final Short Chunk Packet

Successful MASSO Link and SendToMasso captures confirmed that the final short chunk is formatted differently than earlier notes.

For a final chunk shorter than 1422 bytes:

```text
chunk_length = actual remaining byte count
data = actual remaining bytes
trailing pad = 4 bytes
total length = 17 + remaining_bytes
```

Because:

```text
2 checksum
2 magic
1 type
4 chunk index
4 chunk length
N data
4 final trailing bytes
= 17 + N
```

Example confirmed successful upload:

```text
File size: 49,369 bytes
Full chunks: 34
Final chunk index: 34
Final chunk length: 1021
Final UDP payload length: 1038
```

Calculation:

```text
17 + 1021 = 1038
```

A previous failed implementation used only 3 trailing bytes on the final short chunk; MASSO did not ACK that final chunk. Adding the 4th trailing byte made the upload succeed.

### Data Chunk Response

Response length: 10 bytes.

Type: `0x0B`.

The ACK appears to contain the next expected chunk number.

Observed behavior:

```text
Send chunk 0  -> ACK indicates 1
Send chunk 1  -> ACK indicates 2
Send chunk 34 -> ACK indicates 35
```

Example final ACK:

```text
45 a7 03 00 0b 00 23 00 00 00
```

`0x23` is decimal `35`, indicating MASSO accepted chunk 34 and advanced to expected chunk 35.

Working interpretation:

```python
ack_next_chunk = int.from_bytes(ack[5:7], "big")
expected_next = chunk_index + 1
chunk_ok = (ack[4] == 0x0B and ack_next_chunk == expected_next)
```

This byte order should be verified with more captures, but it matched observed chunk ACKs.

---

## File Upload Process

Recommended process:

1. Connect to controller.
2. Confirm status packets are being received.
3. Do not upload while machine is running or faulted.
4. Send start-upload packet with folder/path, filename, and file size.
5. Wait for type `0x0A` ACK.
6. Confirm ACK status bytes are `00 00`.
7. Send full data chunks:
   - chunk length field = `1422`
   - 1422 data bytes
   - 3 trailing pad bytes
8. Send final short chunk:
   - chunk length field = actual remaining bytes
   - actual remaining bytes of data
   - 4 trailing pad bytes
9. Wait for type `0x0B` ACK after each chunk.
10. Confirm ACK advances to the next expected chunk number.

---

## Filename and Folder Notes

- ASCII filenames are used in observed captures.
- Backslash `\` is used as the path separator.
- Forward slash `/` should not be used.
- Directories must already exist on MASSO; no folder-create packet has been decoded.
- The earlier 15-character filename limit applies only to the short fixed-size start-upload format and should not be treated as universal.
- The folder-aware format supports longer filenames and separate target folders.

---

## Error Handling Notes

- Start-upload ACK type `0x0A` may still indicate failure; inspect bytes 5-6.
- If start-upload status is `f7 00`, treat as rejected/failed and do not send chunks.
- If a chunk ACK is not received, resend the same chunk.
- MASSO Link appears to retry chunks quickly.
- The final chunk is especially sensitive to packet length/padding.

---

## Checksum Calculation

CRC16-CCITT:

- Polynomial: `0x1021`
- Initial value: `0x0000`
- Input data: all bytes after checksum
- Output: little-endian 2 bytes

Example:

```python
def calculate_checksum(data: bytes) -> bytes:
    crc = 0x0000
    poly = 0x1021

    for byte in data:
        crc ^= byte << 8
        for _ in range(8):
            if crc & 0x8000:
                crc = (crc << 1) ^ poly
            else:
                crc <<= 1
            crc &= 0xFFFF

    return crc.to_bytes(2, "little")
```

---

## Implementation Notes

- Keepalive packets are normally sent about once per second.
- MASSO status packets arrive frequently, around 10 packets per second in observed captures.
- The UI should disable uploads when the machine is running, faulted, or recently stopped.
- A stopped debounce of about 1.5 seconds is recommended.
- Store last successful upload time locally if showing “file sent X seconds ago”; this appears to be a UI-side MASSO Link feature, not a decoded controller timestamp.

---

## Known Observed Values

### Status Packet

| Field | Value | Meaning |
|---|---:|---|
| Byte 5 | `0x00-0x64` | Progress percent |
| Byte 6 | `0x00` | Not running |
| Byte 6 | `0x02` | Running |
| Byte 7 | `0xFF` | Normal/no fault observed |
| Byte 7 | `0x15` | Torch breakaway |
| Byte 12 | `0x01` | Normal/no user prompt |
| Byte 12 | `0x00` | Waiting for user input/tool change |

### Upload ACK

| Packet Type | Bytes 5-6 | Meaning |
|---|---|---|
| `0x0A` | `00 00` | Start upload accepted |
| `0x0A` | `f7 00` | Start upload rejected/failed |
| `0x0B` | next chunk | Data chunk accepted, next expected chunk |

---

## Open Questions

- Complete mapping of byte 7 fault/status codes.
- Feed hold: inferred from stalled line number or represented by a dedicated byte?
- Exact folder-aware start-upload packet structure.
- Whether MASSO officially expects client TX to be bound to `11000-11050`, or whether MASSO Link's ephemeral TX source port is valid/intentional.
- Whether the discovery packet final byte is always the current month.
- Maximum confirmed filename/path lengths for folder-aware upload.
- Whether remote directory listing, delete, rename, or browse packets exist.

---
