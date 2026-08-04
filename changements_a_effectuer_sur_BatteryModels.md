# Changements à effectuer sur BatteryModels

**Version cible** : v1.4.0 (API additive, 100 % rétro-compatible)
**Date** : Août 2026
**Statut** : Spécification — à implémenter dans `src/BatteryModels.h`

---

## 1. Contexte

Les seuils de détection de `BatteryModels.h` (tension de float, bande de tolérance, dead-band de courant, temps de repos OCV, comptage de cycle) sont actuellement **codés en dur et identiques pour les 15 technologies**. Certains sont dérivés de formules globales (`V_tol = max(0.1 V, V_nom × 0.015)`, `I_float_max = max(0.6 A, cap × 0.02)`, repos OCV 0.5 h sauf LiFePO4 2 h).

L'objectif est de :
1. Rendre ces seuils **configurables par technologie** (valeurs par défaut adaptées à chaque chimie) ;
2. Permettre une **surcharge compile-time** (`#ifndef`) et **runtime** (setters) ;
3. Exposer un jeu de réglages **Kalman** (`P`, `Q`, `R`, seuils ΔAh/ΔSoC, REST_LONG, confiance) que la librairie sœur `BatteryKalman` consommera comme valeurs par défaut (voir `changements_a_effectuer_sur_BatteryKalman.md`).

---

## 2. Inventaire des seuils actuels dans `src/BatteryModels.h`

| Fonction | Ligne(s) | Seuil actuel | Rôle |
|---|---|---|---|
| `getVoltageToleranceBand()` | 774-778 | `max(0.1 V, V_nom × 0.015)` | Bande de proximité FLOAT / ABSORPTION |
| `detectChargeState()` | 780-812 | dead-band courant `0.05 A` ; `I_float_max = max(0.6 A, cap × 0.02)` | Détection DISCHARGE / FLOAT / REST / ABSORPTION / BULK |
| `getPeukertReferenceRate()` | 822-830 | `C/20` (plomb, NiFe) ; `C` (lithium, autres) | Taux de référence Peukert |
| `updateCoulombCount()` | 846-856 | cycle si `\|ΣAh\| ≥ C_nom × 0.8` | Comptage des cycles |
| `isOcvMeasurementValid()` | 901-907 | courant `≤ 0.05 A` ; repos `0.5 h` (LiFePO4 : `2 h`) | Validité d'une mesure OCV |

---

## 3. Nouveau struct `DetectionConfig`

À ajouter dans `src/BatteryModels.h` (à côté de `TechParams`) :

```cpp
struct DetectionConfig {
    float current_deadband_A;   // Dead-band courant rest/charge/décharge (A)
    float float_current_min_A;  // I_float_max = max(float_current_min_A, cap*float_current_factor)
    float float_current_factor; // Facteur capacité pour I_float_max (par défaut 0.02)
    float v_tol_pct;            // Fraction de V_nom pour V_tol (par défaut 0.015)
    float v_tol_min_V;          // Valeur plancher de V_tol (par défaut 0.1 V)
    float ocv_rest_hours;       // Repos minimum avant mesure OCV (h)
    float cycle_thr_pct;        // Seuil de comptage cycle (fraction de C_nom, par défaut 0.8)
};
```

### Intégration dans `TechParams`

```cpp
struct TechParams {
    ...
    DetectionConfig detect;   // NOUVEAU v1.4.0
    KalmanTuning    kalman;   // NOUVEAU v1.4.0 (voir §4)
    ...
};
```

### Refactorisation des fonctions concernées

- `getVoltageToleranceBand()` : remplacer `max(0.1f, V_nom*0.015f)` par `max(getParams().detect.v_tol_min_V, V_nom * getParams().detect.v_tol_pct)`.
- `detectChargeState()` : utiliser `getParams().detect.current_deadband_A` pour `in_rest`/`charging`/`discharging` et `max(getParams().detect.float_current_min_A, cap * getParams().detect.float_current_factor)` pour `I_float_max`.
- `isOcvMeasurementValid()` : remplacer `0.05f`, `0.5f` et le cas spécial LiFePO4 par `getParams().detect.current_deadband_A` et `getParams().detect.ocv_rest_hours`. Le cas spécial LiFePO4 (2 h) devient simplement une valeur `ocv_rest_hours = 2.0` dans `TECH_PARAMS[4]`.
- `updateCoulombCount()` : remplacer `C_nominal * 0.8f` par `C_nominal * getParams().detect.cycle_thr_pct`.

---

## 4. Nouveau struct `KalmanTuning`

Regroupe les paramètres que `BatteryKalman` consommera comme défauts par technologie (voir le document compagnon).

