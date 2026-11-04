# Mystic Light controller (USB `1462:7D86`)

Part of the [MSI MEG Z790 ACE reference](README.md). The tags (CAPTURED, PUBLISHED, OBSERVED, HOST, DERIVED,
INFERRED, UNKNOWN) are explained there.

The board's RGB lighting is driven by a small USB HID microcontroller on the board itself. It has no sensors and
no fan control: it only stores and runs lighting effects and accepts per-LED colour frames.

## 1. Identity

| Item | Value | Tag |
|---|---|---|
| USB ID | `1462:7D86` (the product ID equals the board's model number, MS-7D86), `bcdDevice` 0x0001 | HOST |
| Class | HID `03/00/00`, one interface, one top-level collection: usage page `0x0001`, usage `0x0000` | HOST |
| Bus-reported description | `MYSTIC LIGHT` | HOST |
| Serial number string | 12 characters; the first four are the board PID (`7D86`) | HOST |
| Windows driver | inbox `HidUsb` (`input.inf`). No vendor driver, no WinUSB | HOST, OBSERVED 2026-11-02 after MSI Center was removed |
| Microcontroller | Nuvoton NUC126 family; MSI's software calls the controller family "7B10" | INFERRED from identifiers in MSI's software (`NuvotonMCU_7B10`, `NUC126_7B10_Set_LED_Data`) and the APROM/LDROM terminology |
| Firmware | APROM 0.5, LDROM 1.5 (BIOS 1.G0) | OBSERVED 2026-11-02 with the query in section 9 |
| Protocol family | OpenRGB's "185-byte" Mystic Light class: the board answers a GET of feature report `0x52` with 185 bytes | PUBLISHED (OpenRGB lists `0x7D86` in its tested-board table), OBSERVED |

## 2. Zones

| Zone | Kind | LEDs | `0x52` slot offset | MSI Center's name | What it lights on the observed board |
|---|---|---|---|---|---|
| JRGB1 | 12 V 4-pin RGB header | 1 (one colour for the strip) | 1 | `JRGB1` | nothing connected |
| JPIPE1 | on-board LEDs | 18 | 11 | `IO / PCH Cover` (INFERRED) | the dragon on the I/O cover and the LED array on the chipset cover under the graphics card, one chain (OBSERVED 2026-10-27) |
| JPIPE2 | on-board LEDs | 4 | 21 | `M.2_1` (INFERRED) | the "ACE" lettering above the first PCIe slot (OBSERVED 2026-10-27) |
| JRAINBOW1 | 5 V ARGB header | 0..200 | 31 | `JARGB_V2_1` (CAPTURED) | on the observed PC, the ARGB lead of a Lian Li UNI HUB (12 fans × 16 LEDs) |
| JRAINBOW2 | 5 V ARGB header | 0..240 | 42 | `JARGB_V2_2` (CAPTURED) | nothing connected |
| JRAINBOW3 | 5 V ARGB header | 0..240 | 53 (the slot other boards use for JCORSAIR) | `JARGB_V2_3` (CAPTURED) | nothing connected |
| ONBOARD | not a visible zone on this board | 0 | 74 | (MSI's "SELECT ALL") | the sync master and the direct-mode switch (sections 4 and 7) |

The zone set, LED counts and maxima were first published by OpenRGB (its board row for `0x7D86`: no on-board
zone, JPIPE1 = 18, JPIPE2 = 4, one JRGB header, per-LED direct mode, no minimum firmware). The physical locations
are OBSERVED by lighting one zone at a time. "IO / PCH Cover" and "M.2_1" are MSI Center's names for the two
on-board zones; which name belongs to which zone could not be settled from the capture (applying "Off" to `M.2_1`
alone changed nothing on the wire), so that pairing is INFERRED.

## 3. Transport

| Report | Direction | Length (with ID) | Use | Tag |
|---|---|---|---|---|
| Feature `0x52` | GET and SET | 185 | the whole lighting configuration (section 4) | PUBLISHED, CAPTURED, OBSERVED |
| Feature `0x53` | SET | 725 | one per-LED frame (section 7) | PUBLISHED, CAPTURED, OBSERVED |
| Output `0x01` / input `0x01` | write, then read | 64 / 64 | firmware version query (section 9) | PUBLISHED, OBSERVED |
| Feature `0x81` | GET | 242 | MSI's "Gen2" ARGB scan (section 10) | CAPTURED |
| Feature `0x50`, `0x51` | | | used by a different Mystic Light family (761-byte boards); **do not send** to this board | PUBLISHED |

Byte 0 of every buffer in this document is the report ID, and every length includes it. RGB triplets are sent
as R, G, B. No field is wider than one byte, so byte order never matters.

- **Everything MSI Center exchanges with the board is a feature report on the control pipe** (CAPTURED
  2026-10-30, about two hours of use): `SET_REPORT(Feature 0x52)` with wLength 185, `GET_REPORT(Feature 0x52)`,
  `SET_REPORT(Feature 0x53)` with 725, and `GET_REPORT(Feature 0x81)` with 242. No `0x01` query was ever sent.
  The interrupt IN endpoint carried only an occasional 1-byte `40` report.
- On Windows the feature reports go through `HidD_SetFeature` / `HidD_GetFeature` (or
  `IOCTL_HID_GET_FEATURE`, which also returns the length). Windows rejects a SET buffer shorter than the
  collection's `FeatureReportByteLength` and truncates a longer one, so size SET buffers to that value and pad
  with zeros (PUBLISHED by hidapi's Windows backend; INFERRED for this collection, whose caps were not recorded).
- No elevation is needed: the inbox HID driver lets any user open the collection (PUBLISHED by OpenRGB's
  documentation; on the bench the controller was driven from a service, so a non-elevated open was not tested).
- HID handles are shareable. Several programs can write the controller at the same time, and the last write
  wins (section 11).
- **SET has no acknowledgement.** The only success signal is the transfer itself. Read the report back to check
  (section 5).

## 4. The configuration report `0x52` (185 bytes)

### 4.1 Layout

| Offset | Size | Slot | On this board |
|---|---|---|---|
| 0 | 1 | report ID `0x52` | |
| 1 | 10 | JRGB1 (zone data) | JRGB1 |
| 11 | 10 | JPIPE1 (zone data) | 18 on-board LEDs |
| 21 | 10 | JPIPE2 (zone data) | 4 on-board LEDs |
| 31 | 11 | JRAINBOW1 (zone data + LED count) | ARGB header 1 |
| 42 | 11 | JRAINBOW2 (zone data + LED count) | ARGB header 2 |
| 53 | 11 | JCORSAIR slot, read as zone data + LED count | ARGB header 3 (JRAINBOW3) |
| 64 | 10 | spare slot | preserve |
| 74 | 10 | ONBOARD (zone data) | sync master; effect `0x25` here = direct per-LED mode |
| 84..173 | 9 × 10 | further on-board slots | preserve |
| 174 | 10 | JRGB2 | not present on this board; preserve |
| 184 | 1 | save flag: `0` = apply only (volatile), `1` = also store in the controller's flash | per write |

The layout was first published by OpenRGB; the slot assignments for this board are CAPTURED (each per-header
apply in MSI Center changed only that header's slot) and OBSERVED.

### 4.2 Zone data (10 bytes, plus 1 for the ARGB headers)

| +Offset | Field | Encoding |
|---|---|---|
| +0 | effect | effect ID, section 4.4 |
| +1..+3 | colour 1 | R, G, B |
| +4 | speed and brightness | bits 0-1 speed (0 slow, 1 medium, 2 fast); bits 2-6 brightness 0..10 (0 off, 10 = 100 %); bit 7 = "sync JRGB", meaningful on the ONBOARD slot |
| +5..+7 | colour 2 | R, G, B (what each effect does with it is UNKNOWN; MSI Center always writes colour 2 = colour 1, CAPTURED) |
| +8 | colour flags | bit 0 sync ONBOARD, bit 1 sync JRAINBOW1, bit 2 sync JRAINBOW2, bit 3 sync JCORSAIR/JRAINBOW3, bit 4 sync JPIPE1, bit 5 sync JPIPE2, bit 6 UNKNOWN, **bit 7 = use the custom colours** (0 = the effect's own rainbow or random colours) |
| +9 | padding | `0`; MSI Center and OpenRGB write `0x04` in the JRAINBOW3 slot (CAPTURED); MSI's filler slots carry `0x05` |
| +10 | LED count (ARGB headers only) | the number of LEDs the hardware effect runs over; MSI's default is 100 |

Speed and brightness byte, DERIVED from the encoding: brightness 10 / speed 0 = `0x28`, brightness 10 / speed 1 =
`0x29`, brightness 10 / speed 2 = `0x2A`, brightness 5 / speed 1 = `0x15`; with the sync-JRGB bit, `0x29 | 0x80` =
`0xA9`. MSI Center's sliders: speed maximum 2 and middle 1, brightness maximum 10 and middle 6 (CAPTURED).

To decode a readback: effect = +0, speed = (+4) & 0x03, brightness = ((+4) >> 2) & 0x1F, custom colour =
(+8) & 0x80 ≠ 0. (Masking the brightness with the pre-shifted `0x7C` after shifting, or reading bit 7 as
"rainbow", gives wrong values: brightness 10 then reads as 8.)

### 4.3 The ONBOARD slot and sync

The board has no zone of its own in the ONBOARD slot (offset 74); the controller uses it as a master that other
zones can follow through the sync bits in its colour flags (offset 82) and the sync-JRGB bit in its offset 78.

- **MSI Center's "SELECT ALL"** (CAPTURED 2026-10-30) writes the effect and colour into the ONBOARD slot, sets
  offset 78 bit 7 (for example `0xAA` = speed 2, brightness 10, sync JRGB), writes colour flags `0xB1` for
  custom-colour styles (sync ONBOARD, JPIPE1, JPIPE2, custom colour) or `0x31` for the effect's own colours, and
  padding `0x04`. JRGB1, JPIPE1 and JPIPE2 keep their old slot bytes and follow the master. The three ARGB headers
  are written in their own slots, with LED count 100.
- So with MSI's usual setting, the on-board LEDs and JRGB1 follow ONBOARD while the three ARGB headers run on
  their own slots (OBSERVED 2026-10-27: header sync bits off, on-board sync on).
- A write to a zone whose sync bit is set in offset 82 may appear to do nothing, because the zone follows the
  master (INFERRED from the bit names). Read offset 82 first. Clearing those bits changes the user's MSI sync
  setup; restore them when done.

### 4.4 Effect IDs

| ID | Name | Notes |
|---|---|---|
| 0 | Off | OpenRGB does not offer it; brightness 0 or black Static is the safe "off" (PUBLISHED). MSI Center's "Off" on one on-board zone sent no change (CAPTURED) |
| 1 | Static | |
| 2 | Breathing | CAPTURED from MSI Center |
| 3 | Flashing | |
| 4 | Double flashing | |
| 5 | Lightning | |
| 7 | Meteor | CAPTURED |
| 15 | Colour ring | |
| 16 | Planetary | |
| 17 | Double meteor | CAPTURED from MSI Center. See the warning below |
| 18 | Energy | |
| 19 | Blink | |
| 20 | Clock | |
| 21 | Colour pulse | |
| 22 | Colour shift | |
| 23 | Colour wave | |
| 24 | Marquee | |
| 26 | Rainbow wave (MSI Center calls it "Rainbow") | CAPTURED, with the custom-colour bit cleared |
| 27 | Visor | |
| 29 | Rainbow flashing | |
| 35 | Colour ring double flashing | |
| 36 | Stack | |
| 37 (`0x25`) | **reserved**: in the ONBOARD slot this value is the per-LED direct mode (section 7) | never send as an effect |
| 38 | Fire | |
| 6, 8-14, 25, 28, 30-34, 39 | MSI Marquee, Water drop, MSI Rainbow, Pop, Rap, Jazz, Play, Movie, Rainbow, JRainbow, Rainbow double flashing, Random, Fan control, Disable 2, Colour ring flashing, Lava | defined by OpenRGB but not offered. **Water drop (8) OBSERVED 2026-11-03** on JRAINBOW1 (volatile, one zone, one minute): the controller took the slot value, stayed enumerated and answered the readback with 8 in the slot, the ARGB strip showed a water-drop animation in the chosen colour, and the next regular effect replaced it; the other fifteen are UNKNOWN |

IDs and names were first published by OpenRGB, which offers the 22 effects listed without a note plus its own
"Direct" pseudo-mode (its internal value 100, never sent on the wire). **Warning:** OpenRGB disabled Mystic Light
for a time after a controller on another MSI board (MPG X570 Gaming Pro Carbon WiFi) stopped responding on USB
after Double meteor was set (OpenRGB issue #389, PUBLISHED). MSI Center sends 17 on this board without harm
(CAPTURED), but treat the effects OpenRGB does not offer as untested: one zone at a time, never saved. One data
point exists: Water drop (8) ran on JRAINBOW1 for a minute without harm (OBSERVED 2026-11-03, see the table); Double
meteor itself was not tried.

## 5. Reading the configuration

`GET_REPORT(Feature 0x52)` returns 185 bytes in the layout of section 4. **It returns the live state, not only the
stored image** (OBSERVED 2026-10-27: a volatile write read back as written). At start-up the same read returned
the image MSI Center had last written (CAPTURED 2026-10-30).

**But the live state lags the write** (OBSERVED 2026-11-02 to 11-04, from the logs of a third-party driver that read
back once right after each volatile write): a `GET_REPORT` issued at once, inside the same mutex hold as the
`SET_REPORT`, returned the **previous** image every time two writes went out back to back (the driver's scene
applied its zones 10 to 20 ms apart), and a read 16 ms after a write still showed the image from before it, while a
read 48 s after the previous write showed the written image, as the 2026-10-27 test with commands seconds apart had.
A read 200 ms after the write returned the written image at the first attempt (OBSERVED 2026-11-04). So the
controller applies a configuration write asynchronously, somewhere between 16 and 200 ms after accepting it. Two
consequences for a host: a readback used to verify a write must wait, and a read-modify-write that follows another
write closely must build on what the host wrote, not on a readback that may predate it, or the second write undoes
the first. MSI Center sidesteps both by always writing the whole image from its own state (section 6).

Things to check in a readback:

- **Offset 74 = `0x25`** means direct per-LED mode was left on (by a crash or another program). Never save an
  image with `0x25` there.
- The sync bits at offsets 78 and 82 (section 4.3).
- The LED counts at offsets 41, 52 and 63.

## 6. Writing an effect

1. Read `0x52` (section 5). **Always start from a fresh readback**; never build an image from zeros, and change
   only the bytes of the zone being set.
2. Write the image as read (the "old" image), then the new image, both with save = 0. OpenRGB sends this pair
   because "Windows Mystic Light sends two consecutive reports" (PUBLISHED); MSI Center's own pattern is longer
   (below).
3. Read back to confirm.

**What MSI Center sends for one Apply** (CAPTURED 2026-10-30): the previous image 5 to 6 times, 16 ms apart,
then the new image built up slot by slot in further writes 16 ms apart, and sometimes the final image three more
times about a second later. At LEDKeeper2's start: one GET of `0x52`, one `0x53` frame of 725 zero bytes, then the
`0x52` image six times and six more a second later.

**Saving.** Offset 184 = `1` stores the image in the controller's flash, so it comes back at power-on. In 44
captured writes MSI Center **never** set it (CAPTURED); it relies on its own service to re-apply the scene at every
start and display wake (see [MSI software and BIOS](msi-software-and-bios.md), section 3). OpenRGB saves only on
an explicit save, and then both halves of the double write carry save = 1 (two flash commits). The flash's write
endurance is UNKNOWN; save rarely, and never in direct mode.

**LED counts.** Write the counts at offsets 41, 52 and 63 (JRAINBOW1 0..200, JRAINBOW2 and 3 0..240). Do not write
0 for a header nobody sized: a hardware effect may then light nothing on it (INFERRED). OpenRGB writes 0 for an
unsized header; MSI writes 100.

**Unused slots.** MSI Center leaves its own filler in slots the board does not use (`25 00 00 00 28 00 00 00 00 00`,
with padding `0x05` in the last four), and it changed one byte (offset 133) during the session for no visible
reason (CAPTURED). Preserve every byte you do not mean to change.

## 7. Direct per-LED mode

Direct mode streams colours from the host instead of running a hardware effect. It is entered by a special `0x52`
image (the "enable message"), then fed with `0x53` frames. Both were first published by OpenRGB; OBSERVED on this
board on 2026-10-27 (all LEDs white for 30 s, non-sync mode, four packets per frame).

### 7.1 Sync and non-sync

- **Sync mode** is used when JRAINBOW1 ≤ 40, JRAINBOW2 ≤ 40 and JRAINBOW3 ≤ 120 LEDs: one `0x53` packet carries
  every LED on the board.
- **Non-sync mode** otherwise: each ARGB header gets its own `0x53` packet (up to 200 / 240 / 240 LEDs), followed
  by the on-board packet. OpenRGB waits 13 ms after each header packet, which gives at most about 25 frames per
  second (PUBLISHED). The highest safe rate is UNKNOWN.

### 7.2 The enable message

A complete `0x52` image, written once with save = 0 when direct mode is entered and again whenever the LED counts
change. Sync mode (DERIVED from OpenRGB's published builder; 185 bytes):

```
52 01 00 00 00 08 00 00 00 80 00 01 00 00 00 2A 00 00 00 80 00 01 00 00 00 2A 00 00 00 80
00 01 00 00 00 29 00 00 00 80 00 28 01 00 00 00 29 00 00 00 80 00 28 01 00 00 00 29 00 00
00 82 00 78 01 00 00 00 28 00 00 00 80 00 25 00 00 00 A9 00 00 00 BF 00 01 00 00 00 28 00
00 00 80 00 01 00 00 00 28 00 00 00 80 00 01 00 00 00 28 00 00 00 80 00 01 00 00 00 28 00
00 00 80 00 01 00 00 00 28 00 00 00 80 00 01 00 00 00 28 00 00 00 80 00 01 00 00 00 28 00
00 00 80 00 01 00 00 00 28 00 00 00 80 00 01 00 00 00 28 00 00 00 80 00 01 00 00 00 2A 00
00 00 80 00 00
```

Checks: length 185, offset 74 = `0x25` (direct), 82 = `0xBF` (full sync), LED counts 41 = 40, 52 = 40, 63 = 120,
184 = 0.

Non-sync mode is the same image with exactly these changes: offset 31 `01` → `25`, 41 `28` → `C8` (200), 42 `01` →
`25`, 52 `28` → `F0` (240), 53 `01` → `25`, 63 `78` → `F0` (240), 82 `BF` → `B1`.

The constant bytes decode with the zone-data rules (INFERRED): JRGB1's `0x08` is brightness 2 (20 %), so a 12 V strip
may run dim in direct mode (UNKNOWN; nothing was connected to JRGB1 here).

### 7.3 Frames (feature `0x53`, 725 bytes)

| Offset | Field | Value |
|---|---|---|
| 0 | report ID | `0x53` |
| 1 | header 0 | `0x25` |
| 2 | channel | `0x06` = on-board LEDs plus the sync blocks; `0x04` = JRAINBOW1 or 2; `0x05` = JRAINBOW3 |
| 3 | index | `0` = JRAINBOW1, `1` = JRAINBOW2 (with channel `0x04`); `0` otherwise |
| 4 | header 3 | `0x00` |
| 5 + 3·i | LED i | R, G, B for i = 0..239 |

LED index map of the channel-`0x06` packet on this board (DERIVED from OpenRGB's published offsets for `0x7D86`):

| LED index | Byte offsets | Zone |
|---|---|---|
| 0..17 | 5..58 | JPIPE1 |
| 18..21 | 59..70 | JPIPE2 |
| 22 | 71..73 | JRGB1 |
| 23..62 | 74..193 | JRAINBOW1 sync block (≤ 40) |
| 63..102 | 194..313 | JRAINBOW2 sync block (≤ 40) |
| 103..222 | 314..673 | JRAINBOW3 sync block (≤ 120) |
| 223..239 | 674..724 | unused, send 0 |

In non-sync mode the headers go in their own packets (`53 25 04 00 00` + 240 LEDs for JRAINBOW1, `53 25 04 01 00`
for JRAINBOW2, `53 25 05 00 00` for JRAINBOW3), then the channel-`0x06` packet with indices 0..22.

Example (DERIVED): a sync frame with JPIPE1 LED 0 = `FF 00 00`, JRGB1 = `00 FF 00`, JRAINBOW1 LED 0 = `00 00 FF` and
JRAINBOW3 LED 119 = `12 34 56` has these non-zero bytes only: `[0]=53 [1]=25 [2]=06 [5]=FF [72]=FF [76]=FF [671]=12
[672]=34 [673]=56`.

### 7.4 Leaving direct mode

Write a normal `0x52` image (for example the image read before direct mode was entered, with offset 74 not `0x25`).
When the host stops sending frames, the LEDs keep the last frame (INFERRED: no keep-alive exists in any published
driver, and nothing timed out during a 30 s still frame, OBSERVED). Direct mode is entered with save = 0, so it does
not survive power loss (INFERRED).

## 8. Example writes (DERIVED from the layout)

Applied to a readback image R:

- **JRAINBOW1 Static red, brightness 10, custom colour:** `[31]=01 [32..34]=FF 00 00 [35]=28 [36..38]=FF 00 00
  [39]=R[39]|0x80 [40]=00`. Nothing changes in the ONBOARD slot.
- **JRGB1 Static green, brightness 10** in OpenRGB's style, which also copies the effect into the ONBOARD master
  because the board has no on-board zone: `[1]=01 [2..4]=00 FF 00 [5]=28 [6..8]=00 FF 00 [9]=R[9]|0x80 [10]=00`,
  plus `[74]=01 [75..77]=00 FF 00 [79..81]=00 FF 00 [82]=(R[82]|0x01)&0xCF [83]=00 [78]=R[78]&0x7F`.
- **Readback decode:** slot bytes `07 FF 80 00 29 FF 80 00 80 00` = Meteor, colour `FF 80 00`, speed 1,
  brightness 10, custom colour on.

## 9. Firmware version (reports `0x01`)

Write output report `0x01` (64 bytes): `01 B0` followed by 62 × `CC` for the application firmware (APROM), or
`01 B6` + 62 × `CC` for the loader (LDROM). Read input report `0x01` (64 bytes): byte 2 is the version, shown as
high nibble . low nibble (`0x05` = 0.5, `0x15` = 1.5). PUBLISHED by OpenRGB, which sends it on every detection;
OBSERVED on this board 2026-11-02 (APROM 0.5, LDROM 1.5).

- Use a read timeout (OpenRGB's read has none, so a silent controller would hang it), and flush the input queue
  first: other programs' replies can arrive on your handle (INFERRED from Windows' per-handle input-report copies).
- MSI's software has checksum commands for both firmware areas (`Get_FwCheckSum_APROM/LDROM`, HOST); the command
  bytes are UNKNOWN (`0xB4` / `0xB8` INFERRED from a comment in OpenRGB). MSI Center 2.x has no screen that sends
  them, so they could not be captured. **Do not send** any `0x01` command other than `0xB0` and `0xB6`.

## 10. Gen2 ARGB scan (feature `0x81`)

MSI Center's "Scan" on the JARGB_V2 page (CAPTURED 2026-10-30) writes the `0x52` image once, then reads
**`GET_REPORT(Feature 0x81)`, 242 bytes**: byte 0 `0x81`, then 15 entries of 16 bytes
`ff ff ff ff 1a ff 00 00 2a 00 ff 00 80 00 1e 00`, the rest `0xFF`. The UI showed one "Gen1" device per header, so
these identical entries are the "nothing Gen2 attached" defaults. The `0x2A` and `0x80` bytes look like the
speed/brightness and colour-flag bytes of section 4.2; `0x1A` and `0x1E` are UNKNOWN. Decoding the entries needs a
Gen2 strip or fan. No SET of `0x81` was seen. MSI's software names the related functions `Gen2_Detect`,
`Gen2_SetStrip`, `Gen2_StripMaxLedCount` and `JARGB_SwitchToGen1` (HOST).

## 11. Standalone behaviour and other writers

- **A hardware effect runs on the controller by itself.** Through an S3 sleep of 20 s the controller stayed on the
  bus and the effect kept running, with nothing re-sent by the host (OBSERVED 2026-11-02). After restarts and a
  power cycle the observed board came back in the scene its host software sent again at start-up.
- **Volatile writes are lost at power removal** and the flash image comes back (INFERRED from the save flag).
  Whether a volatile write survives a warm restart with standby power on is UNKNOWN.
- **MSI's service re-applies its own scene** at start, at every display power-on and around display changes and
  USB arrivals (HOST, from MSI's logs; OBSERVED 2026-10-27 after a monitor power cycle). Any other program's
  setting is overwritten then. See [MSI software and BIOS](msi-software-and-bios.md), section 3.
- MSI's software and LibreHardwareMonitor serialise USB HID transfers to MSI devices with the named mutex
  `Global\Access_USB_Sensors` (HOST, PUBLISHED). Holding it around a read-modify-write keeps the write from
  interleaving with MSI's, but does not stop MSI from overwriting later.
- **Whether the BIOS's LED settings** (such as "EZ LED Control" or lighting in S5) gate this controller is UNKNOWN.

## 12. Do not send

| What | Why |
|---|---|
| Any `0x01` command other than `0xB0` / `0xB6` | checksum and boot-loader commands exist in MSI's software; their encoding is UNKNOWN |
| Feature reports `0x50`, `0x51` | another Mystic Light family's commands |
| A `0x52` image built from zeros | always read first; unknown bytes must be preserved |
| Effect `0x25` in any slot, or OpenRGB's value 100 | `0x25` switches the ONBOARD slot to direct mode; 100 is an OpenRGB-internal value |
| Save = 1 with offset 74 = `0x25` | a stored direct-mode image would come back dark or undefined at power-on |
| The effects OpenRGB does not offer, saved, or several zones at once | behaviour UNKNOWN; one controller of this family on another board went off the bus after an effect was set |
| A firmware update | not described here at all; MSI ships boot-loader and MCU reset tools (`FWRescue.exe`, `ResetMCU\JT1Toucher`, which works through port I/O, not HID) that are not documented and should not be run |
