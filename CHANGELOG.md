# Changelog — Dell Latitude 3500 Hackintosh Tahoe

Tous les changements notables pour ce projet sont documentés dans ce fichier.

Format basé sur [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

---

## [v2.1] — 2026-08-23

### 🔧 Fixed

- **Audio clarification**: AppleALC + KDK + MyKextInstaller is the confirmed working stack for Tahoe (not VoodooHDA)
  - Post-install procedure: install KDK, run MyKextInstaller, restore AppleHDA.kext to `/System/Library/Extensions`
  - `alcid=19` for Latitude 3500 verified stable
  - `csr-active-config=0x08030003` required (SIP partially disabled for audio framework access)

- **SSDT-PNLF backlight fix documented**: `_UID = 0x13` (0x10 caused brightness capped at ~2%)
  - Applies to Coffee Lake / Whiskey Lake-U (i5-8365U)
  - NVRAM GUID `4D1EDE05-38C7-4A6A-9CC6-4BCCA8B38C14` for brightness persistence

- **Trackpad I2C initialization order clarified**: VoodooI2C dependency load sequence matters
  - VoodooI2CServices → VoodooGPIO → VoodooInput → VoodooI2C → VoodooI2CHID
  - SSDT-XOSI + `_OSI→XOSI` patch required for DELL08BD recognition

### ✨ Added

- **Comprehensive hardware specifications table** in README
- **BIOS configuration guide** with exact Dell UEFI settings (SATA=AHCI, SecureBoot=Off, etc.)
- **Troubleshooting section** covering:
  - Brightness capped to low value → SSDT-PNLF _UID fix
  - No Bluetooth → Intel chipset unsupported in Tahoe → USB dongle recommendation
  - WiFi absent/slow → itlwm kext enable check, HeliPort install
  - Sleep/wake issues → BIOS Wake-on-LAN, USB port mapping
- **USB cartography** (`hardware Details/usb.json`) for reference and troubleshooting
- **GitHub ISSUE_TEMPLATE** for bug reports with required info (Tahoe version, BIOS, logs)
- This CHANGELOG.md

### 🔄 Changed

- **Bluetooth stack removed entirely** from bootloader configuration
  - ❌ IntelBluetoothFirmware.kext (Tahoe incompatibility)
  - ❌ IntelBTPatcher.kext (caused kernel panics on both AC9560 and AX200)
  - ❌ BlueToolFixup.kext (no longer needed)
  - ❌ Boot-arg: `-ibtcompatbeta` removed
  - ✅ **Recommended solution**: USB Bluetooth dongle (CSR8510 or BCM20702, ~25 EUR, zero configuration)

- **NVMe format reminder in README**: KXG60ZNV512G MUST stay in 512e sector format
  - 4Kn reformatting causes APFS snapshot reversion loop → infinite installation loop
  - Use `nvme format` command cautiously; verify with `nvme id-ns -H /dev/nvme0n1`

- **OpenCore Timeout value corrected** to `10` seconds (was causing installer reboot loops at `0`)

- **Framebuffer DVMT patching validated**:
  - Stolen memory: `0x00003C00` (~32 MB, locked by Dell BIOS)
  - Framebuffer memory: `0x00009000` (4 MB for VRAM allocation)
  - Device-ID spoof: `0x3E9B` (Coffee/Whiskey Lake compatibility)
  - ig-platform-id: `0x3E9B0000`

### 📚 Documentation Improvements

- README now includes:
  - ✅ Feature support matrix (working / limited / impossible)
  - ✅ Step-by-step installation guide (2-3 hours end-to-end)
  - ✅ Post-installation audio setup procedure
  - ✅ BIOS settings table (10+ key toggles)
  - ✅ Device Properties framebuffer config (all fields explained)
  - ✅ Security & privacy notes (SIP, FileVault, iCloud compatibility)

### 🔍 Known Issues

- **Bluetooth**: No native Bluetooth on this platform under Tahoe (Intel chipset limitation)
  - Workaround: USB Bluetooth 4.0 dongle (CSR8510 detected natively, no kexts)
  - Dongle does NOT support Continuity features (iCloud handoff, AirDrop, etc.)

- **DRM Video**: 1080p max (no 4K streaming)
  - Reason: SIP partially disabled for audio framework
  - Affects: Netflix, Apple TV+, Disney+ in browser (fine via native apps)

- **Continuity**: Never supported (WiFi is CNVi/itlwm, not native macOS handoff)

- **macOS 27+**: Not possible (Tahoe = final x86 release; macOS 27 = ARM only)

### 🧪 Tested Configuration

- **OS**: macOS 26 Tahoe (26.0, 26.0.1)
- **Bootloader**: OpenCore 1.0.7+
- **BIOS**: Dell Latitude 3500 BIOS v1.36.0 (UEFI)
- **CPU**: Intel Core i5-8365U (Whiskey Lake-U, CPUID 806EC)
- **GPU**: Intel UHD Graphics 620 (DVMT 32 MB)
- **RAM**: 16 GB DDR4-2400 (dual-channel)
- **SSD**: Kioxia KXG60ZNV512G 512 GB (NVMe Gen3, 512e format)
- **WiFi**: Intel Wireless-AC AX200 (CNVi, via itlwm + HeliPort)
- **Audio**: Realtek ALC3271 → AppleALC (layout-id=19) via KDK + MyKextInstaller
- **Trackpad**: DELL08BD (I2C0, address 0x2C) via VoodooI2C + VoodooI2CHID
- **Ethernet**: Realtek RTL8111 GbE

### 🔗 References

- [OpenCore Documentation](https://github.com/acidanthera/OpenCorePkg)
- [Dortania Hackintosh Guide](https://dortania.github.io)
- [VoodooI2C Project](https://github.com/VoodooI2C/VoodooI2C)
- [OpenIntelWireless](https://github.com/OpenIntelWireless)

---

## [v2.0] — 2026-07-15

### ✨ Initial Release for macOS 26 Tahoe

- **Full OpenCore bootloader configuration**
  - SMBIOS: MacBookPro16,2 (Tahoe-compatible, MacBookPro15,2 unsupported)
  - Custom GUID enabled for Dell-specific quirks
  - All ACPI patches (OSID→XSID, _OSI→XOSI, ECDV→XSTA, PNLF→XNLF)

- **GPU acceleration verified**
  - Metal 3 support, QuickSync H.264/H.265/VP9 encoding
  - Framebuffer patches for DVMT 32 MB limitation
  - HDMI output functional with connector patching

- **Audio stack** (initial)
  - AppleALC + layout-id 19 for ALC3271 codec
  - KDK + MyKextInstaller post-installation

- **Trackpad multitouch** via VoodooI2C
  - 2-finger scroll, 3-finger swipe, gesture support
  - DELL08BD (Cypress protocol) on I2C0

- **WiFi connectivity**
  - Intel AC9560 (original) or AX200 (tested drop-in replacement)
  - itlwm kext + HeliPort UI (no native WiFi menu)
  - ~150 Mbps real-world throughput

- **Battery & power management**
  - Accurate battery percentage display (±5%)
  - CPU power management via SSDT-PLUG
  - Sleep/wake stable, no phantom wakeups

- **USB port mapping**
  - 18 ports detected (13×USB 2.0 + 5×USB 3.x)
  - Internal ports (Bluetooth, webcam, card reader) excluded from SMC limit
  - External ports functional

- **Comprehensive README** (French + English)
  - Hardware specifications
  - Step-by-step installation (2-3 hours)
  - Troubleshooting guide
  - Feature matrix (working/limited/impossible)

### ⚠️ Known Limitations (v2.0)

- **Bluetooth**: Intel AC9560 shows device but non-functional (driver incompatibility under Tahoe)
- **Continuity**: Never supported (WiFi is CNVi, lacks native integration)
- **DRM video**: Limited to 1080p (SIP partial disable required)
- **FileVault**: Partial incompatibility with SIP settings (not recommended)

---

## [v1.0] — 2026-06-01

### 🚀 Early Tahoe Proof-of-Concept

- Initial OpenCore EFI configuration for Latitude 3500
- Basic system boot and installation verified
- Core ACPI tables (SSDT-EC, SSDT-PLUG, SSDT-XOSI)
- iGPU acceleration working (Metal, basic rendering)
- Trackpad PS/2 fallback (no I2C yet)

### ⚠️ Known Issues (v1.0)

- Trackpad only PS/2, no multitouch
- WiFi driver hunt in progress
- Audio codec probing
- Bluetooth untested

---

## Contribution Notes

### For Maintainers

- **Test before tagging**: Boot OpenCore picker, verify each major component (audio, WiFi, trackpad, display)
- **SSDT compilation**: Use `iasl` to compile `.dsl` → `.aml`; verify plist patch hex values match
- **BIOS updates**: Dell may release firmware after v1.36.0 — test compatibility before recommending
- **KDK + MyKextInstaller**: Procedure may change in future Tahoe point releases; document any divergence
- **Hardware variations**: Some Latitude 3500 units may have different codecs/wifi cards — encourage users to report via GitHub Issues

### For Users

- **Backup your EFI** before editing config.plist
- **Use GenSMBIOS** to generate fresh SMBIOS values (never reuse from this repo)
- **Read the BIOS guide carefully** — wrong settings cause boot loops
- **Post logs, not screenshots** when reporting bugs (ioreg, system logs, OpenCore boot messages)

---

## Version Numbering

- **Major (v#)**: Large feature additions or platform changes (e.g., switch to Tahoe)
- **Minor (#.#)**: Bug fixes, new component support, documentation improvements
- **Patch (#.#.#)**: Reserved for typo/cosmetic fixes (rarely used)

Latest recommended version: **v2.1** (current stable)

---

*Last updated: August 23, 2026*  
*Tested with: macOS Tahoe 26.0.1 final, OpenCore 1.0.7+, BIOS v1.36.0*
