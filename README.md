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

## [2.8.2] 
### Corrections v2.8.2
- Recovery : coupure finale conditionnelle (comme boucle principale).
  Bug : le blueprint coupait pompe et power inconditionnellement en fin de recovery,
  sans verifier si le chauffage ou les bulles etaient actifs (variable skip_coupure
  inexistante dans le recovery). Ajout de skip_coupure_reboot avant la coupure finale.

## [2.8.1] 
### Corrections v2.8.1
- Condition entree recovery changee de restant >= v_duree_min a restant > 0.
  Bug : 14 min restantes avec duree_min=15 declenchait "objectif deja atteint"
  alors que duree_par_cycle_reboot applique deja le minimum via | max v_duree_min.

## [2.8.0] 
### Corrections v2.8.0
- ha_start pose spa_recovery_running=ON immediatement (avant le delai de 30s)
  pour bloquer le trigger power_on qui se declenche en meme temps au reboot
  et qui ecrasait le snapshot existant (bug : minutes_filtrees = 0).
- power_on : condition supplementaire fenetre 120s apres demarrage HA
  comme filet de securite supplementaire.

## [2.7.8] 
### Corrections v2.7.8
- Recovery apres reboot : au lieu d un seul bloc "rattrapage",
  recalcul des cycles restants (via spa_cycle_en_cours) et repartition
  des minutes restantes sur ces cycles avec intervalles proportionnels.
  Exemple : 2 cycles restants sur 60 min objectif, 30 min filtrees
  2 cycles de 15 min chacun avec intervalle calcule sur la plage restante.
- Recalcul en temps reel au debut de chaque cycle de rattrapage pour
  absorber les ecarts (comme la boucle principale).

## [2.7.7] 
### Corrections v2.7.7

- Recovery apres reboot : au lieu d un seul bloc "rattrapage",
  recalcul des cycles restants (via spa_cycle_en_cours) et repartition
  des minutes restantes sur ces cycles avec intervalles proportionnels.
  Exemple : 2 cycles restants sur 60 min objectif, 30 min filtrees →
  2 cycles de 15 min chacun avec intervalle calcule sur la plage restante.
- Recalcul en temps reel au debut de chaque cycle de rattrapage pour
  absorber les ecarts (comme la boucle principale).

## [2.7.6] 
### fix bug lors du reboot

- Remplacement de toute la logique de recovery complexe (datetimes, timestamps,
  fenetre inter-cycle, spa_state, spa_cycles_restants, spa_session_start)
  par une approche simple et fiable.
- Au demarrage HA : si on est dans la plage horaire ET qu une session etait
  active (spa_snapshot_fait=on), on coupe la pompe proprement, on attend
  pause_min_off minutes, puis on relance un cycle de rattrapage base sur
  les minutes reellement filtrees (pump_time - snapshot).
- Suppression des entites devenues inutiles :
    input_number.spa_cycles_restants
    input_number.spa_session_start
    input_datetime.spa_fin_cycle_prevu
    input_datetime.spa_fin_pause_prevue
  Ces entites peuvent etre supprimees de configuration.yaml.
- spa_state conserve (idle/filtering/pause/bonus) car utile pour le dashboard.
- Une seule instance de recovery possible grace au verrou spa_recovery_running.

## [2.7.5] 
### fix bug lors du reboot

- BUG FONDAMENTAL resolu : spa_fin_pause_prevue et spa_fin_cycle_prevu
  pouvaient contenir des valeurs perimees d une session precedente.
  Le blueprint calculait secondes_fin_pause < -7200 = false donc
  tombait dans le cas "pause depassee" et relancait immediatement.
- Ajout de input_number.spa_session_start : timestamp Unix du debut
  de session. Ecrit au demarrage de chaque session.
- Toute datetime de pause ou de cycle est consideree valide UNIQUEMENT
  si elle est posterieure au debut de la session courante.
  Une valeur perimee est donc systematiquement ignoree.