```cpp
struct KalmanTuning {
    // --- P : covariance de l'erreur (état [C_hat, dC_dCycle]) ---
    float p_init_C;           // Variance initiale pour C (Ah²)
    float p_min_C;            // Variance minimale pour C (Ah²)
    float p_init_aging;       // Variance initiale pour dC/dCycle
    float p_min_aging;        // Variance minimale pour aging

    // --- Q : bruit de processus ---
    float q_aging_uncertainty; // Incertitude sur le taux aging
    float q_capacity_drift;    // Dérive de capacité entre cycles

    // --- R : bruit de mesure (estimation en ligne) ---
    float r_init;             // R initial
    float r_min;              // R minimum
    float r_max;              // R maximum
    float r_smooth_alpha;     // Lissage exponentiel de R_estimate

    // --- Segments ΔAh / ΔSoC (apprentissage de capacité) ---
    float seg_min_dAh_abs;    // Seuil absolu minimal (Ah)
    float seg_min_dAh_pct;    // Seuil relatif minimal (% de C) — NOUVEAU
    float seg_min_dsoc[3];    // {bootstrap, coarse, refine/track} (% SoC)
    float seg_dsoc_high_conf; // dSoC pour confiance élevée (%)
    bool  ocv_segment_allowed;// false => pas de segment OCV (NiFe, Sodium)

    // --- Détection de batterie changée ---
    float   battery_change_thr;   // Écart relatif (% de C_hat)
    uint8_t battery_change_count; // Nombre d'outliers consécutifs

    // --- Porte de stabilité REST_LONG (référence OCV fiable) ---
    float   rest_long_min_min;         // Repos minimum (min)
    float   rest_long_stable_mv;       // Écart tension crête toléré (V)
    float   rest_long_stable_min;      // Fenêtre de stabilité (min)
    uint8_t rest_long_stable_min_samples; // Nb min d'échantillons dans la fenêtre
    uint8_t rest_long_stable_max_samples; // Taille max de l'anneau

    // --- Confiance ---
    float conf_float;          // Confiance segment FLOAT (0.95)
    float conf_rest_base;      // Confiance REST_LONG de base (0.70)
    float conf_rest_span;      // Bonus de stabilité REST_LONG (0.15)
    float conf_phase_coarse;   // Seuil phase COARSE (0.3)
    float conf_phase_refine;   // Seuil phase REFINE (0.6)
};
```

> **Note d'architecture** : `BatteryModels` ne fait qu'**exposer** ces valeurs (elles décrivent la chimie de la batterie). C'est `BatteryKalman` qui les **consomme**. Pour un modèle non-Kalman, elles restent inutilisées (aucune dépendance ajoutée).

---

## 5. API `BatteryModel` — getters et setters

### Getter par technologie

```cpp
const DetectionConfig& getDetectionConfig() const;   // retourne getParams().detect
const KalmanTuning&    getKalmanTuning() const;      // retourne getParams().kalman
```

> **Attention PROGMEM** : `getParams()` renvoie une référence sur le cache (`_paramsCache`), pas directement sur PROGMEM. Les getters doivent donc retourner une copie ou être basés sur `_paramsCache`. Les setters d'override doivent être stockés dans des membres d'instance (RAM), pas dans PROGMEM.

### Overrides runtime (priorité la plus haute)

Chaque setter stocke une valeur d'instance qui écrase le défaut de la technologie :

```cpp
void setCurrentDeadband(float amps);
void setVoltageTolerance(float pct, float min_V);
void setFloatCurrentMax(float min_A, float factor);
void setOcvRestHours(float hours);
void setCycleThreshold(float pct);

void setKalmanP(float init_C, float min_C, float init_aging, float min_aging);
void setKalmanQ(float aging_uncertainty, float capacity_drift);
void setKalmanR(float init, float min, float max, float smooth_alpha);
void setKalmanSegmentThresholds(float min_dAh_abs, float min_dAh_pct,
                                float dSoc_boot, float dSoc_coarse, float dSoc_track,
                                float dSoc_high_conf, bool ocv_allowed);
void setKalmanBatteryChange(float thr, uint8_t count);
void setKalmanRestLong(float min_min, float stable_mv, float stable_min,
                       uint8_t min_samples, uint8_t max_samples);
void setKalmanConfidence(float conf_float, float conf_rest_base, float conf_rest_span,
                         float conf_phase_coarse, float conf_phase_refine);

void resetOverrides();   // réinitialise tous les overrides runtime
```

### Hiérarchie de résolution (à documenter dans le header)

