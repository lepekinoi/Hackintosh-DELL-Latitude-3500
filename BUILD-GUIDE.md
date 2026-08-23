# Hackintosh Dell Latitude 3500 — macOS Tahoe 26
## Guide complet A → Z (OpenCore 1.0.7+)

**Cible matérielle — confirmée par le BIOS 1.36.0, le DSDT et HWiNFO**

| Composant | Modèle | Source |
|---|---|---|
| Modèle | Dell Latitude 3500, fabriqué 09/2019 | Overview |
| BIOS | **1.36.0** (18/12/2024) — dernière version publiée | cctk |
| CPU | **Intel Core i5-8365U** @ 1.60 GHz, 4C/8T, 15 W | Overview |
| CPUID | **`806EC`** → Whiskey Lake-U confirmé | Overview |
| Cache | L2 1024 kB / L3 6144 kB · microcode F0 | Overview |
| iGPU | Intel UHD Graphics 620, VBIOS 9.0.1085 | Devices |
| **Mémoire vidéo** | **32 Mo** 🔴 | Devices |
| GPU dédié | **Aucun** ✅ | Devices |
| RAM | 16 Go DDR4 2400 MHz, 2×8 Go, **dual channel** | Memory |
| **SSD** | **Kioxia/Toshiba KXG60ZNV512G 512 Go NVMe Gen3** ✅ | System Config |
| Baie SATA 2,5" | **Vide et désactivée** (SATA-0 = OFF) — config mono-disque | System Config |
| Wi-Fi | **Intel AX200NGW (PCIe)** — remplace l'AC9560 CNVi d'origine ✅ | §12.1 |
| Bluetooth | Intel `0x8087:0x0029` — ❌ non fonctionnel sous Tahoe (§12.4) | — |
| Audio | **Realtek ALC3204** (confirmé — Dell l'appelle ALC3271) | Live Linux, `codec#0` |
| Écran | 15,6" FHD **1920×1080** | Devices |
| MAC Ethernet (LOM) | **`E4-54-E8-3F-4A-73`** | Devices |
| MAC Pass-Through | `E4-54-E8-3F-4A-74` (à désactiver) | Devices |

### Ce que ces relevés changent par rapport à la version initiale du guide

| Point | Résolution |
|---|---|
| CPU | i5-**8365U** (variante vPro), pas 8265U. Même architecture, même config OpenCore |
| GPU Nvidia MX130 | ❌ **Absent** → tout ce qui concernait `SSDT-dGPU-Off` et `-wegnoegpu` est supprimé |
| DVMT | **32 Mo confirmé par le BIOS** → les patchs framebuffer ne sont plus une hypothèse, ils sont obligatoires |
| Codec audio | ✅ **ALC3204 confirmé** (voir §11.3) |
| Écran | FHD 1920×1080 confirmé |
| Adresse ROM iServices | Déjà connue : `E454E83F4A73` — plus besoin de la chercher après installation |
| **SSD** | ✅ KXG60ZNV512G en place — le bon choix, SN850X écarté (§0.5) |
| Trackpad | ❓ **Seule inconnue restante** — voir §1.1 |

---

# SOMMAIRE

0. Ce qui ne fonctionnera pas + choix du stockage
1. Phase 0 — Inventaire matériel *(quasi résolu)*
2. Phase 1 — Réglages BIOS Dell *(avec tableau de priorités)*
3. Phase 2 — Outils et fichiers à télécharger
4. Phase 3 — Clé USB d'installation — **A.** depuis Windows · **B.** depuis macOS
5. Phase 4 — Génération des SSDT (ACPI)
6. Phase 5 — Arborescence EFI complète
7. Phase 6 — config.plist section par section
8. Phase 7 — Premier boot et installation
9. Phase 8 — Post-installation
10. Phase 9 — Cartographie USB
11. Phase 10 — Audio sous Tahoe (AppleALC + AppleHDA réinjecté, ou VoodooHDA)
12. Phase 11 — Wi-Fi et Bluetooth Intel
13. Phase 12 — Batterie, luminosité, veille
14. Phase 13 — iServices, OTA, finitions
15. Dépannage — pannes classiques et leurs causes
16. Checklist finale
17. Annexe A — Rapport matériel depuis Windows
18. **Annexe B — Analyse des relevés fournis** *(trackpad, EC, BIOS, exports restants)*

---

# 0. CE QUI NE FONCTIONNERA PAS

À lire avant d'investir un week-end. Rien de tout cela n'est contournable sur ce matériel.

**Définitivement impossible**
- **Continuité complète** : AirDrop, Handoff, Sidecar, Presse-papiers universel, Déverrouillage par Apple Watch, Appels iPhone, Recopie iPhone. Les cartes Wi-Fi Intel ne sont pas reconnues nativement par macOS, et les cartes Broadcom qui le seraient ont leur Wi-Fi cassé sous Tahoe.
- **Touch ID / Apple Pay** : pas de puce T2.
- **Mises à jour majeures** : macOS 26 Tahoe est le dernier système Apple supportant x86. macOS 27 est exclusivement Apple Silicon. Ta machine est en fin de ligne dès le premier boot.
- **FileVault + Secure Boot complet** : possible mais incompatible avec SIP abaissé (voir §11).

**Dégradé mais utilisable**
- **Audio analogique** : `AppleHDA.kext` retiré par Apple à partir de Tahoe beta 2. Solution retenue et confirmée sur cette machine : `AppleALC` + réinjection d'`AppleHDA` (extrait d'un KDK beta 1 ou d'un Sequoia) via MyKextInstaller — layout-id `19`. VoodooHDA reste une solution de repli plus simple mais de moindre qualité. Nécessite SIP partiellement désactivé dans les deux cas.
- **Bluetooth** : fonctionne (souris, clavier, casque A2DP) via les kexts OpenIntelWireless, mais sans les fonctions de continuité.
- **Wi-Fi** : fonctionne via itlwm + HeliPort, sans menu Wi-Fi natif dans macOS. `AirportItlwm`, qui offrirait ce menu, est cassé sous Tahoe (§12.2).
- **DRM** : Netflix/Apple TV+ en 1080p navigateur seulement, pas de 4K.

**Fonctionne bien**
- Accélération graphique complète, QuickSync (encodage/décodage vidéo matériel), Metal.
- Gestion d'alimentation CPU native (SpeedStep, C-states, Turbo).
- Ethernet, USB, webcam, lecteur SD, HDMI (vidéo + audio).
- Veille/réveil, batterie, luminosité, trackpad multitouch.
- Dual-boot Windows sans conflit.

## 0.5 — Stockage : le KXG60ZNV512G est le disque retenu

**Décision arrêtée : Kioxia/Toshiba KXG60ZNV512G 512 Go, disque unique, dédié à macOS.** Le WD_BLACK SN850X a été écarté volontairement.

Pourquoi le XG6 l'emporte sur cette machine :

| Critère | KXG60ZNV512G (Kioxia XG6) | WD_BLACK SN850X |
|---|---|---|
| Interface vs slot | PCIe **3.0** x4 → **100 % du slot exploité** | PCIe 4.0 dans un slot 3.0 → moitié du débit théorique perdue |
| Veille / réveil | Propres. `NVMeFix` en confort, pas en secours | APST/ASPM mal digérés par `IONVMeFamily` → paniques au sortir de veille |
| Consommation au repos | Faible → autonomie préservée | Élevée, très sensible sur un châssis 15 W |
| Secteurs | 512e — **ne jamais reformater en 4K** (§0.5 bis) | 512e par défaut |
| Réputation hackintosh | Famille XG utilisée par Apple chez des OEM | Fonctionne, mais demande de l'entretien |
| Capacité | 512 Go | 1 To |

Le seul compromis est la capacité. Pour un macOS Tahoe seul, sans dual-boot, c'est confortable : ~35 Go pour le système, le reste pour toi. La capacité n'aurait été un vrai sujet que pour un Windows + macOS sur le même disque.

➡️ **Le SN850X reste pertinent** — mais sur une machine PCIe Gen4, où il sera à sa place. Garde-le pour ça.

### Configuration physique de la machine

D'après la capture BIOS : le M.2 (`M.2 PCIe SSD-0/SATA-2`) est occupé par le XG6, et la baie 2,5" (`SATA-0`) est vide et désactivée. C'est exactement la configuration mono-disque souhaitée — rien à changer.

## 0.5 bis — 🔴 Formatage du disque : rester en 512e, ne PAS passer en 4K

**Correction majeure par rapport à une recommandation initiale.** Le XG6 supporte un format 4096 octets natif, et le gain théorique (moins d'amplification d'écriture, meilleur alignement avec APFS) est réel sur le papier. **Mais sur cette machine, en pratique, le 4Kn casse l'installation.**

**Ce qui a été observé** : disque reformaté en `nvme format -l 1` (4096 octets) → installation macOS qui semble avancer normalement (défilement verbeux propre, montage APFS réussi, `block size: 4096 block count: ...` confirmé) → puis boucle infinie de redémarrage. À chaque cycle : `com.apple.MobileSoftwareUpdate` monte le volume, tente une vérification, échoue avec `disallowing unsupported code signature page shift: 14` / `unable to create a validated cs_blob object: 85`, puis `revert_to_snapshot` annule et redémarre. Indéfiniment, sans jamais progresser.

**Cause probable** : la chaîne de vérification de signature de code utilisée par l'installateur x86 (`x86legacyap`) ne gère pas proprement un volume APFS en secteurs 4096 natifs à cette étape précise de l'installation. C'est un problème spécifique à l'installateur hackintosh sur x86, pas une limitation générale d'APFS.

**Le correctif qui a fonctionné** : reformater le disque en **512e** (le format standard) puis relancer l'installation depuis zéro.

```bash
# Lister les formats LBA disponibles
sudo nvme id-ns -H /dev/nvme0n1 | grep "LBA Format"

# Repérer la ligne à 512 octets (généralement index 0) et l'appliquer
sudo nvme format /dev/nvme0n1 -l 0

# Vérifier
sudo nvme id-ns -H /dev/nvme0n1 | grep "in use"
```

> 🔴 **Ne reformate PAS en 4096 octets sur cette plateforme.** Si le disque a déjà été formaté en 4K, il faut le reformater en 512e **avant** de relancer l'installation — un simple nouvel effacement APFS depuis l'Utilitaire de disque ne suffit pas, le format de secteur reste inchangé tant que `nvme format` n'est pas exécuté à nouveau.

> Le gain de performance du 4Kn était de toute façon marginal (quelques pourcents de latence en écriture). Ça ne valait pas le blocage que ça a provoqué.

### Kext associé

`NVMeFix.kext` reste dans la liste (§3.3). Sur le XG6 il n'est pas indispensable au démarrage, mais il améliore la gestion d'alimentation NVMe et ne coûte rien.

---

# 1. PHASE 0 — INVENTAIRE MATÉRIEL

✅ **Résolu.** Les relevés fournis (DSDT, dump BIOS cctk, rapport HWiNFO) ont tranché toutes les questions ouvertes. Analyse détaillée en **Annexe B**.

| Point | Statut | Preuve |
|---|---|---|
| **Trackpad** | ✅ **`DELL08BD`** sur I2C0, adresse `0x2C` | `trackpad.txt` + DSDT |
| GPU dédié Nvidia | ✅ **Absent** | Aucun périphérique PCI Nvidia dans HWiNFO |
| iGPU | ✅ `8086:3EA0` | HWiNFO PCI |
| DVMT | ✅ 32 Mo → patchs framebuffer obligatoires | BIOS Overview |
| Ethernet | ✅ `10EC:8168` rev 15 | HWiNFO PCI |
| Wi-Fi CNVi | ✅ `8086:9DF0` sub `42348086` | HWiNFO PCI |
| Audio | ✅ `8086:9DC8` (Cannon Lake-LP cAVS) | HWiNFO PCI |
| Contrôleur EC | ✅ nommé **`ECDV`** | DSDT ligne 57426 |
| Champs EC batterie | ✅ **tous en 8 bits** → ECEnabler inutile | DSDT `Field (ECOR…)` |
| AWAC + RTC | ✅ les deux présents → SSDT-AWAC requis | DSDT lignes 3644 / 18681 |
| GPI0 | ✅ présent, `INT3450` | DSDT ligne 16905 |
| Lecteur d'empreintes | ✅ Absent physiquement | Déclaré en ACPI mais aucun device USB |
| Capteur de luminosité | ✅ Absent physiquement | `ALSD` déclaré, rien côté matériel |
| Mode SATA | ✅ **déjà en AHCI** | `EmbSataRaid=Ahci` |

### 1.1 Il reste ces exports à produire

Voir **§B.6** pour la liste précise et les commandes.

### 1.2 BIOS — mis à jour en 1.36.0 ✅

Tu es passé de 1.26.0 à **1.36.0** (18/12/2024). J'avais affirmé à tort que la 1.26.0 était la dernière — elle ne l'était pas. Bien joué d'avoir vérifié.

**Le timing est bon** : un flash BIOS réinitialise les réglages et efface les entrées de boot NVRAM. L'avoir fait *avant* de construire l'EFI t'évite de devoir tout refaire ensuite.

Le DSDT a été réextrait sur cette version et **toutes les conclusions précédentes tiennent** : `ECDV`, `AWAC` + `RTC`, `GPI0`, `HPET`, `XHC`, champs EC en 8 bits. Aucune option DVMT ni CFG Lock n'est apparue.

### 1.3 Sauvegarde

Le XG6 va accueillir macOS, donc tout ce qu'il contient sera effacé.

- Sauvegarde le dossier de relevés **hors de la machine** — c'est ton bien le plus précieux à ce stade.
- Le SN850X que tu as retiré peut servir de disque de secours : y cloner ton Windows actuel te permet de revenir en arrière en échangeant les deux M.2.

---

# 2. PHASE 1 — RÉGLAGES BIOS DELL

## 2.0 Tableau de priorités — l'essentiel en un coup d'œil

Tous les réglages n'ont pas le même poids. Voici le tri, du bloquant au cosmétique.

### 🔴 Niveau 1 — Bloquant : sans ça, rien ne démarre

| Réglage | Valeur | Catégorie BIOS | Symptôme si mal réglé |
|---|---|---|---|
| **SATA/NVMe Operation** | **AHCI/NVMe** | System Configuration | Aucun disque visible dans l'Utilitaire de disque |
| **Secure Boot Enable** | **OFF** | Secure Boot | OpenCore refusé au démarrage |
| **Enable Legacy Option ROMs** | **OFF** | Boot Options | Blocage `gIO`, erreurs GPU |
| **Boot Mode / Boot List** | **UEFI** | Boot Options | macOS ne démarre pas du tout |
| **Enable USB Boot Support** | **ON** | System Configuration | Clé d'installation invisible |
| **Fastboot** | **Thorough** | POST Behavior | Ports USB non initialisés → `Still waiting for root device` |

### 🟠 Niveau 2 — Stabilité : ça démarre, mais ça se comporte mal

| Réglage | Valeur | Catégorie BIOS | Symptôme si mal réglé |
|---|---|---|---|
| **Intel SGX Enable** | **Disabled** | Intel SGX | Carte mémoire perturbée, slide KASLR instable |
| **Intel SpeedStep** | **Enabled** | Performance | CPU bloqué en fréquence, chauffe permanente |
| **C-States Control** | **Enabled** | Performance | Ventilateurs à fond, autonomie effondrée |
| **Intel TurboBoost** | **Enabled** | Performance | Performances divisées par deux |
| **Hyper-Thread Control** | **Enabled** | Performance | 4 threads au lieu de 8 |
| **Multi Core Support** | **All** | Performance | Cœurs désactivés |
| **Wake on LAN/WLAN** | **Disabled** | Power Management | Réveil immédiat après mise en veille |
| **USB Wake Support** | **OFF** | Power Management | Réveils intempestifs |
| **Computrace / Absolute** | **Disabled** | Security | Interférences ACPI bas niveau |
| **MAC Address Pass-Through** | **Disabled** | POST Behavior | Adresse MAC instable → iServices qui décrochent |
| **Virtualization (VT-x)** | **Enabled** | Virtualization | Machines virtuelles impossibles |
| **VT for Direct I/O (VT-d)** | Disabled si l'option existe | Virtualization | Sinon → `DisableIoMapper = YES` |
| **Hard Drive Free Fall Protection** | **OFF** | System Configuration | Accéléromètre inutile sur SSD, non géré |
| **UEFI Boot Path Security** | **Never** | Boot Options | Mot de passe demandé à chaque boot USB |

### ⚪ Niveau 3 — Neutre : macOS s'en moque, décide selon tes autres usages

| Réglage | Recommandation | Pourquoi c'est neutre |
|---|---|---|
| **TPM 2.0 Security** | **Laisse activé** | macOS ne le cherche pas. Requis par Windows 11 |
| **PTT / Intel Platform Trust** | **Laisse activé** | Idem — c'est le TPM firmware Intel |
| SMART Reporting | ON | Sans effet |
| Enable Camera | **ON** | Sinon pas de webcam sous macOS non plus |
| Enable Audio / Micro / Speaker | ON | |
| Enable SD Card | ON | |
| Fn Lock Options | Ton choix | Confort clavier |
| Full Screen Logo | OFF | Cosmétique |
| Battery Charge Configuration | Ton choix | Géré par le firmware, pas par macOS |
| SupportAssist OS Recovery | OFF | Utilitaire Dell, inutile ici |

> **La distinction Niveau 2 / Niveau 3 mérite d'être comprise.** Un réglage de niveau 2 est désactivé parce qu'il **interfère** avec macOS. Un réglage de niveau 3 est simplement ignoré : le désactiver ne t'apporte rien et peut te fermer des portes ailleurs. Beaucoup de tutoriels mélangent les deux et te font tout désactiver « par précaution » — c'est du bruit.

---

## 2.0 bis — État de TON BIOS (dump `bios-dump-ok.txt`, version 1.36.0)

### ✅ Tout ce que tu as appliqué depuis le dernier passage

| Clé cctk | Valeur | Commentaire |
|---|---|---|
| `VtForDirectIo` | **Disabled** | VT-d coupé — voir note ci-dessous sur `DisableIoMapper` |
| `UefiBootPathSecurity` | **Never** | Plus d'invite au boot USB |
| `WakeOnDock` | **Disabled** | |
| `EmbNic1` | **Enabled** | PXE retiré, démarrage plus rapide |
| `UefiNwStack` | **Disabled** | |
| `MediaCard` / `SdCard` | **Enabled** | Lecteur SD activé — il apparaît dans la carte USB (§10) |
| `BootOrder` | `+hdd.1,+hdd.2,-embnicipv4,-embnicipv6` | Boot réseau désactivé |

### ✅ Déjà bon depuis le départ

`EmbSataRaid=Ahci` · `SecureBoot=Disabled` · `Fastboot=Thorough` · `Speedstep`/`SpeedShift`/`CStatesCtrl`/`TurboMode`=`Enabled` · `CpuCore=CoresAll` · `LogicProc=Enabled` · `CpuXdSupport=Enabled` · `SoftGuardEn=Disabled` · `Absolute=Disabled` · `MacAddrPassThru=Disabled` · `HdFreeFallProtect=Disabled` · `WakeOnLan=Disabled` · `WakeOnAc=Disabled` · `BlockSleep=Disabled` · `Sata0=Disabled` / `Sata2=Enabled` · `Camera`/`IntegratedAudio`/`Microphone`=`Enabled`

### ✅ Régression corrigée

`UsbWake` était repassé à `Enabled` lors du flash 1.36.0. Le dump `bios-dump-ok.txt` confirme qu'il est de nouveau à **`Disabled`**. Plus rien à modifier côté BIOS.

### ⚠️ Un point à garder en tête pour le formatage 4K

`SedBlockSidAuthentication=Enabled` — cette fonction verrouille les commandes TCG Opal du disque. **Si `nvme format` échoue** avec une erreur d'accès, désactive-la temporairement :

```
cctk.exe --SedBlockSidAuthentication=Disabled
```

Ne la désactive pas préventivement : essaie d'abord, elle ne pose problème que sur certains firmwares SSD.

### 💡 Conséquence de `VtForDirectIo=Disabled`

VT-d est maintenant coupé au niveau du firmware, donc `DisableIoMapper` n'est théoriquement plus nécessaire.

**Garde-le quand même à `YES`.** Certains firmwares continuent d'exposer la table `DMAR` même VT-d désactivé, et le quirk ne coûte rien. Tu pourras le passer à `NO` plus tard, une fois le système stable, si tu veux vérifier — mais ce n'est pas un gain mesurable.

### ⚪ Réglages neutres relevés au passage

`TpmSecurity=Enabled` / `TpmActivation=Enabled` — neutres pour macOS, laisse tel quel (§2.4).
`PrimaryBattChargeCfg=Adaptive` — les valeurs `CustomChargeStart=50` / `CustomChargeStop=90` sont ignorées tant que le mode est `Adaptive`. Si tu veux plafonner la charge à 90 % pour préserver la batterie, passe en `PrimaryBattChargeCfg=Custom`.
`KeyboardIllumination=Dim`, `FanSpeed=Auto`, `LidSwitch=Enabled`, `FnLockMode=EnableSecondary` — confort, sans effet sur macOS.

> **Toujours aucune option DVMT ni CFG Lock** dans le dump 1.36.0. Les patchs framebuffer et `AppleXcpmCfgLock = YES` restent obligatoires. C'est définitif.

---

Accès : **F2** au démarrage. Ton BIOS 1.36.0 utilise la nouvelle interface Dell « BIOS Setup » — panneau de catégories à gauche, interrupteurs ON/OFF plutôt que des cases à cocher.

> 🔴 **Active d'abord le mode Advanced** — l'interrupteur en haut à droite de chaque page. Sur tes captures il est déjà activé. Sans lui, la moitié des options décrites ci-dessous reste invisible.

> ⚠️ **AVANT TOUT** : si BitLocker est actif sur le Toshiba, désactive-le (`Panneau de configuration → Chiffrement de lecteur BitLocker → Désactiver`) avant de toucher au mode SATA/NVMe. Sinon, écran de récupération au redémarrage.

**Catégories de ton BIOS** (vues sur la capture 3) : Overview · Boot Options · System Configuration · Video · Security · Passwords · Secure Boot · Expert Key Management · Performance · Power Management · Wireless · POST Behavior · Virtualization · Maintenance · System Logs · SupportAssist.

### 2.1 Boot Options

*(remplace l'ancienne catégorie « General » des BIOS Dell antérieurs)*

| Option | Valeur | Raison |
|---|---|---|
| Boot Mode / Boot List Option | **UEFI** ou "UEFI with Secure Boot OFF" | macOS ne démarre qu'en UEFI |
| **Enable Legacy Option ROMs** | **OFF** | Équivalent du CSM. Provoque des blocages `gIO` et des erreurs GPU |
| Enable Attempt Legacy Boot | **OFF** | |
| **Enable UEFI Boot Path Security** | **Never** ou "Always Except Internal HDD" | Évite une demande de mot de passe à chaque boot USB |
| Enable Secure Boot | **OFF** (voir aussi §2.5) | |
| Boot Sequence | Ton disque macOS en premier une fois installé | |

### 2.2 System Configuration

C'est la page de ta capture 3. **Fais défiler vers le haut** : l'option la plus importante se trouve juste au-dessus de l'interrupteur `SATA-0` que tu vois à l'écran.

| Option | Valeur | Raison |
|---|---|---|
| **SATA/NVMe Operation** | **AHCI/NVMe** (pas "RAID On") | 🔴 **LE réglage critique.** Voir ci-dessous |
| SATA-0 | Actuellement **OFF** — laisse ainsi | La baie 2,5" est vide |
| M.2 PCIe SSD-0/SATA-2 | **ON** | C'est ton disque système |
| Enable SMART Reporting | ON | Sans effet sur macOS |
| USB Configuration → Enable USB Boot Support | **ON** | 🔴 Indispensable pour la clé d'installation |
| USB Configuration → Enable External USB Port | **ON** | |
| Audio → Enable Audio / Microphone / Internal Speaker | **ON** | |
| Miscellaneous Devices → **Enable Camera** | **ON** ✅ *(déjà correct)* | |
| Miscellaneous Devices → **Enable Hard Drive Free Fall Protection** | **OFF** | Accéléromètre Dell inutile sur SSD, et non géré par macOS |
| Miscellaneous Devices → Enable Secure Digital (SD) Card | ON | |
| Miscellaneous Devices → Secure Digital Card Boot | OFF | |
| Miscellaneous Devices → Secure Digital Card Read-Only Mode | OFF | |

> 🔴 **Piège NVMe/RAID — s'applique aussi à ton disque M.2.** Beaucoup pensent que le mode "RAID On" ne concerne que le SATA : faux. Avec Intel RST activé, le contrôleur NVMe est présenté sous une forme propriétaire que macOS ne sait pas lire. **Résultat : aucun disque visible dans l'Utilitaire de disque.**
>
> Si Windows est installé sur le Toshiba en mode RAID On, la bascule vers AHCI/NVMe le rendra non-démarrable (`INACCESSIBLE_BOOT_DEVICE`). Conversion sans réinstallation :
> 1. Sous Windows, invite de commandes **administrateur** : `bcdedit /set {current} safeboot minimal`
> 2. Redémarrer → F2 → System Configuration → **SATA/NVMe Operation = AHCI/NVMe** → Apply Changes
> 3. Windows démarre en mode sans échec et charge le pilote NVMe standard
> 4. Invite de commandes admin : `bcdedit /deletevalue {current} safeboot`
> 5. Redémarrer normalement
>
> Si tu pars sur une installation macOS seule qui écrase le contenu du XG6, tu peux ignorer la procédure ci-dessus : bascule simplement en AHCI/NVMe et installe par-dessus.

> 🔴 **Piège SATA/RAID :** si Windows a été installé en mode "RAID On", basculer en AHCI le rendra non-démarrable (écran bleu `INACCESSIBLE_BOOT_DEVICE`). Procédure de conversion sans réinstallation :
> 1. Sous Windows, invite de commandes **administrateur** : `bcdedit /set {current} safeboot minimal`
> 2. Redémarrer → F2 → passer SATA Operation sur **AHCI** → sauvegarder
> 3. Windows démarre en mode sans échec, installe le pilote AHCI
> 4. Invite de commandes admin : `bcdedit /deletevalue {current} safeboot`
> 5. Redémarrer normalement

### 2.3 Video

Ta capture 1 le confirme noir sur blanc : **Video Memory = 32 MB**. Le BIOS Latitude n'expose aucun réglage DVMT Pre-Allocated pour le modifier, alors que macOS en attend 64 Mo.

➡️ **Ce n'est donc plus une hypothèse** : les patchs `framebuffer-patch-enable`, `framebuffer-stolenmem` et `framebuffer-fbmem` sont **obligatoires** dans le config.plist (§7.3). Sans eux, tu auras une panique noyau systématique sur `AppleIntelCFLGraphicsFramebuffer`.

Rien à modifier dans cette page du BIOS.

### 2.4 Security

| Option | Valeur | Raison |
|---|---|---|
| **Secure Boot → Secure Boot Enable** | **Disabled** | 🔴 Obligatoire, OpenCore n'est pas signé Microsoft |
| Expert Key Management | Non touché | |
| Intel SGX Enable | **Disabled** 🟠 | Voir §2.6 |
| **TPM 2.0 Security** | **Laisse activé** ⚪ | Voir encadré ci-dessous |
| **PTT Security / Intel Platform Trust** | **Laisse activé** ⚪ | Idem — c'est le TPM firmware Intel |
| **Computrace / Absolute** | **Disabled** | Module de traçage bas niveau, source de problèmes ACPI |
| CPU XD Support (Execute Disable) | **Enabled** | Requis par macOS |
| Admin/System Password | À ta guise | |
| Master Password Lockout | Décoché | |

> ### 🔐 TPM : activer ou désactiver ?
>
> **macOS ignore complètement le TPM.** Il ne le cherche pas, ne l'utilise pas, et ne sait même pas qu'il existe. VirtualSMC émule la puce SMC d'Apple, et aucune étape de la chaîne de démarrage macOS ne dépend d'un TPM. Le réglage est donc **strictement neutre** pour le hackintosh.
>
> Ce qui décide, c'est Windows :
>
> | Ton scénario | TPM 2.0 / PTT |
> |---|---|
> | macOS seul | Indifférent → **laisse activé** |
> | Dual-boot Windows 11 | **Enabled** — obligatoire pour W11 |
> | Windows jetable pour les relevés | **Ne touche à rien** |
>
> Beaucoup de guides recommandent de le désactiver « par précaution ». C'est un principe d'hygiène (« coupe ce qui ne sert pas »), pas une réponse à un bug documenté. Aucun cas connu de TPM cassant une installation Tahoe.
>
> **Deux pièges :**
> - Ne lance **jamais** `Clear TPM` si BitLocker est actif → destruction des clés de scellement, il ne te reste que ta clé de récupération à 48 chiffres.
> - Passer `TPM State` sur Disabled avec BitLocker actif déclenche une demande de clé au démarrage suivant. Suspends-le d'abord : `manage-bde -protectors -disable C:`
>
> **Ne confonds pas TPM et Secure Boot.** Secure Boot doit impérativement être désactivé (OpenCore n'est pas signé Microsoft). Le TPM, lui, peut rester allumé sans conséquence.

### 2.5 Secure Boot

| Option | Valeur |
|---|---|
| Secure Boot Enable | **Disabled** 🔴 |
| Secure Boot Mode | Deployed Mode (sans effet une fois désactivé) |

### 2.6 Intel Software Guard Extensions (SGX)

| Option | Valeur |
|---|---|
| **Intel SGX Enable** | **Disabled** 🟠 |
| Enclave Memory Size | 32 MB (sans effet une fois SGX désactivé) |

> ### 🧩 SGX : pourquoi désactiver, cette fois pour une vraie raison
>
> Contrairement au TPM, SGX n'est pas neutre. Quand il est activé, le firmware réserve une zone de mémoire physique dédiée aux enclaves — la **PRM** (Processor Reserved Memory). Cette zone est retirée de la carte mémoire présentée au système.
>
> Conséquence côté OpenCore : la carte mémoire devient plus fragmentée, ce qui réduit le nombre de valeurs de slide KASLR utilisables. Dans les cas défavorables, on tombe sur le message `OCABC: Only N/256 slide values are usable!` et sur des échecs de démarrage aléatoires — le genre de panne qui n'arrive qu'une fois sur trois et qu'on met des heures à diagnostiquer.
>
> **macOS n'utilise SGX pour rien.** Le désactiver ne coûte donc strictement rien du côté Apple.
>
> Le seul usage réel côté Windows : certains services de streaming s'en servaient historiquement pour la lecture 4K protégée (Netflix via PlayReady SL3000). Intel a depuis déprécié SGX sur ses processeurs grand public et ces services sont passés à autre chose. Ce n'est plus un argument.
>
> ➡️ **Disabled**, sans hésitation.

### 2.7 Performance

| Option | Valeur | Raison |
|---|---|---|
| Multi Core Support | **All** | |
| Intel SpeedStep | **Enabled** | 🔴 Requis pour la gestion d'alimentation native |
| C-States Control | **Enabled** | 🔴 Requis, sinon la machine chauffe en permanence |
| Intel TurboBoost | **Enabled** | |
| Hyper-Thread Control | **Enabled** | |

### 2.8 Power Management

| Option | Valeur | Raison |
|---|---|---|
| AC Behavior / Wake on AC | Décoché | |
| Auto On Time | Disabled | |
| USB Wake Support | **Décoché** | Évite les réveils intempestifs |
| Wireless Radio Control | Décoché | |
| **Wake on LAN/WLAN** | **Disabled** | 🔴 Cause classique de réveils immédiats après veille |
| Block Sleep | **Décoché** | |
| Peak Shift | Disabled | |
| Advanced Battery Charge Configuration | Disabled | |
| Primary Battery Charge Configuration | Adaptive ou Custom | Sans effet sur macOS |
| Type-C Connector / Thunderbolt (si présent) | **Disabled** pour l'installation | Réactivable après |

### 2.9 POST Behavior

| Option | Valeur | Raison |
|---|---|---|
| **Fastboot** | **Thorough** | 🔴 "Minimal" saute l'initialisation matérielle → USB non détecté |
| Extend BIOS POST Time | 0 s | |
| Fn Lock Options | Lock Mode Secondary (recommandé — touches multimédia directes) | Confort |
| Full Screen Logo | Décoché | |
| Warnings and Errors | Prompt on Warnings and Errors | |
| **MAC Address Pass-Through** | **Disabled** | 🔴 Important — voir ci-dessous |

> **Pourquoi désactiver le MAC Pass-Through :** ton BIOS expose deux adresses — la vraie carte Ethernet (`E4-54-E8-3F-4A-73`) et une adresse de substitution (`E4-54-E8-3F-4A-74`) utilisée par la fonction Pass-Through pour les stations d'accueil. Si cette fonction reste active, l'adresse vue par macOS peut changer selon que tu es sur dock ou non. Or l'adresse MAC sert de base au champ `ROM` qui valide tes iServices : une adresse instable = iMessage qui se déconnecte. Désactive, et macOS verra toujours `E4:54:E8:3F:4A:73`.

### 2.10 Virtualization Support

| Option | Valeur | Raison |
|---|---|---|
| Virtualization (VT-x) | **Enabled** | |
| **VT for Direct I/O (VT-d)** | **Disabled** si l'option existe | Sinon : `DisableIoMapper = YES` dans le config |
| Trusted Execution | Décoché | |

> Sur beaucoup de Latitude, VT-d n'est pas désactivable séparément. Ce n'est pas grave : `DisableIoMapper = YES` fait le travail proprement (§7.4).

### 2.11 Wireless

| Option | Valeur |
|---|---|
| Wireless Switch → WLAN / Bluetooth | Cochés |
| Wireless Device Enable → WLAN / Bluetooth | Cochés |

### 2.12 Maintenance / System Logs

Rien à modifier.

### 2.13 Options absentes sur ce BIOS

| Option manquante | Contournement |
|---|---|
| **CFG Lock (MSR 0xE2)** | `Kernel → Quirks → AppleXcpmCfgLock = YES` |
| **DVMT Pre-Allocated 64 Mo** | Patchs `framebuffer-stolenmem` + `framebuffer-fbmem` |
| **Above 4G Decoding** | Non nécessaire sur cette plateforme |
| **CSM explicite** | Remplacé par "Enable Legacy Option ROMs" (à décocher) |
| **XHCI Hand-off** | Remplacé par `UEFI → Quirks → ReleaseUsbOwnership = YES` |

---

# 3. PHASE 2 — OUTILS ET FICHIERS

Tout se prépare depuis Windows. Crée un dossier de travail, par exemple `C:\Hack\`.

### 3.1 Outils (dossier `C:\Hack\Tools\`)

| Outil | Source | Usage |
|---|---|---|
| **ProperTree** | `github.com/corpnewt/ProperTree` | Éditeur de plist. **Le seul autorisé** |
| **GenSMBIOS** | `github.com/corpnewt/GenSMBIOS` | Génération des numéros de série |
| **SSDTTime** | `github.com/corpnewt/SSDTTime` | Génération automatique des SSDT |
| **MountEFI** | `github.com/corpnewt/MountEFI` | Montage de la partition EFI |
| **gibMacOS** | `github.com/corpnewt/gibMacOS` | Téléchargement de macOS depuis Windows |
| **USBToolBox** | `github.com/USBToolBox/tool` | Cartographie USB depuis Windows |

> Ces outils sont en Python. Installe **Python 3.x** depuis python.org en cochant "Add Python to PATH". Les scripts `.bat` fournis feront le reste.

> ❌ **N'utilise jamais** OpenCore Configurator, OCC, Hackintool en mode édition de config, ou tout autre "configurateur" graphique. Ils injectent des clés Clover et corrompent le plist. ProperTree uniquement.

### 3.2 OpenCore

`github.com/acidanthera/OpenCorePkg/releases` → **OpenCore-1.0.7-RELEASE.zip** (ou plus récent).

Télécharge aussi la version **DEBUG** : tu en auras besoin au premier dépannage.

### 3.3 Kexts — liste complète pour cette machine

Télécharge la **dernière release** de chaque dépôt (pas les sources).

#### Fondations — obligatoires, dans cet ordre de chargement

| Kext | Dépôt | Rôle |
|---|---|---|
| `Lilu.kext` | acidanthera/Lilu | Socle de patch. **Toujours en premier** |
| `VirtualSMC.kext` | acidanthera/VirtualSMC | Émulation de la puce SMC Apple |
| `WhateverGreen.kext` | acidanthera/WhateverGreen | Correctifs graphiques iGPU |

#### Plugins VirtualSMC (dans `VirtualSMC.kext/Contents/Resources/` du zip)

| Kext | Rôle |
|---|---|
| `SMCProcessor.kext` | Température CPU |
| `SMCSuperIO.kext` | Vitesse des ventilateurs |
| `SMCBatteryManager.kext` | 🔴 Batterie — indispensable sur portable |
| `SMCLightSensor.kext` | ❌ **Ne pas ajouter** — `ALSD` est déclaré en ACPI mais aucun capteur physique n'existe sur ce modèle |
| `SMCDellSensors.kext` | Contrôle des ventilateurs Dell (optionnel, à ajouter après le premier boot réussi) |

> ⚠️ N'ajoute **jamais** `SMCLightSensor` ou `SMCDellSensors` en aveugle. Teste d'abord sans, ajoute ensuite.

#### Stockage

| Kext | Dépôt | Rôle |
|---|---|---|
| `NVMeFix.kext` | acidanthera/NVMeFix | Gestion d'alimentation NVMe. Requis pour le SN850X |

#### Réseau

| Kext | Dépôt | Rôle |
|---|---|---|
| `RealtekRTL8111.kext` | Mieze/RTL8111_driver | Ethernet Realtek |
| `itlwm.kext` | OpenIntelWireless/itlwm | Wi-Fi Intel — **seule option sous Tahoe**, voir §12.2. Ne PAS utiliser `AirportItlwm` |
| `IntelBluetoothFirmware.kext` | OpenIntelWireless/IntelBluetoothFirmware | Firmware BT Intel |
| ~~`IntelBTPatcher.kext`~~ | idem | 🔴 **NE PAS ACTIVER** — panique noyau + boucle de redémarrage (§12.4) |
| `BlueToolFixup.kext` | acidanthera/BrcmPatchRAM | 🔴 Obligatoire depuis Monterey |

#### Entrées (clavier / trackpad)

| Kext | Dépôt | Rôle |
|---|---|---|
| `VoodooPS2Controller.kext` | acidanthera/VoodooPS2 | Clavier PS/2 + touches Fn |
| `VoodooI2C.kext` | VoodooI2C/VoodooI2C | Bus I2C (**si trackpad I2C**) |
| `VoodooI2CHID.kext` | idem (satellite) | Trackpad HID I2C |

> 🔴 **VoodooI2C ne se charge pas seul — trois plugins internes sont obligatoires, et souvent oubliés.** Ils vivent dans `VoodooI2C.kext/Contents/PlugIns/` et **doivent être déclarés dans `Kernel → Add` avant `VoodooI2C.kext` lui-même** :
>
> | Plugin | Rôle |
> |---|---|
> | `VoodooI2CServices.kext` | Services de base requis par VoodooI2C |
> | `VoodooGPIO.kext` | 🔴 Gère l'interruption `GpioInt` du trackpad sur le contrôleur GPIO (`\_SB.PCI0.GPI0`, `INT3450` sur cette machine) |
> | `VoodooInput.kext` | Requis par `VoodooI2CHID` pour le multitouch |
>
> **Symptôme si ces plugins manquent** : au démarrage, le verbeux affiche `Dependency com.alexandred.VoodooI2CServices was not found for kext com.alexandred.VoodooI2C` et `Symbol __ZN10VoodooGPIO9metaClassE has 0-value`. VoodooI2C se charge quand même, mais sans son support GPIO — le trackpad reste inerte.
>
> **Le `VoodooInput` de `VoodooPS2Controller`** (voir plus bas) doit alors être **désactivé** — les deux paquets embarquent chacun leur copie, charger les deux crée un conflit de version. Garde uniquement celui de `VoodooI2C`.

> **Si trackpad I2C** : dans `VoodooPS2Controller.kext/Contents/PlugIns/`, **supprime** `VoodooInput.kext`... non — garde VoodooInput, mais **désactive** `VoodooPS2Mouse.kext` et `VoodooPS2Trackpad.kext` dans le config.plist (`Enabled = NO`). Deux pilotes de trackpad simultanés = conflit garanti.
>
> **Si trackpad PS/2** : n'ajoute ni VoodooI2C ni VoodooI2CHID, et garde `VoodooPS2Trackpad.kext` activé.

#### Divers

| Kext | Dépôt | Rôle |
|---|---|---|
| ~~`ECEnabler.kext`~~ | 1Revenger1/ECEnabler | ⚪ **Non nécessaire** — ton EC n'expose que des champs 8 bits (§5.3). Garde-le sous la main en filet |
| `RestrictEvents.kext` | acidanthera/RestrictEvents | 🔴 Nom du CPU + mises à jour OTA |
| `BrightnessKeys.kext` | acidanthera/BrightnessKeys | Touches de luminosité |
| `CPUFriend.kext` + `CPUFriendDataProvider.kext` | acidanthera/CPUFriend | Optionnel — affinage de la gestion d'alimentation, **après** que tout fonctionne |
| `HibernationFixup.kext` | acidanthera/HibernationFixup | Optionnel — seulement si problèmes de veille prolongée |
| `USBMap.kext` / `UTBMap.kext` | Généré par toi | §10 |

#### ❌ Kexts à NE PAS mettre

| Kext | Pourquoi |
|---|---|
| `AppleALC.kext` | ✅ **Requis** — voir §11.1. Nécessite AppleHDA réinjecté via MyKextInstaller/SimpleLoader |
| `USBInjectAll.kext` | Obsolète, remplacé par la cartographie USB |
| `VoodooHDA.kext` | ⚠️ Ne va **pas** dans l'EFI. Il s'installe dans `/Library/Extensions` (§11) |
| `NullEthernet`, `FakeSMC`, `AppleIntelE1000e` | Anciens/inutiles ici |
| `AirportBrcmFixup` | Aucune carte Broadcom |

### 3.4 Pilotes UEFI (`.efi`)

| Fichier | Provenance | Obligatoire |
|---|---|---|
| `OpenRuntime.efi` | OpenCorePkg/X64/EFI/OC/Drivers | 🔴 Oui |
| `HfsPlus.efi` | acidanthera/OcBinaryData → Drivers | 🔴 Oui |
| `ResetNvramEntry.efi` | OpenCorePkg | Recommandé (entrée "Reset NVRAM" au menu) |
| `OpenCanopy.efi` | OpenCorePkg | Optionnel (interface graphique) |
| `AudioDxe.efi` | OpenCorePkg | Optionnel (carillon de démarrage) |

> Supprime tous les autres `.efi` du dossier `Drivers`. Chaque driver inutile est une source de bug potentielle.

---

# 4. PHASE 3 — CLÉ USB D'INSTALLATION

Clé de **16 Go minimum**. Deux méthodes selon ce dont tu disposes : depuis Windows (§4.A), ou depuis un Mac / un hackintosh déjà fonctionnel (§4.B).

---

## 4.A — Depuis Windows

USB 2.0 de préférence (meilleure compatibilité au boot que l'USB 3.0 sur les firmwares Dell).

### 4.A.1 Téléchargement de macOS Tahoe

La méthode « image de récupération » est la plus fiable :

```
cd C:\Hack\Tools\gibMacOS
macrecovery.bat
```

Puis, selon le menu, sélectionne **macOS Tahoe (26)**. Le script télécharge `BaseSystem.dmg` et `BaseSystem.chunklist` (≈ 700 Mo). Le reste du système sera téléchargé pendant l'installation — prévois une connexion Ethernet, le Wi-Fi ne sera pas fonctionnel dans l'installateur.

> Si la commande directe est nécessaire :
> ```
> python macrecovery.py -b Mac-937A206F2EE63C01 -m 00000000000000000 download
> ```
> (board-id iMac20,1 — adapte selon le SMBIOS retenu)

### 4.A.2 Formatage de la clé

Via **diskpart** (invite de commandes admin) :

```
diskpart
list disk
select disk X          ← remplace X par le numéro de ta clé. VÉRIFIE DEUX FOIS.
clean
convert gpt
create partition primary
format fs=fat32 quick label="TAHOE"
assign
exit
```

> 🔴 `clean` efface tout le disque sélectionné. Une erreur de numéro et tu effaces ton SSD.

### 4.A.3 Copie des fichiers

Sur la clé, crée l'arborescence suivante :

```
TAHOE:\
├── com.apple.recovery.boot\
│   ├── BaseSystem.dmg
│   └── BaseSystem.chunklist
└── EFI\
    ├── BOOT\
    │   └── BOOTx64.efi
    └── OC\
        ├── ACPI\
        ├── Drivers\
        ├── Kexts\
        ├── Resources\
        ├── Tools\
        ├── OpenCore.efi
        └── config.plist
```

Prends `EFI\BOOT\BOOTx64.efi`, `EFI\OC\OpenCore.efi` et le dossier `Resources` dans `OpenCore-1.0.7-RELEASE\X64\EFI\`.

Vide entièrement `ACPI`, `Drivers`, `Kexts` et `Tools` — tu vas les remplir toi-même. Copie `Docs\Sample.plist` et renomme-le `config.plist`.

---

## 4.B — Depuis macOS

Utile si tu as accès à un vrai Mac, ou si tu disposes déjà d'un premier hackintosh fonctionnel (même une VM macOS) pour préparer la clé du Latitude. Cette méthode produit un installateur **complet** (12-15 Go) plutôt qu'une image de récupération minimale — elle télécharge tout d'un coup, donc pas besoin d'Ethernet pendant l'installation elle-même une fois la clé prête.

### 4.B.1 Obtenir l'application « Installer macOS Tahoe »

**Sur un Mac déjà en Tahoe ou capable de le proposer** (App Store) :

`Réglages Système → Général → Mise à jour de logiciels` → si Tahoe est proposé en mise à niveau, télécharge-le sans lancer l'installation. L'app `Installer macOS Tahoe.app` se dépose dans `/Applications`.

**Méthode fiable, en ligne de commande** — fonctionne même si l'App Store ne propose pas la mise à niveau directement :

```bash
softwareupdate --list-full-installers
```

Repère la ligne correspondant à Tahoe (26.x), puis :

```bash
softwareupdate --fetch-full-installer --full-installer-version 26.0
```

Adapte le numéro de version à celui affiché par la commande précédente. Le téléchargement (12-15 Go) se fait directement dans `/Applications/Install macOS Tahoe.app`.

> Cette commande fonctionne aussi bien sur un vrai Mac que sur un hackintosh déjà installé — c'est le moyen le plus direct d'obtenir l'installateur complet sans dépendre de l'App Store.

### 4.B.2 Préparer la clé USB

Branche la clé (16 Go minimum), puis identifie-la :

```bash
diskutil list external
```

Note son identifiant (`disk4`, par exemple — **pas** `disk4s1`, le disque entier).

Efface-la en Mac OS étendu, table GUID :

```bash
diskutil eraseDisk JHFS+ TAHOE GPT /dev/disk4
```

> 🔴 Vérifie deux fois le numéro de disque. `eraseDisk` efface intégralement le support ciblé, sans confirmation supplémentaire.

### 4.B.3 Créer le support d'installation

L'utilitaire intégré à l'app fait tout le travail — partitionnement, copie du système, rendu bootable :

```bash
sudo "/Applications/Install macOS Tahoe.app/Contents/Resources/createinstallmedia" --volume /Volumes/TAHOE
```

Confirme par `y` puis ton mot de passe. Compte 15 à 25 minutes. La clé est reformatée au passage et renommée **`Install macOS Tahoe`** — normal, ignore l'avertissement de disque illisible si le Finder le montre entre-temps.

### 4.B.4 Ajouter l'EFI OpenCore

`createinstallmedia` produit une clé bootable en soi, mais sans OpenCore elle démarrera l'installateur Apple pur, inutilisable sur ce matériel. Il faut y ajouter la partition EFI.

```bash
diskutil list disk4
```

Repère la partition EFI (`disk4s1`, généralement 200 Mo, type `EFI`) :

```bash
sudo diskutil mount disk4s1
```

Elle se monte sous `/Volumes/EFI`. Copie ton dossier `EFI` complet dessus — celui que tu as construit avec OpenCore, tes kexts, tes SSDT et ton `config.plist` :

```bash
sudo cp -R ~/Hack/EFI /Volumes/EFI/
```

Vérifie l'arborescence finale :

```
/Volumes/EFI/EFI/
├── BOOT/
│   └── BOOTx64.efi
└── OC/
    ├── ACPI/          ← tes 6 .aml
    ├── Drivers/
    ├── Kexts/
    ├── Resources/
    ├── Tools/
    ├── OpenCore.efi
    └── config.plist
```

Démonte proprement :

```bash
sudo diskutil unmount /Volumes/EFI
```

### 4.B.5 Démarrage sur le Latitude

Identique à la méthode Windows (§8.1) : **F12** au logo Dell, sélection de la clé en UEFI, menu OpenCore, entrée **« macOS Installer »** ou **« Install macOS Tahoe »**.

> 💡 **Avantage de cette méthode** : l'installateur complet ne nécessite pas de télécharger le reste du système pendant l'installation. Tu peux donc t'en passer d'Ethernet une fois la clé prête — utile si tu prépares tout depuis un endroit sans accès filaire au Latitude.

---

# 5. PHASE 4 — GÉNÉRATION DES SSDT

Les SSDT sont de petites tables ACPI qui corrigent le firmware Dell pour macOS. **SSDTTime** les génère à partir de tes propres tables — bien plus fiable que de récupérer celles d'un autre.

**Tu as besoin de cinq SSDT :** `PLUG`, `EC-USBX`, `PNLF`, `AWAC`, `XOSI`.

## 5.1 Préparer les tables ACPI

SSDTTime a besoin des tables de **ta** machine, en BIOS 1.36.0. Deux voies :

**Voie A — laisser SSDTTime dumper** (le plus simple)

Lance `SSDTTime.bat` en **administrateur**, puis option **`D`** (*Dump the current system's ACPI tables*). Il extrait tout et se positionne dessus automatiquement.

**Voie B — fournir un dossier de tables**

Si tu as déjà tes `.dat` via `acpidump.exe -b` :
```
C:\Hack\Rapport\ACPI\
├── dsdt.dat
├── ssdt1.dat
├── ssdt2.dat
└── ...
```
Depuis SSDTTime, utilise l'option de sélection de dossier et pointe dessus.

> 🔴 **Fournis l'ensemble des tables, pas seulement le DSDT.** Certaines méthodes référencées vivent dans les SSDT ; sans elles, SSDTTime peut mal détecter et produire des SSDT incorrects.

> ⚠️ Ces tables doivent venir du **BIOS 1.36.0**. Si tu as encore des dumps de la 1.26.0 qui traînent, jette-les.

## 5.2 Les cinq options à lancer

Lance-les dans cet ordre. Chaque option écrit son `.aml` dans `SSDTTime\Results\`.

### Option `3` — FakeEC Laptop → `SSDT-EC-USBX.aml`

Prends bien la variante **Laptop** (option 3), pas la simple (option 2) : la partie `USBX` définit les courants d'alimentation USB, nécessaire sur portable.

**Ce que SSDTTime va trouver** : ton contrôleur embarqué s'appelle **`ECDV`**, pas `EC`. C'est le cas favorable — l'outil crée un faux device `EC` sans avoir à renommer le device réel, donc **aucun patch de renommage à reporter**.

### Option `4` — PLUG → `SSDT-PLUG.aml`

**Ce que SSDTTime va trouver** : `Processor (PR00…PR07)` dans `Scope (_SB)`. Le SSDT ciblera `\_SB.PR00`.

Ton DSDT ne contient **aucun** `plugin-type`, donc ce SSDT est bien nécessaire — c'est lui qui active `X86PlatformPlugin` et toute la gestion d'alimentation CPU native.

### Option `6` — AWAC → `SSDT-AWAC.aml`

**Ce que SSDTTime va trouver** : `AWAC` (`ACPI000E`) **et** `RTC` (`PNP0B00`) coexistent dans ton DSDT. C'est exactement le cas que cette option traite — le SSDT désactive `AWAC` sous Darwin et force l'horloge héritée.

Selon la détection, le fichier peut s'appeler `SSDT-AWAC.aml` ou `SSDT-RTC0.aml`. Les deux conviennent, garde celui produit.

### Option `9` — PNLF → `SSDT-PNLF.aml`

**Ce que SSDTTime va trouver** : un device `LCD` avec une méthode `_BCM`, mais aucun `PNLF`. macOS a besoin de ce dernier pour exposer le curseur de luminosité.

> 🔴 **Vérifie la valeur `_UID` générée — c'est le point qui a posé le plus de problème sur cette machine.** SSDTTime propose par défaut `_UID = 0x10` (16), correspondant à la plage Skylake/Kaby Lake, `PWMMax = 0x056C`. **Ce n'est pas la bonne valeur pour Whiskey Lake.** Sur ce CPU, la bonne ligne est `_UID = 0x13` (19), Coffee Lake et plus récent, `PWMMax = 0xFFFF`.
>
> **Symptôme si tu gardes `0x10` par erreur** : le curseur de luminosité fonctionne visuellement dans les Réglages Système, mais le rétroéclairage plafonne très bas même au maximum. macOS calcule sur une échelle 47 fois trop petite (`0x056C` = 1388 au lieu de `0xFFFF` = 65535), donc "100%" envoyé au contrôleur PWM correspond en réalité à environ 2% de sa pleine puissance.
>
> **Corrige impérativement `_UID` à `0x13`** dans le `.dsl` avant compilation, ou ouvre l'`.aml` produit dans ProperTree et vérifie la valeur avant de le copier dans l'EFI.

Si l'outil demande le type de rétroéclairage, choisis la variante **Coffee Lake / Skylake+** — ton iGPU est une UHD 620 sur Whiskey Lake.

### Option `0` — XOSI → `SSDT-XOSI.aml` + patch de renommage

🔴 **C'est le SSDT critique de ce projet.** Sans lui, ton trackpad est détecté mais inerte (§5.3).

**Question posée** : la version de Windows maximale à déclarer.

**Réponse : `Windows 2015`.**

Voici pourquoi. Ton DSDT teste dix chaînes `_OSI` et en déduit la variable `OSYS` :

| Chaîne testée | `OSYS` obtenu |
|---|---|
| `Windows 2009` | `0x07D9` |
| **`Windows 2012`** | **`0x07DC`** ← seuil du trackpad |
| `Windows 2013` | `0x07DD` |
| **`Windows 2015`** | **`0x07DF`** ← la plus haute déclarée |

Le trackpad exige `OSYS >= 0x07DC`. Déclarer `Windows 2015` donne `0x07DF`, ce qui satisfait ce seuil et tous les autres tests conditionnels du DSDT. Descendre plus bas ferait rater des chemins de code ; monter plus haut n'apporterait rien puisque le DSDT ne teste rien au-delà.

**SSDTTime va signaler la présence d'une méthode `OSID`.** C'est normal : ton DSDT en contient une, qui appelle `_OSI` avec des constantes nommées (`WXP`, `WLG`, `WIN7`, `WIN8`, `WN81`). Le renommage les couvre aussi, c'est le comportement voulu.

> 💡 **Le renommage ne casse pas Windows.** La méthode `XOSI` générée ne retourne « vrai » pour les versions Windows que **sous Darwin** ; dans tous les autres cas elle relaie vers `_OSI` inchangé. Ton dual-boot n'est pas affecté.

## 5.3 Rappel — pourquoi XOSI est indispensable ici

Ton trackpad est `ACPI\DELL08BD`, déclaré sous `\_SB.PCI0.I2C0.TPD0` :

| Paramètre | Valeur |
|---|---|
| `_HID` | Méthode retournant `ITPN` = `DELL08BD` |
| `_CID` | `PNP0C50` — HID sur bus I2C |
| Bus | I2C0 (`8086:9DE8`), adresse `0x2C`, 400 kHz |
| Interruption | GpioInt via `\_SB.PCI0.GPI0` |
| `_STA` | `0x0F` en permanence |

`_STA` ne dépend d'aucune variable : le trackpad est **toujours** visible. Le problème est ailleurs :

```
Method (_CRS) {
    If ((OSYS < 0x07DC)) { Return (SBFI) }
    If ((TPDM == Zero))  { Return (ConcatenateResTemplate (I2CM (...), SBFG)) }
    Return (ConcatenateResTemplate (I2CM (...), SBFI))
}
```

Sans XOSI, la première branche retourne `SBFI` **seul** — un descripteur d'interruption nu, **sans ressource `I2cSerialBus`**. VoodooI2C n'a rien à quoi se raccrocher.

➡️ **Symptôme d'un oubli** : trackpad présent dans IORegistry, parfaitement mort. Pas d'erreur, pas de message. C'est le genre de panne qu'on cherche des heures.

## 5.4 Les options à NE PAS lancer

| Option | Pourquoi la sauter |
|---|---|
| `1` FixHPET | Aucun conflit d'IRQ détecté sur ta machine. À garder en réserve si l'audio ou les timers font des siennes |
| `2` FakeEC (non-laptop) | Remplacée par l'option 3 |
| `5` PMC | Concerne les cartes de bureau 300-series pour la NVRAM native. Sans objet sur un portable CNL-LP |
| `7` USB Reset | Sert au mapping USB par SSDT — tu utilises `UTBMap.kext` (§10) |
| `8` PCI Bridge | Aucun pont manquant |
| `A` Fix DMAR | Sert quand VT-d n'est pas désactivable. Le tien est à `VtForDirectIo=Disabled`, et tu gardes `DisableIoMapper = YES` en filet |

Et deux SSDT que d'autres guides ajouteraient, inutiles ici :

- **`SSDT-dGPU-Off`** — pas de GPU dédié sur ta machine
- **`SSDT-BATT`** — tes champs EC sont tous en 8 bits (§5.5)
- **`SSDT-GPI0`** — `GPI0` (`INT3450`) existe déjà et est fonctionnel

## 5.5 Batterie — rien à faire

Sur beaucoup de Dell, le contrôleur embarqué expose des champs de plus de 8 bits que macOS ne sait pas lire, ce qui impose `ECEnabler.kext` ou un `SSDT-BATT` sur mesure.

**Ton DSDT ne présente pas ce problème.** La région `ECOR` est intégralement découpée en octets :

```
OperationRegion (ECOR, EmbeddedControl, Zero, 0xFF)
Field (ECOR, ByteAcc, Lock, Preserve)
{
    EC00,   8,
    EC01,   8,
    ...
}
```

Zéro champ de 16, 32 ou 64 bits. Les devices `BAT0` et `BAT1` lisent ces octets directement.

➡️ `ECEnabler.kext` non nécessaire, `SSDT-BATT` non nécessaire.

## 5.6 Résultat réel de la génération — six SSDT, quatre patches

⚠️ **Mise à jour après vérification des fichiers produits.** SSDTTime a détecté deux conflits de nommage supplémentaires dans ton DSDT que je n'avais pas identifiés à l'avance. Le résultat final diffère donc de ce qui était annoncé plus haut — voici la version exacte à utiliser.

### Six fichiers `.aml`, pas cinq

| Fichier | Rôle | Vérifié contre ton DSDT |
|---|---|---|
| `SSDT-EC.aml` | Faux `EC` (`ACID0001`), actif sous Darwin | Cohérent — ton contrôleur réel s'appelle `ECDV` |
| `SSDT-PLUG.aml` | `plugin-type = 1` sur `\_SB.PR00` | Cible confirmée — c'est bien ton premier processeur |
| `SSDT-RTCAWAC.aml` | Force `STAS = 1`, désactive `AWAC`, garde `RTC` | Cohérent — les deux coexistent dans ton DSDT |
| `SSDT-XOSI.aml` | Renvoie vrai pour Windows jusqu'à 2015, sous Darwin | Liste vérifiée, va bien jusqu'à `"Windows 2015"` |
| `SSDT-PNLF.aml` | Device `PNLF`, **`_UID = 0x13`** (Coffee Lake/Whiskey Lake, PWMMax `0xFFFF`) | ⚠️ Pas `0x10` — voir encadré §5.2 |
| 🆕 `SSDT-SBUS-MCHC.aml` | Crée `MCHC` (absent de ton DSDT) + confirme `BUS0` sous le SMBus | Ton DSDT ne déclare aucun `MCHC` — nécessaire |

Copie les **six** `.aml` dans `EFI/OC/ACPI/`.

### Quatre patches dans `config.plist → ACPI → Patch`, pas un

`patches_OC.plist` contient quatre renommages. **L'ordre dans lequel ProperTree les insère est important** — respecte celui du fichier généré :

| # | Comment | Find (ASCII) | Replace (ASCII) | Pourquoi |
|---|---|---|---|---|
| 1 | `OSID to XSID rename` | `OSID` | `XSID` | Doit précéder le renommage `_OSI`, sinon ta méthode `OSID` (qui appelle `_OSI`) interférerait avec le patch suivant |
| 2 | `_OSI to XOSI rename` | `_OSI` | `XOSI` | Le renommage principal — active `SSDT-XOSI.aml` |
| 3 | `ECDV _STA to XSTA Rename` | `y\x00\x14\t_STA` | `y\x00\x14\tXSTA` | Désactive le `_STA` du vrai `ECDV`, pour laisser la main au faux `EC` du SSDT |
| 4 | `PNLF to XNLF Rename` | `PNLF` | `XNLF` | 🔴 **Ton DSDT contient déjà un champ nommé `PNLF`** (un tampon de calibration à la ligne 4083, `PNL7…PNLO`). Sans ce renommage, le nouveau device `PNLF` entrerait en collision de nom avec ce champ existant — l'ACPI refuserait de charger |

**Valeurs en Data (hexadécimal) à saisir dans ProperTree**, si tu préfères passer par le type Data plutôt que String :

```
1) Find: 4F534944   Replace: 58534944
2) Find: 5F4F5349   Replace: 584F5349
3) Find: 790014095F535441   Replace: 7900140958535441
4) Find:504E4C46   Replace: 584E4C46
```

> 💡 **Le plus simple** : n'écris rien à la main. Ouvre `patches_OC.plist` directement dans **ProperTree**, sélectionne le contenu de `ACPI → Add` et `ACPI → Patch`, copie-colle dans ton `config.plist`. C'est exactement pour ça que SSDTTime génère ce fichier séparé — le recopier à la main introduirait un risque d'erreur inutile sur des valeurs binaires.

### Un fichier à laisser de côté

Si SSDTTime a aussi produit `SSDT-Bridge.aml` ou `SSDT-PMC.aml` lors d'une passe précédente : **ne les ajoute pas**. Aucun pont PCI manquant ni contrôleur PMC 300-series identifié sur cette machine — ce sont des templates génériques sans rapport avec ton DSDT.

## 5.7 Vérification avant de passer au config.plist

- [ ] **Six** `.aml` présents dans `EFI/OC/ACPI/` : `EC`, `PLUG`, `RTCAWAC`, `XOSI`, `PNLF`, `SBUS-MCHC`
- [ ] Aucun `DSDT.aml`, `SSDT-Bridge.aml` ni `SSDT-PMC.aml` dans ce dossier
- [ ] `ACPI → Add` contient les six entrées, toutes en `Enabled = YES`
- [ ] `ACPI → Patch` contient **quatre** entrées, dans l'ordre : `OSID→XSID`, `_OSI→XOSI`, `ECDV _STA→XSTA`, `PNLF→XNLF`
- [ ] Les quatre patches ont `Count = 0` et `Limit = 0` (portée globale — c'est la valeur par défaut de SSDTTime, ne pas la modifier)

# 6. PHASE 5 — ARBORESCENCE EFI COMPLÈTE

Voici à quoi doit ressembler ton EFI une fois tout rassemblé.

```
EFI/
├── BOOT/
│   └── BOOTx64.efi
└── OC/
    ├── ACPI/
    │   ├── SSDT-PLUG.aml
    │   ├── SSDT-EC-USBX.aml
    │   ├── SSDT-PNLF.aml
    │   ├── SSDT-AWAC.aml
    │   └── SSDT-XOSI.aml            (si trackpad I2C)
    │
    ├── Drivers/
    │   ├── OpenRuntime.efi
    │   ├── HfsPlus.efi
    │   └── ResetNvramEntry.efi
    │
    ├── Kexts/
    │   ├── Lilu.kext
    │   ├── VirtualSMC.kext
    │   ├── SMCProcessor.kext
    │   ├── SMCSuperIO.kext
    │   ├── SMCBatteryManager.kext
    │   ├── WhateverGreen.kext
    │   ├── NVMeFix.kext
    │   ├── ECEnabler.kext
    │   ├── RestrictEvents.kext
    │   ├── BrightnessKeys.kext
    │   ├── RealtekRTL8111.kext
    │   ├── itlwm.kext               (jamais AirportItlwm sous Tahoe)
    │   ├── IntelBluetoothFirmware.kext
    │   ├── IntelBTPatcher.kext      (présent mais DÉSACTIVÉ - §12.4)
    │   ├── BlueToolFixup.kext
    │   ├── VoodooPS2Controller.kext
    │   ├── VoodooI2C.kext           (si trackpad I2C)
    │   ├── VoodooI2CHID.kext        (si trackpad I2C)
    │   └── USBMap.kext              (ajouté en phase 9)
    │
    ├── Resources/
    │   ├── Audio/
    │   ├── Font/
    │   ├── Image/
    │   └── Label/
    │
    ├── Tools/
    │   └── OpenShell.efi
    │
    ├── OpenCore.efi
    └── config.plist
