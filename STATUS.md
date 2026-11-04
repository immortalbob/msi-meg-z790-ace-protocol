# Status and change log

Part of the [MSI MEG Z790 ACE reference](README.md). One row per item; the tags are explained in the README.
"Captured" refers to the 2026-10-30 capture of MSI Center.

## Mystic Light controller `1462:7D86`

| Item | Layout | Captured | Observed | Open |
|---|---|---|---|---|
| transport | feature reports on the control pipe; inbox HID | yes | yes | the collection's exact report descriptor and caps |
| `0x52` configuration, 185 bytes | complete (PUBLISHED); slot offsets for this board confirmed | yes (GET and SET) | yes (2026-10-27) | colour-flags bit 6; why MSI changed offset 133 |
| readback | GET returns the live state, applied with a delay: a read right after the write (and 16 ms after it) still shows the previous image, a read 200 ms after it shows the written one | yes | yes (2026-10-27 live; 2026-11-02 to 11-04 the lag) | the exact delay (between 16 and 200 ms) |
| zone map | JRGB1, JPIPE1 (18), JPIPE2 (4), JRAINBOW1-3; physical locations | per-header names yes | yes (2026-10-27) | which of MSI's names "IO / PCH Cover" / "M.2_1" is JPIPE1 or JPIPE2 |
| effect IDs | complete list (PUBLISHED) | 2, 7, 17, 26 | several of the 22 offered ones, Rainbow wave in daily use; Water drop (8, not offered by OpenRGB) on JRAINBOW1 without harm (2026-11-03) | the other 15 not offered by OpenRGB; effect 0 |
| speed and brightness | complete | yes (slider positions) | yes | |
| colour 2 | field known | MSI always writes colour 2 = colour 1 | no | what each effect does with a distinct colour 2 |
| sync bits (ONBOARD slot) | complete for MSI's use | yes ("SELECT ALL" = `0xB1` / `0x31`) | yes | |
| MSI's write pattern | old image ×5-6, new image slot by slot, 16 ms apart | yes | | |
| save flag (offset 184) | complete | never set by MSI (44 writes) | not used | flash endurance |
| LED counts | complete; MSI writes 100 | yes | yes | what a count of 0 does |
| direct mode: enable message | complete (PUBLISHED; DERIVED bytes) | | yes, non-sync (2026-10-27) | sync mode on the hardware; JRGB1 brightness in direct mode |
| direct mode: `0x53` frames | complete; index map for this board | one all-zero frame at MSI's start | yes (2026-10-27) | maximum frame rate |
| firmware query `0x01` `B0`/`B6` | complete (PUBLISHED) | never sent by MSI Center | yes (2026-11-02: 0.5 / 1.5) | |
| checksum commands | | not capturable (no screen in MSI Center 2.x) | | command bytes; **do not send** |
| Gen2 scan, feature `0x81` | 242 bytes, 15 × 16-byte entries | yes (no Gen2 device attached) | | the entry fields; needs a Gen2 strip |
| standalone behaviour | effects run on the controller | | effect kept running through S3 (2026-11-02) | volatile state across a warm restart; BIOS LED options |
| firmware update, MCU reset | | | | **deliberately not described** |

## NCT6687D fan and sensor controller

| Item | Layout | Observed | Open |
|---|---|---|---|
| identity, ports, base | complete (PUBLISHED) | NCT6687D at base `0x0A20`, EC firmware 1.0 (11/01/22), 2026-11-02 | |
| access sequence and mutex | complete (PUBLISHED) | yes | |
| temperatures 0-6 | complete | yes | meaning of 5/6 (`T_SEN1/2`?) |
| temperatures 7-10 | registers known | no | meaning on this board |
| voltages | decode complete | partly | rail names other than +12V, +5V, Vcore, DIMM |
| fan RPM slots 0-7 | complete | CPU_FAN1 and SYS_FAN2 with devices attached | |
| fan RPM slots 8-16 | registers known | no | PUMP_FAN2 at slot 8? |
| manual control `0xA00`/`0xA01`/`0xA28`/`0x160` | complete (PUBLISHED) | every channel held and handed back (2026-10-27); all eight at `0xFF` at once and handed back (2026-11-03) | the Linux `0xCF8` handshake on this board; 0 % on these headers |
| channel ↔ header map | bit order PUBLISHED | channels 0 and 3 (2026-10-26/27); channel 7's output register stays `0x80` under a manual `0xFF` (2026-11-03) | channels 1, 2, 4-7 by spin test; what channel 7 drives, if anything |
| `0xA28` + i under BIOS control | | `0x66` on all channels (after earlier hand-backs; again 2026-11-03) | whether the BIOS writes it |
| sleep and restart | | `0xA00` stayed `0x00` through S3 and restarts (2026-11-02) | a manual setting across S3 |
| PUMP_FAN2 control | | | no register known |
| EC curve programming | known only for the `-R`/`DR` variant | | for the plain NCT6687D |
| alarm, limit, beep registers | names only | | |

## MSI software and BIOS

| Item | Status |
|---|---|
| MSI Center's footprint (services, drivers, task, packages) | complete for 2.0.74.0 / SDK 3.2026.0911.01 (HOST) |
| LEDKeeper2 re-apply triggers | from MSI's logs (HOST); display wake OBSERVED 2026-10-27 |
| MSI Center fan engine under Memory Integrity | fails to load its driver (HOST) |
| BIOS reinstall chain (`ACPI\MBID0002`, `msiswacpi.inf`, software component) | complete (HOST); the BIOS option removing the device OBSERVED 2026-11-02 on BIOS 1.G0 |
| clean removal and leftovers | complete (OBSERVED 2026-11-02) |
| the BIOS option's menu path on 1.G0 | not recorded |
| `LightKeeperService` HTTP port with the SDK enabled | UNKNOWN |

## Change log

- **2026-11-04, update.** Mystic Light: the configuration readback is the live state **but lags the write** (section
  5 of the lighting document): a read right after a `SET_REPORT`, or 16 ms after it, still returns the previous image,
  a read 200 ms after it returns the written image; found in the logs of a third-party driver whose immediate
  verification read called the readback "stored" at every start when two zones were written back to back. Water
  drop (8), one of the effect IDs OpenRGB does not offer, ran on JRAINBOW1 without harm (section 4.4). NCT6687D:
  all eight channels held at `0xFF` at once and handed back; channel 7's output register (`0x160`+7) stays at `0x80`
  under a manual `0xFF`, unlike channels 0 to 6, so what channel 7 drives, if anything, stays open (sections 3, 4.4
  and 5); a second reading of the outputs under BIOS control. README bench table extended; rows above updated.
- **2026-11-02, first edition.** Written from one USBPcap capture of MSI Center driving the Mystic Light controller
  (2026-10-30, 14 segments), read-only inspection of MSI's installed software, bench sessions with independently
  written code (2026-10-26 to 2026-11-02, including the removal of MSI Center), and the open-source projects credited
  in the README. New compared with what was public before: which parts of the Mystic Light "185-byte" family apply to
  this board and where its zones are; what MSI Center actually sends (only feature reports, the write pattern, the
  never-set save flag, "SELECT ALL" semantics, MSI's filler and paddings, the Gen2 scan report `0x81`); that the
  configuration readback is the live state; the controller's behaviour through sleep; the NCT6687D's base address,
  firmware, header map and register values under BIOS control on this board; the hand-back pitfall in a released
  sensor library; why MSI Center never controlled this board's fans; how the BIOS reinstalls MSI software through an
  ACPI device and a software-component driver package; and a verified clean-removal procedure.
