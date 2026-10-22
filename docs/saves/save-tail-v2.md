# Save tail v2: non-retail state inside a retail `.sav`

Status: **implemented in OpenBA** (`main`, commit `b4c274c07`, 2026-10-21); proposed here as the shared
way for an engine, hook or tool to keep its own state in a save that retail Banished still opens.
Version 1 (2026-10-12) is still read; nothing writes it any more.

The reference implementation is [`src/ext/savetail.h`](https://github.com/fran-cap/OpenBA/blob/main/src/ext/savetail.h)
and `savetail.cpp` in OpenBA; the rationale and the retail evidence are in OpenBA's
`docs/extensions/handins/SAVETAIL.md`, `EXTSAVE.md` and `EXTHOSTS.md`.

## 1. Where the bytes go

### The retail file

A Banished `.sav` is written by the Runtime's persistence worker:

```
u32 fileHeader   version(16) | blockCount(16)      version 2, blockCount 2
u32 magic        0x80808080
u32 crc          CRC-32 over the descriptors, then each block's payload
blockCount x { u32 id, u32 offset, u32 size }
block 0          the metadata record (name, thumbnail, mod list)
block 1          the game state: one compressed stream written by GameUpdateLoop::SaveRestore
```

### Not at the end of the file

Retail's save-list job requires the file size to equal `12 + 12 * blockCount + sum(block sizes)`
exactly, and a third block fails the block-count check. A file with bytes appended after its last block
is listed with an empty description, a black thumbnail and an empty mod list. **No byte may follow the
last block, and no third block may be added.**

### Inside block 1, after the last retail Serialize

Retail's reader stops after the master serializer's own 4-byte trailer and closes the compressed stream
without comparing its position with the stream's size. Bytes written into block 1's compressed stream
after the last retail `Serialize` and before the stream closes are therefore:

- ignored by retail (the save loads and plays as if they were not there);
- covered by the block's size and the file CRC automatically, because both are computed over whatever
  block 1 holds.

The tail is written there, as raw bytes through the compressed stream (no retail `IO::Serializer`
header around it). A reader takes every byte from its position after the last retail `Serialize` to the
end of the decompressed block-1 stream.

### Zero bytes when there is nothing to say

A writer with no state to record writes **nothing**: no magic, no header. A save with no tail is
byte-identical to a retail save, which is what lets a byte-for-byte resave check hold for an unmodified
game.

## 2. The chunk (version 2)

Little-endian throughout.

```
u32  magic           0x53545845   ("EXTS" as bytes: 45 58 54 53)
u32  version         2
u32  count
count x {
  u8   kind          1 = extension record, 2 = host record (section 3)
  u8   flags         bit 0 = must-understand; other bits 0 on write, ignored on read
  u32  idLength      1..64
  u8   id[idLength]  [a-z0-9_]+, no NUL
  i32  formatVersion >= 1, the version of this record's payload layout
  u32  payloadLength
  u8   payload[payloadLength]
}
u32  bodyLength      byte count of everything above, magic through the last payload
u32  crc             CRC-32 of those bytes
```

- **CRC-32** is IEEE 802.3: reflected, polynomial `0xEDB88320`, init and xorout `0xFFFFFFFF`
  (zlib's `crc32`). Check value: `"123456789"` -> `0xCBF43926`.
- **The 8-byte trailer is fixed for every version.** A reader finds it from the end, so a cut tail
  reads as truncated and a damaged one as a bad checksum before any record is trusted.
- **Version 1** has no `kind` and `flags` bytes; every version-1 record is an extension record with
  must-understand set. Otherwise identical.

## 3. Record kinds

| kind | name | flags written | meaning |
| --- | --- | --- | --- |
| 1 | extension | must-understand | A rule set the town was played under. The set of extension records is the town's **lock set**: a reader that does not know an id, or knows an older `formatVersion`, must not run the town. The payload is that extension's per-town state, or empty when it has none. |
| 2 | host | none | Per-town **presentation** state of a shared surface (in OpenBA: the extension toolbar window's position and page). Never part of the lock set and never fatal. |
| other | reserved | either | A later kind needs no new chunk version: an unknown kind without must-understand is dropped, an unknown kind with it makes the save unloadable. |

The test for which kind a piece of state is: if a reload must restore it for the town to play on as
saved, it is an extension record; if losing it only resets something the player sees, it is a host
record.

Ids of the two kinds are separate namespaces. Host records are written only beside at least one
extension record, so a town with no locked rule set still writes zero bytes.

## 4. Writing

1. Collect one extension record per active rule set, then one host record per host, in a stable order.
2. If there is no extension record, write nothing (section 1) and stop.
3. Encode as in section 2, with `version` 2, and write the bytes into block 1's compressed stream
   after the last retail `Serialize` and before the stream closes.

A payload's encoder may run on a save worker thread (OpenBA's background save does): it reads only
plain state owned by its own extension or host, never UI objects.

## 5. Reading

Take the bytes from the position after the last retail `Serialize` to the end of the decompressed
block-1 stream, then:

| Step | Check | Outcome on failure |
| --- | --- | --- |
| 1 | No bytes left | A retail or tail-free save: load normally. |
| 2 | The bytes start with the magic | Not a tail: ignore the bytes (log), load normally. |
| 3 | At least 20 bytes; `bodyLength` equals the size minus 8 | Truncated: **refuse the save**. |
| 4 | CRC of the body matches | Bad checksum: **refuse**. |
| 5 | `version` is 1 or 2 | Newer chunk version: **refuse**. |
| 6 | Every record parses inside the body, ids valid, `formatVersion` >= 1, no bytes left over | Malformed: **refuse**. |
| 7 | Split by kind | Unknown kind with must-understand: **refuse**. Without: drop it (log). |
| 8 | Every extension id known to this reader, `formatVersion` not newer than the reader's, no id twice | **Refuse.** |
| 9 | Lock the town's rule set to exactly the saved extension ids | |
| 10 | Hand each extension payload to its decoder | Decoder rejects it: **refuse** (the state is damaged). |
| 11 | Hand each host payload to its decoder | Unknown host, newer host `formatVersion`, or rejected payload: drop it, reset that host to defaults, log. Never refuse. |

"Refuse" means the town is not played on. OpenBA shows the engine's fatal error box at this point,
because the world is already deserialized and has no recoverable failure path; a tool that only reads
saves can simply report the save as unsupported.

## 6. The block-0 mod-list entries

Block 0's mod list (retail's `SaveMetaData`, serializer version 3) carries, after the real enabled mods,
one entry per extension record so the save list and retail itself can see them:

```
String name     the extension's display name
String path     "ext:<id>"
u32             0           (retail: package CRC)
u32             0x010007FF  (retail: the game version the package was built with; 1.0.7)
u32             formatVersion   (retail: the mod author's version)
```

`String` is retail's encoding: `u32 n` = byte length including the NUL, with bit 31 set as the UTF-8
flag, then `n` bytes.

`ext:<id>` can never match a real package path, so **retail shows its own `modsMissing` modal once per
entry** (Yes continues, No aborts) and, on Yes, loads the town without the extensions. A reader that
knows the extension takes the `ext:` entries out of the mod check and relies on the tail instead. Host
records get no mod-list entry.

## 7. Compatibility

| Save | Retail | A v1-only reader | A v2 reader |
| --- | --- | --- | --- |
| No tail | loads | loads | loads |
| v1 tail | `modsMissing` per entry, then loads without the extensions | loads | loads |
| v2 tail | `modsMissing` per entry, then loads without the extensions | refuses (newer chunk version) | loads |

## 8. Payloads registered so far

OpenBA's two payloads, as examples of the documentation every payload carries beside its encoder.

**Extension `restricted_foods`, formatVersion 1** (foods citizens may not eat in this town):

```
u32 count
count x { u8 kind (1 = blocked food), u32 length (1..1024), u8 path[length] }
```

Foods are named by the template path of their raw-material description, never by index, because mods
change the load order between sessions; a path the game no longer has matches nothing. The decoder
refuses a short or over-long payload, an unknown kind, an empty, over-long or control-character path,
and a path listed twice (compared without case).

**Host `toolbar`, formatVersion 1** (the extension toolbar window):

```
u8  open      0 or 1
i32 x         upper-left corner, UI scaled pixels
i32 y
u32 length    0..64
u8  page[length]   the showing page's id, [a-z0-9_]; empty = the first page
```

## 9. Worked example

One extension record (`example_b`, formatVersion 1, empty payload, as a rule set with no state writes)
and the `toolbar` host record (open at 24, 120, first page). 77 bytes, CRC `0x03C81CEB`:

```
0000  45 58 54 53 02 00 00 00 02 00 00 00 01 01 09 00   magic, version 2, count 2, kind 1, flags 1, idLength 9 ...
0010  00 00 65 78 61 6d 70 6c 65 5f 62 01 00 00 00 00   "example_b", formatVersion 1, payloadLength 0 ...
0020  00 00 00 02 00 07 00 00 00 74 6f 6f 6c 62 61 72   kind 2, flags 0, idLength 7, "toolbar"
0030  01 00 00 00 0d 00 00 00 01 18 00 00 00 78 00 00   formatVersion 1, payloadLength 13, open 1, x 24, y 120 ...
0040  00 00 00 00 00 45 00 00 00 eb 1c c8 03            page length 0, bodyLength 69, crc
```

## 10. Open questions

- [ ] Id registry: a shared list of extension and host ids, or a per-project prefix (`openba_`,
  `bmp_`, `unb_`) so two projects cannot claim the same id?
- [ ] Multiplayer (layer 4 of the [multiplayer proposal](../multiplayer/comparison-and-proposal.md#layer-4-saves)):
  the room, seat-to-player map, outbox and last ids seen as an extension record (must-understand, a town
  saved in a session cannot be played on alone without it) or as a new record kind?
- [ ] A third kind for state that survives an extension being turned off and on again, carried through
  untouched by readers that do not know it. Not built; no extension needs it yet.
- [ ] Can the retail hook (banished-multiplayer) write into block 1's stream after the last retail
  `Serialize`, or does it need a different carrier?