```

**Ordre de chargement des kexts** — règle absolue : un plugin se charge **après** sa dépendance. `Lilu` en premier, `VirtualSMC` avant ses plugins SMC*, `VoodooI2C` avant `VoodooI2CHID`.

Dans ProperTree, `Ctrl+Shift+R` (**OC Clean Snapshot**) lit ton dossier EFI et remplit automatiquement les sections `ACPI → Add`, `Kernel → Add`, `UEFI → Drivers` et `Misc → Tools` dans le bon ordre. Utilise-le systématiquement après chaque ajout de fichier.

---

# 7. PHASE 6 — CONFIG.PLIST SECTION PAR SECTION

Ouvre `config.plist` dans **ProperTree**. Toutes les valeurs non mentionnées restent à leur valeur par défaut du `Sample.plist`.

> **Règle d'or** : ne supprime jamais une clé. OpenCore n'a pas de valeurs de repli — une clé absente provoque un refus de démarrage.

## 7.1 ACPI

### ACPI → Add
Rempli automatiquement par le Clean Snapshot. Vérifie que chaque `.aml` de ton dossier apparaît avec `Enabled = YES`.

### ACPI → Delete
Vide.

### ACPI → Patch

**Uniquement si tu utilises SSDT-XOSI** (pas avec GPI0) :

| Clé | Type | Valeur |
|---|---|---|
| Comment | String | `Change _OSI to XOSI` |
| Enabled | Boolean | **YES** |
| Count | Number | `0` |
| Limit | Number | `0` |
| Find | Data | `5F4F5349` |
| Replace | Data | `584F5349` |

### ACPI → Quirks
Tout sur `NO` (valeurs par défaut).

## 7.2 Booter

### Booter → Quirks

| Quirk | Valeur | Note |
|---|---|---|
| AvoidRuntimeDefrag | **YES** | |
| DevirtualiseMmio | NO | |
| DisableSingleUser | NO | |
| DisableVariableWrite | NO | |
| DiscardHibernateMap | NO | |
| EnableSafeModeSlide | **YES** | |
| **EnableWriteUnprotector** | **NO** | Voir note ci-dessous |
| ForceBooterSignature | NO | |
| ForceExitBootServices | NO | |
| ProtectMemoryRegions | NO | Chromebooks uniquement |
| ProtectSecureBoot | NO | |
| ProtectUefiServices | NO | |
| ProvideCustomSlide | **YES** | |
| ProvideMaxSlide | `0` | |
| **RebuildAppleMemoryMap** | **YES** | |
| ResizeAppleGpuBars | `-1` | |
| SetupVirtualMap | **YES** | |
| SignalAppleOS | NO | |
| **SyncRuntimePermissions** | **YES** | |

> ⚠️ **Si tu restes bloqué sur `EB|LOG:EXITBS:START`** au premier boot : inverse ce couple → `RebuildAppleMemoryMap = NO` et `EnableWriteUnprotector = YES`. C'est fréquent sur les firmwares OEM anciens. Teste la combinaison recommandée d'abord.

## 7.3 DeviceProperties

### DeviceProperties → Add → `PciRoot(0x0)/Pci(0x2,0x0)`

Cette entrée n'existe pas dans le Sample — crée-la manuellement (clic droit → New Child sous `Add`, type Dictionary).

| Clé | Type | Valeur | Rôle |
|---|---|---|---|
| `AAPL,ig-platform-id` | Data | **`00009B3E`** | Framebuffer UHD 620 pour portable |
| `device-id` | Data | **`9B3E0000`** | 🔴 Obligatoire : UHD 620 sur CPU Coffee/Whiskey Lake doit se faire passer pour un 0x3E9B |
| `framebuffer-patch-enable` | Data | **`01000000`** | Active les patchs suivants |
| `framebuffer-stolenmem` | Data | **`00003001`** | 🔴 Compense le DVMT bloqué à 32 Mo |
| `framebuffer-fbmem` | Data | **`00009000`** | 🔴 Idem |

Ces trois derniers sont **obligatoires** sur ce BIOS Dell puisqu'aucun réglage DVMT n'est exposé. Sans eux : panique noyau systématique au chargement d'`AppleIntelCFLGraphicsFramebuffer`.

### DeviceProperties → Add → `PciRoot(0x0)/Pci(0x1f,0x3)`

Supprime cette entrée si elle existe (elle contient un `layout-id` AppleALC devenu inutile sous Tahoe).

### DeviceProperties → Delete
Vide.

## 7.4 Kernel

### Kernel → Add
Rempli par le Clean Snapshot. Vérifie l'ordre et désactive les plugins PS/2 non voulus :

| Kext | Enabled |
|---|---|
| `VoodooPS2Mouse.kext` | **NO** (si trackpad I2C) |
| `VoodooPS2Trackpad.kext` | **NO** (si trackpad I2C) |
| `VoodooInput.kext` | YES |
| `VoodooPS2Keyboard.kext` | YES |

### Kernel → Emulate
Tout vide / `0`.

### Kernel → Force / Block
Vides.

### Kernel → Patch
Vide pour l'instant. On y reviendra si tu choisis un SMBIOS non supporté (§7.7).

### Kernel → Quirks

| Quirk | Valeur | Note |
|---|---|---|
| AppleCpuPmCfgLock | NO | Ivy Bridge et antérieurs uniquement |
| **AppleXcpmCfgLock** | **YES** | 🔴 Le BIOS Dell n'expose pas CFG Lock |
| AppleXcpmExtraMsrs | NO | |
| AppleXcpmForceBoost | NO | |
| CustomPciSerialDevice | NO | |
| **CustomSMBIOSGuid** | **YES** | 🔴 **Spécifique Dell** |
| DisableIoMapper | **YES** | VT-d non désactivable dans ce BIOS |
| DisableIoMapperMapping | NO | |
| DisableLinkeditJettison | **YES** | |
| DisableRtcChecksum | **YES** | Évite les réinitialisations BIOS après reboot (fréquent sur Dell) |
| ExtendBTFeatureFlags | NO | |
| ForceAquantiaEthernet | NO | |
| ForceSecureBootScheme | NO | |
| IncreasePciBarSize | NO | |
| LapicKernelPanic | NO | HP uniquement |
| LegacyCommpage | NO | |
| PanicNoKextDump | **YES** | |
| PowerTimeoutKernelPanic | **YES** | |
| ProvideCurrentCpuInfo | NO | |
| SetApfsTrimTimeout | `-1` | |
| ThirdPartyDrives | NO | NVMe non concerné |
| **XhciPortLimit** | **NO** | 🔴 Ne fonctionne plus depuis macOS 11.3. Il **faut** cartographier les USB (§10) |

### Kernel → Scheme
`KernelArch = x86_64`, `KernelCache = Auto`, le reste par défaut.

## 7.5 Misc

### Misc → Boot

| Clé | Valeur |
|---|---|
| HibernateMode | `None` |
| HibernateSkipsPicker | NO |
| **HideAuxiliary** | **YES** |
| LauncherOption | `Disabled` (→ `Full` après installation, §9.3) |
| LauncherPath | `Default` |
| PickerAttributes | `17` |
| PickerMode | `Builtin` (ou `External` si OpenCanopy) |
| PickerVariant | `Auto` |
| PollAppleHotKeys | **YES** |
| ShowPicker | **YES** |

> ⚠️ **`Timeout = 0` semble tentant** (le picker attend indéfiniment, aucun risque de relance automatique sur la mauvaise entrée) mais se retourne contre toi pendant l'installation macOS, qui redémarre plusieurs fois d'affilée : sans présence devant la machine à chaque cycle pour valider, l'installation reste bloquée sur le menu entre deux redémarrages. Une valeur de `10` secondes est le bon compromis — le picker attend un choix manuel mais relance automatiquement la dernière entrée sélectionnée si personne n'intervient, ce qui convient à un cycle d'installation multi-reboot sans surveillance constante.
| TakeoffDelay | `0` |
| Timeout | `10` |

### Misc → Debug

Pour la phase d'installation (avec la version **DEBUG** d'OpenCore) :

| Clé | Valeur |
|---|---|
| AppleDebug | **YES** |
| ApplePanic | **YES** |
| DisableWatchDog | **YES** |
| DisplayDelay | `0` |
| DisplayLevel | `2147483650` |
| SysReport | NO |
| **Target** | **`67`** |

Une fois le système stable, repasse en RELEASE avec `Target = 3`.

### Misc → Security

| Clé | Valeur | Note |
|---|---|---|
| AllowSetDefault | **YES** | |
| ApECID | `0` | |
| AuthRestart | NO | |
| BlacklistAppleUpdate | **YES** | |
| DmgLoading | `Signed` | |
| EnablePassword | NO | |
| ExposeSensitiveData | `6` | |
| HaltLevel | `2147483648` | |
| **ScanPolicy** | **`0`** | 🔴 Sinon la clé USB n'apparaît pas |
| **SecureBootModel** | **`Disabled`** | Voir note |
| **Vault** | **`Optional`** | 🔴 C'est un mot, pas un booléen. Sensible à la casse |

> **SecureBootModel** : `Disabled` est requis pour les mises à jour OTA sous Tahoe (§14.3). Si tu veux conserver le Secure Boot Apple, laisse `Default` et ajoute le kext **iBridged** (github.com/Carnations-Botanica/iBridged).

### Misc → Serial / Entries
Par défaut / vides.

### Misc → Tools
`OpenShell.efi` avec `Enabled = YES` pendant la phase de mise au point.

## 7.6 NVRAM

### NVRAM → Add → `7C436110-AB2A-4BBB-A880-FE41995C9F82`

| Clé | Type | Valeur |
|---|---|---|
| `boot-args` | String | voir ci-dessous |
| `csr-active-config` | Data | `00000000` (→ `03080000` post-install si AppleALC+AppleHDA, `03000000` si VoodooHDA — §11) |
| `prev-lang:kbd` | String | `fr-FR:1` |
| `run-efi-updater` | String | `No` |

**boot-args pour l'installation** (verbose, pour voir les erreurs) :

```
-v debug=0x100 keepsyms=1 -ibtcompatbeta
```

**boot-args après installation** (une fois tout stable) :

```
-ibtcompatbeta revpatch=sbvmm,cpuname
```

Détail de chaque argument :

| Argument | Rôle |
|---|---|
| `-v` | Mode verbeux — indispensable au dépannage, à retirer ensuite |
| `debug=0x100` | Empêche le redémarrage automatique lors d'une panique noyau |
| `keepsyms=1` | Affiche les symboles dans les paniques |
| **`-ibtcompatbeta`** | 🔴 **Spécifique macOS 26** — requis pour le Bluetooth Intel |
| `revpatch=sbvmm` | Débloque les mises à jour OTA (avec RestrictEvents) |
| `revpatch=cpuname` | Affiche le vrai nom du CPU dans "À propos de ce Mac" |
| ~~`-wegnoegpu`~~ | ❌ Inutile ici — pas de GPU dédié sur ta machine |
| `-igfxnotelemetryload` | À ajouter si gel au démarrage sur le logo Apple |
| `igfxonln=1` | À ajouter si l'écran externe HDMI n'est pas détecté à chaud |

> Ces arguments se cumulent, séparés par des espaces. `revpatch` prend ses options séparées par des virgules **sans espace**.

### NVRAM → Add → `4D1FDA02-38C7-4A6A-9CC6-4BCCA8B30102`
Par défaut.

### NVRAM → Delete
Laisse les entrées par défaut (elles forcent la réécriture de `boot-args` à chaque démarrage, pratique en phase de réglage).

### NVRAM → WriteFlash
**YES**

## 7.7 PlatformInfo

C'est le point qui change le plus par rapport à un guide Sequoia.

### Choix du SMBIOS

Tahoe n'accepte que quatre modèles. Pour un portable, deux stratégies :

**Option A — recommandée : `MacBookPro16,2`**

| Avantage | Inconvénient |
|---|---|
| SMBIOS officiellement supporté par Tahoe | Modèle Ice Lake — les vecteurs de fréquence CPU ne correspondent pas exactement à Whiskey Lake |
| Mises à jour OTA fonctionnelles | Gestion d'alimentation potentiellement sous-optimale |
| Aucun patch supplémentaire | Corrigeable avec CPUFriend |

**Option B — `MacBookPro15,2` + contournement du board-id**

Meilleure correspondance CPU (quad-core 15 W), donc gestion d'alimentation plus fine, mais nécessite un patch pour contourner la vérification du board-id, et les mises à jour OTA deviennent aléatoires.

➡️ **Commence par l'option A.** Si tes fréquences CPU restent bloquées ou si la machine chauffe, passe à l'option B ou installe CPUFriend.

### Génération des identifiants

```
cd C:\Hack\Tools\GenSMBIOS
GenSMBIOS.bat
```
→ Option 1 (télécharger MacSerial) → Option 3 → saisir `MacBookPro16,2`

Reporte les valeurs :

| Sortie GenSMBIOS | Destination dans PlatformInfo → Generic |
|---|---|
| `Type` | `SystemProductName` |
| `Serial` | `SystemSerialNumber` |
| `Board Serial` | `MLB` |
| `SmUUID` | `SystemUUID` |

| Clé | Valeur |
|---|---|
| **`ROM`** | **`E454E83F4A73`** (type Data) — ta vraie MAC Ethernet, déjà connue ✅ |
| `SpoofVendor` | **YES** |

> 💡 **Gain de temps** : normalement il faut renseigner une valeur bidon puis la corriger après installation. Comme le BIOS t'a donné l'adresse LOM (`E4-54-E8-3F-4A-73`), tu peux mettre la bonne valeur dès maintenant. Dans ProperTree, choisis le type **Data** et saisis les 12 caractères hexadécimaux **sans les tirets** : `E454E83F4A73`.
| `AdviseFeatures` | NO |
| `MaxBIOSVersion` | NO |
| `ProcessorType` | `0` |
| `SystemMemoryStatus` | `Auto` |

> 🔴 **Le numéro de série doit être INVALIDE.** Teste-le sur `checkcoverage.apple.com` : tu dois obtenir « Impossible de vérifier la couverture pour ce numéro de série ». Si Apple reconnaît le numéro, régénère — sinon tu risques de faire bannir le vrai propriétaire du Mac.

### PlatformInfo — racine

| Clé | Valeur | Note |
|---|---|---|
| `Automatic` | **YES** | |
| `UpdateDataHub` | **YES** | |
| `UpdateNVRAM` | **YES** | |
| `UpdateSMBIOS` | **YES** | |
| **`UpdateSMBIOSMode`** | **`Custom`** | 🔴 **Spécifique Dell**, à coupler avec `CustomSMBIOSGuid = YES` |
| `CustomMemory` | NO | |

## 7.8 UEFI

### UEFI → APFS

| Clé | Valeur |
|---|---|
| `EnableJumpstart` | YES |
| `MinDate` | `0` |
| `MinVersion` | `0` |

### UEFI → Drivers
Rempli par le Clean Snapshot. Vérifie que **`OpenRuntime.efi` a `LoadEarly = YES`** — les autres restent à `NO`.

### UEFI → Input / Output / Audio
Par défaut. (`UIScale = 0` dans Output.)

### UEFI → Quirks

| Quirk | Valeur | Note |
|---|---|---|
| ActivateHpetSupport | NO | |
| DisableSecurityPolicy | NO | Passe à YES si les drivers `.efi` refusent de se charger |
| EnableVectorAcceleration | YES | |
| ForceOcWriteFlash | NO | |
| ForgeUefiSupport | NO | |
| **ReleaseUsbOwnership** | **YES** | 🔴 Firmwares portables OEM |
| **RequestBootVarRouting** | **YES** | 🔴 Sinon Dell efface l'entrée de boot OpenCore |
| ResizeGpuBars | `-1` | |
| TscSyncTimeout | `0` | |
| UnblockFsConnect | NO | HP uniquement |

### UEFI → ReservedMemory / ProtocolOverrides
Par défaut.

## 7.9 Validation avant boot

OpenCorePkg fournit un validateur — **utilise-le systématiquement** :

```
cd C:\Hack\OpenCore-1.0.7-RELEASE\Utilities\ocvalidate
ocvalidate.exe D:\EFI\OC\config.plist
```

Tu dois obtenir `0 problems found`. Corrige tout ce qui remonte avant d'aller plus loin.

---

# 8. PHASE 7 — PREMIER BOOT ET INSTALLATION

### 8.1 Démarrage sur la clé

Redémarre → **F12** au logo Dell → sélectionne la clé USB en mode UEFI.

Le menu OpenCore doit apparaître. Sélectionne **« macOS Base System »** ou **« Install macOS Tahoe »**.

Le défilement en mode verbeux est long (2 à 5 minutes). C'est normal.

### 8.2 Utilitaire de disque

Une fois dans l'installateur :

`Utilitaire de disque` → menu **Présentation → Afficher tous les appareils** (crucial, sinon tu ne verras que les volumes)

**Si tu veux garder Windows** : sélectionne l'espace non alloué que tu auras préparé sous Windows (Gestion des disques → réduire la partition Windows d'au moins 80 Go). Puis Partitionner.

**Si tu dédies le disque à macOS** : sélectionne le disque physique entier → Effacer :
- Nom : `Macintosh HD`
- Format : **APFS**
- Schéma : **Table de partition GUID**

### 8.3 Installation

Quitte l'Utilitaire de disque → Installer macOS Tahoe → sélectionne le volume.

Le système télécharge ~14 Go puis redémarre **plusieurs fois** (3 à 5 en général). À chaque redémarrage :

1. Boot sur la clé USB (F12)
2. Dans le menu OpenCore, sélectionne l'entrée **« macOS Installer »** ou le nom de ton volume
3. Ne choisis pas l'installateur de la clé aux redémarrages intermédiaires

Compte 45 à 90 minutes au total.

### 8.4 Assistant de configuration

Quand macOS démarre pour la première fois :
- **Branche un câble Ethernet** — le Wi-Fi ne fonctionne pas encore
- Tu peux ignorer la connexion à l'Apple ID pour l'instant (on réglera les iServices en §14)

---

# 9. PHASE 8 — POST-INSTALLATION

### 9.1 Vérifications immédiates

Ouvre le Terminal et contrôle :

```bash
# Accélération graphique — doit indiquer 1536 Mo de VRAM, pas 7 Mo
system_profiler SPDisplaysDataType | grep -A2 "Chipset"

