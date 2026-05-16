# 🛁 SPA Filtration — Home Assistant Blueprint

**Gestion intelligente de la filtration du SPA Bestway Lay-Z-SPA**  
Blueprint Home Assistant — automatisation complète basée sur la température, l'utilisation réelle et la plage horaire.

---

## ✨ Fonctionnalités

| Fonctionnalité | Description |
|---|---|
| 🌡️ **3 paliers de température** | Durée de filtration adaptée à la température de l'eau |
| 🔄 **Cycles dynamiques** | Recalcul à chaque cycle selon la température réelle |
| 💧 **Bonus utilisation** | Filtration prolongée selon le temps réel d'utilisation des bulles |
| 🌙 **Plage nocturne** | Fonctionne sur 22h→8h sans coupure à minuit |
| ⏭️ **Skip coupure intelligent** | Pompe et power maintenus si chauffage ou bulles actifs |
| 🏊 **Cycle bonus final** | Filtration supplémentaire automatique après utilisation des bulles |
| 🔌 **Rattrapage WiFi** | Cycle de rattrapage après reconnexion ESP8266 |
| 🛡️ **Watchdog sécurité** | Alerte si pompe allumée anormalement longtemps |
| 🚨 **Stop d'urgence** | Arrêt immédiat via bouton dédié |
| 🔧 **Vérification entités** | Contrôle au démarrage des entités manquantes |

---

## 📦 Fichiers

```
spa_filtration.yaml          # Blueprint principal — gestion filtration
spa_bulles_auto_off.yaml     # Blueprint compagnon — coupure automatique des bulles
configuration.yaml           # Entités internes requises (à ajouter à votre config HA)
```

> **Pourquoi deux blueprints ?**  
> La coupure automatique des bulles est une sécurité (enfants, oubli) qui doit fonctionner
> même pendant un long cycle de filtration. En séparant les deux automations, chacune
> fonctionne de façon indépendante et fiable.

---

## 🔧 Installation

### Étape 1 — Entités internes

Ajoutez ce bloc dans votre `configuration.yaml` :

```yaml
input_number:
  spa_snapshot_pump_time:
    name: "SPA — Snapshot pompe"
    min: 0
    max: 99999
    step: 0.001
    mode: box
  spa_snapshot_air_time:
    name: "SPA — Snapshot bulles"
    min: 0
    max: 99999
    step: 0.001
    mode: box
  spa_duree_objectif:
    name: "SPA — Objectif filtration (min)"
    min: 0
    max: 1440
    step: 1
    mode: box
  spa_bonus_minutes:
    name: "SPA — Bonus minutes"
    min: 0
    max: 480
    step: 1
    mode: box
  spa_cycle_en_cours:
    name: "SPA — Cycle en cours"
    min: 0
    max: 8
    step: 1
    mode: box

input_boolean:
  spa_snapshot_fait:
    name: "SPA — Snapshot effectue"
  spa_stoppe:
    name: "SPA — Arret urgence actif"
  spa_recovery_running:       # NOUVEAU v2.7
    name: "SPA — Recovery en cours"

input_datetime:
  spa_fin_cycle_prevu:        # NOUVEAU v2.6
    name: "SPA — Fin de cycle prevue"
    has_date: true
    has_time: true
  spa_fin_pause_prevue:       # NOUVEAU v2.7
    name: "SPA — Fin de pause prevue"
    has_date: true
    has_time: true

input_select:
  spa_state:                  # NOUVEAU v2.7 — remplace spa_cycle_actif
    name: "SPA — Etat"
    options:
      - idle
      - filtering
      - pause
      - bonus
    initial: idle

input_button:
  spa_reset:
    name: "SPA — Reset manuel"
    icon: mdi:restart
  spa_stop:
    name: "SPA — Stop urgence"
    icon: mdi:stop-circle-outline
```

### Étape 2 — Importer les blueprints

#### Via URL (recommandé)

