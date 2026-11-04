# Fan and sensor controller (Nuvoton NCT6687D)

Part of the [MSI MEG Z790 ACE reference](README.md). The tags are explained there.

The board's fan headers, temperatures and voltages belong to a Nuvoton NCT6687D Super I/O with an embedded
controller (EC). It is not a USB device: the host reaches it through LPC I/O ports, which on Windows needs a kernel
driver. The EC runs the BIOS's Smart Fan curves by itself; a host can read every sensor and override any header with
a fixed duty.

Most register facts here were first published by the Linux `nct6687d` driver and by LibreHardwareMonitor (see
Credits in the README). This document collects them for this board and adds what was OBSERVED on it.

## 1. Identity

| Item | Value | Tag |
|---|---|---|
| Chip | Nuvoton NCT6687D, plain variant (chip ID `CR20:CR21` = `0xD5 0x92`), not the `-R`/`DR` variant some later MSI boards use | PUBLISHED (both drivers take the plain branch for Z790 boards); OBSERVED |
| Super I/O config ports | `0x2E` (index) / `0x2F` (data); drivers also probe `0x4E` / `0x4F` | PUBLISHED; the chip was detected (OBSERVED) |
| Hardware-monitor base | `0x0A20` (logical device `0x0B`, `CR60:CR61`) | OBSERVED 2026-11-02 |
| EC firmware | 1.0, build date 11/01/22 | OBSERVED 2026-11-02, decoded from `0x604`-`0x609` (section 4.5) |
| Board | MEG Z790 ACE (MS-7D86) rev 1.0, BIOS 1.G0 (2026-03-03) | HOST |
| Sensor-library model | LibreHardwareMonitor has no board entry for MS-7D86 (only the sibling MEG Z790 GODLIKE MAX, MS-7D85), so it shows generic channel and sensor names | PUBLISHED |

## 2. Access

### 2.1 Ports

| Path | Ports | Use |
|---|---|---|
| Super I/O configuration | `0x2E` / `0x2F`; enter with `0x87 0x87`, leave with `0xAA` | detection, finding the base address |
| EC hardware-monitor window | `base+4` page, `base+5` index, `base+6` data (here `0x0A24`..`0x0A26`) | every register in section 4 |

The EC space is paged: a 16-bit register address splits into page = address >> 8 and index = address & 0xFF.

