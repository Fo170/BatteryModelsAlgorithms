# Changements à effectuer sur BatteryKalman

**Version cible** : v1.6.0 (API additive, rétro-compatible avec v1.5.3)
**Date** : Août 2026
**Statut** : Spécification — à implémenter dans `src/BatteryKalman.h` (repo `Fo170/BatteryKalman`)
**Document compagnon** : `changements_a_effectuer_sur_BatteryModels.md` (BatteryModels v1.4.0 expose les valeurs par défaut par technologie)

---

## 1. Contexte

Les constantes P/Q/R et les seuils ΔAh/ΔSoC de `BatteryKalman.h` sont actuellement **globales** : un seul jeu de valeurs pour toutes les technologies, surchargeables uniquement à la compilation via `#ifndef`. Or la qualité de l'apprentissage dépend fortement de la chimie (plateau OCV plat LiFePO4/LTO, OCV non fiable NiFe/Sodium, petites capacités NiMH/Alkaline, polarisation lente plomb).

BatteryModels v1.4.0 introduit `BatteryModel::getKalmanTuning()` qui expose un `KalmanTuning` **par technologie**. BatteryKalman doit **consommer** ces défauts, tout en gardant :
- la surcharge compile-time existante (`#ifndef`),
- la surcharge runtime (nouveaux setters),
- le comportement actuel quand le modèle ne fournit pas de tuning (modèle « custom », autre librairie).

---

## 2. Inventaire des constantes actuelles (`src/BatteryKalman.h`)

| Constante | Ligne(s) | Valeur | Rôle |
|---|---|---|---|
| `KALMAN_P_INIT_C` | 48-49 | 500.0 | Variance initiale P pour C (Ah²) |
| `KALMAN_P_INIT_AGING` | 51-52 | 1e-6 | Variance initiale pour dC/dCycle |
| `KALMAN_P_MIN_C` | 54-55 | 0.10 | Variance minimale pour C |
| `KALMAN_P_MIN_AGING` | 57-58 | 1e-7 | Variance minimale pour aging |
| `Q_AGING_UNCERTAINTY` | 62-63 | 2e-6 | Incertitude taux aging |
| `Q_CAPACITY_DRIFT` | 65-66 | 0.01 | Dérive de capacité entre cycles |
| `R_INIT` / `R_MIN` / `R_MAX` / `R_SMOOTH_ALPHA` | 70-81 | 4.0 / 0.5 / 100.0 / 0.1 | Bruit de mesure + lissage |
| `SEG_MIN_DAH` | 92-93 | 0.20 Ah | Seuil absolu de fermeture de segment |
| `SEG_MIN_DSOC_P0/P1/P2` | 95-102 | 3.0 / 5.0 / 8.0 % | ΔSoC min par phase |
| `SEG_DSOC_HIGH_CONF` | 104-105 | 15.0 % | ΔSoC pour confiance élevée |
| `BATTERY_CHANGE_THR` / `COUNT` | 109-112 | 0.30 / 3 | Détection batterie changée |
| `REST_LONG_MIN_MS` | 127-128 | 15 min | Repos minimum avant référence OCV |
| `REST_LONG_STABLE_MV` | 130-131 | 0.020 V | Écart tension crête toléré |
| `REST_LONG_STABLE_MS` | 133-134 | 30 min | Fenêtre de stabilité |
| `REST_LONG_STABLE_MIN/MAX_SAMPLES` | 136-142 | 6 / 32 | Taille de l'anneau |
| Phases / confiance | 317-321, 396-397, 434-440, 517-518, 637-639 | 0.2 / 0.95 / 0.70 / 0.15 / 0.3 / 0.6 | Mapping confiance → phase, confiance segment |

Fonctions à modifier : `predictKalman()` (~275), `applyKalmanUpdate2D()` (313), `updateREstimate()` (408), `updatePhaseFromConfidence()` (434), logique REST_LONG (472-492), `updateSegmentAndKalman()` (610-668), init/reset (813-821).

---

## 3. Hiérarchie de résolution des valeurs

À implémenter de façon centralisée (une fonction `getTuning()` interne) :

```
1. Override runtime (setters)              — priorité la plus haute
2. Modèle : model->getKalmanTuning()       — si le modèle l'expose (BatteryModels v1.4+)
3. Macro compile-time BATTERYKALMAN_*      — si définie avant #include (comportement actuel)
4. Défaut interne                          — valeurs actuelles (inchangées)
```