# Gestion d'alimentation — X86PlatformPlugin doit être chargé
kextstat | grep -i x86platform

# Kexts chargés
kextstat | grep -v com.apple
```

Si tu vois **7 Mo de VRAM** et un fond d'écran uni sans transparence : l'accélération n'est pas active → revoir §7.3.

### 9.2 Installation de l'EFI sur le SSD

Sur macOS, monte la partition EFI du disque interne :

```bash
diskutil list                    # repère le disque interne, ex. disk0
sudo diskutil mount disk0s1      # la partition EFI est généralement s1
```

Copie ton dossier `EFI` complet depuis la clé USB vers la racine de la partition EFI montée.

> Sur ce Dell, la partition EFI créée par Windows fait généralement 100 Mo. Ton EFI OpenCore pèse 15 à 25 Mo — largement suffisant.

Redémarre **sans la clé USB** : F12 → « UEFI: OS Boot Manager » ou l'entrée créée automatiquement.

**Si l'entrée n'apparaît pas** dans le menu de boot Dell : passe `Misc → Boot → LauncherOption` de `Disabled` à **`Full`** dans le config.plist de l'EFI interne, puis redémarre une fois depuis la clé. OpenCore s'enregistrera lui-même dans la NVRAM du firmware.

### 9.3 Bascule en version RELEASE

Une fois le système stable :
- Remplace `OpenCore.efi` et `BOOTx64.efi` par les versions **RELEASE**
- ⚠️ Remplace **aussi** `OpenRuntime.efi` — un mélange de versions DEBUG/RELEASE empêche le démarrage
- `Misc → Debug → Target` → `3`
- `boot-args` → retire `-v debug=0x100 keepsyms=1`

---

# 10. PHASE 9 — CARTOGRAPHIE USB

> # ✅ CARTOGRAPHIE TERMINÉE ET VALIDÉE
>
> `usb.json` contient les **10 ports attendus**, tous typés correctement, appairages HS↔SS cohérents, aucun port en trop ni manquant.
>
> Il ne reste qu'à générer `UTBMap.kext` (§10.3).

## 10.1 Ta cartographie validée

**Contrôleur unique** : `\_SB.PCI0.XHC` — `8086:9DED`, 18 ports déclarés.

| Port | Rôle physique | Compagnon | Type | Sélectionné ? |
|---|---|---|---|---|
| **1** | USB-A 3.x n°1 — côté HS | ↔ 13 | `3` | ✅ |
| **13** | USB-A 3.x n°1 — côté SS | ↔ 1 | `3` | ✅ |
| **3** | USB-A 3.x n°2 — côté HS | ↔ 15 | `3` | ✅ |
| **15** | USB-A 3.x n°2 — côté SS | ↔ 3 | `3` | ✅ |
| **4** | USB-C (recharge) — côté HS | ↔ 16 | `9` | ✅ |
| **16** | USB-C (recharge) — côté SS | ↔ 4 | `9` | ✅ |
| **2** | USB-A 2.0 seul (aucun compagnon) | — | `0` | ✅ |
| **6** | `Integrated_Webcam_HD` (`0BDA:5520`) | — | `255` | ✅ |
| **7** | `USB2.0-CRW` — lecteur de carte SD | — | `255` | ✅ |
| **10** | `Intel Wireless Bluetooth` (`8087:0AAA`) | — | `255` | ✅ |

**Nouveauté** : le port **7** est apparu, il héberge le lecteur de carte SD (`USB2.0-CRW`). C'est le résultat direct de `MediaCard=Enabled` / `SdCard=Enabled` dans le BIOS. Déclare-le en **interne (`255`)**.

**Cible : 10 ports sélectionnés** — bien en dessous de la limite de 15, donc `XhciPortLimit` reste à `NO`.

Ports internes sans usage : 5, 8, 9, 11, 12, 14, 17, 18. Laisse-les décochés.

## 10.2 Notes sur les types

### Le port 4/16 : USB-C de recharge

Vérifié physiquement : c'est bien un connecteur USB-C, et c'est **le port d'alimentation du portable**.

Le type `9` est le bon. La distinction entre les deux types Type-C se joue sur le nombre de ports SuperSpeed rattachés au connecteur :

| Type | Quand l'utiliser |
|---|---|
| `8` — Type-C **avec** commutateur | Deux ports SS pour un connecteur : retourner le câble change de port SS |
| `9` — Type-C **sans** commutateur | Un seul port SS, l'orientation est gérée par le contrôleur |

Ton port 4 n'a qu'un compagnon, le 16 → **type 9**.

> 🔴 **Ne laisse jamais ce port décoché.** C'est ton port d'alimentation. La recharge elle-même est gérée par le contrôleur embarqué et fonctionnera quoi qu'il arrive, mais un port d'alimentation non déclaré perturbe la gestion de veille.

**DisplayPort Alt Mode** : ce port supporte probablement la sortie vidéo. Si elle ne fonctionne pas, c'est un sujet de patching de connecteurs WhateverGreen — à traiter après installation, jamais avant. Voir §15.

### Tableau de référence des types

| Type | Valeur | Tes ports |
|---|---|---|
| USB 2.0 | `0` | 2 |
| USB 3.0 | `3` | 1, 3, 13, 15 |
| Type-C sans commutateur | `9` | 4, 16 |
| **Interne** | **`255`** | 6 (webcam), 7 (lecteur SD), 10 (Bluetooth) |

> Marquer le Bluetooth en **interne (`255`)** est ce qui permet le réveil par clavier ou souris Bluetooth et une veille correcte.

## 10.3 Générer le kext

La sélection est faite. Il ne reste que la génération.

1. Lance USBToolBox — il recharge `usb.json` automatiquement s'il est dans le même dossier
2. Menu principal → **Build kext**
3. Copie `UTBMap.kext` dans `EFI/OC/Kexts/`
4. **OC Clean Snapshot** dans ProperTree (`Ctrl+Shift+R`)

> **Vérifie le mode de génération.** Dans les réglages de l'outil, une option produit une carte utilisant les classes natives d'Apple plutôt que celles d'USBToolBox.
>
> | Mode | Ce qu'il faut embarquer |
> |---|---|
> | Classes natives | `UTBMap.kext` seul |
> | Classes USBToolBox | `USBToolBox.kext` **puis** `UTBMap.kext`, dans cet ordre |
>
> Se tromper d'ordre de chargement dans `Kernel → Add` rend la carte inopérante sans message d'erreur.

# 11. PHASE 10 — AUDIO SOUS TAHOE

**Apple a retiré `AppleHDA.kext` de macOS à partir de la beta 2 de Tahoe** — les Mac Intel équipés d'une puce T2 n'en ont plus besoin, et Tahoe étant la dernière version supportant l'Intel, Apple a simplement suivi ce chemin. Sans ce composant, `AppleALC` n'a plus rien à patcher : haut-parleurs internes et prise jack restent muets par défaut.

**Ce qui fonctionne encore sans rien faire** : audio HDMI/DisplayPort, casques et cartes son USB — ces chemins ne dépendent pas d'`AppleHDA`.

Deux solutions existent. **La première (§11.1) est celle qui fonctionne réellement et qui a été validée sur cette machine — AppleALC + réinjection d'`AppleHDA`.** VoodooHDA (§11.2) reste documentée en repli, plus simple mais de moins bonne qualité.

## 11.1 Solution retenue et confirmée : AppleALC + AppleHDA réinjecté

C'est plus lourd à mettre en place que VoodooHDA, mais le résultat est fidèle à un vrai Mac : détection de jack correcte, meilleure qualité, pas de panneau de réglage tiers.

### Pourquoi une simple copie de fichier ne suffit pas

`AppleHDA` est censé faire partie de la collection de noyau système, scellée cryptographiquement avec le reste du volume (`SystemKernelExtensions.kc`). Contrairement à `Lilu` ou `VoodooI2C`, injectés par OpenCore au démarrage EFI, `AppleHDA` doit être **intégré à cette collection scellée** — ce qui impose de désceller le volume système, littéralement reconstruire la collection avec le kext ajouté, puis rescelle. C'est un mécanisme du même ordre que le patching de volume racine d'OCLP, avec les mêmes contraintes : à refaire après chaque mise à jour majeure de macOS.

### Prérequis

| Élément | Détail |
|---|---|
| Kext | `AppleALC.kext`, version **1.9.5 ou plus récente**, dans `EFI/OC/Kexts/` et activé dans `Kernel → Add` (position : juste après `Lilu.kext`) |
| `csr-active-config` | `03080000` dans `NVRAM → Add → 7C436110-...` (ajoute `CSR_ALLOW_UNAUTHENTICATED_ROOT` au SIP partiellement désactivé) |
| **`AppleHDA.kext`** | 🔴 **Point critique** — voir ci-dessous, ne pas le chercher dans un KDK Tahoe récent |

> ⚠️ **Retire `csr-active-config` de `NVRAM → Delete`** s'il y figure — sinon la valeur est réécrite à chaque démarrage.

### Où trouver `AppleHDA.kext` — le vrai piège

**Apple l'a retiré à partir de la beta 2 de Tahoe.** Les KDK correspondant aux versions publiques de Tahoe (26.x) ne le contiennent **plus** — chercher dedans est une perte de temps garantie. Deux sources valables seulement :

- Le KDK de la **beta 1** de Tahoe, la seule version qui l'a encore embarqué
- Le dossier `/System/Library/Extensions/AppleHDA.kext` d'une installation **macOS 15 Sequoia** — fonctionne aussi bien

### Outil d'installation : MyKextInstaller

`github.com/Mirone/MyKextInstaller` — l'app la plus simple, signée et notarisée. Elle propose de **télécharger `AppleHDA.kext` directement depuis l'application**, sans avoir à le chercher toi-même dans un KDK — c'est la voie la plus fiable.

*(Alternative : SimpleLoader de Laobamac, interface SwiftUI, disponible en plusieurs langues via un fork traduit — `github.com/perez987/SimpleLoader`. Fonctionnellement équivalent.)*

### Procédure

1. Lance MyKextInstaller
2. Laisse l'app télécharger `AppleHDA.kext` elle-même (ou fournis-le si tu l'as extrait d'un Sequoia)
3. Installe le kext
4. **🔴 Quand l'app demande de redémarrer : choisis impérativement « Restart Now », jamais « Later ».**

> **C'est le détail qui fait toute la différence.** L'intégration dans la collection scellée n'est effective qu'au prochain démarrage propre. Un redémarrage différé laisse le temps à d'autres processus (Spotlight, Time Machine, mises à jour en attente) d'interférer avec le remontage du volume — c'est la cause la plus probable d'un échec silencieux où le kext semble installé mais ne charge jamais.

### Vérification après redémarrage

```bash
kextstat | grep "AppleHDA"
```

Trois lignes doivent apparaître : `com.apple.driver.AppleHDAController`, `com.apple.driver.AppleHDA`, `com.apple.driver.AppleHDAHardwareConfigDriver`. Si rien n'apparaît, le kext n'a jamais intégré la collection — reprends à l'étape du redémarrage.

Confirmation supplémentaire :
```bash
ioreg -l -w0 | grep -i "AppleHDAController"
```
Doit apparaître comme enfant du device `HDEF`.

### Trouver le bon layout-id — méthode et résultat confirmé sur cette machine

Ton codec est un **Realtek ALC236** (`Vendor Id: 0x10ec0236`, révision `0x100002`), que Dell référence en interne sous l'appellation `ALC3204`/`ALC3271`. Layouts officiellement supportés pour cette famille :

```
3, 11, 12, 13, 14, 15, 16, 17, 18, 19, 23, 36, 54, 55, 68, 69, 99
```

**✅ `layout-id = 19` fonctionne sur ce Latitude 3500 — confirmé.** C'est le layout Lenovo IdeaPad 500-14ISK à l'origine, mais il couvre exactement la configuration de pins observée sur cette machine (haut-parleur interne, micro interne, jack casque avec détection).

Pour tester un layout par itération, un seul à la fois, dans `boot-args` :
```
alcid=19
```
Une fois confirmé fonctionnel, rends-le permanent dans `DeviceProperties → Add → PciRoot(0x0)/Pci(0x1F,0x3)` :

| Clé | Type | Valeur |
|---|---|---|
| `layout-id` | Data | `13000000` (19 en hexadécimal little-endian) |

Puis **retire le boot-arg `alcid=`** — il a toujours la priorité la plus haute et masquerait la `DeviceProperty`.

### Rappel important

Une mise à jour majeure de macOS Tahoe **efface la réinjection** — `AppleHDA.kext` disparaît à nouveau du volume scellé et doit être réinstallé après chaque montée de version. Garde MyKextInstaller ou SimpleLoader sous la main.

---

## 11.2 Solution de repli : VoodooHDA

Plus simple, ne touche pas au volume scellé, mais qualité audio inférieure et détection de jack généralement absente. À réserver si la réinjection d'`AppleHDA` échoue ou si tu préfères éviter de toucher au SIP au niveau `UNAUTHENTICATED_ROOT`.

**Étape 1 — Abaisser SIP**

Dans `config.plist → NVRAM → Add → 7C436110-...` :

```
csr-active-config = 03000000
```

(Note : `03000000` suffit pour VoodooHDA — pas besoin du bit `UNAUTHENTICATED_ROOT` requis par la réinjection d'AppleHDA.)

Retire aussi `csr-active-config` de `NVRAM → Delete` s'il y figure. Redémarre.

**Étape 2 — Installer VoodooHDA**

Télécharge la dernière release sur `github.com/CloverHackyColor/VoodooHDA` (paquet contenant `VoodooHDA.kext` et `VoodooHDA.prefpane`).

```bash
sudo cp -R ~/Downloads/VoodooHDA.kext /Library/Extensions/
cp -R ~/Downloads/VoodooHDA.prefpane ~/Library/PreferencePanes/

