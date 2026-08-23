---
name: Feature request
about: Suggérer une amélioration ou une nouvelle fonctionnalité
title: "[FEATURE] "
labels: enhancement
assignees: ''

---

## 💡 Description

[Explique succinctement ce que tu aimerais améliorer ou ajouter]

**Exemple :** "Ajouter le support du WiFi AX211 (plus récent que AX200)"

---

## 🎯 Cas d'Usage

Pourquoi cette fonctionnalité est utile ?

- Pour qui ? [Ex: utilisateurs avec SSD NVMe Gen4]
- Quel problème résout-elle ? [Ex: améliorer la compatibilité hardware]
- Impact estimé ? [Ex: 5% des utilisateurs Latitude 3500 ont ce composant]

---

## 🔍 Alternatives Considérées

Y a-t-il une workaround actuelle ?

**Exemple :**
- Actuellement : utiliser USB dongle Bluetooth
- Proposal : supporte les dongles USB natifs sans kext
- Avantage : simplifie l'installation post-boot

---

## 🛠️ Contexte Technique

[Optional mais utile]

- Quel composant est affecté ? [Ex: WiFi, audio, stockage]
- Référence hardware ? [Ex: Intel AX211 VID:8086 PID:2725]
- Lien vers datasheet / documentation ? [Ex: GitHub OpenIntelWireless]

**Exemple pour NVMe :**
```
Nouveau drive : Samsung 990 Pro
Interface : NVMe Gen4 PCIe
Secteur : 4Kn (problématique : APFS loop)
Proposition : documentation + note dans config pour 4Kn avoidance
```

---

## 📋 Checklist

- [ ] J'ai cherché si cette feature existe déjà ou est documentée
- [ ] J'ai vérifié qu'elle n'est pas dans Roadmap / Issues fermés
- [ ] Je fournirai des détails techniques si demandé
- [ ] Je peux tester la solution si elle est proposée

---

## 🎁 Contribution

Es-tu prêt à contribuer (PR) ?

- [ ] Oui, je peux soumettre un PR
- [ ] Oui, mais j'ai besoin de guidance
- [ ] Non, j'attends que quelqu'un d'autre le fasse

---

**Merci d'améliorer ce projet !** 🙌
