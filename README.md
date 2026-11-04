# MSI MEG Z790 ACE: board protocol reference

An independent description of everything a host program can talk to on the MSI MEG Z790 ACE (MS-7D86) motherboard
besides the CPU and the chipset itself: the lighting controller, the fan and sensor controller, and the way MSI's
own software and the BIOS interact with them. It is meant as the single reference for running this board without
MSI's software.

| Part | How the host reaches it | What it does | Document |
|---|---|---|---|
| Mystic Light controller | USB HID `1462:7D86`, Windows inbox driver | the board's RGB: three ARGB headers, one 12 V RGB header, 22 on-board LEDs; hardware effects and per-LED streaming | [msi-mystic-light-7d86.md](msi-mystic-light-7d86.md) |
| Nuvoton NCT6687D Super I/O / EC | LPC port I/O (`0x2E`, hardware monitor at `0x0A20`), needs a kernel driver | 8 fan channels, fan speeds, 11 temperatures, 14 voltages; runs the BIOS fan curves | [nct6687d-fan-sensor-controller.md](nct6687d-fan-sensor-controller.md) |
| DDR5 temperature sensors | chipset SMBus | one SPD5118 hub per module | [nct6687d-fan-sensor-controller.md](nct6687d-fan-sensor-controller.md), section 7 |
| MSI software and the BIOS | services, kernel drivers, an ACPI device | MSI Center, its re-apply behaviour, how the BIOS reinstalls it, how to remove it cleanly | [msi-software-and-bios.md](msi-software-and-bios.md) |

Coverage per item and change history: [STATUS.md](STATUS.md).

## The board at a glance (observed unit)

| Item | Value |
|---|---|
| Board | MEG Z790 ACE (MS-7D86) rev 1.0, BIOS 1.G0 (2026-03-03) |
| Mystic Light | USB `1462:7D86`, firmware APROM 0.5 / LDROM 1.5; zones JRGB1, JPIPE1 (18 LEDs: I/O-cover dragon and chipset cover), JPIPE2 (4 LEDs: "ACE" lettering), JRAINBOW1-3 (ARGB, up to 200 / 240 / 240 LEDs) |
| Fan and sensor controller | Nuvoton NCT6687D (plain), EC firmware 1.0 (build 11/01/22); channel 0 = CPU_FAN1, 1 = PUMP_FAN1, 2-6 = SYS_FAN1-5, 7 = "SYS_FAN6" |
| Headers listed for the board | 1 CPU fan, 2 pump fans, 5 system fans, 2 `T_SEN` thermistor inputs |
| Windows drivers needed | none from MSI: the lighting controller uses the inbox HID driver; the fan controller needs any signed port-I/O driver (PawnIO was used) |
| MSI software at observation time | MSI Center 2.0.74.0 with MSI Center SDK 3.2026.0911.01; removed on 2026-11-02 (the board kept working) |

## Why this exists

The Mystic Light protocol family and the NCT6687D registers were already known from open-source software (see
Credits), but scattered across code for many boards, with nothing that said which parts apply to this board, what
MSI's own software actually sends, or how the pieces behave together. On 2026-10-30 the USB traffic between MSI
Center and the lighting controller was captured with USBPcap while every lighting function was exercised. From
2026-10-26 to 2026-11-04 both controllers were driven on the same board by independently written third-party code,
and MSI's software was removed on 2026-11-02. This reference writes down what was seen, what earlier projects published,
and what is still unknown.

## Scope

- The lighting controller's HID reports, layouts, value sets, sequences and behaviour; the fan controller's ports,
  registers, sequences and behaviour; what MSI's software installs and does.
- Not covered: CPU tuning (MSRs, power limits), memory timings, the BIOS setup itself beyond the options named,
  firmware updates of either controller (deliberately not described), and MSI's user interface.
- One unit was observed, with the versions in the table above. Other BIOS or firmware versions may differ.

## How to read the tags

Every statement carries one of these tags.

| Tag | Meaning |
|---|---|
| **CAPTURED** | seen on the USB bus in the capture of MSI Center listed below |
| **PUBLISHED** | first published by an open-source project or by MSI, which is named in Credits; where a capture or the bench re-checked it, the statement says so |
| **OBSERVED** | seen while driving this board with independently written third-party code (not the vendor software), with the date |
| **HOST** | read on the observing PC without driving the hardware: Windows device properties, the vendor's installed files and their string listings, INF files, logs, registry value names, socket and service listings |
| **DERIVED** | computed by hand from a PUBLISHED or CAPTURED layout (for example a complete byte array); not itself captured |
| **INFERRED** | a conclusion drawn from the evidence, not read from any source; the reasoning is given |
| **UNKNOWN** | a gap, with a note on what would close it |

## Capture and bench sessions

Times are local time of the observing PC (UTC-7).

**The capture** (2026-10-30, 07:23 to 13:07, 14 segments with a reboot in between). USBPcap on the root hub
carrying the board's lighting controller, filtered afterwards to that device. MSI Center 2.x with its LEDKeeper2
process owned the controller; one action per segment: the global lighting switch, a restart of MSI's lighting
services, styles (Breathing, Meteor, Double Meteor, Rainbow) on "SELECT ALL" and on single headers, a colour change,
the speed and brightness sliders, one on-board zone set to Off, and the JARGB_V2 "Scan"; LED power saving and the
third-party RGB setting were toggled earlier in the session. The raw files are not published (they hold other traffic of the
same root hub); every frame that matters is described in the documents.

