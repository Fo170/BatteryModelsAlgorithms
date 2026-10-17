# 📦 REGISTRY SUBMISSION GUIDE - v1.2.0

**Date**: 2026-07-20  
**Status**: Ready for submission  
**Release**: v1.2.0 (GitHub)

---

## 🎯 Deux Registries à Soumettre

### **1. Arduino Library Manager**
Official Arduino registry for IDE integration

### **2. PlatformIO Library Registry**
PlatformIO dependency management

---

## 📋 PRÉ-REQUIS (VÉRIFIÉS ✅)

- [x] `library.json` present et valide
- [x] `README.md` avec installation instructions
- [x] `LICENSE` file (GPLv3)
- [x] GitHub repository public
- [x] Git tags: v1.2.0 créé
- [x] Release GitHub: v1.2.0 créée
- [x] Code complet: src/BatteryModels.h

**Status**: ✅ Tous les pré-requis validés

---

## 1️⃣ SOUMISSION ARDUINO LIBRARY MANAGER

### **Process**

#### **Étape 1: Vérifier library.json**

Doit contenir:
```json
{
  "name": "BatteryModels",
  "version": "1.2.0",
  "description": "...",
  "keywords": [...],
  "authors": [{...}],
  "license": "GPL-3.0-only",
  "frameworks": "arduino",
  "platforms": [...],
  "repository": {
    "type": "git",
    "url": "https://github.com/Fo170/BatteryModels.git"
  }
}
```

**Status**: ✅ library.json complété et valide

---

#### **Étape 2: Fork Arduino Library Registry**

1. Aller à: https://github.com/arduino/library-registry
2. Fork le repository
3. Clone locally

```bash
git clone https://github.com/YOUR-USERNAME/library-registry.git
cd library-registry
```

---

#### **Étape 3: Ajouter BatteryModels au Registry**

Créer fichier: `libraries/BatteryModels.txt`

```
name=BatteryModels
url=https://github.com/Fo170/BatteryModels
maintainer=Olivier Fournet <olivier.fournet@free.fr>
sentence=Parametric battery models with OCV tables and Thevenin dynamics
paragraph=Complete battery modeling library for 15 technologies (Lead-acid, Lithium, NiMH, Alkaline, etc.) with advanced thermal compensation (Arrhenius, RC thermal, OCV polynomial). Precision ±1.4% SoC over -30 to +80°C.
category=Device Control
url=https://github.com/Fo170/BatteryModels
architectures=*
types=Contributed
```

---

#### **Étape 4: Créer Pull Request**

1. Commit et push vers votre fork
2. Créer PR vers `arduino/library-registry:main`
3. PR title: `Add BatteryModels library`
4. PR description:
```
Add BatteryModels v1.2.0 to Arduino Library Manager

Library: BatteryModels
Repository: https://github.com/Fo170/BatteryModels
Version: 1.2.0
License: GNU GPLv3

This is a parametric battery modeling library supporting 15 battery technologies with:
- Advanced thermal compensation (Arrhenius exponential, RC thermal, OCV polynomial)
- Thevenin equivalent circuit model
- Coulomb counting with Peukert correction
- SoC estimation (OCV-based and Coulomb counting)
- Cross-platform support (Arduino, PlatformIO, ESP32, ARM)

Installation via Arduino IDE: Sketch > Include Library > Manage Libraries > Search "BatteryModels"
```

---

#### **Étape 5: Arduino Review**

- Arduino team reviews PR (typically 1-2 weeks)
- May ask for clarifications
- Once merged → available in Arduino IDE

**Timeline**: 1-2 semaines pour approval

---

## 2️⃣ SOUMISSION PLATFORMIO LIBRARY REGISTRY

### **Process**

#### **Étape 1: Vérifier library.json pour PlatformIO**

```json
{
  "name": "BatteryModels",
  "version": "1.2.0",
  "description": "...",
  "keywords": [...],
  "repository": {
    "type": "git",
    "url": "https://github.com/Fo170/BatteryModels"
  },
  "authors": [{...}],
  "license": "GPL-3.0-only",
  "frameworks": "arduino",
  "platforms": ["atmelavr", "espressif32", "espressif8266", "teensy", "ststm32", "atmelsam"],
  "headers": "BatteryModels.h"
}
```

