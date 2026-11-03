# MSI software and BIOS on the MEG Z790 ACE

Part of the [MSI MEG Z790 ACE reference](README.md). The tags are explained there.

This document describes what MSI's Windows software installs for this board, how it interacts with the two
controllers, how the BIOS brings it back on a fresh Windows, and how it was removed from the observed PC on
2026-11-02 with the board fully working afterwards. Everything here is HOST (read from the installed software,
its logs and the Windows device tree) or OBSERVED, unless tagged otherwise.

## 1. What MSI Center installed

Observed versions: MSI Center 2.0.74.0 (a Microsoft Store package, `9426MICRO-STARINTERNATION.MSICenter`) with
**MSI Center SDK** 3.2026.0911.01 (a classic installer in `C:\Program Files (x86)\MSI\MSI Center`, which holds the
modules, services and drivers), Mystic Light component 3.0.25.5.

| Kind | Items |
|---|---|
| Services | `MSI_Center_Service` (`MSI_Central_Service.exe`), `Mystic_Light_Service`, `LightKeeperService`, `MSI_Case_Service` |
| User-session processes | `LEDKeeper2.exe` (applies Mystic Light scenes), started and restarted by `Mystic_Light_Service` (INFERRED from its strings: `LEDKeeper2.exe`, `MSI Task Host - LEDKeeper2_Host`, `WTSGetActiveConsoleSessionId`, `CreateProcessAsUser`) |
| Kernel drivers | **`MSIO`** (`C:\Windows\System32\drivers\MsIo64.sys`, version 1.3, **System start: loaded at every boot**); `NTIOLib_MysticLight`, `NTIOLib_CC_COMM`, `NTIOLib_FWUpdate` (copies of `NTIOLib_X64.sys` inside the MSI Center folder, demand start). Both families give a program raw port access |
| Scheduled task | `\MSI Task Host - MSI.ToastServer` |
| Store packages | MSI Center, and MSI GameBar 3.0.23.0 |
| Bundled installs | the SDK's uninstall also removed `ENE_X_AIC_HAL`, `ENE Video Capture Box HAL` and `WD P40 Game Drive` (Windows Installer log), so the SDK installs third-party hardware layers silently |
| Settings | `HKLM\SOFTWARE\WOW6432Node\MSI\MSI Center\Component\<module>` (CoreLiquid, Mystic Light, and so on), `C:\ProgramData\MSI` |
| Other MSI modules present | AI Cooling, Case, Cube Fan, Gaming Gear, Gaming Mode, Graphics Fan Tool, Nahimic, Power Supply Unit, User Scenario, Voice Control, Live Update (`Support`). On the observed PC the PSU, Case and Cube Fan modules found no hardware |

Named mutexes in MSI's binaries (string listings): `Global\Access_USB_Sensors` (USB HID transfers to MSI devices,
including the Mystic Light board at start-up and the CoreLiquid AIO), `Global\Access_ISABUS.HTP.Method`,
`Global\Access_SMBUS.HTP.Method` and `Global\Access_EC`. LibreHardwareMonitor uses the same names (PUBLISHED), so
programs that honour them do not interleave their bus transactions.

## 2. Fans: MSI Center did nothing on this PC

With Windows' Memory Integrity (HVCI) on, MSI Center's fan engine logged on every start, every day for at least a
month: `Memory integrity : Enabled`, `LoadDriver : Fails`, `MB Not Support!`. Its port-I/O driver is blocked under
Memory Integrity, so the fan headers always ran on the BIOS Smart Fan curves. The fan controller is documented in
[nct6687d-fan-sensor-controller.md](nct6687d-fan-sensor-controller.md).

## 3. Lighting: how MSI re-applies

`LEDKeeper2` writes its scene to the Mystic Light controller (and to the GPU and DIMM lighting it manages):