### Détection de la disponibilité du tuning

Le modèle (interface abstraite) ne possède pas forcément `getKalmanTuning()`. Deux approches possibles :

- **A (recommandée)** : la classe `BatteryKalman` est construite avec un `BatteryModel*` concret. Ajouter une méthode à l'interface abstraite : `virtual const KalmanTuning* getKalmanTuning() const { return nullptr; }` (défaut `nullptr` = non disponible). BatteryModels v1.4+ surcharge et renvoie l'adresse d'un objet statique/copiable. Tout modèle existant compile sans modification.
- **B (sans toucher à l'interface)** : essai de détection par `dynamic_cast`/préprocesseur — fragile, à éviter.

Le `KalmanTuning` doit être **copié en interne** au premier accès (pas conservé comme pointeur sur le modèle, pour éviter les fuites si le modèle est réaffecté).

---

## 4. Nouvelle API publique (setters runtime)

```cpp
void setTuning(const KalmanTuning& t);          // remplace tout le tuning runtime

void setP(float init_C, float min_C, float init_aging, float min_aging);
void setQ(float aging_uncertainty, float capacity_drift);
void setR(float init, float min, float max, float smooth_alpha);
void setSegmentThresholds(float min_dAh_abs, float min_dAh_pct,
                          float dSoc_boot, float dSoc_coarse, float dSoc_track,
                          float dSoc_high_conf, bool ocv_allowed);
void setBatteryChange(float thr, uint8_t count);
void setRestLong(float min_minutes, float stable_mv, float stable_minutes,
                 uint8_t min_samples, uint8_t max_samples);
void setConfidence(float conf_float, float conf_rest_base, float conf_rest_span,
                   float conf_phase_coarse, float conf_phase_refine);

void resetTuning();   // revient à la hiérarchie modèle > macro > défaut
```

> Les setters écrasent aussi les valeurs déjà lues depuis le modèle. `resetTuning()` repasse en mode automatique.

---

## 5. Modifications fonctionnelles

### 5.1 Utilisation du tuning partout où les constantes sont lues

Remplacer chaque usage de constante par le champ correspondant de `getTuning()` :

| Usage actuel | Ligne(s) | Remplacé par |
|---|---|---|
| `KALMAN_P_INIT_C` (init P, reset, p_factor) | 194, 307, 396, 813 | `t.p_init_C` |
| `KALMAN_P_MIN_C` | 378-384 | `t.p_min_C` |
| `KALMAN_P_INIT_AGING` / `P_MIN_AGING` | 194-195, 381 | `t.p_init_aging` / `t.p_min_aging` |
| `KALMAN_P_MIN_C` | 384 | `t.p_min_C` |
| `Q_AGING_UNCERTAINTY` | 304-305, 318, 349, 361 | `t.q_aging_uncertainty` |
| `Q_CAPACITY_DRIFT` | 304 | `t.q_capacity_drift` |
| `R_INIT` | 198-199, 820-821 | `t.r_init` |
| `R_MIN` / `R_MAX` | 418, 423 | `t.r_min` / `t.r_max` |
| `R_SMOOTH_ALPHA` | 421-422 | `t.r_smooth_alpha` |
| `SEG_MIN_DAH` | 652 | `max(t.seg_min_dAh_abs, C_hat × t.seg_min_dAh_pct/100)` (voir §5.3) |
| `SEG_MIN_DSOC_P0/P1/P2` | 648-650 | `t.seg_min_dsoc[phase]` |
| `SEG_DSOC_HIGH_CONF` | 104-105 (défini, non utilisé actuellement) | `t.seg_dsoc_high_conf` (à brancher ou documenter) |
| `BATTERY_CHANGE_THR` / `COUNT` | 344-346 | `t.battery_change_thr` / `t.battery_change_count` |
| `REST_LONG_*` | 492, 517-518, 637-639 | `t.rest_long_*` |
| confiance FLOAT / REST_LONG | 613, 634, 637-639 (+ syncs 512, 525, 568, 794) | `t.conf_float` / `t.conf_rest_base` / `t.conf_rest_span` |
| seuils de phase | 438-440 | `t.conf_phase_coarse` / `t.conf_phase_refine` |

### 5.2 Gating des segments OCV (NiFe, Sodium)

Actuellement, un segment est fermé par deux références : passage `State_FLOAT` (SoC=100 %, fiable partout) et passage `State_REST_LONG` (référence OCV). Pour NiFe/Sodium (`model->isOcvReliable() == false`), la référence OCV est fausse → `C_measured = ΔAh/ΔSoC` est faussé.

**Changement** dans `updateSegmentAndKalman()` (ligne 629-646) : fermer un segment par REST_LONG **uniquement si** `t.ocv_segment_allowed == true` (ou, à défaut, si `model->isOcvReliable()`). La fermeture par FLOAT reste toujours autorisée (référence 100 % fiable).

```cpp
} else if (mppt_state == State_REST_LONG && mppt_state_prev != State_REST_LONG) {
    if (!getTuning().ocv_segment_allowed) {  // NiFe/Sodium : ne pas utiliser la référence OCV
        startNewSegment(data->SoC_fused, 0.5f);
        return;
    }
    ...
}
```

> `isOcvUnreliable()` (ligne 855) existe déjà : `return !model->isOcvReliable();`. Réutiliser ce prédicat comme fallback si le tuning n'expose pas `ocv_segment_allowed`.

### 5.3 Seuil ΔAh relatif (petites batteries NiMH/Alkaline)

`SEG_MIN_DAH = 0.20 Ah` représente ~10 % d'une batterie AA (2 Ah). Remplacer le test de la ligne 652 :

```cpp
// Avant
if (dAh < SEG_MIN_DAH || dSoC < min_dSoC || seg.n == 0) {

// Après
float min_dAh = max(getTuning().seg_min_dAh_abs,
                    kalman->C_hat * getTuning().seg_min_dAh_pct / 100.0f);
if (dAh < min_dAh || dSoC < min_dSoC || seg.n == 0) {
```

### 5.4 REST_LONG : paramètres temporels en minutes → millisecondes

Le `KalmanTuning` de BatteryModels expose `rest_long_min_min` / `rest_long_stable_min` en **minutes** (portables) ; BatteryKalman les convertit en `ms` (`× 60000UL`) au chargement du tuning, pour rester cohérent avec `millis()`.

---

## 6. Rétro-compatibilité et versionnage

- **Aucune constante supprimée** : les `#ifndef` restent, ils servent désormais de **niveau 3** de la hiérarchie (fallback). Un modèle sans `getKalmanTuning()` et sans setter runtime → **comportement v1.5.3 identique**.
- Le constructeur `BatteryKalman(SoCData*, KalmanState2D*, BatteryModel*, Coulomb*)` est inchangé.
- **Numéros de version à mettre à jour** :
  1. `src/BatteryKalman.h` en-tête : `Version: 1.6.0`
  2. `library.json` de BatteryKalman → `"version": "1.6.0"` + description
  3. `README.md` de BatteryKalman (badges/callouts)
  4. `CHANGELOG.md` de BatteryKalman (entrée v1.6.0)
- **Dépendance** : BatteryKalman v1.6.0 fonctionne avec BatteryModels **≥ 1.3.0** (tuning optionnel) et tire le meilleur parti de **≥ 1.4.0**.

---

## 7. Documentation à mettre à jour

- `MIGRATION.md` : chapitre « v1.5.3 → v1.6.0 » (nouveaux setters, hiérarchie, aucun breaking change).
- `README.md` : section configuration (setters runtime + macros).
- `BATTERYMODELS_COMPATIBILITY.md` : mentionner le pont `getKalmanTuning()` (BatteryModels v1.4+).
- Renvoyer vers `changements_a_effectuer_sur_BatteryModels.md` pour les valeurs par technologie.

---

## 8. Tests de validation recommandés

1. **Régression** : comportement v1.5.3 sans setter ni modèle v1.4 (mêmes courbes d'apprentissage).
2. **LiFePO4** : segment avec `dSoC` petit doit être rejeté (`seg_min_dsoc = {5,8,10}`), `R` plus grand → convergence plus lente mais stable.
3. **NiFe/Sodium** : aucun segment REST_LONG ne doit fermer ; seuls les passages FLOAT alimentent le Kalman.
4. **NiMH 2 Ah** : un segment de 0.10 Ah doit être **rejeté** (seuil relatif 1 % = 0.02 Ah, mais le seuil absolu 0.20 Ah… vérifier que `max(0.20, C×1%)` = 0.20 → rejeté), et un segment de 0.25 Ah accepté.
5. **Plomb** : `REST_LONG_STABLE_MV = 30 mV` tolère la dérive lente → la référence REST_LONG est atteinte plus souvent qu'avec 20 mV.
