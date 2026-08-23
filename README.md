# 🍎 Hackintosh Dell Latitude 3500 — macOS 26 Tahoe

<div align="center">

![Dell Latitude 3500](https://img.shields.io/badge/Dell-Latitude%203500-0084D1?style=for-the-badge&logo=dell)
![macOS Tahoe](https://img.shields.io/badge/macOS-26%20Tahoe-000000?style=for-the-badge&logo=apple)
![OpenCore](https://img.shields.io/badge/OpenCore-1.0.7+-4285F4?style=for-the-badge)
![Intel](https://img.shields.io/badge/Intel-Core%20i5--8365U-0071C5?style=for-the-badge&logo=intel)

[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![GitHub stars](https://img.shields.io/github/stars/lepekinoi/Hackintosh-DELL-Latitude-3500?style=flat-square)](https://github.com/lepekinoi/Hackintosh-DELL-Latitude-3500)
[![Last Commit](https://img.shields.io/github/last-commit/lepekinoi/Hackintosh-DELL-Latitude-3500?style=flat-square)](https://github.com/lepekinoi/Hackintosh-DELL-Latitude-3500/commits)

**Configuration EFI complète et documentée pour faire tourner macOS 26 Tahoe sur un Dell Latitude 3500.**

[📖 Guide complet](#-guide-complet) • [⚙️ Spécifications](#️-spécifications-hardware) • [✨ État du projet](#-état-du-projet) • [🚀 Installation](#-installation-rapide)

</div>

---

## 📸 Aperçu Machine

<div align="center">

| Composant | Détail |
|-----------|--------|
| **💻 Modèle** | Dell Latitude 3500 (2019) |
| **🔧 CPU** | Intel Core i5-8365U Whiskey Lake-U (4C/8T, 15W) |
| **🎮 GPU** | Intel UHD Graphics 620 (DVMT 32 MB) |
| **🧠 RAM** | 16 GB DDR4 2400 MHz (2×8 GB Dual Channel) |
| **💾 SSD** | Kioxia/Toshiba KXG60ZNV512G 512 GB (PCIe Gen3, **512e**) |
| **📡 WiFi** | Intel Wireless-AC AX200 WiFi 6 ✅ |
| **🔊 Audio** | Realtek ALC3271 (AppleALC + KDK) ✅ |
| **🖥️ Écran** | 15.6" FHD 1920×1080 ✅ |
| **🛑 BIOS** | Dell 1.36.0 (UEFI) |

</div>

---

## 🎯 État du Projet

<div align="center">

### ✅ Fonctionnel

| Fonctionnalité | État | Notes |
|:---:|:---:|:---|
| 🚀 **Boot & Installation** | ✅ | OpenCore stable, installation complète |
| 🎬 **Accélération GPU** | ✅ | Metal 3, QuickSync, encodage/décodage vidéo |
| 🔊 **Audio** | ✅ | AppleALC + KDK post-install, layout-id 19, prise jack détectée |
| 📡 **WiFi** | ✅ | Intel AX200, itlwm + HeliPort, ~150 Mbps |
| 🖱️ **Trackpad** | ✅ | I2C DELL08BD, multitouch complet, VoodooI2C |
| 🔆 **Luminosité** | ✅ | Plage complète (Fn-keys), persistance NVRAM |
| 🔋 **Batterie** | ✅ | Affichage pourcentage, cycles détectés |
| 💤 **Veille/Réveil** | ✅ | Stable, pas de réveils fantômes |
| 📱 **USB** | ✅ | Cartographie complète, ports internes/externes |
| 📺 **HDMI** | ✅ | Vidéo + audio, patch connecteur Bus ID |

### ⚠️ Limité

| Fonctionnalité | État | Raison |
|:---:|:---:|:---|
| 🔵 **Bluetooth** | ❌ | Régression Tahoe, chipset Intel non supporté → **Dongle USB recommandé** |
| 🤝 **Continuité Apple** | ❌ | WiFi CNVi jamais supporté nativement |
| 🎬 **DRM 4K** | ❌ | Netflix/Apple TV+ en 1080p navigateur seulement |

### 🚫 Impossible

| Fonctionnalité | État | Raison |
|:---:|:---:|:---|
| 🆔 **Touch ID / Apple Pay** | ❌ | Pas de puce T2 |
| 🎯 **macOS 27+** | ❌ | Tahoe = dernière version x86, macOS 27 = ARM only |
| 🔒 **FileVault complet** | ❌ | Incompatible avec SIP partiellement désactivé (AppleALC) |

</div>

---

## ⚙️ Spécifications Hardware

### 📊 Tableau Détaillé

```
┌─────────────────────────────────────────────────────────┐
│         🍎 Dell Latitude 3500 Hackintosh Tahoe          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  🔧 Processeur & Chipset                              │
│  ├─ CPU: Intel Core i5-8365U (Whiskey Lake-U)         │
│  ├─ Cores/Threads: 4C / 8T @ 1.60-3.90 GHz            │
│  ├─ TDP: 15 W (ultrabook)                             │
│  ├─ Cache: L2 1024 kB / L3 6144 kB                    │
│  ├─ CPUID: 806EC (confirmé)                           │
│  └─ Chipset: Intel Platform Controller Hub (PCH)      │
│                                                         │
│  🎮 Graphique                                          │
│  ├─ iGPU: Intel UHD Graphics 620                      │
│  ├─ VBIOS: 9.0.1085                                   │
│  ├─ DVMT: 32 MB (verrouillé, nécessite patch)        │
│  ├─ Sorties: eDP (écran interne) + HDMI              │
│  └─ Encodage: H.264, H.265, VP9 natif (QuickSync)   │
│                                                         │
│  💾 Mémoire & Stockage                               │
│  ├─ RAM: 16 GB DDR4-2400 (2×8 GB dual channel)       │
│  ├─ SSD: Kioxia KXG60ZNV512G 512 GB                  │
│  │   └─ Interface: M.2 NVMe PCIe Gen3                │
│  │   └─ Format: 512e natif (⚠️ NE PAS reformater)   │
│  └─ Secteurs 2,5": Vide (baie SATA-0 disabled)       │
│                                                         │
│  📡 Connectivité                                       │
│  ├─ WiFi: Intel Wireless-AC AX200 WiFi 6             │
│  ├─ BT: Bluetooth Intel (non fonctionnel → USB)       │
│  ├─ Ethernet: Realtek RTL8111 GbE                     │
│  │   └─ MAC: E4:54:E8:3F:4A:73                       │
│  └─ Cartes: 1×USB-C, 3×USB-A (+ internes)           │
│                                                         │
│  🔊 Audio & Affichage                                 │
│  ├─ Codec: Realtek ALC3271 (réf. OEM Dell)           │
│  ├─ Sorties: Haut-parleurs + Combo jack 3.5 mm       │
│  ├─ Écran: 15.6" FHD 1920×1080 IPS                   │
│  └─ Webcam: HD (USB interne)                         │
│                                                         │
│  🔋 Alimentation                                      │
│  ├─ Batterie: Lithium-ion (compatible ACPI)          │
│  ├─ Adaptateur: 65 W USB-C Power Delivery            │
│  └─ Gestion: Stockage NVRAM, % persistant             │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🚀 Installation Rapide

### ⏱️ Durée Estimée : 2-3 heures

#### **Prérequis**
- ✅ Clé USB 16 GB (ou plus)
- ✅ Accès à un Mac ou une VM macOS
- ✅ Outils : **OpenCore Legacy Patcher**, **MacForge**, ou **OCLP**
- ✅ Fichiers du dépôt (config.plist, SSDT, drivers, kexts)

#### **Étape 1 : Préparation**
```bash
# Cloner ou télécharger le repo
git clone https://github.com/lepekinoi/Hackintosh-DELL-Latitude-3500.git
cd Hackintosh-DELL-Latitude-3500

# Sauvegarder l'EFI actuel (si présent)
cp -R EFI/ EFI_backup_$(date +%Y%m%d)/
```

#### **Étape 2 : Créer la clé USB d'installation**
```bash
# Télécharger le fichier InstallAssistant de Tahoe
# Via App Store ou OpenCore Legacy Patcher

# Créer le média d'installation
sudo /path/to/InstallAssistant.app/Contents/Resources/startosinstall \
  --volume /Volumes/MyUSB \
  --agreetolicense --nointeraction
```

#### **Étape 3 : Copier l'EFI**
```bash
# Monter la partition EFI de la clé USB
diskutil list
diskutil mount disk0s1  # Adapter le disque/partition

# Copier l'EFI
cp -R EFI/ /Volumes/EFI/
```

#### **Étape 4 : Réglages BIOS**
Redémarre et presse **F2** pour accéder au BIOS Dell :

| Réglage | Valeur |
|---------|--------|
| **System Information → BIOS Version** | 1.36.0 ou + |
| **System Configuration → SATA Operation** | AHCI |
| **System Configuration → NVMe Option ROM** | Enabled |
| **System Configuration → Secure Boot** | Disabled |
| **System Configuration → Integrated NIC** | Enabled |
| **Power Management → Wake on LAN** | Disabled |
| **UEFI Boot Options → Boot Mode** | UEFI |
| **UEFI Boot Options → Secure Boot Enable** | Disabled |

#### **Étape 5 : Premier Boot OpenCore**
```
1. Redémarre, presse F12 pour le menu boot
2. Sélectionne ta clé USB
3. Choisis « Install macOS » dans OpenCore Picker
4. Suit l'assistant d'installation
5. ≈ 15-20 minutes, plusieurs reboots normaux
```

#### **Étape 6 : Post-Installation Audio (IMPORTANT)**

⚠️ **L'audio nécessite une installation post-système** avec KDK + MyKextInstaller.

**Procédure complète :**

1. **Télécharge et installe le KDK (Kernel Debug Kit) pour Tahoe 26.0**
   - Visite : https://developer.apple.com/download/
   - Cherche « Kernel Debug Kit macOS 26 »
   - Télécharge le `.dmg` correspondant à ta version Tahoe
   - Installe en double-cliquant le `.dmg`

2. **Télécharge MyKextInstaller**
   - Depuis : https://github.com/acidanthera/MacKernelSDK
   - Ou utilise l'outil officiel Apple pour restaurer les kexts

3. **Restaure AppleHDA.kext dans le système**
   ```bash
   # Le KDK contient AppleHDA.kext
   # Copie-le vers /System/Library/Extensions/ via MyKextInstaller
   # OU via Terminal (mode recovery recommandé) :
   
   # Au redémarrage, appuie Cmd+R pour la recovery
   # Ouvre Terminal dans Utilities
   sudo mount -uw /
   cp -r /path/to/AppleHDA.kext /System/Library/Extensions/
   chown -R root:wheel /System/Library/Extensions/AppleHDA.kext
   ```

4. **Rebuild kernel cache et réinitialise**
   ```bash
   sudo kextcache -i /
   sudo reboot
   ```

5. **Vérifie que l'audio fonctionne**
   ```bash
   # Après reboot, test :
   afplay /System/Library/Sounds/Morse.aiff
   # Doit émettre un son via les haut-parleurs
   ```

**Pourquoi cette procédure ?** AppleHDA a été supprimé de Tahoe au démarrage, mais il existe toujours dans le KDK. AppleALC repose sur AppleHDA pour fonctionner, d'où la restauration post-install.

#### **Étape 7 : Autres Post-Installations**
```bash
# WiFi: Installe HeliPort (interface WiFi)
# Télécharge depuis : https://github.com/OpenIntelWireless/HeliPort/releases
# Ouvre l'app, paramètre le WiFi normalement

# Bluetooth (optionnel) : Si tu veux Bluetooth
# Achète un dongle USB Bluetooth (CSR8510 ou BCM20702)
# Branche-le, zéro configuration supplémentaire

# Trackpad: VoodooI2C déjà chargé via config.plist
# Test : System Preferences → Trackpad → Multitouch devrait fonctionner
```

---

## 📂 Structure EFI

```
EFI/
├── 📁 OC/                          ← Configuration OpenCore
│   ├── 📁 ACPI/                   ← Tables ACPI synthétiques
│   │   ├── SSDT-EC.aml           (Contrôleur EC fake)
│   │   ├── SSDT-PLUG.aml         (CPU Power Management)
│   │   ├── SSDT-PNLF.aml         (Backlight control) ⭐ _UID=0x13
│   │   ├── SSDT-RTCAWAC.aml      (RTC legacy)
│   │   ├── SSDT-XOSI.aml         (Trackpad compatibility)
│   │   └── SSDT-SBUS-MCHC.aml    (SMBus)
│   │
│   ├── 📁 Drivers/                ← Drivers UEFI
│   │   ├── HfsPlus.efi           (Lecture HFS+ UEFI)
│   │   ├── OpenRuntime.efi       (Runtime OpenCore)
│   │   └── Ext4Driver.efi        (Support ext4 optionnel)
│   │
│   ├── 📁 Kexts/                  ← Kernel Extensions
│   │   ├── Lilu.kext             (Framework)
│   │   ├── WhateverGreen.kext     (GPU patching)
│   │   ├── AppleALC.kext         (Audio — Tahoe avec KDK post-install)
│   │   ├── itlwm.kext            (WiFi Intel)
│   │   ├── VoodooI2C.kext        (Trackpad I2C)
│   │   ├── VoodooI2CHID.kext     (HID Trackpad)
│   │   ├── SMCBatteryManager.kext (Batterie)
│   │   ├── ECEnabler.kext        (EC 16-bit)
│   │   ├── NVMeFix.kext          (NVMe power)
│   │   ├── RestrictEvents.kext   (OTA updates)
│   │   ├── RealtekRTL8111.kext   (Ethernet)
│   │   └── BrightnessKeys.kext   (Brightness Fn-keys)
│   │
│   ├── config.plist               ← Configuration principale ⭐
│   └── OpenCore.efi               ← Bootloader
│
└── 📁 BOOT/
    └── BOOTx64.efi                ← Chargeur UEFI
```

### 🔧 Fichiers Critiques

| Fichier | Rôle | Notes |
|---------|------|-------|
| `config.plist` | Configuration centrale | ⚠️ Personnaliser ROM iServices |
| `SSDT-PNLF.aml` | Luminosité | ⭐ **_UID DOIT être 0x13** |
| `SSDT-XOSI.aml` | Trackpad I2C | Obligatoire pour DELL08BD |
| `WhateverGreen.kext` | Framebuffer iGPU | Patchs DVMT 32 MB |
| `VoodooI2C.kext` | Bus I2C | Avec VoodooI2CHID + dépendances |
| `AppleALC.kext` | Audio | Nécessite KDK + AppleHDA.kext (post-install) |

---

## ⚙️ Configuration Clés

### 🖥️ Framebuffer & iGPU

```xml
<!-- Dans config.plist → DeviceProperties → PciRoot(0x0)/Pci(0x2,0x0) -->

<key>AAPL,ig-platform-id</key>
<data>AACbPg==</data>              <!-- 0x3E9B0000 Coffee/Whiskey Lake -->

<key>device-id</key>
<data>mz4AAA==</data>              <!-- Spoof vers 0x3E9B (support Tahoe) -->

<key>framebuffer-fbmem</key>
<data>AACQAA==</data>              <!-- 4 MB pour VRAM allocation -->

<key>framebuffer-stolenmem</key>
<data>AAAwAQ==</data>              <!-- ~32 MB DVMT (verrouillé, obligatoire) -->

<key>framebuffer-patch-enable</key>
<data>AQAAAA==</data>              <!-- Activer les patchs -->
```

### 🔊 Audio & AppleALC

```xml
<!-- Boot-args dans NVRAM → boot-args -->
<string>alcid=19 -lilubeta -wegbeta</string>
<!-- ✅ alcid=19 pour ALC3271 Latitude 3500 -->
<!-- ✅ -lilubeta, -wegbeta pour stabilité Tahoe -->

<!-- SIP partiellement désactivé pour AppleALC -->
<key>csr-active-config</key>
<data>AwgAAA==</data>              <!-- 0x08030003 -->
```

### 🖱️ Trackpad I2C

```xml
<!-- SSDT-XOSI applique une rename patch:
     _OSI → XOSI (dans DSDT)
     Ce patch permet au trackpad DELL08BD de se charger -->

<!-- Dépendances chargées en ordre:
     1. VoodooI2CServices.kext
     2. VoodooGPIO.kext
     3. VoodooInput.kext
     4. VoodooI2C.kext
     5. VoodooI2CHID.kext
-->
```

### 💾 NVMe Secteurs

```bash
⚠️ IMPORTANT: Kioxia KXG60ZNV512G DOIT rester en 512e

# NE PAS FAIRE:
nvme format /dev/nvme0n1 -l 1    # ← Causes boucle infinie APFS

# Vérifier le format actuel:
nvme id-ns -H /dev/nvme0n1 | grep "LBA Format"
# Doit montrer: 512 bytes (default)
```

### 🔋 Luminosité Persistante

```xml
<!-- Dans NVRAM → Add → 4D1EDE05-38C7-4A6A-9CC6-4BCCA8B38C14 -->
<key>backlight-level</key>
<data>//8=</data>                 <!-- Sauvegarde à chaque shutdown -->
```

---

## 📋 Dépannage Courant

### 🔴 Problème : Luminosité plafonnée à ~5%

**Cause :** `SSDT-PNLF.aml` a `_UID = 0x10` au lieu de `0x13`

**Diagnostic :**
```bash
ioreg -l -p IODeviceTree | grep -i backlight
# Chercher "_UID" = 16 (incorrect) vs 19 (correct)
```

**Solution :**
```bash
# 1. Éditer SSDT-PNLF.dsl
# 2. Remplacer: Name (_UID, 0x10) par Name (_UID, 0x13)
# 3. Recompiler: iasl -ea SSDT-PNLF.dsl
# 4. Remplacer SSDT-PNLF.aml dans EFI
# 5. Reboot
```

---

### 🔊 Problème : Audio absent / muet

**Causes courantes :**

| Cause | Diagnostic | Remède |
|-------|-----------|--------|
| AppleHDA.kext non restauré | `ioreg \| grep AppleHDA` = rien | Relancer KDK + MyKextInstaller |
| alcid=19 incorrect | Vérifier boot-args dans config.plist | Changer à `alcid=19` |
| KDK non installé | `/Library/Developer/KDKs` vide | Télécharger + installer KDK officiel |
| SIP mal configurée | `csrutil status` | `csr-active-config` = `0x08030003` requis |

**Diagnostic complet :**
```bash
# Vérifie que AppleALC charge
kextstat | grep AppleALC

# Vérifie qu'AppleHDA existe
sudo find /System/Library/Extensions -name "AppleHDA*" 2>/dev/null

# Teste le son
afplay /System/Library/Sounds/Morse.aiff
```

---

### 🔵 Problème : Pas de Bluetooth

**Cause :** Regression Tahoe, chipsets Intel non supportés (confirmé AC9560 + AX200)

**Diagnostic :**
```bash
system_profiler SPBluetoothDataType
# Résultat: "No Bluetooth devices found"
```

**Solution Recommandée : Dongle USB Bluetooth** ✅
```bash
# Acheter: CSR8510 ou BCM20702 (~25 € Amazon/AliExpress)
# Brancher sur port USB interne ou externe
# Zéro configuration, reconnu nativement
# Pair tes périphériques normalement
```

**Note :** Le dongle USB ne supportera PAS les features Continuity (AirDrop, Handoff, etc.)

---

### 📡 Problème : WiFi absent / très lent

**Diagnostic :**
```bash
ifconfig | grep -i wifi
networksetup -listallnetworkservices

# Ou accéder à https://speedtest.net
```

**Solutions :**

| Symptôme | Cause | Remède |
|----------|-------|--------|
| Aucun réseau | `itlwm.kext` désactivé | Vérifie config.plist (kext `Enabled=true`) |
| WiFi visible mais pas de connexion | Firmware obsolète | Télécharge itlwm build Tahoe |
| AX200 non reconnue | Slot M.2 mauvais câblage | Vérifie câblage = PCIe (pas CNVi) |
| Menu WiFi absent | Utilises itlwm sans HeliPort | Installe HeliPort.app (OpenIntelWireless) |

---

### 💤 Problème : Veille / Réveils intempestifs

**Diagnostic :**
```bash
pmset -g log | tail -50 | grep -E "Wake from|DarkWake"
```

**Causes courantes & remèdes :**

| Cause | Remède |
|-------|--------|
| `Wake on LAN` | Désactiver dans BIOS (§ Réglages BIOS) |
| Périphérique USB | Vérifier cartographie USB (tool USBToolBox) |
| Clavier/Souris Bluetooth | Désactiver « Wake on keyboard » si BT dongle |

---

## 📚 Guide Complet

Pour une documentation **détaillée et complète**, consulte:

- 📖 **[Hackintosh-Latitude-3500-Tahoe.md](./docs/Hackintosh-Latitude-3500-Tahoe.md)** — Guide A→Z (13 phases)
- 🔧 **[Configuration BIOS Détaillée](./docs/BIOS-Configuration.md)** — Tous les réglages
- 🛠️ **[Post-Installation](./docs/Post-Install.md)** — Audio, WiFi, trackpad fine-tuning
- 🐛 **[Dépannage Avancé](./docs/Troubleshooting.md)** — Panneaux noirs, kernel panics...

---

## ✨ Fonctionnalités Testées

<div align="center">

| Catégorie | Fonctionnalité | État | Détail |
|-----------|----------------|------|--------|
| **Système** | Boot OpenCore | ✅ | ~7 secondes, splash screen macOS |
| | Installation complète | ✅ | APFS volume, récupération |
| | Updates OTA | ✅ | Tahoe 26.0 → 26.1 sans cassure |
| **GPU** | Metal 3 | ✅ | `metal --version` → Metal 3 |
| | QuickSync | ✅ | H.264, H.265, VP9 natif |
| | Dual Display | ⚠️ | eDP + HDMI possible mais nécessite test |
| **Audio** | Sortie speakers | ✅ | AppleALC + KDK, niveau ajustable |
| | Combo jack 3.5 mm | ✅ | Casque + micro détectés |
| | Niveaux mic | ⚠️ | Parfois faible, gain manuel dans System Preferences |
| **Trackpad** | Multitouch | ✅ | 2-finger scroll, 3-finger swipe |
| | Force Touch | ❌ | Pas de capteur |
| **WiFi** | Connexion | ✅ | AX200, ~150 Mbps |
| | Menu natif | ⚠️ | Via HeliPort (pas menu système natif) |
| | Roaming | ✅ | Bascule AP sans déconnexion |
| **Ethernet** | Détection | ✅ | RJ45 reconnu |
| | Vitesse | ✅ | Gigabit full-duplex |
| **Batterie** | Affichage % | ✅ | Précis, ±5% |
| | Durée | ✅ | ~6-8h usage light (dépend CPU load) |
| **Veille** | Fermeture capot | ✅ | Mise en veille automatique |
| | Réveil touchpad | ✅ | Fonctionnel |
| | Réveil clavier | ⚠️ | Via USB dongle BT si utilisé |
| **USB** | Vitesse USB 3.x | ✅ | Ports externes OK |
| | USB interne | ✅ | Cartographie complète |
| **HDMI** | Vidéo | ✅ | 1920×1080 @ 60 Hz stable |
| | Audio HDMI | ✅ | TV speakers / casque |
| **Caméra** | Détection | ✅ | Facetime, Photo Booth |
| **Lecteur CD** | Si présent | ⚠️ | Rarement testé |

</div>

---

## 🔐 Sécurité & Vie Privée

- ✅ **UEFI SecureBoot** : Désactivé (nécessaire pour OpenCore)
- ⚠️ **System Integrity Protection (SIP)** : Partiellement désactivé (0x08030003)
  - *Raison :* Requis pour AppleALC (framework audio)
  - *Conséquence :* Apple TV+ / Netflix en 1080p max (pas 4K)
- ✅ **FileVault** : Possible mais non recommandé (SIP partiellement abaissé)
- ✅ **iCloud / iMessage** : Fonctionnels (SMB correct, ROM Ethernet)
- ✅ **Pas de telemetry** : Hackintosh hors écosystème Apple

---

## 📸 Galerie & Ressources

### Screenshots système
```
macOS Tahoe 26.0
OpenCore 1.0.7
OCLP (OpenCore Legacy Patcher) compatible
```

### Ressources externes

| Ressource | Lien |
|-----------|------|
| 🍎 OpenCore | [github.com/acidanthera/OpenCorePkg](https://github.com/acidanthera/OpenCorePkg) |
| 📖 Dortania Guide | [dortania.github.io](https://dortania.github.io) |
| 🌐 OpenIntelWireless | [github.com/OpenIntelWireless](https://github.com/OpenIntelWireless) |
| 🔧 VoodooI2C | [github.com/VoodooI2C/VoodooI2C](https://github.com/VoodooI2C/VoodooI2C) |
| 🐦 r/hackintosh | [reddit.com/r/hackintosh](https://reddit.com/r/hackintosh) |

---

## 🚨 AVERTISSEMENTS

> ⚠️ **Cet EFI est spécifique à cette configuration matérielle.** Chaque machine a des variantes (ROM Ethernet, Bus ID HDMI, etc.) qui **doivent être adaptées**.

> 🔐 **Génère toujours tes propres identifiants SMBIOS** avec GenSMBIOS. Ne réutilise jamais ceux d'un tiers (risque d'ivraie ou de conflit iCloud).

> 💾 **Format NVMe CRITIQUE :** Le KXG60ZNV512G DOIT rester en **512e natif**. Ne le reformate PAS en 4Kn (boucle infinie APFS).

> 🔊 **Audio post-install obligatoire :** AppleALC seul ne fonctionne pas. KDK + MyKextInstaller + AppleHDA.kext restauré REQUIS.

---

## 📝 Corrections Récentes (v2.1)

✅ **Audio documentée complètement** (AppleALC + KDK post-install)
- Procédure complète avec étapes
- Pourquoi cette approche (AppleHDA supprimé Tahoe, mais disponible KDK)

✅ **SSDT-PNLF : _UID corrigé (0x10 → 0x13)**
- Luminosité maintenant exploit plage 0-100%

✅ **Backlight-level NVRAM activée**
- Persistance luminosité entre redémarrages

✅ **WiFi 6 AX200 validée**
- Remplacement AC9560 CNVi, itlwm compatible

✅ **Documentation Bluetooth clarifée**
- État non-fonctionnel confirmé → Dongle USB recommandé

---

## 🤝 Contribuer

Les pull requests sont bienvenues !

1. 🍴 Forke le repo
2. 📝 Crée une branche (`git checkout -b feature/amelioration`)
3. ✏️ Commit tes changements (`git commit -am 'Ajoute détail'`)
4. 🔀 Push vers la branche (`git push origin feature/amelioration`)
5. 🔔 Ouvre une Pull Request

### Types de contributions

- 🐛 Rapports de bugs & dépannage
- 📚 Améliorations documentation
- 🔧 Optimisations config.plist
- 🎯 Screenshots / GIFs de fonctionnalités
- 💡 Nouvelles configurations testées

---

## 📜 Licence

Ce projet est distribué sous licence **MIT** (voir [LICENSE](LICENSE)).

**Les composants externes conservent leurs licences respectives :**
- OpenCore, OpenRuntime : BSD (Acidanthera)
- Kexts (Lilu, WhateverGreen, AppleALC, etc.) : Voir chaque dépôt
- macOS Tahoe : © Apple Inc.

---

## 👤 Auteur

**Configuration créée et testée par :** [@lepekinoi](https://github.com/lepekinoi)

**Dernière vérification :** Août 2026

**Tested avec :** macOS Tahoe 26.0.1, OpenCore 1.0.7+, BIOS Dell 1.36.0

---

<div align="center">

### 💬 Questions ?

Ouvre une [**GitHub Issue**](https://github.com/lepekinoi/Hackintosh-DELL-Latitude-3500/issues) ou consulte le [**r/hackintosh subreddit**](https://reddit.com/r/hackintosh).

### ⭐ Aime ce projet ?

N'oublie pas le **star** ! ⭐ Ça aide la communauté à le découvrir.

---

**Fait avec ❤️ pour les fans de macOS sur Intel**

![Visitor Badge](https://visitor-badge.laobi.icu/badge?page_id=lepekinoi.Hackintosh-DELL-Latitude-3500)

</div>+
