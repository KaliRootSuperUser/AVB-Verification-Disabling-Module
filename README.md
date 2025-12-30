# AVB-Disabler-with-Rescue — Separated AVB Disabling & Boot Recovery Module

A separated module for disabling Android Verified Boot (AVB) and automatically recovering from boot failures.

## 1. Project Introduction

This project is a separated solution for disabling Android Verified Boot (AVB) and for boot recovery on Android devices. It addresses two core problems:

1. Securely disable AVB verification to bypass system signature restrictions and allow installation of custom modules.
2. Mitigate the "brick" risk caused by disabling AVB by providing both automatic and manual recovery mechanisms to ensure device usability.

Compared with traditional solutions, this project uses an "adaptation layer + emergency layer + verification layer" to improve compatibility across more device models, lower operational difficulty, and strengthen safety controls. It is only recommended for legitimate development and testing scenarios (disabling AVB reduces device security — do not use on sensitive daily-driver devices).

## 2. Core Features

### 2.1 AVB Disabler Module (AVB-Disabler)

- Multi-device adaptation: Automatically detects the device model and loads the corresponding vbmeta partition path (supports mainstream models such as Samsung S22, Xiaomi Mi13, Google Pixel 7, including dynamic-partition devices).
- Pre-installation checks: Automatically checks Android version (Android 8.0+ required), Magisk version (Magisk 20.4+ required), and bootloader unlock status; installation is aborted if requirements are not met.
- Security reminder: Shows a risk notice via Magisk during installation and requires user confirmation to proceed.
- Property spoofing: Spoofs system properties (e.g., `ro.boot.verifiedbootstate=green`) to evade AVB-state checks used by some systems.

### 2.2 Boot Recovery Module (Brick-Rescue)

- Automatic backup: On first boot, automatically backs up the original vbmeta partition to `/data/adb/avb_backup/` to enable recovery.
- Intelligent boot detection: Uses a 30-second timeout (tuned for low-end devices) and checks both `sys.boot_completed` and Magisk service status to reduce false positives.
- Tiered recovery mechanism:
  - 3 boot failures: automatically enter safe mode and retry.
  - 5 boot failures: automatically restore the original vbmeta, disable the AVB module, and reboot; the device should return to normal.
- Offline emergency recovery: Provides `offline_recovery.zip` (includes Fastboot tools for Windows/Linux/macOS) for recovery via a computer when ADB is unavailable.
- Status logging: Generates `/sdcard/boot_failure_report.txt` with failure reasons and recovery suggestions for troubleshooting.

### 2.3 Combined Installation Package (AVB-Disabler-With-Rescue)

- One-click dual install: Flashing once installs both the AVB disabler and the rescue module.
- Incremental updates: Detects installed module versions and updates only changed files to save install time.
- Device whitelist: Checks `docs/device_list.md` for verified devices before installation and warns for unsupported models.

## 3. Requirements

### Device

- Android 8.0 or later.
- Bootloader unlocked.
- Magisk 20.4 or later installed and properly activated (supports Magisk Hide).
- At least 100 MB free storage for vbmeta backups and logs.

### Computer (optional — for emergency recovery)

- Android SDK Platform Tools installed (ADB and Fastboot).
- Windows / Linux / macOS supported (offline recovery package includes multi-OS Fastboot tools).

## 4. Installation Guide

### Method 1 — Combined package (recommended)

1. Download `AVB-Disabler-With-Rescue.zip` from the project root.
2. Open Magisk Manager → Modules → Install from storage → select the ZIP file.
3. Wait for installation to complete (a risk notice will appear; tap Confirm).
4. Reboot the device. Modules become active automatically (check status in Magisk Manager).

### Method 2 — Install modules separately (advanced)

1. Download `AVB-Disabler.zip`, install via Magisk, and reboot.
2. Download `Brick-Rescue.zip`, install via Magisk, and reboot again.
3. Verify: check `/data/adb/modules/` for folders `avbdisabler` and `brickrescue` — presence indicates successful installation.

## 5. Manual Recovery Operations

If the device fails to boot (e.g., stuck on boot logo), use one of the following recovery methods:

### 5.1 Recovery via ADB (recommended — requires ADB debugging enabled)

1. Connect the device to a computer and open a terminal/command prompt.
2. Run:
```bash
adb shell sh /data/adb/modules/brickrescue/system/bin/avb_recovery --force
```
3. The device will restore the original vbmeta and reboot. After reboot, the AVB disabler will be disabled and the device should boot normally.

### 5.2 Recovery via Recovery mode (no ADB)

1. Boot into TWRP or another custom recovery (commonly by holding Power + Volume Up).
2. Use the File Manager to delete `/data/adb/modules/avbdisabler`.
3. Reboot; the AVB disabler module will be inactive and the device should resume normal boot.

### 5.3 Recovery via Fastboot (extreme cases — cannot enter system or recovery)

1. Extract `emergency/offline_recovery.zip` on a computer and locate the Fastboot tool for your OS.
2. Boot the device into Fastboot mode (commonly Power + Volume Down) and connect to the computer.
3. Execute (you must have the original `vbmeta.img` from official firmware):
```bash
# For Windows
fastboot.exe flash vbmeta vbmeta_original.img

# For Linux/macOS
./fastboot flash vbmeta vbmeta_original.img
```
4. Then run:
```bash
fastboot reboot
```