- Reset de spa_fin_pause_prevue et spa_fin_cycle_prevu au debut de
  chaque session pour eliminer les vieilles valeurs.

## [2.7.4] 
### fix bug lors du reboot

- BUG : spa_fin_pause_prevue ecrite APRES spa_state=pause. Si reboot entre
  les deux, la datetime etait invalide (-1) et le blueprint lancait
  immediatement le cycle suivant sans respecter la pause.
  Correction : spa_fin_pause_prevue ecrite EN PREMIER, avant spa_state=pause.
- Recovery pause avec datetime invalide (-1) : applique v_pause_min_off
  comme delai minimal plutot que de lancer immediatement.
- Meme correction appliquee a spa_fin_cycle_prevu : datetime ecrite avant
  spa_state=filtering pour la meme raison.

## [2.7.3] 
### fix bug lors du reboot

- BUG DEFINITIF resolu : etat_au_reboot toujours idle meme pendant la pause.
  Cause racine : le passage a idle en fin de pause etait encore present dans
  le code malgre le fix v2.7.2. Supprime definitivement.
- Ajout de input_number.spa_cycles_restants : persiste le nombre de cycles
  restants a executer. Permet une recovery robuste sans dependre uniquement
  du spa_state. Mis a jour avant chaque cycle, decrement apres chaque cycle.
- Recovery pause : utilise spa_cycles_restants pour savoir combien de cycles
  lancer apres la pause, meme si spa_state etait idle au reboot.
- Le spa_state reste sur pause jusqu a ce que le cycle suivant set filtering.
  Aucun passage par idle entre fin de pause et debut du cycle suivant.

Nouvelle entite requise dans configuration.yaml
  input_number.spa_cycles_restants

## [2.7.2] 
### fix bug lors du reboot

- BUG : reprise ignoree si reboot survient entre fin de pause et demarrage
  du cycle suivant. Le state repassait a idle apres le delay de pause mais
  avant que filtering soit ecrit par le cycle suivant.
  Correction : suppression du passage a idle en fin de pause. L etat pause
  est maintenu jusqu au prochain set filtering. La fenetre dangereuse
  est eliminee.
- Cas pause dans recovery renforce : si secondes_fin_pause <= 0 mais
  que des cycles restent, un cycle de rattrapage est lance immediatement.
    
## [2.7.1] 
### Corrigé

BUG CRITIQUE : TypeError: can't subtract offset-naive and offset-aware datetimes
as_datetime() sur une valeur de input_datetime retourne un datetime sans
timezone (naive), alors que now() retourne un datetime avec timezone (aware).
Python refuse de les soustraire, ce qui faisait planter toute la logique de
reprise au démarrage de HA.
Correction : as_local(as_datetime(...)) appliqué aux deux calculs de secondes
restantes (secondes_fin_cycle et secondes_fin_pause).

## [2.7]
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

## [2.6]
### Added
- Reprise automatique après reboot Home Assistant (cycle / bonus)
- Snapshot persistant du cycle (`spa_fin_cycle_prevu`)
- Protection contre conflit de recovery au démarrage

---

## [2.5]
### Added
- Optimisation logique temps (`v_now_min`)
- Fallback température capteur KO → mode sécurité (filtration max)
- Paramètre `pause_min_off` (temps OFF minimum entre cycles)
- `notifications_debug` activable/désactivable

---

## [2.4]
### Added
- Notification de début de cycle enrichie
- Reset automatique du snapshot en fin de session

---

## [2.3]
### Fixed
- Correction notification de fin de session manquante
- Clarification logique de coupure SPA

---

## [2.2]
### Fixed
- Correction erreur blueprint Home Assistant (`choose` requis)

---

## [2.1]
### Fixed
- Bug critique : pompe non arrêtée après activation manuelle SPA
- Ajout arrêt explicite dans `power_on`

---

## [2.0]
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
