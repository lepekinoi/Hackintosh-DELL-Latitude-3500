# Audit matériel — collecte des relevés

Méthodologie de collecte des informations matérielles nécessaires à la construction d'un EFI OpenCore. Cette procédure est **générique** : elle s'applique à n'importe quelle machine, pas seulement au Latitude 3500.

Les résultats obtenus sur cette machine précise sont analysés dans l'Annexe B du [guide de construction](BUILD-GUIDE.md).

---


Objectif : produire un dossier de relevés qui fige la réalité technique de la machine, pour ne plus travailler par hypothèses.

## A.1 La pile d'outils

**Les trois indispensables :**

| Outil | Licence | Ce qu'il apporte |
|---|---|---|
| **HWiNFO64** (portable) | Gratuit | 🔴 **Le pilier.** Rapport exhaustif : tous les périphériques PCI avec VEN/DEV/SUBSYS, contrôleurs I2C, SMBIOS, SPD des barrettes, capteurs, NVMe. Export TXT/HTML/CSV |
| **acpidump** (paquet ACPICA) | Gratuit | 🔴 Dump des **tables ACPI** (DSDT, SSDT, FACP, DMAR…) sans installer de pilote. L'archive contient aussi `iasl` pour les décompiler |
| **USBToolBox** | Gratuit | Cartographie des ports USB (§10) |

**Les compléments utiles :**

| Outil | Pourquoi |
|---|---|
| **AIDA64 Extreme** (essai 30 j) | Rapport le plus lisible du marché, avec un assistant qui génère un HTML complet en un clic. Redondant avec HWiNFO mais plus agréable à relire |
| ~~**RWEverything**~~ | ❌ Son pilote est bloqué sur Windows 11 (liste des pilotes vulnérables Microsoft). Remplacé par `acpidump` |
| **PCI-Z** | Identifie les périphériques PCI inconnus par leurs VEN/DEV quand Windows n'a pas le pilote |
| **USBDeview** (NirSoft) | Liste tous les périphériques USB jamais branchés, avec leur VID/PID |
| **Dell Command \| Configure** | Spécifique Dell — exporte **toute** la configuration BIOS, y compris des options non affichées dans l'interface F2. Voir §A.5 |
| **SSDTTime** | Génère les SSDT à partir des tables dumpées (§5) |

> **Pourquoi HWiNFO plutôt que CPU-Z ou Speccy** : ces derniers donnent un aperçu, pas un inventaire. HWiNFO descend au niveau des identifiants PCI et des contrôleurs I2C — exactement ce dont on a besoin pour choisir les kexts.

## A.2 Procédure de collecte

