# Bluetooth Intel sous macOS 26 Tahoe — Journal d'investigation

## Statut : non résolu — solution de contournement en place (dongle USB)

Ce document existe pour une raison précise : **économiser le temps de la prochaine personne** qui rencontre exactement ce symptôme. Neuf variables indépendantes ont été testées méthodiquement sur ce Dell Latitude 3500, avec deux cartes Intel différentes. Aucune n'a résolu le problème. Si tu es ici parce que ton Bluetooth Intel ne fonctionne pas sous Tahoe, la conclusion courte est : **ce n'est probablement pas ta configuration — passe directement à la section Solution en bas de page.**

---

## Le symptôme

Le matériel est détecté, identifié correctement, le kext se charge — mais ne finalise jamais son attachement.

```
+-o IntelBluetoothFirmware  <class IntelBluetoothFirmware, !registered, !matched, active, busy 0>
    "IOProbeScore" = 94000
    "IOMatchedAtBoot" = Yes
    "idProduct" = 41              ← ou 2730 selon la carte, voir plus bas
    "IOProviderClass" = "IOUSBHostDevice"
```

`IOProbeScore = 94000` : le kext *veut* s'attacher, avec un score de correspondance élevé.
`IOMatchedAtBoot = Yes` : il a été associé au démarrage.
`!registered, !matched` : mais l'attachement final n'aboutit jamais, de façon permanente.

**Côté système**, `system_profiler SPBluetoothDataType` continue d'afficher un profil fantôme hérité du SMBIOS déclaré (`MacBookPro16,2`) :

```
Bluetooth Controller:
    Chipset: BCM_4350C2
    Vendor ID: 0x004C (Apple)
    Address: NULL
    State: Off
```

Ce n'est pas le vrai matériel — c'est le contrôleur Bluetooth théorique d'un vrai MacBook Pro 16", qu'Apple continue d'exposer par défaut sans jamais céder la main au vrai périphérique Intel.

---

## Matériel confirmé au niveau USB

Le device Bluetooth est bien énuméré, avec les bons identifiants, dans les deux configurations testées :