**Status**: ✅ library.json compatible PlatformIO

---

#### **Étape 2: PlatformIO Account**

1. Créer compte: https://platformio.org/auth/sign-up
2. Vérifier email
3. Login sur: https://platformio.org/

---

#### **Étape 3: Submit Library**

1. Aller à: https://platformio.org/lib/submit
2. Remplir le formulaire:

```
Library Name: BatteryModels
Repository URL: https://github.com/Fo170/BatteryModels
Description: Parametric battery models with OCV tables and Thevenin dynamics
Keywords: battery, battery-model, ocv, soc, thevenin, thermal-compensation
Frameworks: Arduino
Platforms: atmelavr, espressif32, espressif8266, teensy, ststm32, atmelsam
License: GNU General Public License v3.0
Documentation URL: https://github.com/Fo170/BatteryModels/blob/main/README.md
Repository Type: Git
```

---

#### **Étape 4: PlatformIO Review**

- Validates library.json
- Tests download and compilation
- Typically automated (few hours)

**Timeline**: 2-24 heures pour availability

---

#### **Étape 5: Verify Installation**

Once approved, users can install:

```ini
[env:example]
platform = espressif32
board = esp32dev
lib_deps =
    BatteryModels
```

---

## ✅ SOUMISSION CHECKLIST

### **Arduino Library Manager**
- [ ] library.json valide (checked with online validator)
- [ ] Fork arduino/library-registry
- [ ] Ajouter libraries/BatteryModels.txt
- [ ] Créer PR
- [ ] Attendre approval (1-2 semaines)
- [ ] Disponible dans Arduino IDE

### **PlatformIO Registry**
- [ ] library.json PlatformIO-compatible
- [ ] Créer compte PlatformIO
- [ ] Soumettre via https://platformio.org/lib/submit
- [ ] Attendre validation (2-24 heures)
- [ ] Vérifier installation: `platformio lib install BatteryModels`

---

## 📊 TIMELINE ESTIMÉ

```
Jour 0:
  ✅ BatteryModels v1.2.0 release GitHub

Jour 1:
  [ ] Soumettre à PlatformIO Registry
  [ ] Soumettre à Arduino Library Manager

Jour 2:
  ✅ PlatformIO approval (généralement)

Jour 14:
  ✅ Arduino approval (estimé)

Jour 15:
  🎉 BatteryModels disponible dans:
     - Arduino IDE Library Manager
     - PlatformIO Library Registry
     - GitHub Release v1.2.0
```

---

## 🔗 USEFUL LINKS

**Arduino Library Registry**:
- Documentation: https://create.arduino.cc/library-registry-converter/
- Registry: https://github.com/arduino/library-registry
- Submission: Via PR on registry repo

**PlatformIO Registry**:
- Submit: https://platformio.org/lib/submit
- Registry: https://platformio.org/lib
- Documentation: https://docs.platformio.org/en/latest/librarymanager/

---

## 📝 NOTES

- **GPLv3 License**: Both registries support GPLv3 ✓
- **Cross-platform**: arduino-only not required; PlatformIO supports more ✓
- **Documentation**: README.md sufficient for both ✓
- **Updates**: Future releases automatically updated if git tag follows semver ✓

---

## 🎯 COMMANDES RAPIDES

### **Arduino Registry (manual)**

```bash
# Fork and clone
git clone https://github.com/YOUR-USERNAME/library-registry.git
cd library-registry

# Add library
echo 'name=BatteryModels
url=https://github.com/Fo170/BatteryModels
maintainer=Olivier Fournet <olivier.fournet@free.fr>
sentence=Parametric battery models with OCV tables and Thevenin dynamics
paragraph=Complete battery modeling library for 15 technologies with advanced thermal compensation.
category=Device Control
architectures=*
types=Contributed' > libraries/BatteryModels.txt

# Commit and push
git add libraries/BatteryModels.txt
git commit -m "Add BatteryModels library"
git push origin main

# Create PR on GitHub web interface
```

### **PlatformIO Registry (web form)**

Visit: https://platformio.org/lib/submit
Fill form with library details (as shown above)
Submit

---

**Status**: Ready for Submission  
**Next Action**: Choose to submit or wait for user decision

---

**Version**: 1.2.0  
**Date**: 2026-07-20