**Bench sessions** (OBSERVED; third-party code, MSI's software stopped or, later, removed):

| Date | What was checked |
|---|---|
| 2026-10-26 to 10-27 | every fan channel held at a fixed duty and handed back to the BIOS with `0xA00` read back; which header carries which tach; every lighting zone set on its own and mapped to the board; the readback is the live state; direct mode in non-sync mode; restoring the start-up configuration; MSI's re-apply after a monitor power cycle |
| 2026-10-28 to 11-01 | the controllers under a Windows service across reboots, shutdowns and sleep |
| 2026-11-02 | firmware query; EC state and identity bytes; an S3 sleep with both controllers read before and after; the removal of MSI Center, the BIOS option and the reinstall device |
| 2026-11-03 | one hidden lighting effect (Water drop, 8) on JRAINBOW1; all eight fan channels at `0xFF` at once during a thermal emergency and handed back afterwards; channel 7's output register not following its command; a second reading of the outputs under BIOS control |
| 2026-11-04 | the configuration readback's lag behind a write, from the driver logs of 2026-11-02 to 11-04 and a readback 200 ms after the write |

## Sources

1. A USB capture of MSI Center driving the lighting controller, made with USBPcap on the owner's own PC.
2. Read-only inspection of the same PC: Windows device properties, the vendor's installed files (string listings,
   INF files, logs; nothing executed, decompiled or disassembled), registry value names, service, process and socket
   listings.
3. The open-source projects listed under Credits and MSI's public Mystic Light SDK documentation, for the PUBLISHED
   statements.
4. For the OBSERVED statements: the controllers' own replies and behaviour when driven by independently written code.

No vendor source code was used or seen. The only vendor strings reproduced are identifiers and file names from the
installed software, INF directives, log lines, and values that appear on the wire.

## Credits

Projects are linked to their repositories; the facts were taken from their source trees (default branch) as of
October 2026.

- **[OpenRGB](https://gitlab.com/CalcProgrammer1/OpenRGB)** first published the Mystic Light "185-byte" protocol:
  the report IDs and lengths, the configuration layout and zone-data fields, the effect IDs, the speed and brightness
  encoding, the sync bits, the direct-mode enable message and frame layout with the sync and non-sync rules, the
  firmware query, and the board table entry for `0x7D86`. Its issue #389 records the controller lock-up on another
  board.
- **[LibreHardwareMonitor](https://github.com/LibreHardwareMonitor/LibreHardwareMonitor)** published the NCT6687D
  access sequence with the `0xFF` token, the register arrays for temperatures, voltages, fans and control, the manual
  control sequence, the channel bit order, the shared mutex names, and the DIMM temperature path (with
  **[RAMSPDToolkit](https://github.com/Blacktempel/RAMSPDToolkit)**).
- **[nct6687d](https://github.com/Fred78290/nct6687d)** (Linux kernel driver for the NCT6687) published the
  register map, the `0xCF8` engine handshake, the identity bytes `0x604`-`0x609`, the initialisation writes and the
  suspend handling.
- **[PawnIO.Modules](https://github.com/namazso/PawnIO.Modules)** provides the signed port-I/O module used to reach
  the EC on Windows.
- **[hidapi](https://github.com/libusb/hidapi)** documents the Windows rules for feature-report buffer lengths.
- **MSI** published the Mystic Light SDK reference (v1.0.0.04, 2019). MSI is the vendor whose hardware and software
  were observed; product, zone and command names are MSI's.

Nothing from these projects (code, comments or prose) is copied here, only the protocol facts they established;
byte layouts and constants they already publish are stated with credit.

## Licence

The documents in this folder are released under the **Creative Commons Attribution 4.0 International licence
(CC BY 4.0)**: use, copy, translate and build on them for any purpose, with credit to this work. See `LICENSE`.
No scripts are published with this reference.

## How to capture more

The capture was made with [USBPcap](https://desowin.org/usbpcap/) on Windows 11.

1. Run `USBPcapCMD.exe` once without arguments to list the root hubs and find the one carrying `1462:7D86`.
2. From an elevated prompt, capture that whole root hub (`-A`) with injected descriptors:
   `USBPcapCMD.exe -d \\.\USBPcap1 -A --inject-descriptors -o mysticlight.pcap`. Stop with Ctrl+C.
3. Keep only the controller's address. **After a sleep or a reboot the injected descriptor list can carry stale
   addresses** (here the controller was listed at one address but talked at another, twice): find it by its traffic,
   control transfers carrying `SET_REPORT` / `GET_REPORT` for reports `0x52`, `0x53` and `0x81`. The root hub usually
   carries your keyboard and mouse too, so delete the raw file once it is filtered.
4. Do one thing at a time in MSI Center and write down the time of each action. Each USBPcap record starts with
   `u16 headerLength, u64 irpId, u32 status, u16 function, u8 info, u16 bus, u16 device, u8 endpoint,
   u8 transferType, u32 dataLength`; transfer type 2 is control.

Still uncaptured, in case you want to contribute: the firmware and checksum screen (MSI Center 2.x has none for this
board), a JARGB_V2 scan with a Gen2 strip attached, MSI's host-computed styles (CPU temperature, music), and the
"third party RGB" mode while a third-party program streams. See [STATUS.md](STATUS.md).

## Safety

Apart from the OBSERVED statements, nothing here was tested beyond observing the vendor software; anyone driving
their own board does so at their own risk. In particular:

- a controller of this lighting family on another board stopped responding on USB after a rarely used effect was set;
  stick to the effects MSI Center itself uses, one zone at a time, and never save untested ones to flash;
- the fan controller has no watchdog: a program that crashes with a fan channel in manual mode leaves that fan at the
  last duty, possibly stopped, until the channel is handed back or the power is removed;
- never send firmware, boot-loader or checksum commands, and never run MSI's MCU reset or rescue tools.