```
1. Override runtime (setter)       — priorité la plus haute
2. Macro compile-time BATTERYMODELS_* — si définie avant #include
3. Valeur par défaut de la technologie (TECH_PARAMS / TECH_PARAMS[...].detect|kalman)
```

### Surcharge compile-time

Toutes les macros sont optionnelles et **globales** (écrasent le défaut par technologie) :

```cpp
#ifndef BATTERYMODELS_DEADBAND_A
#define BATTERYMODELS_DEADBAND_A        0.05f
#endif
#ifndef BATTERYMODELS_OCV_REST_HOURS
#define BATTERYMODELS_OCV_REST_HOURS    0.5f
#endif
#ifndef BATTERYMODELS_CYCLE_THR_PCT
#define BATTERYMODELS_CYCLE_THR_PCT     0.8f
#endif
#ifndef BATTERYMODELS_V_TOL_PCT
#define BATTERYMODELS_V_TOL_PCT         0.015f
#endif
```

> Implémentation : les valeurs `TECH_PARAMS` restent les vraies valeurs par tech ; les macros compile-time s'appliquent dans le getter si définies ; les setters runtime écrasent tout (membres initialisés à une sentinelle `-1`/`NAN` = « non surchargé »).

---

## 6. Valeurs par défaut proposées par technologie

### 6.1 `DetectionConfig`

| Tech | deadband (A) | v_tol_pct | ocv_rest (h) | cycle_thr | Justification |
|---|---|---|---|---|---|
| UNKNOWN | 0.05 | 1.5 % | 0.5 | 0.8 | Valeurs prudentes génériques |
| FLOODED | 0.05 | 1.5 % | 1.0 | 0.8 | Polarisation lente → repos plus long |
| AGM | 0.05 | 1.5 % | 1.0 | 0.8 | Idem plomb |
| GEL | 0.05 | 1.5 % | 1.0 | 0.8 | Idem plomb |
| LIFEPO4 | 0.05 | 1.0 % | **2.0** | 0.8 | Plateau OCV plat → mesure OCV lente (conservé de v1.3) |
| LION | 0.05 | 1.5 % | 0.5 | 0.8 | OCV raide → repos court suffisant |
| LIPO | 0.05 | 1.5 % | 0.5 | 0.8 | Idem |
| LTO | 0.05 | 1.0 % | 2.0 | 0.8 | Plateau plat, faible dVocv/dSoC |
| LMNO | 0.05 | 1.5 % | 0.5 | 0.8 | Idem lithium |
| NMC | 0.05 | 1.5 % | 0.5 | 0.8 | Idem |
| NCA | 0.05 | 1.5 % | 0.5 | 0.8 | Idem |
| NIFE | 0.05 | 2.0 % | 1.0 | 0.8 | OCV non fiable (voir §6.2) |
| SODIUM | 0.05 | 1.5 % | 1.0 | 0.8 | OCV non fiable (voir §6.2) |
| NIMH | 0.05 | 1.5 % | 1.0 | 0.8 | Petite capacité, hystérésis modérée |
| ALKALINE | 0.05 | 1.5 % | 1.0 | 0.8 | Décharge seule |

### 6.2 `KalmanTuning` — jeu de défauts recommandé

| Tech | p_init_C | q_capacity_drift | r_init | dSoC {boot,course,fin} | seg_min_dAh_pct | ocv_segment | REST_LONG stable (mV) |
|---|---|---|---|---|---|---|---|
| UNKNOWN | 500 | 0.010 | 4.0 | {3, 5, 8} | 0.5 % | oui | 20 |
| FLOODED | 600 | 0.030 | 6.0 | {3, 5, 8} | 0.5 % | oui | 30 |
| AGM | 500 | 0.025 | 5.0 | {3, 5, 8} | 0.5 % | oui | 30 |
| GEL | 500 | 0.025 | 5.0 | {3, 5, 8} | 0.5 % | oui | 30 |
| LIFEPO4 | 300 | 0.008 | 8.0 | {5, 8, 10} | 0.5 % | oui | 20 |
| LION | 200 | 0.005 | 3.0 | {2, 3, 5} | 0.5 % | oui | 20 |
| LIPO | 200 | 0.005 | 3.0 | {2, 3, 5} | 0.5 % | oui | 20 |
| LTO | 300 | 0.006 | 7.0 | {5, 8, 10} | 0.5 % | oui | 20 |
| LMNO | 200 | 0.005 | 3.0 | {2, 3, 5} | 0.5 % | oui | 20 |
| NMC | 200 | 0.005 | 3.0 | {2, 3, 5} | 0.5 % | oui | 20 |
| NCA | 200 | 0.005 | 3.0 | {2, 3, 5} | 0.5 % | oui | 20 |
| NIFE | 800 | 0.030 | 12.0 | {3, 5, 8} | 0.5 % | **non** | 50 |
| SODIUM | 300 | 0.008 | 8.0 | {3, 5, 8} | 0.5 % | **non** | 20 |
| NIMH | 400 | 0.015 | 6.0 | {4, 6, 8} | **1.0 %** | oui | 30 |
| ALKALINE | 400 | 0.010 | 6.0 | {4, 6, 8} | **1.0 %** | oui | 30 |

