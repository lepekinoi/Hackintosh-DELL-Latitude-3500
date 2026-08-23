---
name: Bug report
about: Rapporter un problème technique avec la configuration Hackintosh
title: "[BUG] "
labels: bug
assignees: ''

---

## 🐛 Description du Problème

[Décris brièvement ce qui ne fonctionne pas — en une ou deux phrases]

**Exemple :** "Le trackpad ne se charge pas au boot, pas de multitouch"

---

## 📋 Informations Système

**IMPORTANT** — sans ces infos, on ne peut pas te aider :

- **Tahoe version** : (26.0 / 26.0.1 / autre ?)
  ```bash
  # Commande pour vérifier :
  sw_vers
  ```

- **BIOS Dell** : (doit être 1.36.0+)
  ```bash
  # Visible dans Préférences Système > À propos > BIOS / Firmware
  ```

- **OpenCore version** : (1.0.7+ ?)
  ```bash
  # Visible au boot dans OpenCore picker
  ```

- **EFI version** : (quelle branche du repo ? v2.1 ? custom ?)

---

## 🔧 Étapes pour Reproduire

1. ...
2. ...
3. ...

**Résultat observé :** [ce qui se passe]
**Résultat attendu :** [ce qui devrait se passer]

---

## 📊 Logs & Diagnostiques

### ✅ REQUIS : IORegistry

[Sans ça, on ne peut rien diagnostiquer — c'est obligatoire]

```bash
# Terminal — copie/colle ces commandes :
ioreg -l -p IODeviceTree > ~/ioreg.txt
cat ~/ioreg.txt
```

Puis **copie/colle tout le résultat ci-dessous** dans un bloc de code :

<details>
<summary>ioreg (cliquez pour voir le contenu)</summary>

```
[COLLE LE RESULTAT DE `ioreg -l -p IODeviceTree` ICI]
```

</details>

---

### ✅ REQUIS : Boot Log OpenCore

Lors du prochain boot :

1. Appuie sur **Cmd + V** dans le picker OpenCore (menu de démarrage)
   - Cela affiche les logs de boot
   - Prends une capture d'écran OU note le texte

2. Ou cherche le fichier log sur macOS (après installation) :
   ```bash
   # Si tu as accès au système :
   log stream --predicate 'eventMessage contains[cd] "OpenCore"' > ~/opencore_boot.log
   cat ~/opencore_boot.log
   ```

<details>
<summary>OpenCore boot log (cliquez pour voir)</summary>

```
[COLLE LES LOGS ICI]
```

</details>

---

### 📌 OPTIONNEL mais utile : System Log

Si le système boot mais qu'un composant échoue (audio, WiFi, trackpad) :

```bash
# Affiche les 100 dernières lignes du système log :
log show --predicate 'level == error OR level == critical' --last 1h | head -100
```

Ou cherche une erreur spécifique :

```bash
# Audio :
log show --predicate 'process == "kernel" AND message contains "ALAC"' --last 1h

# WiFi :
log show --predicate 'process contains "itlwm"' --last 1h

# Trackpad I2C :
log show --predicate 'message contains "VoodooI2C"' --last 1h
```

<details>
<summary>System log (optionnel)</summary>

```
[LOGS ICI]
```

</details>

---

### 📷 Screenshots

Ajoute des captures d'écran si pertinent :

- Écran noir / kernel panic → capture du message d'erreur
- Problème d'affichage → capture HDMI / écran externe
- Problème de batterie → capture Préférences Système > Batterie

---

## 🎯 Checklist de Diagnostic

Avant de créer l'issue, essaie ces vérifications :

- [ ] J'ai vérifié que le problème n'existe pas sur une install vanilla macOS (pour éliminer le hardware)
- [ ] J'ai comparé ma config.plist avec la v2.1 du repo (pour trouver mes modifications)
- [ ] J'ai cherché le problème dans la section Troubleshooting du README
- [ ] J'ai collecté ioreg et boot logs (requis)
- [ ] Mon BIOS est à jour (1.36.0+) et configuré selon le guide BIOS

---

## 📝 Contexte Additionnel

[Toute info supplémentaire utile]

**Exemple :**
- J'ai modifié le device-id pour...
- J'ai remplacé le SSD et depuis...
- C'était OK avant la mise à jour OpenCore vers...

---

## 🔗 Ressources Utiles

Si tu trouves la solution avant une réponse, partage-la ! Ça aide d'autres.

- [Hackintosh.com Guide](https://hackintosh.com)
- [Dortania Troubleshooting](https://dortania.github.io/OpenCore-Install-Guide/)
- [r/hackintosh megathread](https://reddit.com/r/hackintosh)

---

**Merci de fournir les logs — ça accélère le diagnostic !** ⚡