- at its own start;
- at every **display power-on** (two passes about 3.5 s apart), and around display-configuration changes and USB
  arrivals (MSI's Mystic Light log, several times a day); OBSERVED 2026-10-27 after a monitor power cycle;
- continuously (a loop with a 50 ms wait) while a host-computed style such as a CPU-temperature colour or a
  weather style is active (INFERRED from the log's thread names and the settings that exist).

It never set the controller's save flag in two hours of captured use (CAPTURED), so the board's own flash image is
whatever was last saved by something else. Any other program's lighting is overwritten at the next trigger while
LEDKeeper2 runs. The wire format of these writes is in [msi-mystic-light-7d86.md](msi-mystic-light-7d86.md),
section 6.

MSI's local SDK service (`LightKeeperService`) held a UDP socket on `127.0.0.1:14621` and no TCP listener while its
SDK setting was off (HOST, socket listing 2026-10-18). MSI's public Mystic Light SDK (`MysticLight_SDK.dll`, `MLAPI_*` functions,
documented in MSI's PDF reference) talks to MSI's software, not to the hardware, and needs MSI's applications
installed (PUBLISHED by MSI).

## 4. How the BIOS reinstalls MSI software

**The chain** (HOST unless tagged; OBSERVED on 2026-11-02):

1. The BIOS option **"MSI Driver Utility Installer"** exposes an ACPI device, **`ACPI\MBID0002\1`**, named "MSI
   Software Component Interface", with hardware IDs `ACPI\VEN_MBID&DEV_0002`, `ACPI\MBID0002` and `*MBID0002`.
2. Windows Update installs MSI's driver package **`msiswacpi.inf`** for it (provider Micro-Star INT'L CO., LTD.,
   class System, signed by the Microsoft Windows Hardware Compatibility Publisher). Two versions were in the driver
   store: 1.0.0.15 (2026-01-07) and 1.0.0.16 (2026-07-07).
3. The package installs **no driver** (`Needs = NO_DRV`). It copies `MSI_Driver_Utility_Installer.exe` and
   `ModulInterface.dll` into the driver store and registers them as a **software component** (`AddSoftware`,
   `SoftwareType = 1`, `SoftwareBinary = MSI_Driver_Utility_Installer.exe`, `SoftwareArguments = /task`), which
   Windows runs when the device is installed (INFERRED from how software components work).
4. The program keeps a copy in `C:\Program Files (x86)\MSI\MSI_Driver_Utility_Installer`; its log shows it
   enumerating the PC's devices and recording the board model and BIOS version. It is MSI's way of offering its
   drivers and MSI Center on a new Windows (INFERRED from its name and web reports). On the observed PC it last ran
   on 2026-07-29.

The same INF matches many ACPI IDs in four families (`MBID0001`-`000F`, `MBAD0001`-`000F`, `DTID0001`-`000F`, and
`DTAD0001` onwards; the listing read was cut after `DTAD000C`), so the mechanism is shared across MSI's boards and
desktops (INFERRED from the ID names).

**Turning it off.** Set "MSI Driver Utility Installer" to **Disabled** in the BIOS (web reports place it under the
advanced settings, near the Windows OS configuration; the exact path on BIOS 1.G0 was not recorded). After a
restart, `ACPI\MBID0002` is **gone** from the device tree (OBSERVED 2026-11-02, BIOS 1.G0). On the first attempt the
setting had not actually been saved, and the device was still there after two restarts: **check the device tree after
changing the option.** Web reports say that BIOS updates and a Windows update (late 2025) switched the option back on
for some users (PUBLISHED by Born's Tech and Windows World; not seen here), so check again after every BIOS update.

Then delete the driver packages (elevated):

```
pnputil /enum-drivers           (find the oemNN.inf entries whose original name is msiswacpi.inf)
pnputil /delete-driver oemNN.inf /uninstall
```

Delete them only after the device is gone; while it exists, Windows Update would install the package again.

## 5. Removing MSI Center (done 2026-11-02)

What worked on the observed PC, with the board's lighting and fans fully working on Windows' own drivers afterwards:

1. **Before:** export `HKLM\SOFTWARE\WOW6432Node\MSI` and copy `C:\ProgramData\MSI` (the scenes and component
   settings); check firmware updates while MSI Center still exists (it is the only updater for the Mystic Light
   controller; the BIOS can be updated with M-Flash from a USB stick).
2. **Uninstall** the MSI Center app, then **MSI Center SDK**, in Settings > Apps (MSI Afterburner and RivaTuner
   Statistics Server, if installed, are separate entries).
3. **Restart** (not Shut down: with Windows' Fast Startup on, Shut down hibernates the kernel and loaded drivers stay
   loaded), set the BIOS option to Disabled (section 4), and restart again.
4. **Remove what the uninstallers leave.** On the observed PC they left:

   | Leftover | Removal |
   |---|---|
   | the `MSIO` driver service and `MsIo64.sys`, still loading at every boot | `sc delete MSIO`, restart, then delete the file |
   | the service `MSI_Center_Service` (its program already gone) | `sc delete MSI_Center_Service` |
   | the driver service `NTIOLib_MysticLight` and its `.sys` | `sc delete NTIOLib_MysticLight`, delete the folder |
   | the scheduled task `\MSI Task Host - MSI.ToastServer` | delete the task |
   | `C:\Program Files (x86)\MSI` (MSI Center, MSI_Driver_Utility_Installer, MSI Companion) and `C:\ProgramData\MSI` | delete |
   | the MSI Center and MSI GameBar Store packages, still **staged for the SYSTEM account** after the per-user uninstall (only an elevated `Get-AppxPackage -AllUsers` shows them) | `Remove-AppxPackage -AllUsers` |
   | the `msiswacpi.inf` driver packages | section 4 |
   | `HKLM\SOFTWARE\WOW6432Node\MSI` | delete (after the export) |

5. **Check:** the Mystic Light controller binds to `input.inf` (`HidUsb`), no `MSIO` or `NTIOLib` service is
   registered, `ACPI\MBID0002` is absent, and no MSI service, task, package or folder is left. All of this held on the
   observed PC after the final restart, and the board lights came up in their scene (OBSERVED 2026-11-02).

Keep: the Realtek audio packages with `msi` in their names (`rtdusbad_msi.inf`, `extrtxusb_msi_rtk.inf`) are the
board's onboard audio drivers, not MSI Center.

## 6. What is lost without MSI Center

- The Mystic Light controller's firmware updates (a temporary MSI Center install would be needed).
- Live Update for drivers and BIOS (BIOS updates work with M-Flash).
- Host-computed lighting styles (CPU temperature, weather, music) unless another program streams them; per-LED
  direct mode (section 7 of the Mystic Light document) makes that possible.
- MSI's extras (Nahimic, Gaming Mode, User Scenario performance presets; CPU limits can be set in the BIOS instead).