**Champs non listés (défauts communs pour toutes les techs)** :

| Champ | Valeur | Note |
|---|---|---|
| p_min_C | 0.10 | Idem constantes actuelles |
| p_init_aging / p_min_aging | 1e-6 / 1e-7 | |
| q_aging_uncertainty | 2e-6 | |
| r_min / r_max / r_smooth_alpha | 0.5 / 100 / 0.1 | |
| seg_min_dAh_abs | 0.20 Ah | **Remplacé par** `max(seg_min_dAh_abs, C × seg_min_dAh_pct/100)` |
| seg_dsoc_high_conf | 15.0 % | |
| battery_change_thr / count | 0.30 / 3 | |
| rest_long_min / stable_ms / min_samples / max_samples | 15 min / 30 min / 6 / 32 | |
| conf_float / rest_base / rest_span | 0.95 / 0.70 / 0.15 | |
| conf_phase_coarse / refine | 0.3 / 0.6 | |

### 6.3 Justifications physiques

- **LiFePO4 / LTO** : plateau OCV très plat (20-90 % SoC) → un petit ΔSoC mesuré est très bruité : `C_measured = ΔAh/ΔSoC` explose si `dSoC` est bruité. Il faut un **ΔSoC minimum plus grand** et un **R (bruit de mesure) plus grand**, plus un repos OCV long (2 h).
- **Li-ion (LION/LIPO/LMNO/NMC/NCA)** : OCV raide et fiable → seuils ΔSoC bas, R faible, `P_INIT_C` plus petit (apprentissage rapide, haute confiance).
- **Plomb (FLOODED/AGM/GEL)** : polarisation lente après charge/décharge → **REST_LONG_STABLE plus tolérant (30 mV au lieu de 20)** car la tension plomb dérive de 30-50 mV/h ; `q_capacity_drift` plus élevé (vieillissement plomb plus marqué).
- **NiFe / Sodium** : `isOcvReliable()=false` → `ocv_segment_allowed=false` : les segments basés sur la référence OCV REST_LONG sont **exclus** ; seul le passage FLOAT (référence 100 %) alimente l'apprentissage. NiFe : hystérésis énorme → R très grand, `P_INIT_C` grand.
- **NiMH / Alkaline** : petites capacités (2-5 Ah) → `SEG_MIN_DAH = 0.20 Ah` représente jusqu'à 10 % de C. Le nouveau **seuil relatif** `seg_min_dAh_pct = 1 %` corrige ce biais. Alkaline : décharge seule, apprentissage uniquement sur décharge.
- **Alkaline** : non rechargeable — la détection `State_FLOAT` reste pertinente mais les segments REST_LONG doivent reposer sur la décharge.

---

## 7. Rétro-compatibilité et versionnage

- **API additive uniquement** : aucun champ existant de `TechParams` n'est renommé ; les nouvelles valeurs sont initialisées via `_paramsCache` avec les valeurs de `TECH_PARAMS` mises à jour.
- Le constructeur, `detectChargeState()`, `isOcvMeasurementValid()` conservent leurs signatures. Seul leur **comportement interne** devient piloté par la config.
- **Numéros de version à mettre à jour** :
  1. `src/BatteryModels.h` ligne 5 : `BatteryModels v1.4.0 - ...`
  2. `library.json` → `"version": "1.4.0"` + description
  3. `README.md` badges/callouts
  4. `CHANGELOG.md` (entrée v1.4.0 en français, tableaux)
- **Vérification** : mock `Arduino.h` + `g++ -fsyntax-only` (comme en développement), + vérifier que les 4 exemples compilent toujours.

---

## 8. Documentation à mettre à jour

- `README.md` : tableau des fonctionnalités + section configuration (macros `BATTERYMODELS_*`, setters).
- `AGENTS.md` : mise à jour des « Conventions & gotchas » et du « Adding a new battery type » (ajouter les champs `detect`/`kalman` aux étapes 4).
- `CHANGELOG.md` : entrée v1.4.0.
- Nouveau chapitre : les valeurs de `KalmanTuning` étant consommées par BatteryKalman, renvoyer vers `changements_a_effectuer_sur_BatteryKalman.md`.