sudo chmod -R 755 /Library/Extensions/VoodooHDA.kext
sudo chown -R root:wheel /Library/Extensions/VoodooHDA.kext

sudo kmutil install --update-all
```

**Étape 3 — Autoriser le chargement**

`Réglages Système → Confidentialité et sécurité` → un message signale un logiciel système bloqué → **Autoriser**. Redémarre.

**Étape 4 — Vérifier**

`Réglages Système → Son` doit lister les sorties. Le panneau `VoodooHDA` dans les Réglages permet d'ajuster les niveaux si le son est saturé ou trop faible.

**Ce que ça coûte par rapport à la solution AppleALC :**

| Conséquence | Détail |
|---|---|
| Qualité audio | Inférieure à AppleHDA : bruit de fond possible, égalisation moins bonne |
| Détection du jack | Souvent non automatique — bascule manuelle des sorties |
| Micro interne | Fonctionne généralement, parfois avec un niveau faible |
| Apple TV / DRM | Le contenu protégé peut refuser de se lire avec SIP abaissé |

## 11.3 Ton codec — pour référence

`ALC3271`/`ALC3204` (appellations Dell) = **Realtek ALC236** en interne, confirmé via dump codec sous Linux :

```bash
cat /proc/asound/card0/codec#0 | head -20
```

Utile pour toute recherche future — cherche « ALC236 » plutôt que les références Dell propriétaires.

## 11.4 Alternative pragmatique

Un petit **DAC USB-C** à 15-20 € contourne tout le problème : SIP reste inchangé, qualité supérieure, aucune configuration, aucune réinjection à refaire après mise à jour. Sur un portable dont les haut-parleurs internes sont de toute façon médiocres, ça reste une option à considérer si la réinjection AppleHDA devient trop contraignante à maintenir dans la durée.

---

# 12. PHASE 11 — WI-FI ET BLUETOOTH INTEL

## 12.1 Le slot M.2 A/E accepte le CNVi **et** le PCIe

⚠️ **Correction d'une affirmation antérieure de ce guide.** Une version précédente concluait, à partir du sous-système Intel `4234:8086` relevé sur la carte AC9560 d'origine, que le slot était « câblé CNVi uniquement » et qu'aucune carte PCIe n'y fonctionnerait. **C'était une déduction abusive** : cet identifiant prouve seulement que la carte *installée* était du CNVi, pas que le connecteur soit dépourvu de lignes PCIe.

**Vérifié en pratique sur cette machine** : une Intel **AX200NGW**, qui est une carte **PCIe** et non CNVi, fonctionne parfaitement dans ce slot. Le connecteur porte donc bien les deux jeux de signaux.

### Cartes compatibles — tableau de référence

| Carte | Interface | Tient dans le slot ? | Wi-Fi sous Tahoe | Bluetooth sous Tahoe |
|---|---|---|---|---|
| **AC 9560** (d'origine) | CNVio Gen1 | ✅ | ✅ itlwm + HeliPort | ❌ (§12.4) |
| **AX200** ✅ *retenue* | **PCIe** | ✅ **validé** | ✅ itlwm + HeliPort, Wi-Fi 6 | ❌ (§12.4) |
| AX210 | PCIe | ✅ probable | ✅ itlwm | ❌ cas d'échec documentés |
| **AX201 / AX211** | CNVio2 | ❌ **impossible** | — | — |
| DW1820A / DW1560 | PCIe Broadcom | ✅ physiquement | ❌ **cassé sous Tahoe** | ✅ probablement |

**Pourquoi AX201 et AX211 sont exclues** : elles utilisent l'interface **CNVio2**, qui exige un chipset 400-series et un CPU de 10e génération minimum. Le Cannon Point-LP (300-series) de cette machine ne parle que le CNVio première génération. C'est une limite électrique — aucune modification de BIOS ne peut la contourner.

> 💡 **Pas de whitelist Wi-Fi chez Dell.** Contrairement à d'autres constructeurs, Dell n'impose aucune liste blanche de cartes — confirmé par leurs propres modérateurs. Le seul obstacle est électrique, jamais logiciel. Inutile d'envisager un BIOS modifié, d'autant que ce modèle applique `Signed Firmware Update`.

## 12.2 Wi-Fi — itlwm + HeliPort (seule option viable sous Tahoe)

1. `itlwm.kext` dans `EFI/OC/Kexts/`
2. Installer **HeliPort** (`github.com/OpenIntelWireless/HeliPort`) dans `/Applications`
3. L'ajouter aux ouvertures automatiques (`Réglages Système → Général → Ouverture`)

**À quoi s'attendre** : le Wi-Fi apparaît comme une interface Ethernet générique (`en0`) et **n'apparaît jamais dans les Réglages Wi-Fi de macOS**. C'est normal, pas un dysfonctionnement — la connexion se pilote exclusivement depuis l'icône HeliPort dans la barre de menus.

> ❌ **AirportItlwm n'est pas une alternative sous Tahoe.** Ce kext frère offrirait un menu Wi-Fi natif, mais il est cassé depuis macOS 15 Sequoia et ne fonctionne pas sur macOS 26. Le dépôt officiel de support AX210 le confirme explicitement, et au moins un utilisateur a endommagé son installation en tentant de le forcer sur Tahoe 26.4. **Ne l'installe pas.**

**Vérification** :
```bash
ifconfig en0                     # doit montrer une inet 192.168.x.x une fois connecté
system_profiler SPNetworkDataType
```

## 12.3 Antennes — branchement Dell

| Câble | Connecteur carte | Repère |
|---|---|---|
| **Blanc** | **MAIN** = **2** | Triangle plein |
| **Noir** | **AUX** = **1** | Triangle vide |

Sur l'AX200NGW, l'étiquette porte les mentions `MAIN 2` et `AUX 1` avec les deux triangles.

> ⚠️ **Vérifie que le connecteur AUX est bien clipsé.** Un retour d'expérience signale qu'avec AUX débranché, le Bluetooth ne se connecte à rien alors que le Wi-Fi reste normal. Ces connecteurs U.FL sont minuscules : appuie à la verticale jusqu'au clic, jamais en biais.

## 12.4 🔴 Bluetooth — non fonctionnel sous Tahoe

**État : non résolu.** Neuf variables indépendantes testées, deux cartes Intel différentes, même résultat. Le diagnostic complet fait l'objet d'un document séparé :

📄 **[BLUETOOTH-INVESTIGATION.md](./BLUETOOTH-INVESTIGATION.md)**

### Résumé du symptôme

Le matériel est correctement énuméré (`0x8087:0x0029` pour l'AX200, `0x8087:0x0AAA` pour la 9560), le kext se charge avec un score de correspondance élevé, mais l'attachement ne se finalise jamais :

```
+-o IntelBluetoothFirmware  <class IntelBluetoothFirmware, !registered, !matched, active>
```

macOS continue d'exposer un profil fantôme `BCM_4350C2` hérité du SMBIOS `MacBookPro16,2`, avec `Address: NULL` et `State: Off`.

### 🔴 `IntelBTPatcher.kext` — à ne PAS activer

Ce kext provoque une **panique noyau et une boucle de redémarrage**, reproduite à l'identique sur les deux cartes testées. C'est un bug connu, documenté sur le dépôt officiel ([issue #486](https://github.com/OpenIntelWireless/IntelBluetoothFirmware/issues/486)) et rapporté sur d'autres puces Intel (AX200, AX210).

Son rôle est marginal — il ne sert qu'à la compatibilité descendante vers d'anciens périphériques Bluetooth 4.x. **Laisse-le désactivé définitivement.**

### Ce qui reste dans la configuration

`IntelBluetoothFirmware.kext` et `BlueToolFixup.kext` sont conservés dans l'EFI, activés. Ils ne causent aucune instabilité et se réactiveront d'eux-mêmes si une future version du kext corrige le problème. Le boot-arg `-ibtcompatbeta` reste également en place.

### ✅ Solution retenue : dongle Bluetooth USB

- Chipset **CSR8510** ou **Broadcom BCM20702**
- 10-15 €, reconnu nativement par macOS, aucun kext ni configuration
- Occupe un port USB-A (deux restent disponibles sur ce modèle)
- Pas de fonctions Continuité — de toute façon indisponibles avec une carte Wi-Fi Intel

---

# 13. PHASE 12 — BATTERIE, LUMINOSITÉ, VEILLE

### 13.1 Batterie

Avec `SMCBatteryManager.kext` + `ECEnabler.kext`, le pourcentage doit s'afficher correctement.

**Vérification** :
```bash
system_profiler SPPowerDataType
```
Contrôle : capacité, nombre de cycles, état de charge cohérents.

**Si le pourcentage est absurde** (0 %, 100 % figé, valeur aberrante) : le contrôleur embarqué expose des champs 16 bits. ECEnabler devrait suffire ; sinon il faut un `SSDT-BATT` sur mesure — le guide « Getting Started With ACPI » de Dortania détaille la procédure de découpage des champs EC.

### 13.2 Luminosité — le correctif complet

Deux ingrédients sont nécessaires pour un rétroéclairage pleinement fonctionnel sur ce modèle, et les deux ont posé problème en pratique.

**1. `_UID` correct dans `SSDT-PNLF.aml`**

SSDTTime génère par défaut `_UID = 0x10` (plage Skylake, `PWMMax = 0x056C`). Sur Whiskey Lake, il faut **`_UID = 0x13`** (plage Coffee Lake et plus récent, `PWMMax = 0xFFFF`). Avec la mauvaise valeur, le curseur de luminosité bouge dans les Réglages Système mais plafonne à quelques pourcents de la pleine puissance, même au maximum — l'échelle utilisée est 47 fois trop petite. Voir le mécanisme détaillé en §5.2.

**2. `backlight-level` dans le bon GUID NVRAM**

Pour forcer une luminosité correcte dès le démarrage (utile pendant l'installation, où l'écran peut sinon rester très sombre) :

Dans `NVRAM → Add → 4D1EDE05-38C7-4A6A-9CC6-4BCCA8B38C14` (variables Apple — **pas** le GUID des boot-args) :

| Clé | Type | Valeur |
|---|---|---|
| `backlight-level` | Data | `FFFF` (pleine échelle, calée sur `_UID = 0x13`) |

Et dans `NVRAM → Delete` du **même GUID**, ajoute `backlight-level` — sans quoi la valeur n'est posée qu'une fois et macOS l'écrase ensuite avec son propre réglage.

**3. Correctifs WhateverGreen complémentaires**

Dans `DeviceProperties → Add → PciRoot(0x0)/Pci(0x2,0x0)` :

| Clé | Type | Valeur |
|---|---|---|
| `enable-backlight-registers-fix` | Data | `01000000` |
| `enable-backlight-smoother` | Data | `01000000` |

Ces deux clés corrigent l'état incohérent que certains firmwares Coffee/Whiskey Lake laissent dans les registres PWM, et lissent les transitions.

**Touches Fn (F11/F12)** : gérées par `BrightnessKeys.kext`. Si le curseur système fonctionne mais pas les touches, il faut remapper les codes ADB via l'application **VoodooPS2 Keyboard Preference Pane**, ou éditer `Info.plist` de `VoodooPS2Keyboard.kext` (section `Custom ADB Map`). Codes utiles : Luminosité + `0x90`, Luminosité − `0x91`.

### 13.3 Veille

**Configuration recommandée** — désactive l'hibernation, qui fonctionne rarement bien sur Hackintosh :

```bash
sudo pmset -a hibernatemode 0
sudo pmset -a autopoweroff 0
sudo pmset -a standby 0
sudo pmset -a proximitywake 0
sudo pmset -a powernap 0
sudo pmset -a tcpkeepalive 0
sudo rm -f /var/vm/sleepimage
sudo touch /var/vm/sleepimage
sudo chflags uchg /var/vm/sleepimage
```

**Diagnostic des réveils intempestifs** :

```bash
pmset -g log | grep -e "Wake from" -e "DarkWake"
```

Causes classiques et remèdes :

| Cause | Remède |
|---|---|
| `Wake on LAN` | Désactiver dans le BIOS (§2.8) |
| Périphérique USB | Cartographie USB incorrecte — repasser les ports internes en type `255` |
| `XHC` / `RP0x` | Ajouter un `SSDT-USBW` ou `SSDT-GPRW` (SSDTTime option 7) |
| Réveil immédiat après fermeture du capot | Vérifier `SSDT-EC-USBX` et la cartographie USB |

### 13.4 Gestion d'alimentation CPU

**Vérification** avec Hackintool (`github.com/benbaker76/Hackintool`) → onglet Power :
- Les fréquences doivent varier de ~400 MHz (repos) à ~3900 MHz (Turbo)
- Si le CPU reste bloqué à une fréquence unique : `SSDT-PLUG` mal chargé, ou SMBIOS inadapté

**Si les fréquences sont mauvaises avec MacBookPro16,2** : c'est le décalage Ice Lake / Whiskey Lake. Deux options :
1. Installer **CPUFriend** + générer un `CPUFriendDataProvider.kext` adapté au i5-8265U
2. Basculer sur `MacBookPro15,2` avec le contournement de board-id

### 13.5 Sortie vidéo externe — HDMI et VGA

**❌ Port VGA/RGB : non réparable, n'y passe pas de temps.** Apple a retiré tout encodeur de sortie analogique VGA des pilotes iGPU Intel depuis plusieurs générations de macOS. Aucun patch ne peut créer un support que le pilote ne contient pas.

**✅ Port HDMI : fonctionnel après patch de connecteur.** Par défaut, `AAPL,ig-platform-id = 00009B3E` ne déclare que des connecteurs DisplayPort (`connector-type = 00040000`) sur les sorties externes — aucun connecteur de type HDMI (`00080000`). Résultat sans patch : détection nulle (`system_profiler SPDisplaysDataType` ne montre qu'un seul écran, l'interne), même écran branché.

**Diagnostic préalable**, une fois macOS installé :

```bash
ioreg -l -w0 | grep -i "connector-type"
```

Si aucune ligne ne retourne `00080000`, le HDMI n'est pas déclaré et le patch ci-dessous est nécessaire.

**Le patch, valeurs confirmées fonctionnelles sur ce modèle** — dans `DeviceProperties → Add → PciRoot(0x0)/Pci(0x2,0x0)` :

| Clé | Type | Valeur | Décodage |
|---|---|---|---|
| `framebuffer-con1-enable` | Data | `01000000` | Active le connecteur d'index 1 |
| `framebuffer-con1-alldata` | Data | `01 01 0900 00080000 87010000` | Index `01`, **Bus ID `01`**, Pipe `09`, type **HDMI** (`00080000`), flags par défaut |

> 🔴 **Le Bus ID (`01` ici) est spécifique à chaque modèle de carte mère et ne se devine pas.** Sur ce Latitude 3500, la valeur qui fonctionne est **`01`** — trouvée par itération après avoir testé successivement `05`, `04`, `06`, `02`. Sur une autre machine, même très proche, cette valeur peut différer.
>
> **Symptôme d'un Bus ID incorrect** : ralentissement de quelques secondes au branchement (macOS tente de lire l'EDID sur le mauvais canal DDC, échoue, réessaie, abandonne), sans qu'aucun écran externe n'apparaisse. C'est le signe que le connecteur est presque bon — il ne reste qu'à changer cet octet.
>
> **Méthode d'itération si tu dois la refaire sur une machine différente** : modifie uniquement le second octet de `framebuffer-con1-alldata` (`01 XX 0900...`), teste dans l'ordre `01, 02, 04, 05, 06`, et vérifie après chaque redémarrage :
> ```bash
> system_profiler SPDisplaysDataType
> ```
> Un second bloc `Display:` avec `Connection Type: HDMI` confirme la bonne valeur.

**Ne touche pas à `con0`** (écran interne) pendant cette manipulation — il n'est pas concerné par ce patch et continue de fonctionner normalement pendant toute l'itération.

**Bonus observé** : après ce patch, l'écran interne est passé de 24 bits à 30 bits de profondeur de couleur (`ARGB2101010`) — bénéfice collatéral, sans action supplémentaire.

**Sortie DisplayPort par l'USB-C** — non testée sur cette machine faute d'adaptateur disponible, mais la méthode serait identique : patcher `framebuffer-con2-alldata` avec un type `00040000` (DP) et itérer sur le Bus ID de la même façon.

---

# 14. PHASE 13 — iSERVICES, OTA, FINITIONS

### 14.1 Correction des iServices (iMessage, FaceTime, App Store)

**Prérequis : l'Ethernet doit être `en0` et marqué « intégré ».**

```bash
# Vérifier l'interface
ifconfig en0