## 6. Troubleshooting (FAQ)

Q1: Device won't boot after installation  
A1: Boot into TWRP and delete `/data/adb/modules/avbdisabler`, or run:
```bash
adb shell sh /data/adb/modules/brickrescue/system/bin/avb_recovery --force
```
to restore the original vbmeta.

Q2: The rescue module didn't trigger automatic recovery after boot failures  
A2:
1. Confirm Magisk is activated (check `/data/adb/magisk/` for `magiskd.pid`).
2. Ensure `/data/adb/brick_rescue` directory permissions are `0755` (modify via Recovery if needed).

Q3: "Device not supported" shown during install  
A3: Check `docs/device_list.md` for verified devices. For niche models, manually edit `AVB-Disabler/device_adapt/vbmeta_paths.conf` to add your device's vbmeta partition path (you must determine the device's partition layout yourself).

Q4: After restoring vbmeta, Magisk shows "Not installed"  
A4: Re-flash the Magisk patched boot image (`magisk_patched-xxx_boot.img`) via Fastboot or re-install Magisk via Recovery.

## 7. File Structure

AVB-Disabler-with-Rescue/  
├── AVB-Disabler/                # AVB Disabler Module  
│   ├── device_adapt/            # Device adaptation files  
│   │   ├── vbmeta_paths.conf    # Device model → partition path mappings  
│   │   └── prop_override.sh     # Device-specific property override script  
│   ├── common/                  # Core scripts  
│   │   ├── post-fs-data.sh      # Script executed after filesystem is mounted  
│   │   └── system_hook/         # System hook scripts  
│   ├── system/                  # System configuration files  
│   ├── check_compatibility.sh   # Compatibility check script  
│   └── module.prop              # Module metadata  
│  
├── Brick-Rescue/                # Boot Recovery Module  
│   ├── emergency/               # Emergency recovery assets  
│   │   ├── offline_recovery.zip # Offline recovery package  
│   │   └── fastboot_cmd.txt     # Fastboot command templates  
│   ├── status/                  # Status logs  
│   ├── system/bin/              # Core binaries (e.g., avb_stealthd, diag_system, etc.)  
│   └── module.prop              # Module metadata  
│  
├── AVB-Disabler-With-Rescue/    # Combined installer package  
│   ├── modules/                 # Sub-module ZIP files  
│   ├── verify/                  # Verification files (MD5, compatibility checks)  
│   └── scripts/                 # Installation scripts  
│  
├── docs/                        # Documentation  
│   ├── troubleshooting.md       # Troubleshooting guide  
│   └── device_list.md           # Verified device list  
│  
├── .gitignore  
├── LICENSE  
└── README.md

## 8. Notes

1. Legality & Safety
- Use only for legitimate development and testing. It is prohibited to use this project to break into others' devices, invade privacy, or carry out illegal activities.
- Disabling AVB reduces device security and increases exposure to malicious software. Do not store sensitive data (banking info, passwords) on devices with AVB disabled.
- Back up important data before proceeding (TWRP full backup recommended or export important files to a computer).

2. Compatibility additions
- Supports Android 14 and vbmeta v3 format.
- For dynamic-partition devices, ensure the correct path (e.g., `/dev/block/mapper/vbmeta`) is configured in `device_adapt/vbmeta_paths.conf`.
- Some vendors (e.g., Huawei, OPPO) may have additional locking mechanisms that must be addressed first (e.g., Huawei FRP).

3. Permission statement
- Project owner: "好无聊" (Hao Wuliao). Users are permitted to read, use, and make non-destructive improvements. Commercial sale, sublicensing, or modification of core logic (e.g., splitting modules or changing recovery mechanisms) is prohibited.

## 9. License

This project is licensed under the Apache License 2.0 (a permissive open-source license). Summary of key points:

- Rights: Free to use, copy, distribute, and modify the code and documentation; permitted to create derivative works and use them commercially.
- Obligations: When distributing copies, retain copyright/patent/trademark notices and attribution to the project owner ("Hao Wuliao"); add clear change notices to modified source files; include the full Apache 2.0 license text with distributed copies; patent grants from contributors are non-exclusive and worldwide and must be preserved for downstream users; do not use the project's name or logo for promotion without written permission.
- Disclaimer: Provided "as is" — no warranties about usability, stability, or security. The project owner and the Apache Software Foundation are not liable for damages (device damage, data loss, commercial loss, etc.).

See the `LICENSE` file in the repository root or https://www.apache.org/licenses/LICENSE-2.0 for the full license text.

## 10. Contact

If you encounter unsupported device adaptation issues or find bugs, please report via:

- GitHub Issues: open an issue in this repository and include device model, Android version, reproduction steps, and logs.
- Email: cailin44@outlook.com

Please include "AVB-Disabler-with-Rescue Feedback" in the subject to help with triage.

Note: Some content in the original README was truncated or elided with "[...]" — this translation follows the provided text and retains such omissions where present.