| Carte | idVendor | idProduct | Reconnu par macOS ? |
|---|---|---|---|
| Intel AC 9560 (CNVi, d'origine) | `0x8087` (Intel) | `0x0AAA` | ✅ Énuméré, jamais matched |
| Intel AX200NGW (PCIe, remplacement) | `0x8087` (Intel) | `0x0029` | ✅ Énuméré, jamais matched |

Le swap de carte a été fait spécifiquement pour tester si le problème venait du chipset CNVi de la 9560. **Résultat identique sur les deux** — ce n'est donc pas une histoire de CNVi vs PCIe, ni de génération de puce.

---

## Les neuf variables testées, dans l'ordre chronologique

| # | Test | Méthode | Résultat |
|---|---|---|---|
| 1 | `IntelBTPatcher.kext` actif | Kext activé dans `Kernel → Add` | 🔴 **Panique noyau / boucle de redémarrage**, reproduite à l'identique sur les deux cartes. Bug connu, voir [issue #486](https://github.com/OpenIntelWireless/IntelBluetoothFirmware/issues/486) |
| 2 | `IntelBTPatcher.kext` retiré | Désactivé définitivement | Boucle résolue, mais `!matched` persiste |
| 3 | `-lilubeta` → `-lilubetaall` | Boot-arg, autorise tous les plugins Lilu sur OS non reconnu | Aucun effet |
| 4 | `-wegbeta` → `-wegbetaall` | Idem pour WhateverGreen | Aucun effet |
| 5 | `-ibtcompatbeta` | Vérifié présent dans `nvram boot-args` | Présent, sans effet |
| 6 | Nettoyage NVRAM ciblé | `nvram -d bluetoothExternalDongleFailed` et variantes | Aucun effet |
| 7 | Reset NVRAM complet | Via `ResetNvramEntry.efi` dans le menu OpenCore | Aucun effet |
| 8 | Cartographie USB personnalisée | `UTBMap.kext` avec port Bluetooth déclaré | `!matched` |
| 9 | Carte USB par défaut (`USBDefault`) | `UTBMap.kext` désactivé | `!matched` — **pire** : le device disparaît même de `system_profiler SPUSBDataType` |

## Piste écartée après vérification : le mapping USB

Une hypothèse sérieuse portait sur un possible décalage entre le port déclaré dans `usb.json` et le port réel identifié par macOS (`Identifiant de l'emplacement` dans le rapport système). Un décalage a bien été observé (webcam -1, lecteur SD -1, Bluetooth -3 selon les relevés), mais :

- Le décalage n'est pas linéaire — pas cohérent avec une simple erreur de correspondance de port
- Basculer sur la carte USB par défaut (qui élimine toute question de mapping) donne un résultat **identique ou pire**

Cette piste reste techniquement non totalement épuisée (un test précis avec le port Bluetooth réassigné au bon index n'a pas donné de résultat concluant), mais les éléments disponibles n'en font pas la cause principale.

---

## Logs — absence totale de trace

Fait notable : aucune méthode de journalisation n'a produit la moindre ligne concernant ce kext.

```bash
sudo dmesg | grep -i IntelFir                              # vide
log show --last 15m --predicate 'senderImagePath contains "IntelBluetoothFirmware"'   # 0 entrées
log show --last 15m --predicate 'eventMessage contains "IntelBluetoothFirmware"'      # 0 entrées
```

Sur d'autres configurations documentées en ligne, `dmesg` montre normalement une séquence `Driver init()` → `Driver Probe()` → `Driver Start()` → `Found interface`. Ici, rien de tout cela n'apparaît — le kext semble s'exécuter dans un silence complet, sans jamais atteindre ses points de log habituels.

---

## Hypothèse retenue

Deux pistes restent ouvertes, sans confirmation possible depuis l'extérieur du projet OpenIntelWireless :

**1. Régression de la pile Bluetooth de Tahoe elle-même.** Plusieurs utilisateurs sur les forums Olarila et tonymacx86 rapportent le même symptôme sur macOS 26, y compris avec des cartes AX210 — donc indépendamment du modèle précis de puce Intel. Un fil dédié (« AX210 Tahoe Fix ») propose des correctifs (dont `-lilubetaall`, déjà testé ici sans succès), signe que le problème est identifié côté communauté sans solution stable à ce jour.

**2. Conflit d'attribution avec le nouveau système DriverKit d'Apple.** Une partie de l'énumération USB de ce périphérique porte l'entitlement `com.apple.developer.driverkit.builtin`, signalant une possible réclamation exclusive du port par la pile DriverKit d'Apple (normalement réservée aux vrais Mac à puce T2) avant que le kext legacy `IntelBluetoothFirmware` (basé sur l'ancienne API `IOUSBHostDevice`) n'ait la possibilité de s'y attacher. Cette piste n'a pas pu être confirmée ou infirmée avec les outils disponibles en ligne de commande standard.

---

## Solution retenue : dongle Bluetooth USB

Après épuisement des pistes logicielles raisonnables, la solution appliquée est un dongle Bluetooth USB externe :

- **Chipset recommandé** : CSR8510 ou Broadcom BCM20702
- **Coût** : 10-15 €
- **Configuration requise** : aucune — reconnu nativement par macOS sans kext ni pilote tiers
- **Limite** : pas de partage d'antenne interne, pas de fonctions Continuité (de toute façon absentes avec une carte Wi-Fi CNVi)

Ce dongle occupe un port USB-A. Sur ce modèle, deux ports USB-A restent disponibles après cet usage.

---

## Si tu veux poursuivre l'investigation

Pistes non explorées faute de temps ou d'outils adaptés :

- **Comparer avec une machine Sequoia** identique pour confirmer si le même kext fonctionne sur la version précédente de macOS — trancherait définitivement entre « régression Tahoe » et « configuration spécifique »
- **Extraire et inspecter le binaire `IntelBluetoothFirmware`** avec un désassembleur pour identifier le point d'échec exact dans le code du kext plutôt que d'observer seulement son état final
- **Tester avec `AirportItlwm`** au lieu d'`itlwm` pour le Wi-Fi — non tenté ici car documenté comme cassé sur Tahoe indépendamment du Bluetooth, mais l'interaction entre les deux composants n'a pas été isolée
- **Contacter directement les mainteneurs d'OpenIntelWireless** via une issue GitHub détaillée, en citant ce document — c'est la voie la plus susceptible d'aboutir à une vraie réponse, ce diagnostic dépassant ce qu'un outillage de bord peut confirmer

Si tu obtiens un résultat différent avec une correction de ce diagnostic, une pull request ou une issue sur ce dépôt est bienvenue.

---

*Environnement testé : Dell Latitude 3500, BIOS 1.36.0, macOS 26 Tahoe, OpenCore 1.0.7, IntelBluetoothFirmware/BlueToolFixup dernières versions disponibles au moment du test.*