[![Importer spa_filtration](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https://github.com/thed6bel/Spa-Filtration/blob/main/spa_filtration.yaml)

[![Importer spa_bulles_auto_off](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https://github.com/thed6bel/Spa-Filtration/blob/main/spa_bulles_auto_off.yaml)

#### Manuellement

1. Copiez les fichiers `.yaml` dans `/config/blueprints/automation/`
2. Redémarrez Home Assistant ou rechargez les blueprints

### Étape 3 — Créer les automations

1. **Paramètres → Automatisations → Créer une automatisation → Depuis un blueprint**
2. Sélectionnez **"Gestion filtration SPA Bestway Lay-Z-SPA"**
3. Configurez vos entités et paramètres
4. Répétez pour **"SPA — Coupure automatique des bulles"**

---

## ⚙️ Paramètres principaux

### Entités requises (SPA Bestway Lay-Z-SPA)

| Paramètre | Entité type | Description |
|---|---|---|
| Interrupteur power | `switch.layzspa_power_switch` | Alimentation principale |
| Pompe filtration | `switch.layzspa_pump` | Pompe de filtration |
| Chauffage actif | `binary_sensor.layzspa_heater` | État du chauffage |
| Bulles | `switch.layzspa_airbubbles` | Interrupteur bulles |
| Température eau | `sensor.layzspa_temp_c` | Température en °C |
| Temps pompe | `sensor.layzspa_pump_time` | Compteur cumulatif (heures) |
| Temps bulles | `sensor.layzspa_air_time` | Compteur cumulatif (heures) |
| Connexion WiFi | `binary_sensor.layzspa_connection` | État connexion ESP8266 |

### Paliers de filtration (valeurs par défaut)

| Température | Durée quotidienne |
|---|---|
| < 25°C | 4h (240 min) |
| 25°C – 35°C | 6h (360 min) |
| ≥ 35°C | 8h (480 min) |

> Tous les paramètres sont configurables dans l'interface du blueprint.

### Bonus utilisation

| Durée bulles | Bonus filtration |
|---|---|
| < 10 min | Aucun (seuil anti-accidentel) |
| 10 – 30 min | +60 min (configurable) |
| ≥ 30 min | +120 min (configurable) |

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────┐
│           spa_filtration.yaml               │
│                                             │
│  Triggers : heure_debut, bulles_off,        │
│             reset, stop, connexion,         │
│             power_on                        │
│                                             │
│  Mode : queued (max: 2)                     │
│                                             │
│  ┌─────────────────────────────────────┐    │
│  │  Boucle repeat N cycles             │    │
│  │  - Recalcul température             │    │
│  │  - Calcul bonus air_time            │    │
│  │  - Durée min cycle (15 min)         │    │
│  │  - Skip coupure si chauffage/bulles │    │
│  │  - Watchdog fin de session          │    │
│  └─────────────────────────────────────┘    │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│         spa_bulles_auto_off.yaml            │
│                                             │
│  Trigger : bulles ON                        │
│  Mode : restart                             │
│  → Coupe les bulles après X minutes         │
│  → Fonctionne indépendamment de la          │
│    filtration (même pendant un long delay)  │
└─────────────────────────────────────────────┘
```

---

## 📋 Changelog


Toutes les modifications importantes de ce blueprint SPA sont documentées ici.

---

## [2.7] - 2026-05-16
### Added
- Remplacement de `input_boolean.spa_cycle_actif` par `input_select.spa_state` (idle / filtering / pause / bonus)
- Persistance de l’état complet après reboot HA
- Ajout `input_datetime.spa_fin_pause_prevue` pour gestion des pauses
- Verrou `input_boolean.spa_recovery_running` pour éviter les doubles recoveries
- Garde-fou “zombie state” avec reset sécurité après reboot prolongé

### Changed
- Reprise après reboot étendue aux états pause / bonus / filtering
- Sécurisation des triggers `power_on` et `connexion_retablie`

### Removed
- `input_boolean.spa_cycle_actif`

---

## [2.6] - 2026-05-15
### Added
- Reprise automatique après reboot Home Assistant (cycle / bonus)
- Snapshot persistant du cycle (`spa_fin_cycle_prevu`)
- Protection contre conflit de recovery au démarrage

---

## [2.5] - 2026-05-15
### Added
- Optimisation logique temps (`v_now_min`)
- Fallback température capteur KO → mode sécurité (filtration max)
- Paramètre `pause_min_off` (temps OFF minimum entre cycles)
- `notifications_debug` activable/désactivable

---

## [2.4] - 2026-05-15
### Added
- Notification de début de cycle enrichie
- Reset automatique du snapshot en fin de session

---

## [2.3] - 2026-05-14
### Fixed
- Correction notification de fin de session manquante
- Clarification logique de coupure SPA

---

## [2.2] - 2026-05-14
### Fixed
- Correction erreur blueprint Home Assistant (`choose` requis)

---

## [2.1] - 2026-05-14
### Fixed
- Bug critique : pompe non arrêtée après activation manuelle SPA
- Ajout arrêt explicite dans `power_on`

---

## [2.0] - 2026-05-13
### Added
- Mode `queued` (gestion des triggers pendant delays)
- Séparation du contrôle des bulles en blueprint indépendant
- Watchdog sécurité pompe
- Durées minimum cycle + seuil bonus renforcé
- Fallback température
- Notifications debug optionnelles

---

## [1.0] - Initial
### Added
- Gestion filtration multi-cycles SPA
- Plages horaires (y compris nocturnes)
- Bonus bulles
- Reprise après perte WiFi
- Stop d’urgence

---

## 🔌 Compatibilité

- **Home Assistant** : 2024.8.0+
- **SPA** : Bestway Lay-Z-SPA (via intégration Tasmota / ESP8266)
- **Intégration** : [WiFi-remote-for-Bestway-Lay-Z-SPA](https://github.com/visualapproach/WiFi-remote-for-Bestway-Lay-Z-SPA/tree/master)

---

## 📄 Licence

MIT — libre d'utilisation, de modification et de distribution.

---

*Développé avec ❤️ pour automatiser la filtration du SPA sans y penser.*