**Read one byte** (LibreHardwareMonitor's sequence, PUBLISHED):

1. poll `base+4` until it reads `0xFF` (the "free" token; up to 500 ms), else write `0xFF` to it;
2. write the page to `base+4`;
3. write the index to `base+5`;
4. read `base+6`;
5. write `0xFF` to `base+4` (release).

A write is the same with step 4 writing the value. The Linux driver writes `0xFF`, page, index and data without the
polling and the release, under its own kernel lock (PUBLISHED). The `0xFF` token is a convention among host
programs, not something the EC enforces (INFERRED).

### 2.2 Sharing the bus

Host programs that touch the Super I/O on Windows serialise with the named mutex
**`Global\Access_ISABUS.HTP.Method`**; MSI's software also uses `Global\Access_EC` and, for the SMBus,
`Global\Access_SMBUS.HTP.Method` (HOST, from string listings of MSI's binaries; PUBLISHED by LibreHardwareMonitor
for the ISA and SMBus names). Hold the ISA mutex around every page/index/data sequence. LibreHardwareMonitor waits
only 10 ms for it during updates and writes, and silently skips the work on a timeout, so a long hold elsewhere
makes its writes vanish without an error (PUBLISHED).

### 2.3 Windows drivers

Port I/O needs a kernel driver. Two were seen on this PC:

- **PawnIO** (signed, open source), with its signed `LpcIO` module, which LibreHardwareMonitor embeds. The module
  only allows ports inside base addresses it discovered itself. Its device is accessible to SYSTEM and
  Administrators only (HOST), so reading the board's sensors needs elevation. It loads with Windows' Memory
  Integrity (HVCI) on.
- **MSI's own drivers**: `MSIO` (`MsIo64.sys`) and `NTIOLib_X64.sys` (HOST). With Memory Integrity on, MSI Center's
  fan engine logged `Memory integrity : Enabled`, `LoadDriver : Fails` and `MB Not Support!` on every start, every day
  for at least a month (HOST), so MSI Center never controlled this board's fans on the observed PC.

### 2.4 Writes made when a driver opens the chip

Both LibreHardwareMonitor and the Linux driver write the EC at initialisation: `0x180` bit 7 is set if clear, and
`0x1BB`..`0x1BF` are set to `0x61`..`0x65` (PUBLISHED). Opening a sensor program on this board is therefore not
read-only.

## 3. Channels and headers

The EC has 8 controllable PWM channels. The bit order of the manual-mode register `0xA00` is CPU fan, pump fan,
then system fans 1 to 6 (PUBLISHED by LibreHardwareMonitor).

| Channel | `0xA00` bit | Header (PUBLISHED name) | On the observed board |
|---|---|---|---|
| 0 | 0 | CPU_FAN1 | **CPU_FAN1** (OBSERVED 2026-10-26/27: the tach of an AIO pump plugged in there moved with the pump) |
| 1 | 1 | PUMP_FAN1 | nothing connected |
| 2 | 2 | SYS_FAN1 | nothing connected |
| 3 | 3 | SYS_FAN2 | **SYS_FAN2** (OBSERVED 2026-10-27: a fan hub's tach lead plugged in there followed the hub's speed) |
| 4 | 4 | SYS_FAN3 | nothing connected |
| 5 | 5 | SYS_FAN4 | nothing connected |
| 6 | 6 | SYS_FAN5 | nothing connected |
| 7 | 7 | "SYS_FAN6" | nothing connected; the board is listed with five system headers, so what channel 7 drives is UNKNOWN. **Its output register does not follow a manual command** (OBSERVED 2026-11-03): held at `0xFF` for two minutes with `0xA00` bit 7 set and the command re-written twice, `0x160`+7 stayed at `0x80` (50 %) throughout, while channels 0 to 6 reported their commanded `0xFF`. So channel 7 may have no PWM output at all, or one the EC drives from elsewhere; a host should not expect its readback to confirm a write |

Every channel was held at a fixed duty and handed back to the BIOS without error, and `0xA00` read `0x00` after each
hand-back (OBSERVED 2026-10-27). Headers 2 and 4-7 had nothing plugged in, so their physical positions rest on the
published bit order, not on a spin test.

The board is described with one CPU fan header, **two** pump headers, five system headers and two `T_SEN`
thermistor connectors (from a retailer listing; confirm against the manual). The second pump header has no known
control register; its RPM is probably fan slot 8 (section 4.3), as on the GODLIKE MAX sibling (INFERRED).

## 4. Register map

### 4.1 Temperatures

| Index | Register | LibreHardwareMonitor name (generic) | Notes |
|---|---|---|---|
| 0 | `0x100` | CPU | |
| 1 | `0x102` | System | |
| 2 | `0x104` | VRM MOS | |
| 3 | `0x106` | PCH | |
| 4 | `0x108` | CPU Socket | |
| 5 | `0x10A` | "PCIe x1" | on the GODLIKE MAX sibling these two are named Thermistor 1 and 2, so they may be `T_SEN1`/`T_SEN2` (INFERRED) |
| 6 | `0x10C` | "M2 #1" | as above |
| 7..10 | `0x10E`, `0x110`, `0x112`, `0x114` | not shown for this board | on the sibling: PCIe #1, PCIe #2, M2 #1, M2 #4 (PUBLISHED); meaning here UNKNOWN |

Decode: `°C = (signed byte at addr) + 0.5 × (bit 7 of addr+1)` (PUBLISHED).

### 4.2 Voltages

Fourteen 12-bit readings: `raw = 16 × byte(addr) + (byte(addr+1) >> 4)`, `V = 0.001 × raw`, then × 12 for +12 V,
× 5 for +5 V and × 2 for DIMM (PUBLISHED).

| Index | Register | Name (LibreHardwareMonitor default) |
|---|---|---|
| 0 | `0x120` | +12V (× 12) |
| 1 | `0x122` | +5V (× 5) |
| 2 | `0x124` | Vcore |
| 3 | `0x126` | Voltage #1 |
| 4 | `0x128` | DIMM (× 2) |
| 5 | `0x12A` | CPU I/O |
| 6 | `0x12C` | CPU System Agent |
| 7 | `0x12E` | Voltage #2 |
| 8 | `0x130` | AVCC3 |
| 9 | `0x13A` | CPU Termination |
| 10 | `0x13E` | VRef |
| 11 | `0x136` | VSB |
| 12 | `0x138` | AVSB |
| 13 | `0x13C` | CMOS Battery |

The Linux driver's default table applies × 2 to `0x126` instead of `0x128` (an AMD-board order); the Intel order
above is the plausible one for a Z790 board (INFERRED). Names other than +12V, +5V, Vcore and DIMM are not verified
on this board; sanity-check them (+12 V within 11.4-12.6, +5 V within 4.75-5.25, Vcore below 1.6).

### 4.3 Fan speeds

| Slots | Registers | Decode |
|---|---|---|
| 0..7 (the eight channels) | `0x140` + 2·i | RPM = `(byte(addr) << 8) | byte(addr+1)` (big-endian 16-bit) |
| 8..15 | `0x150`..`0x15E` step 2 | same; not shown by LibreHardwareMonitor for this board. Slot 8 is probably PUMP_FAN2 (INFERRED) |
| 16 | `0x852` | 13-bit count: `count = (byte(0x852) << 5) | (byte(0x853) & 0x1F)`; RPM = 1 350 000 / count for 21 ≤ count < 0x1FFF; `FF F8` means 0 RPM |

All PUBLISHED. OBSERVED on 2026-11-02: CPU_FAN1 2000-2083 RPM (an AIO pump's tach), SYS_FAN2 1080-1126 RPM, the
other channels 0.

### 4.4 Fan control

| Register | Meaning |
|---|---|
| `0xA00` | manual-mode bits, one per channel (section 3). Bit clear = the BIOS curve drives the channel |
| `0xA01` | write request handshake: `0x80` = request, `0x40` = complete |
| `0xA28` + i | commanded duty for channel i in manual mode, 0..255 |
| `0x160` + i | the duty the channel actually outputs, 0..255 (% = value / 2.55) |
| `0xCF8` | engine status used by the Linux driver: bit 3 configuration phase, bit 4 configuration invalid, bit 5 check done, bit 6 lock |

All PUBLISHED. Whether a header outputs PWM or DC follows the BIOS setting; the host writes the same byte either way.

**OBSERVED on 2026-11-02, with every channel under BIOS control:** `0xA00 = 0x00`; `0xA28`..`0xA2F` all `0x66`
(40 %), which the EC ignores while the bit is clear; outputs `0x160`+i: CPU_FAN1 `0x9B` (61 %), a minute later `0xB9`
(73 %), PUMP_FAN1 `0xFF` (100 %, nothing connected), system fans 1-5 `0x99` (60 %), channel 7 `0x80` (50 %).
Every channel had been held and handed back by third-party code before, so whether the BIOS itself leaves `0x66` in
`0xA28`+i is UNKNOWN. The same read on 2026-11-03 (BIOS control, `0xA00 = 0x00`, every command byte `0x66`) gave
CPU_FAN1 `0x9D`, PUMP_FAN1 `0xFF`, SYS_FAN1 and SYS_FAN3-5 `0x99`, **SYS_FAN2 `0x00`** (the BIOS curve had that
header at 0 %; a fan hub's tach lead is all that is plugged in there) and channel 7 `0x80` again, so the outputs
under BIOS control move with the curves except channel 7, which has read `0x80` at every observation.

**All eight channels under manual control at once** (OBSERVED 2026-11-03): a thermal emergency in the host program put
every channel to `0xFF` within one second (`0xA00 = 0xFF`), the outputs of channels 0 to 6 followed, and the hand-back
afterwards cleared every bit with the sequence of section 5.2 (two restore cycles for some channels, as under the
single-channel tests).

### 4.5 Identity bytes

`0x608`.`0x609` = firmware major.minor (decimal), `0x605` / `0x606` / `0x604` = build month / day / year, `0x607` = a
serial byte (PUBLISHED by the Linux driver). OBSERVED: firmware 1.0, build 11/01/22.

## 5. Sequences

### 5.1 Set a fixed duty on one channel

LibreHardwareMonitor's sequence for the plain NCT6687D (PUBLISHED):

1. write `0xA01` = `0x80`, wait 50 ms;
2. read `0xA00`, set bit i, write it back;
3. write `0xA28`+i = duty (0..255; 60 % = 153);
4. write `0xA01` = `0x40`, wait 50 ms.

It reads nothing back. Verify through `0x160`+i on the next read (channel 7 excepted: its output register never
followed a command, section 3). Clamp the percentage yourself before converting
(one library converts with an unclamped `byte(percent × 2.55)`).

The Linux driver uses a stronger handshake for this chip (PUBLISHED; not tried on this board): wait up to 1 s until
`0xCF8` bit 3 = 0 and `0xA01` bit 7 = 0; write `0xA01` = `0x80`; wait until `0xCF8` bit 6 = 0 and bit 3 = 1; change
`0xA00` and `0xA28`+i and read them back; write `0xA01` = `0x40`; wait for `0xCF8` bit 5, then expect bit 4 = 0 and
bit 6 = 1.

### 5.2 Hand a channel back to the BIOS

Clear bit i of `0xA00` inside a `0xA01` request/complete window, then **read `0xA00` back** and repeat if the bit is
still set. The BIOS curve resumes as soon as the bit is clear; the curve itself is not stored in these registers and
is not disturbed by manual mode (INFERRED).

OBSERVED 2026-10-27 for all eight channels (`0xA00` read `0x00` after each).

**Pitfall in a released library.** LibreHardwareMonitorLib 0.9.6 (the nuget.org release current in October 2026)
restores a channel with `mode & ~savedBit` instead of `(mode & ~(1 << i)) | savedBit`. For a channel that started
under the BIOS (saved bit 0), that leaves the bit **set**: the channel stays in manual mode at the saved `0xA28`+i
byte, which may be 0 % (read from the package's code; the source on its main branch has the corrected expression).
Software built on that release must clear the bit itself, check it by readback, and avoid the library's close-time
restore while a channel is unverified.

### 5.3 Read everything

One pass of the page/index/data sequence per register, under the ISA mutex. LibreHardwareMonitor can dump the whole
`0x000`-`0xFFF` space in one hold for diagnostics. Polling faster than about 250 ms gains nothing, because a write
alone takes 2 × 50 ms.

## 6. Standalone behaviour

- **The BIOS curves run in the EC without any host.** A host program that never writes `0xA00` cannot change fan
  behaviour.
- **A manual duty stays until the bit is cleared or power is removed.** There is no watchdog on this path
  (INFERRED: no driver describes one; the Linux "fan control watchdog" is a driver feature for another
  configuration). A host that crashes with a channel at 0 % leaves that fan stopped. A host must hand channels back
  on exit and on its own faults, and should never command a pump below a safe floor.
- **Sleep:** through an S3 sleep of 20 s, `0xA00` stayed `0x00` and the headers stayed on the BIOS curves
  (OBSERVED 2026-11-02). Whether a **manual** setting survives S3 is UNKNOWN; the Linux driver saves and restores
  `0x180` around suspend, which suggests the EC can lose some state (INFERRED). Re-check and re-assert after resume.
- **Restarts:** after full restarts `0xA00` read `0x00` (OBSERVED 2026-11-02). A full power cycle is INFERRED to
  reset the EC to the BIOS configuration; whether a warm restart keeps a manual setting is UNKNOWN.
- **No flash** is involved in any of these registers: fan control and sensor reads are EC RAM only (INFERRED).
- On the observed PC, neither the AIO pump on CPU_FAN1 nor the fan hub on SYS_FAN2 took its speed from the header:
  holding CPU_FAN1 at 100 %, and SYS_FAN2 at 100 % and at 30 %, changed nothing (OBSERVED 2026-10-27). Check what your devices do with a
  header's PWM before relying on it.

## 7. Other sensors on the board's buses

- **DDR5 DIMM temperatures** come from each module's SPD5118 hub on the chipset's SMBus (Intel i801, PCI
  `8086:7A23`), addresses `0x50`-`0x57`. Reading them needs a page-select write to the hub's MR11 register, under
  `Global\Access_SMBUS.HTP.Method` and the controller's hardware semaphore (PUBLISHED by LibreHardwareMonitor and
  RAMSPDToolkit). The DIMMs' RGB controllers sit on the same bus (PUBLISHED by OpenRGB). **Do not write anything
  else on this bus**: the hubs have writable configuration registers, and a wrong write to memory modules can cost
  more than any fan.
- CPU temperatures, power and clocks come from the CPU's own MSRs and are not part of this board.

## 8. Do not

| What | Why |
|---|---|
| Leave a channel in manual mode on exit or crash | no EC watchdog; the duty stays, possibly at 0 % |
| Trust a hand-back without reading `0xA00` | one released library leaves the bit set (section 5.2); writes can also be skipped silently on a mutex timeout |
| Hold `Global\Access_ISABUS.HTP.Method` for long | other programs give up after 10 ms and skip their writes silently |
| Write the EC from two programs | the last writer wins; curves oscillate |
| Write anything on the SMBus beyond the published temperature reads | memory modules |
| Use unsigned port-I/O drivers | blocked under Memory Integrity, and a security risk |
| Write CPU MSRs (power limits, voltage) through a port or MSR module | out of scope; can destabilise or damage the CPU |