Crée un dossier `C:\Hack\Rapport\` et travaille dedans.

**1 — HWiNFO64**

> ⚠️ **Piège au lancement.** Sur l'écran d'accueil, **décoche à la fois « Sensors-only » et « Summary-only »**. Si l'une des deux reste cochée, l'inventaire matériel complet n'est pas construit et **le bouton de rapport n'existe pas**. C'est la cause n°1 de « je ne trouve pas l'option ».

1. Clic droit sur `HWiNFO64.exe` → **Exécuter en tant qu'administrateur**
2. Écran d'accueil : **toutes les cases décochées** → `Start`
3. Le scan tourne, puis une fenêtre **« System Summary »** s'ouvre par-dessus. **Ferme-la** — c'est elle qui masque la fenêtre principale
4. Tu arrives sur la fenêtre principale : arborescence des composants à gauche, détails à droite
5. Clique sur le bouton **« Save Report »** dans la barre d'outils en haut *(ce n'est pas un menu déroulant)*
6. L'assistant demande successivement :
   - le **format** : MHTML, HTML, CSV, TXT ou CDF
   - le **nom et l'emplacement** du fichier
   - les **catégories à inclure** — développe les branches avec le `+` et coche tout
7. `Finish`

Fais l'opération **deux fois** : une en `.html` (lisible, pour parcourir) et une en `.txt` (cherchable au Ctrl+F). Enregistre sous `Rapport\hwinfo-full.html` et `Rapport\hwinfo-full.txt`.

> Si l'écran d'accueil ne s'affiche pas du tout (il peut avoir été désactivé), ouvre `HWiNFO64.INI` à côté de l'exécutable et vérifie que `SensorsOnly=0` et `SummaryOnly=0`.

**2 — Tables ACPI**

> ⚠️ **RWEverything ne se charge plus sur Windows récent.** Son pilote `RwDrv.sys` figure sur la liste des pilotes vulnérables de Microsoft (utilisé dans des attaques BYOVD). Sur Windows 11, l'intégrité de la mémoire et la liste de blocage sont actives par défaut → erreur « driver cannot be loaded », même en administrateur. Deux alternatives, toutes deux meilleures.

**Méthode A — `acpidump.exe` (recommandée sous Windows)**

Outil officiel du projet ACPICA. **Aucun pilote installé** : il passe par l'API Windows `GetSystemFirmwareTable`.

1. Télécharge `iasl-win-*.zip` depuis `github.com/acpica/acpica/releases`
2. Décompresse — l'archive contient `acpidump.exe` **et** `iasl.exe`
3. Invite de commandes **administrateur** :

```
cd C:\Hack\Rapport\ACPI
acpidump.exe -b
```

Le `-b` produit un fichier binaire par table : `dsdt.dat`, `ssdt1.dat`, `ssdt2.dat`, `facp.dat`, `apic.dat`, `dmar.dat`… Tu peux les renommer en `.aml`, SSDTTime accepte les deux extensions.

**Décompilation du DSDT** — uniquement si tu veux le lire toi-même :

```
iasl.exe -e ssdt*.dat -d dsdt.dat
```

> ⚠️ **N'utilise pas `-da` ni `-dl`.** `-da` sert à désassembler *plusieurs* tables dans un espace de noms unique et se met en défaut si on ne lui passe qu'un fichier ; `-dl` force une syntaxe ASL héritée dont tu n'as pas besoin. La bonne option est **`-d`**, et **`-e`** fournit les SSDT pour résoudre les symboles externes sans les désassembler.
>
> **Si le `.dsl` produit est vide** : vérifie d'abord la taille de `dsdt.dat` avec `dir`. Sur ce Latitude il doit peser **275 297 octets**. Un fichier vide ou minuscule signifie que `acpidump.exe` a échoué — presque toujours parce qu'il n'a pas été lancé **en administrateur**. Il n'affiche pas d'erreur franche, il écrit simplement des fichiers vides.
>
> **Les avertissements sont normaux.** Un désassemblage ACPI produit toujours des dizaines de remarques. Le seul critère qui compte : le `.dsl` contient-il du code et commence-t-il par `DefinitionBlock` ?

> 💡 **SSDTTime n'a pas besoin du `.dsl`.** Il lit directement les binaires `.aml` / `.dat`. La décompilation ne sert qu'à la lecture humaine — pour chercher `Device (EC`, `GPI0`, `AWAC` à la main. Si tu ne comptes pas éplucher le fichier, saute cette étape.

**Méthode B — depuis le live Linux (la plus simple)**

Tu vas de toute façon créer cette clé pour le reformatage 4K. Fais le dump **avant** de formater, tant que la machine est intacte :

```bash
mkdir -p /tmp/acpi && cd /tmp/acpi
cp /sys/firmware/acpi/tables/DSDT .
cp /sys/firmware/acpi/tables/SSDT* .
cp /sys/firmware/acpi/tables/FACP .
cp /sys/firmware/acpi/tables/APIC .
cp /sys/firmware/acpi/tables/DMAR . 2>/dev/null
```

Puis copie le dossier sur une clé de données.

**Méthode C — RWEverything : déconseillée**

Il faudrait désactiver l'intégrité de la mémoire (`Sécurité Windows → Sécurité de l'appareil → Isolation du noyau`) puis la liste de blocage des pilotes vulnérables via le registre, et redémarrer. Tu affaiblirais Windows pour obtenir exactement les mêmes fichiers que les méthodes A et B fournissent sans rien toucher.

**Tables à récupérer dans tous les cas** : `DSDT`, tous les `SSDT`, `FACP`, `APIC`, `DMAR`, `MCFG`.

**3 — Liste des périphériques (PowerShell administrateur)**

```powershell
cd C:\Hack\Rapport

Get-PnpDevice | Select-Object Status,Class,FriendlyName,InstanceId |
  Sort-Object Class | Format-Table -AutoSize |
  Out-File -Width 400 devices-all.txt

# Ciblé sur le trackpad — LA réponse qu'on attend
Get-PnpDevice -Class Mouse,HIDClass,System |
  Where-Object { $_.FriendlyName -match "touch|track|I2C|ELAN|SYNA|pointing|souris" } |
  Select-Object FriendlyName,InstanceId,Status |
  Format-List | Out-File -Width 400 trackpad.txt

msinfo32 /report msinfo.txt
```

**4 — USBToolBox** → suivre §10.1, sauvegarder `UTBMap.kext` dans `Rapport\`

**5 — SSDTTime** → option `D` (Dump DSDT). Si le dump échoue sous Windows, pas grave : les tables récupérées à l'étape 2 font exactement le même travail — SSDTTime sait les lire depuis un dossier, il suffit de le lui glisser.

## A.3 Les informations à extraire du rapport

Coche au fur et à mesure — c'est ça, la « réalité technique » de la machine.

| # | Information | Où la trouver | Pourquoi |
|---|---|---|---|
| 1 | **Trackpad : I2C ou PS/2** | `trackpad.txt` / HWiNFO → Devices | 🔴 Détermine VoodooI2C vs VoodooPS2 |
| 2 | Nom ACPI du trackpad (`ELAN0621`…) | idem | Dépannage VoodooI2C |
| 3 | **Contrôleurs I2C Intel** (`8086:9DE8`, `9DE9`…) | HWiNFO → PCI Bus | Confirme la présence du bus I2C |
| 4 | iGPU : `VEN_8086&DEV_3EA0` | HWiNFO → Video Adapter | Confirme le spoof `9B3E0000` |
| 5 | Ethernet : `VEN_10EC&DEV_8168` | HWiNFO → Network | Confirme `RealtekRTL8111.kext` |
| 6 | Contrôleur audio + codec réel | ✅ ALC3204 confirmé (live Linux) | — |
| 7 | Wi-Fi : `VEN_8086&DEV_9DF0` (CNVi) | HWiNFO → Network | Confirme définitivement le montage CNVi |
| 8 | **Câblage du slot M.2 A/E** | HWiNFO → PCI Bus, présence ou non d'un pont PCIe libre | 🔴 Détermine si une DW1560 est envisageable (§12.1) |
| 9 | NVMe : modèle, LBA format, firmware | HWiNFO → Drives | Reformatage 4K (§0.5) |
| 10 | **Nom ACPI du contrôleur embarqué** (`EC`, `EC0`, `H_EC`) | DSDT décompilé | SSDT-EC-USBX |
| 11 | Présence d'un device **`GPI0`** dans le DSDT | DSDT décompilé | XOSI vs GPI0 (§5.3) |
| 12 | Présence de **`AWAC`** / `RTC` | DSDT décompilé | SSDT-AWAC ou SSDT-RTC0 |
| 13 | Champs EC de la batterie **>8 bits** | DSDT décompilé | Valide le besoin d'`ECEnabler` |
| 14 | Liste des ports USB + types | USBToolBox | Cartographie (§10) |
| 15 | **État du CFG Lock** | §A.4 | Valide `AppleXcpmCfgLock` |

Pour les points 10 à 13, il faut décompiler le DSDT :

```
iasl -e ssdt*.dat -d dsdt.dat
```
(iasl est fourni dans la même archive qu'`acpidump` — `github.com/acpica/acpica/releases`)

Puis chercher dans le `.dsl` produit : `Device (EC`, `Device (GPI0`, `AWAC`, `Device (BAT`.

> ✅ **Déjà fait pour toi** — ces quatre points sont analysés en Annexe B à partir du `dsdt.dsl` que tu as fourni. Inutile de refaire l'exercice.

## A.4 Vérifier le CFG Lock

Le BIOS Dell n'expose pas l'option, mais autant savoir où on en est. OpenCorePkg fournit l'outil :

- Copie `Utilities/VerifyMsrE2/VerifyMsrE2.efi` dans `EFI/OC/Tools/` de ta clé
- Ajoute-le dans `Misc → Tools` du config.plist
- Démarre sur la clé, presse **Espace**, lance `VerifyMsrE2`

| Résultat | Conséquence |
|---|---|
| Toutes les lignes en `TRUE` | CFG Lock **actif** → `AppleXcpmCfgLock = YES` (ce qu'on a prévu) |
| Toutes en `FALSE` | CFG Lock déverrouillé → le quirk devient inutile |
| Mélange | Firmware capricieux → laisse le quirk à `YES` |

## A.5 Bonus Dell : options BIOS cachées

**Dell Command | Configure** (téléchargeable sur support.dell.com) permet d'énumérer et de modifier des réglages BIOS que l'interface F2 n'affiche pas.

```
cd "C:\Program Files (x86)\Dell\Command Configure\X86_64"
cctk.exe --outfile C:\Hack\Rapport\bios-dump.txt
```

Le fichier produit liste **tous** les réglages avec leur valeur courante. Ça vaut le coup de le parcourir en cherchant `dvmt`, `cfg`, `vtd`, `sata` — on tombe parfois sur des options absentes de l'interface graphique. Ne modifie rien à l'aveugle, mais si un réglage DVMT apparaissait, ce serait une excellente nouvelle : il rendrait les patchs framebuffer inutiles.

> Réaliste : sur les Latitude de cette génération, le DVMT reste verrouillé. Mais la vérification coûte deux minutes.

## A.6 Livrable attendu

```
C:\Hack\Rapport\
├── hwinfo-full.html
├── hwinfo-full.txt
├── devices-all.txt
├── trackpad.txt
├── msinfo.txt
├── bios-dump.txt
├── UTBMap.kext
└── ACPI\
    ├── DSDT.aml
    ├── DSDT.dsl        (décompilé)
    ├── SSDT-*.aml
    ├── FACP.aml
    ├── APIC.aml
    └── DMAR.aml
```

Avec ce dossier, la construction de l'EFI ne repose plus sur aucune supposition.

---
