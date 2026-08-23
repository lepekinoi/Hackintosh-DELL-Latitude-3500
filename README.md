Dell Latitude 3500 — Hackintosh macOS 26 Tahoe

EFI OpenCore complet et documenté pour faire tourner macOS Tahoe sur un Dell Latitude 3500 (i5-8365U, Whiskey Lake-U). Ce dépôt réunit une configuration fonctionnelle, les SSDT nécessaires, et le détail de chaque problème rencontré — avec sa cause réelle, pas seulement le correctif.

Matériel cible
Composant	Modèle
CPU	Intel Core i5-8365U (Whiskey Lake-U)
iGPU	Intel UHD 620
RAM	16 Go DDR4 2400
Stockage	Kioxia/Toshiba KXG60ZNV512G (NVMe, 512e obligatoire)
Wi-Fi/BT	Intel AC 9560 (CNVi) ou AX200 (PCIe)
Audio	Realtek ALC236 (référencé ALC3271/ALC3204 par Dell)
BIOS	1.36.0
État du projet

✅ Fonctionnel

Démarrage, accélération graphique complète (Metal 3)
Trackpad I2C (DELL08BD) via VoodooI2C
Wi-Fi (itlwm + HeliPort)
Audio interne (AppleALC + AppleHDA réinjecté, layout-id 19)
Luminosité réglable sur toute la plage (curseur + touches Fn)
Sortie HDMI (patch de connecteur nécessaire, voir ci-dessous)
Veille, batterie, USB cartographié

⚠️ Non fonctionnel sous Tahoe

Bluetooth Intel — IntelBluetoothFirmware.kext ne finalise jamais son attachement au périphérique USB (!registered, !matched), reproductible sur deux cartes Intel différentes (AC9560 et AX200). Cause probable : régression de la pile Bluetooth de Tahoe, pas un défaut de configuration. Solution : dongle Bluetooth USB (chipset CSR8510 ou BCM20702, reconnu nativement).

❌ Ne fonctionnera jamais

Port VGA/RGB — aucun encodeur analogique dans les pilotes iGPU Intel depuis plusieurs générations de macOS
Continuité Apple complète (AirDrop, Handoff) — carte Wi-Fi CNVi, pas de puce Broadcom
Points techniques notables

Ce projet documente plusieurs découvertes qui ne sont pas dans les guides génériques :

Le formatage NVMe en 4K natif casse l'installation sur ce modèle précis (boucle infinie de vérification de signature APFS) — rester en 512e
SSDT-PNLF doit utiliser _UID = 0x13, pas la valeur 0x10 générée par défaut par SSDTTime pour cette plateforme — sinon la luminosité plafonne à quelques pourcents
AppleHDA.kext a été retiré des KDK Tahoe à partir de la beta 2 ; il faut l'extraire d'un KDK beta 1 ou d'une installation Sequoia, puis choisir impérativement « Restart Now » (pas « Later ») dans l'outil de réinjection
backlight-level doit être placé dans le GUID NVRAM 4D1EDE05-... (variables Apple), pas dans celui des boot-args — sinon la valeur est silencieusement ignorée
Le patch de connecteur HDMI (framebuffer-con1-alldata) nécessite un Bus ID propre à chaque carte mère, déterminé par itération
Structure du dépôt
EFI/
├── OC/
│   ├── ACPI/          SSDT-EC, PLUG, RTCAWAC, XOSI, PNLF, SBUS-MCHC
│   ├── Kexts/
│   ├── Drivers/
│   └── config.plist
docs/
├── Hackintosh-Latitude-3500-Tahoe.md      guide complet, phase par phase
└── LiveUSB-Linux-Formatage-4K.md          reformatage NVMe en 512e
Avertissement

Ce dépôt reflète une configuration matérielle précise (numéro de série, ROM, UUID retirés du config.plist fourni). Génère tes propres identifiants SMBIOS avec GenSMBIOS avant utilisation — ne réutilise jamais ceux d'un tiers.

Certaines valeurs (Bus ID du connecteur HDMI, layout-id audio) sont spécifiques à cette carte mère et peuvent nécessiter un ajustement sur un autre exemplaire du même modèle.

Licence

À définir par l'auteur — les kexts et OpenCore restent sous leurs licences respectives (BSD/GPL selon le composant, voir dépôts Acidanthera/OpenIntelWireless/VoodooI2C).