# Vérifier le statut "built-in"
system_profiler SPNetworkDataType | grep -A5 "Ethernet"
```

**Si l'Ethernet n'est pas `en0`** :
```bash
sudo rm /Library/Preferences/SystemConfiguration/NetworkInterfaces.plist
sudo rm /Library/Preferences/SystemConfiguration/preferences.plist
```
Redémarre, puis reconfigure les interfaces dans `Réglages Système → Réseau` en ajoutant **d'abord** l'Ethernet.

**Renseigner le ROM** : ✅ déjà fait en §7.7 avec `E454E83F4A73`.

Vérifie simplement que macOS voit bien la même adresse :
```bash
ifconfig en0 | grep ether
```
Tu dois lire `e4:54:e8:3f:4a:73`. Si l'adresse diffère, c'est que le MAC Pass-Through est resté actif dans le BIOS (§2.9).

**Ordre de connexion** :
1. Redémarre
2. Connecte-toi à l'App Store en premier
3. Puis à iMessage/FaceTime

> Si Apple demande un appel au support à la première connexion, c'est généralement que le SMBIOS ou le ROM ne sont pas cohérents. Régénère un nouveau numéro de série et recommence.

### 14.2 Vérification du SMBIOS

```bash
# Modèle détecté
sysctl hw.model

