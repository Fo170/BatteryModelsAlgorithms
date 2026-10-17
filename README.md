# BatteryModelsAlgorithms

**Conteneur** regroupant les librairies Arduino/ESP8266/ESP32 de **modélisation
et d'estimation batterie**. Ce n'est pas une librairie : c'est un *workspace*
qui rassemble des modules **liés et interdépendants** traitant la même
problématique — mesurer, modéliser et estimer l'état d'une batterie.

Chaque librairie garde **son propre dépôt GitHub, sa propre version et son
propre historique**. Ce dépôt racine ne contient que les **pointeurs de
sous-modules** (`.gitmodules`) et cette documentation.

## Les 5 librairies

| Librairie | Version | Rôle | Dépend de |
|---|---|---|---|
| **[BatteryModels](https://github.com/Fo170/BatteryModels)** | 1.6.6 | Modèles physiques : OCV (tables + cubique), Thévenin, thermique (Arrhenius), hystérésis, 2ᵉ RC, gassing, Peukert/η, **AFFRLS** (`RlsIdentifier`), **EKF d'état** (`SocEkf`) | — |
| **[BatteryKalman](https://github.com/Fo170/BatteryKalman)** | 1.7.1 | Filtre de Kalman étendu 2D (capacité + vieillissement), phases Bootstrap → Tracking, observateur passif P1/P2 | BatteryModels, CoulombsAh |
| **[CoulombsAh](https://github.com/Fo170/CoulombsAh)** | 1.2.0 | Comptage coulombs (C) et ampères-heures (Ah) + SoC, Peukert dynamique, rendement η variable | Persistance |
| **[BatteryLifePredictor](https://github.com/Fo170/BatteryLifePredictor)** | 1.3.0 | Prédiction de vie : usure calendaire + cycles (Rainflow), EFC, SOH, EOL, sulfatation | BatteryModels, BatteryKalman |
| **[Persistance](https://github.com/Fo170/Persistance)** | 1.0.0 | Gestionnaire de sauvegardes EEPROM / FRAM / LittleFS (registres nommés, magic + version + CRC16, double tampon) | — |

### Chaîne de dépendances

```
Persistance  ◄──────────────┐
                            │
CoulombsAh ─────────────────┤
      ▲                     │
      │                     │
BatteryModels ◄── BatteryKalman
      ▲                ▲
      │                │
      └── BatteryLifePredictor
```

## Installation

### PlatformIO — tout le workspace

```ini
lib_deps =
    https://github.com/Fo170/BatteryModels.git#v1.6.6
    https://github.com/Fo170/BatteryKalman.git#v1.7.1
    https://github.com/Fo170/CoulombsAh.git#v1.2.0
    https://github.com/Fo170/BatteryLifePredictor.git#v1.3.0
    https://github.com/Fo170/Persistance.git#v1.0.0
```

### Cloner le workspace avec ses sous-modules

```bash
git clone --recurse-submodules https://github.com/Fo170/BatteryModelsAlgorithms.git
```

Ou, sur un clone existant :

```bash
git submodule update --init --recursive
```

## Version de chaque librairie

Chaque librairie déclare sa version dans son propre header :

```c
#define __BATTERYMODELS_H_VERSION__          "1.6.6"
#define __BATTERYKALMAN_H_VERSION__          "1.7.1"
#define __BATTERYLIFEPREDICTOR_H_VERSION__   "1.3.0"
#define __COULOMB_H_VERSION__                "1.2.0"   // CoulombsAh
#define __PERSISTANCE_H_VERSION__            "1.0.0"
```

Un projet hôte peut les **agréger** (ex. page `/version`) sans recopier les
numéros — une seule source de vérité par librairie.

## Conventions

- **Convention de signe** : `I > 0` = **charge**, `I < 0` = **décharge**
  ⇒ `V_terminal = OCV + I·R0 + V1 + V2`.
- **Unités** : `R_int` / `R2` de `BatteryModels` en **mΩ par cellule** ;
  `R0` / `R1` de `RlsIdentifier` en **Ω batterie**.
- **OCV monotone croissant** sur `[0,1]` (l'inverse par bissection l'exige).
- **Pas de membre nommé** `_P`, `_C`, `_r`, `_q`, `_x` (macros du SDK ESP8266).

## Licence

Chaque librairie porte sa propre licence (GPL-3.0-only, sauf mention contraire
dans son dépôt).