# Numéro de série
system_profiler SPHardwareDataType | grep Serial
```

### 14.3 Mises à jour OTA

Sous Tahoe, les mises à jour via `Réglages Système → Mise à jour de logiciels` demandent trois conditions :

1. `RestrictEvents.kext` présent dans `EFI/OC/Kexts/`
2. `boot-args` contenant `revpatch=sbvmm`
3. `Misc → Security → SecureBootModel = Disabled`

> Si tu préfères garder le Secure Boot Apple (`SecureBootModel = Default`), ajoute `iBridged.kext` (github.com/Carnations-Botanica/iBridged) à la place.

**Procédure de mise à jour recommandée** :
1. Sauvegarde ton EFI complet sur une clé USB avant chaque mise à jour
2. Vérifie sur `r/hackintosh` ou le dépôt d'OpenIntelWireless si des kexts doivent être mis à jour
3. Vérifie les versions d'`itlwm` et des kexts Bluetooth avant la mise à jour système
4. Lance la mise à jour

### 14.4 TRIM

Le TRIM est actif par défaut sur NVMe avec APFS. Vérification :
```bash
system_profiler SPNVMeDataType | grep -i trim
```
`TRIM Support: Yes` attendu. La commande `trimforce` ne concerne que les SSD SATA.

### 14.5 Dual-boot Windows

OpenCore détecte automatiquement le Windows Boot Manager si Windows est en UEFI sur le même disque ou un autre.

**Si Windows n'apparaît pas** : vérifie `Misc → Security → ScanPolicy = 0`.

**Pour que le BIOS Dell ne remette pas Windows en premier** : `UEFI → Quirks → RequestBootVarRouting = YES` (déjà configuré §7.8).

> ⚠️ Windows peut réécrire la NVRAM lors de ses grosses mises à jour et reprendre la priorité au boot. Il suffit alors de reprioriser dans le BIOS (F2 → Boot Sequence).

### 14.6 Sauvegarde de l'EFI

Une fois tout fonctionnel :

```bash
sudo diskutil mount disk0s1
cp -R /Volumes/EFI/EFI ~/Desktop/EFI-BACKUP-$(date +%Y%m%d)
```

Archive cette copie ailleurs que sur la machine. C'est ton filet de sécurité pour toutes les mises à jour futures.

---

# 15. DÉPANNAGE — PANNES CLASSIQUES

### `OC: Driver HfsPlus.efi at 1 cannot be loaded - Unsupported!`
Le fichier `HfsPlus.efi` est corrompu ou n'est pas le bon binaire — le cas le plus fréquent est un téléchargement GitHub qui a récupéré la page HTML au lieu du fichier brut (clic droit → Enregistrer sous sur la page, plutôt que le bouton de téléchargement). Un `HfsPlus.efi` légitime pèse environ 40 Ko.
→ Solution rapide : utilise `OpenHfsPlus.efi` (déjà présent dans `X64/EFI/OC/Drivers/` de l'archive OpenCore) à la place. Renomme le chemin dans `UEFI → Drivers`.
→ Solution propre : retélécharge `HfsPlus.efi` depuis `github.com/acidanthera/OcBinaryData` en cliquant sur **Code → Download ZIP**, jamais un clic droit direct sur un fichier de la page.
→ Ce driver est indispensable si ta clé d'installation est en HFS+ (cas de `createinstallmedia`) — sans lui, l'installateur n'apparaît pas dans le menu OpenCore.

### Redémarrages répétés pendant l'installation macOS (5 fois ou plus)
Deux causes bien distinctes à ne pas confondre :

**A — Comportement normal de l'installateur (3 à 5 cycles)** : chaque redémarrage correspond à une étape différente (préparation, copie, application firmware, scellement APFS). Reste devant la machine et resélectionne l'entrée `MacintoshHD` à chaque fois — ou passe `Misc → Boot → Timeout` à une valeur non nulle (`10` secondes recommandé) pour que le picker relance automatiquement sans intervention.

**B — Vraie boucle infinie (5+ cycles, aucune progression)** : signature d'un disque reformaté en **4K natif** (`nvme format -l 1`). Le verbeux montre `handle_mount: setting dev block size to 4096 from 512` suivi de `disallowing unsupported code signature page shift: 14` puis `revert_to_snapshot`, en boucle, sans jamais avancer.
→ **Reformate le disque en 512e** (`nvme format -l 0`) depuis le live Linux, puis relance l'installation depuis zéro (§0.5 bis). C'est la cause la plus probable si tu as suivi une recommandation de formatage 4K trouvée ailleurs.

### Blocage sur `EB|LOG:EXITBS:START`
Firmware sans table `MEMORY_ATTRIBUTE_TABLE`.
→ `Booter → Quirks` : `RebuildAppleMemoryMap = NO`, `EnableWriteUnprotector = YES`

### Panique noyau sur `AppleIntelCFLGraphicsFramebuffer`
DVMT insuffisant.
→ Vérifie les trois patchs `framebuffer-*` (§7.3). C'est **la** panne numéro un sur ce BIOS Dell.

### Fond d'écran uni, 7 Mo de VRAM, interface saccadée
Pas d'accélération graphique.
→ `device-id = 9B3E0000` manquant ou mauvais `AAPL,ig-platform-id`. Reprends §7.3 exactement.

### Aucun disque visible dans l'Utilitaire de disque
→ SATA en mode RAID dans le BIOS (§2.2). Ou `ScanPolicy` ≠ 0.

### La clé USB n'apparaît pas dans le menu OpenCore
→ `Misc → Security → ScanPolicy = 0`

### `OCB: LoadImage failed - Security Violation`
→ Mélange de versions DEBUG/RELEASE entre `OpenCore.efi`, `BOOTx64.efi` et `OpenRuntime.efi`. Les trois doivent venir de la même archive.

### Blocage sur `[EB|#LOG:EXITBS:END]` puis écran noir
→ Ajoute `-igfxnotelemetryload` aux `boot-args`

### `Still waiting for root device`
→ Cartographie USB absente ou incorrecte. Essaie temporairement en branchant la clé sur un port USB 2.0.

### Le BIOS se réinitialise après chaque redémarrage
→ `Kernel → Quirks → DisableRtcChecksum = YES` (fréquent sur Dell)

### Trackpad (`DELL08BD`) détecté mais totalement inerte
C'est le symptôme normal d'un oubli de patch — pas d'une absence de device : `_STA` retourne toujours vrai pour ce trackpad, donc il apparaît dans IORegistry même sans le bon patch.
→ Vérifie `SSDT-XOSI.aml` **et** le patch de renommage `_OSI→XOSI` dans `ACPI → Patch` (§5.2/§5.3). Sans le patch, `_CRS` retourne une interruption nue, sans ressource de bus I2C.
→ Vérifie que **les trois plugins VoodooI2C** (`VoodooI2CServices`, `VoodooGPIO`, `VoodooInput`) sont chargés avant `VoodooI2C.kext` — voir l'encadré de §3.3. C'est la cause la plus fréquente une fois XOSI en place.
→ Vérifie qu'aucun `VoodooPS2Trackpad`/`VoodooPS2Mouse` n'est actif en parallèle, et que le `VoodooInput` de `VoodooPS2Controller` est désactivé (doublon avec celui de VoodooI2C).

### Bluetooth absent ou inactif
Vérifie d'abord `-ibtcompatbeta` dans les `boot-args` — obligatoire sous macOS 26.
→ **Si le boot-arg est présent et que ça ne marche toujours pas** : c'est le problème non résolu documenté en §12.4. Symptôme caractéristique : `ioreg` montre `IntelBluetoothFirmware ... !registered, !matched` et `system_profiler` affiche un contrôleur fantôme `BCM_4350C2` avec `Address: NULL`. Diagnostic complet dans `BLUETOOTH-INVESTIGATION.md`. Solution : dongle USB.

### Boucle de redémarrage après activation des kexts Bluetooth
→ `IntelBTPatcher.kext` est actif. **Désactive-le définitivement** — bug connu provoquant une panique noyau, reproduit sur AC9560 et AX200 ([issue #486](https://github.com/OpenIntelWireless/IntelBluetoothFirmware/issues/486)). Garde uniquement `IntelBluetoothFirmware.kext` et `BlueToolFixup.kext`.

### Wi-Fi absent après une mise à jour de macOS
→ Si tu utilises `AirportItlwm`, il est cassé sous Tahoe. Bascule sur `itlwm.kext` + HeliPort (§12.2). Ne jamais avoir les deux kexts actifs simultanément.

### Ventilateurs à fond en permanence
→ C-States désactivés dans le BIOS (§2.7), ou `SSDT-PLUG` non chargé.

### Réveil immédiat après mise en veille
→ Wake on LAN/WLAN actif dans le BIOS. Puis vérifie la cartographie USB.

### Redémarrage au lieu de l'extinction
→ Vérifie `SSDT-EC-USBX` et la cartographie USB. Un `SSDT-USBW` peut être nécessaire.

### Luminosité qui plafonne bas même au maximum
→ `SSDT-PNLF.aml` généré avec `_UID = 0x10` (Skylake) au lieu de `_UID = 0x13` (Coffee Lake/Whiskey Lake). Voir §5.2 pour le mécanisme exact et la correction.
→ Vérifie aussi que `backlight-level` est placé dans le bon GUID NVRAM : `4D1EDE05-38C7-4A6A-9CC6-4BCCA8B38C14` (variables Apple), **pas** `7C436110-AB2A-4BBB-A880-FE41995C9F82` (boot-args). Dans le mauvais GUID, la valeur est silencieusement ignorée.

### Aucune sortie HDMI détectée (pas d'écran externe, même en `system_profiler`)
Le framebuffer `00009B3E` déclare ses connecteurs externes en DisplayPort par défaut, pas en HDMI, et les laisse désactivés.
→ Patch de connecteurs requis : `framebuffer-con1-enable` + `framebuffer-con1-alldata` avec le type HDMI (`00080000`). Voir §13.5 pour la procédure complète et les valeurs qui fonctionnent sur ce modèle précis.

### Pas de sortie vidéo par l'USB-C
Le framebuffer `00009B3E` expose un jeu de connecteurs figé, calibré pour l'écran interne et le HDMI. La sortie DisplayPort Alt Mode de l'USB-C peut tomber sur un index de connecteur non déclaré.
→ Sujet de patching de connecteurs WhateverGreen (`framebuffer-conX-type`, `framebuffer-conX-index`). Voir §13.5, méthode identique à celle utilisée pour le HDMI.
→ La recharge USB-C, elle, n'est pas concernée : elle est gérée par le contrôleur embarqué, indépendamment du système.

### Le port VGA/RGB ne fonctionne pas et ne fonctionnera jamais
❌ **Non réparable.** Apple a retiré tout support de sortie analogique VGA des pilotes iGPU Intel depuis plusieurs générations de macOS — il n'existe aucun encodeur VGA dans la chaîne graphique. Aucun patch DeviceProperties ne peut créer un support qui n'existe pas dans le pilote. N'investis pas de temps là-dessus.

### Ralentissement de quelques secondes au branchement d'un écran externe, puis stabilisation
Signe d'un **Bus ID incorrect** dans le patch de connecteur (`framebuffer-conX-alldata`). macOS détecte le branchement (HPD fonctionne) mais lit l'EDID sur le mauvais canal DDC, réessaie en boucle quelques secondes, puis abandonne.
→ Le connecteur est presque bon — il ne reste qu'à ajuster l'octet Bus ID. Voir §13.5 pour la méthode d'itération.

### Comment lire une panique noyau
Avec `ApplePanic = YES` et `debug=0x100`, l'écran reste affiché. Photographie-le : la ligne importante est celle qui suit `Backtrace` et mentionne un nom de kext.

---

# 16. CHECKLIST FINALE

**Préalable matériel**
- [x] ~~Type de trackpad~~ → **I2C, `DELL08BD`** confirmé (§B.3)
- [x] ~~BIOS à jour~~ → 1.36.0, dernière version publiée
- [x] ~~Format du disque~~ → **512e** (jamais 4K, §0.5 bis)
- [x] ~~Type du port 4/16~~ → USB-C de recharge, type `9` confirmé (§10.2)
- [x] ~~Lecteur SD~~ → activé dans le BIOS, trouvé sur le port USB 7
- [x] ~~Sélection USB~~ → 10 ports cochés et typés, validés
- [x] ~~`UTBMap.kext`~~ → généré et copié dans l'EFI

**BIOS — conforme (§2.0 bis)**
- [x] SATA/NVMe Operation = `Ahci`
- [x] Secure Boot = `Disabled`
- [x] Intel SGX = `Disabled`
- [x] SpeedStep / C-States / Turbo / Hyper-Threading = `Enabled`
- [x] Wake on LAN / MAC Pass-Through / Free Fall / Computrace = `Disabled`
- [x] Fastboot = `Thorough`
- [x] `VtForDirectIo` = `Disabled` (`DisableIoMapper = YES` gardé malgré tout)
- [x] `UefiBootPathSecurity` = `Never`
- [x] `WakeOnDock` = `Disabled`
- [x] `MediaCard` / `SdCard` = `Enabled`
- [x] `EmbNic1` / `UefiNwStack` : PXE et pile réseau coupés
- [x] `UsbWake` = `Disabled`
- [x] TPM / PTT laissés activés (neutres pour macOS)

**config.plist — construit et validé**
- [x] `CustomSMBIOSGuid = YES`, `UpdateSMBIOSMode = Custom`
- [x] `AppleXcpmCfgLock = YES` (pas d'option CFG Lock dans ce BIOS)
- [x] `DisableRtcChecksum = YES`, `RequestBootVarRouting = YES`
- [x] `AAPL,ig-platform-id = 00009B3E`, `device-id = 9B3E0000`
- [x] Trois patchs framebuffer (`patch-enable`, `stolenmem`, `fbmem`)
- [x] SMBIOS = `MacBookPro16,2`, ROM = `E454E83F4A73`
- [x] `boot-args` contient `-ibtcompatbeta`
- [x] `SecureBootModel = Disabled`
- [x] Pas d'`AppleALC.kext`
- [x] Six SSDT (`EC`, `PLUG`, `RTCAWAC`, `XOSI`, `PNLF` avec `_UID=0x13`, `SBUS-MCHC`)
- [x] Quatre patches ACPI (`OSID→XSID`, `_OSI→XOSI`, `ECDV _STA→XSTA`, `PNLF→XNLF`)
- [x] Trois plugins VoodooI2C (`Services`, `GPIO`, `Input`) avant `VoodooI2C.kext`
- [x] `Misc → Boot → Timeout = 10` (pas `0`, pas `5`)
- [x] `backlight-level = FFFF` dans le bon GUID Apple (`4D1EDE05-...`)
- [x] `ocvalidate` → 0 problème

**Système — validé après installation**
- [x] macOS Tahoe installé et démarre depuis le disque interne
- [x] Accélération graphique (1536 Mo VRAM, Metal 3)
- [x] Trackpad I2C fonctionnel
- [x] Luminosité réglable sur toute la plage (curseur + touches Fn)
- [x] Sortie HDMI fonctionnelle (Bus ID `01` sur ce modèle — §13.5)
- [x] Fréquences CPU variables (à vérifier avec Hackintool)
- [x] ~~Wi-Fi~~ → **AX200 installée**, itlwm + HeliPort fonctionnels
- [x] ~~Bluetooth~~ → **non résoluble sous Tahoe** (§12.4), dongle USB retenu
- [x] ~~Audio~~ → **AppleALC + AppleHDA réinjecté, layout-id 19** — confirmé fonctionnel (§11.1)
- [ ] Batterie : pourcentage à vérifier (`system_profiler SPPowerDataType`)
- [ ] Veille/réveil à tester
- [ ] Ethernet = en0 + iMessage/FaceTime à valider
- [ ] Bascule OpenCore DEBUG → RELEASE une fois tout stable
- [ ] **Sauvegarde de l'EFI archivée hors machine**

❌ **Ne fonctionnera pas sur cette plateforme**
- **Port VGA/RGB** — aucun encodeur analogique dans les pilotes iGPU Intel
- **Bluetooth Intel sous Tahoe** — voir §12.4 et `BLUETOOTH-INVESTIGATION.md`
- **Continuité Apple** (AirDrop, Handoff) — carte Wi-Fi Intel, et le Broadcom qui la permettrait a son Wi-Fi cassé sous Tahoe
- **AX201 / AX211** — interface CNVio2, incompatible avec le chipset 300-series

---

# ANNEXE A — RAPPORT MATÉRIEL

➡️ Cette section a été déplacée dans un document dédié : **[HARDWARE-AUDIT.md](HARDWARE-AUDIT.md)** — méthodologie générique de collecte des relevés matériels, réutilisable sur toute machine.

---

# ANNEXE B — ANALYSE DES RELEVÉS FOURNIS

Sources : `dsdt.dsl` (BIOS 1.36.0), `bios-dump-ok.txt` (cctk), `MDT-C04F505432.LOG` / `.HTM` (HWiNFO), `usb.json` (USBToolBox).

## B.1 Verdict global

**La machine est un très bon candidat.** Aucune anomalie bloquante, plusieurs bonnes surprises, et le point le plus incertain (le trackpad) est tranché. Trois éléments méritaient d'être découverts avant l'installation plutôt qu'après — ils sont détaillés en B.3.

## B.2 Cartographie PCI confirmée

| Fonction | ID PCI | Sous-système | Conséquence |
|---|---|---|---|
| Host bridge | `8086:3E34` | `08BD:1028` | Whiskey Lake-U |
| **iGPU UHD 620** | **`8086:3EA0`** | `08BD:1028` | 🔴 Confirme le spoof `device-id = 9B3E0000` |
| Audio cAVS | `8086:9DC8` | `08BD:1028` | Cannon Lake-LP. Piloté par Intel SST sous Windows |
| **Wi-Fi CNVi** | **`8086:9DF0`** | **`4234:8086`** | 🔴 CNVi confirmé — voir B.4 |
| xHCI USB 3.1 | `8086:9DED` | `08BD:1028` | Contrôleur unique |
| **I2C #0** | `8086:9DE8` | `08BD:1028` | Bus candidat trackpad |
| **I2C #1** | `8086:9DE9` | `08BD:1028` | Bus candidat trackpad |
| Ethernet | `10EC:8168` rev 15 | `08BD:1028` | `RealtekRTL8111.kext` |
| GPIO | `INT34BB` | — | Contrôleur Serial IO GPIO |
| NVMe | KXG60ZNV512G | — | PCIe **x4 8.0 GT/s** = Gen3 x4 pleinement exploité, firmware 10604106 |
| Lecteur SD | `USB2.0-CRW` | port USB 7 | 🆕 Apparu après `MediaCard=Enabled` |

**Aucun périphérique Nvidia.** Le DSDT déclare bien `PEG0`/`PEGP`, mais c'est une déclaration de principe présente sur tous les Latitude — le matériel n'est pas là. Pas de SSDT de désactivation.

## B.3 Les trois découvertes qui changent la configuration

### 1. Le trackpad : `DELL08BD` sur I2C0

`trackpad.txt` donne la réponse : `InstanceId : ACPI\DELL08BD` pour le « Périphérique I2C HID ».

Ce n'est aucun des candidats évidents du DSDT (`SYNA2393`, `DLL077A`, `ATML3432`, `WCOM4831`). C'est le `TPD0` de l'I2C0, dont le `_HID` est une **méthode** qui retourne la variable `ITPN = "DELL08BD"` — un identifiant Dell propriétaire construit sur le `SysId` de la machine (`08BD`).

| Paramètre | Valeur |
|---|---|
| Chemin ACPI | `\_SB.PCI0.I2C0.TPD0` |
| `_HID` | `DELL08BD` (méthode, pas constante) |
| `_CID` | `PNP0C50` — HID sur bus I2C |
| Bus | I2C0 (`8086:9DE8`) |
| Adresse | `0x2C` |
| Vitesse | 400 kHz |
| Interruption | GpioInt via `\_SB.PCI0.GPI0` |
| `_STA` | `0x0F` en permanence |

**Conséquence majeure** : `_STA` ne dépend d'aucune variable — le trackpad est toujours visible. Mais `_CRS` prive le périphérique de sa ressource de bus I2C si `OSYS < 0x07DC`. XOSI reste donc obligatoire, mais le symptôme d'un oubli est « détecté mais inerte », pas « absent ». Détail complet en §5.2.

### 2. L'EC s'appelle `ECDV` et ses champs sont en 8 bits

Deux conséquences favorables : SSDTTime crée un faux device `EC` sans avoir à renommer le device réel, et **`ECEnabler.kext` devient inutile**. Voir §5.3.

### 3. Le BIOS 1.36.0 est configuré à 95 %

Après flash et réglages, il ne reste qu'**une** correction : `UsbWake=Enabled` doit repasser à `Disabled` (le flash l'a remis par défaut). Tout le reste est conforme. Détail en §2.0 bis.

`VtForDirectIo` est maintenant `Disabled` — mais garde `DisableIoMapper = YES` par sécurité.

### 4. La carte USB est complète

10 ports sélectionnés et typés, appairages cohérents. Voir §B.5.

### 5. Deux périphériques I2C annexes repérés

| Device | `_HID` | Statut sous macOS |
|---|---|---|
| `A_CC` | `SMO8810` | Accéléromètre STMicroelectronics (détection de chute Dell). Non supporté, mais **sans `_CID PNP0C50`** donc VoodooI2CHID ne tentera pas de s'y attacher. Inoffensif, rien à faire |
| Dalle | `BOE0802` | Écran BOE. Sans conséquence, mais utile à connaître si la luminosité se comporte mal |

## B.4 Wi-Fi — correction d'une conclusion erronée

Le Wi-Fi d'origine portait le sous-système **`4234:8086`** — un identifiant Intel, signature d'un montage CNVi. Une version antérieure de ce document en concluait que « le slot est câblé CNVi, une carte Broadcom PCIe n'y fonctionnera pas ».

⚠️ **Cette déduction était fausse.** L'identifiant prouve que la carte *installée* était du CNVi, pas que le connecteur soit dépourvu de lignes PCIe.

**Vérifié en pratique** : une **AX200NGW**, carte PCIe, fonctionne dans ce slot. Le connecteur M.2 A/E porte les deux jeux de signaux, comme c'est le cas sur la plupart des plateformes Intel.

Détail complet des cartes compatibles en §12.1.

## B.5 USB — cartographie complète et validée

`usb.json` final : **10 ports sélectionnés**, tous typés, sur le contrôleur unique `\_SB.PCI0.XHC`.

| Ensemble | Ports logiques | Type | Nature |
|---|---|---|---|
| USB-A 3.x n°1 | 1 (HS) + 13 (SS) | `3` | Externe |
| USB-A 3.x n°2 | 3 (HS) + 15 (SS) | `3` | Externe |
| USB-C de recharge | 4 (HS) + 16 (SS) | `9` | Externe |
| USB-A 2.0 | 2 (sans compagnon) | `0` | Externe |
| Webcam | 6 | `255` | Interne |
| Lecteur SD | 7 | `255` | Interne |
| Bluetooth Intel | 10 | `255` | Interne |

Appairages HS↔SS cohérents (`1↔13`, `3↔15`, `4↔16`), aucun port manquant ni superflu, 10 < 15 donc `XhciPortLimit = NO`.

Ports internes inutilisés : 5, 8, 9, 11, 12, 14, 17, 18.

## B.6 Exports — état final

### ✅ Collecte terminée

Tous les relevés nécessaires sont fournis et validés. Il ne reste qu'à générer `UTBMap.kext` depuis `usb.json` (§10.3), ce qui est une opération et non un export.

### ⚪ Optionnel, depuis le live Linux, le moment venu

| Quoi | Commande |
|---|---|
| ~~Codec audio réel derrière ALC3271~~ | ✅ **ALC3204 confirmé** |
| Formats LBA du SSD, avant reformatage | `sudo nvme id-ns -H /dev/nvme0n1 \| grep -i "lba format"` |
| Tables SSDT complètes pour SSDTTime | `acpidump.exe -b` (en administrateur) |

Aucun des trois ne conditionne la construction de l'EFI.

### ✅ Définitivement réglé

| Question | Réponse | Source |
|---|---|---|
| GPU Nvidia MX130 | Absent | HWiNFO PCI |
| Modèle du trackpad | **`DELL08BD`**, I2C0, adresse `0x2C` | `trackpad.txt` + DSDT |
| Rôle de XOSI | Obligatoire — sinon perte de la ressource de bus I2C | DSDT `_CRS` |
| Option DVMT dans le BIOS | Inexistante, y compris en 1.36.0 | cctk |
| Option CFG Lock | Inexistante | cctk |
| Slot M.2 A/E câblé PCIe ? | Non — CNVi (`4234:8086`) | HWiNFO PCI |
| MAC Ethernet | `E4-54-E8-3F-4A-73` | BIOS |
| Champs EC batterie | Tous en 8 bits, `ECEnabler` inutile | DSDT |
| Nom du contrôleur EC | `ECDV` | DSDT |
| AWAC / RTC | Les deux présents → SSDT-AWAC | DSDT |
| Nature du port USB 4/16 | USB-C de recharge, type `9` | Vérification physique |
| Lecteur SD | Port USB 7, interne | `usb.json` |
| Réglages BIOS | Tous conformes, `UsbWake` corrigé | `bios-dump-ok.txt` |
| Version BIOS | 1.36.0, la plus récente | cctk |

## B.7 Récapitulatif — configuration figée (système installé et fonctionnel)

```
SMBIOS ................ MacBookPro16,2
ROM ................... E454E83F4A73
device-id iGPU ........ 9B3E0000
AAPL,ig-platform-id ... 00009B3E
framebuffer-stolenmem . 00003001      (DVMT 32 Mo confirmé)
framebuffer-fbmem ..... 00009000

Disque ................ 512e — JAMAIS 4K (le 4K casse l'install, §0.5 bis)

SSDT ................... EC, PLUG, RTCAWAC, XOSI, PNLF (_UID=0x13 !), SBUS-MCHC
Patch ACPI ............ 4 renommages : OSID→XSID, _OSI→XOSI, ECDV _STA→XSTA, PNLF→XNLF

Quirks Dell ........... CustomSMBIOSGuid = YES
                        UpdateSMBIOSMode = Custom
                        DisableRtcChecksum = YES
Quirks plateforme ..... AppleXcpmCfgLock = YES   (pas d'option BIOS)
                        DisableIoMapper = YES    (VT-d actif)
                        XhciPortLimit = NO

Wi-Fi ................. Intel AX200NGW (PCIe) - remplace l'AC9560 CNVi
                        itlwm + HeliPort (JAMAIS AirportItlwm sous Tahoe)
Bluetooth ............. ❌ non fonctionnel sous Tahoe (voir §12.4)
                        IntelBTPatcher.kext = panique noyau, NE PAS ACTIVER
                        → dongle USB CSR8510 / BCM20702

Trackpad .............. DELL08BD @ \_SB.PCI0.I2C0.TPD0
                        adresse I2C 0x2C, 400 kHz, IRQ via GPI0
Kexts entrée .......... VoodooPS2Controller (clavier seul)
                        VoodooI2CServices + VoodooGPIO + VoodooInput
                        → puis VoodooI2C + VoodooI2CHID (ordre impératif)
                        → VoodooPS2Trackpad, VoodooPS2Mouse et le
                          VoodooInput de VoodooPS2Controller désactivés

Luminosité ............ SSDT-PNLF _UID = 0x13 (PAS 0x10 !)
                        backlight-level = FFFF dans GUID
                        4D1EDE05-38C7-4A6A-9CC6-4BCCA8B38C14
                        (PAS le GUID des boot-args)
                        enable-backlight-registers-fix = 01000000
                        enable-backlight-smoother = 01000000

HDMI .................. framebuffer-con1-enable = 01000000
                        framebuffer-con1-alldata =
                          01 01 0900 00080000 87010000
                        Bus ID 01 (spécifique à ce modèle)
VGA/RGB ............... ❌ jamais fonctionnel sous macOS

USB ................... contrôleur \_SB.PCI0.XHC, 10 ports retenus ✅
                        1+13 / 3+15 : USB-A 3.x    (type 3)
                        4+16        : USB-C PD     (type 9)
                        2           : USB-A 2.0    (type 0)
                        6           : webcam       (type 255)
                        7           : lecteur SD   (type 255)
                        10          : Bluetooth    (type 255)
                        XhciPortLimit = NO (10 < 15)

Misc.Boot.Timeout ..... 10 (jamais 0 pendant l'install multi-reboot)

Audio .................. Codec réel : ALC236 = ALC3204 (Dell dit "ALC3271")
                        ✅ AppleALC + AppleHDA réinjecté (MyKextInstaller)
                        layout-id = 19 (Data: 13000000) — CONFIRMÉ fonctionnel
                        csr-active-config = 03080000
                        AppleHDA extrait d'un KDK beta 1 ou d'un Sequoia
                        (absent des KDK Tahoe ≥ beta 2)
                        🔴 Choisir "Restart Now" jamais "Later" dans l'installeur
                        Repli : VoodooHDA (csr-active-config = 03000000)

Non requis ............ ECEnabler, SMCLightSensor, SSDT-dGPU-Off,
                        SSDT-BATT, SSDT-GPI0, AirportBrcmFixup
```

---

## Ressources

| Ressource | URL |
|---|---|
| Guide Dortania (référence) | dortania.github.io/OpenCore-Install-Guide |
| Prérequis macOS 26 Tahoe | dortania.github.io/OpenCore-Install-Guide/extras/tahoe.html |
| Getting Started With ACPI | dortania.github.io/Getting-Started-With-ACPI |
| Post-Install | dortania.github.io/OpenCore-Post-Install |
| OpenIntelWireless | github.com/OpenIntelWireless |
| Subreddit | reddit.com/r/hackintosh |

---

*Document établi pour Dell Latitude 3500 (BIOS 1.36.0) / i5-8365U Whiskey Lake-U / UHD 620 / 16 Go / Kioxia KXG60ZNV512G / macOS Tahoe 26 / OpenCore 1.0.7. Spécifications validées à partir des relevés BIOS de la machine. Vérifie toujours les dernières versions des kexts avant de commencer — l'écosystème bouge encore, même en fin de vie.*
