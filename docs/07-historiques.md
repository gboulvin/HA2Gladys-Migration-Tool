# 07 — Import des historiques

- **Statut** : 📝 Brouillon
- **Dernière mise à jour** : 2026-11-19
- **Prérequis** : [00 — Vision](00-vision.md) (§ 9 ter, D22 à D24), [01 — Architecture](01-architecture.md) (§ 5.5, § 8), [02 — Lecture HA](02-extraction-ha.md) (§ 5), [03 — Correspondances](03-mapping-devices.md) (§ 3.4)
- **Notes de recherche** : [sources HA](research/ha-data-sources.md) (« R § x »), [code Gladys](research/gladys-codebase.md)

Les références au code Gladys sont relatives au dépôt `GladysAssistant/Gladys` (commit `71edb17`).

---

## 1. Le point bloquant : la purge de Gladys

Gladys **supprime chaque jour** les états plus anciens que la durée de conservation réglée par l'utilisateur. Cette durée est la variable système `DEVICE_STATE_HISTORY_IN_DAYS` (`server/lib/device/device.purgeStates.js`) :
- elle vaut **90 jours** par défaut, à la création du compte (`front/src/actions/signup/signupSetPreferences.js`) ;
- elle se règle dans Paramètres → Système (`SettingsSystemKeepDeviceHistory.jsx`) ;
- `-1` désactive la purge.

**Conséquence** : sans précaution, un historique HA de plusieurs années serait importé, puis **effacé à la purge suivante** pour tout ce qui dépasse 90 jours. D24 (« tout garder ») serait vidé de son sens.

**Règle (H1)** :
1. Avant l'import, l'outil compare la date du plus ancien point à importer à la durée de conservation.
2. Si elle la dépasse, l'utilisateur choisit sur l'écran 8 (05, « Conservation de l'historique ») :
   - **passer la conservation à « illimitée »** (choix par défaut, puisque D24 demande de tout garder ; H1) ;
   - ou l'**allonger** jusqu'à couvrir l'historique importé ;
   - ou **garder son réglage**. Les points trop anciens ne sont alors **pas importés**, plutôt qu'importés puis purgés : on évite une écriture inutile.
3. C'est une **modification d'un réglage existant** : l'ancienne valeur est enregistrée dans le journal et **restaurée en cas d'annulation** (A9, 01 § 5.4 règle 2). Si la valeur restaurée est plus courte, la purge quotidienne de Gladys supprimera ensuite l'historique conservé au-delà (cas des appareils préexistants, § 6) : l'écran d'annulation le précise.
4. L'estimation d'espace disque (§ 5) tient compte du choix.

## 2. Ce qui est importé

| Source HA | Contenu | Rétention HA | Import |
|---|---|---|---|
| États bruts (`states`) | Chaque changement de valeur | 10 jours par défaut (R § 5.1) | Valeurs numériques uniquement |
| Statistiques horaires (`statistics`) | Moyenne, minimum, maximum (mesures) ; index et cumul (compteurs) | **Indéfinie**, pour les capteurs ayant une `state_class` (R § 5.5) | Une valeur par heure (§ 3) |
| Statistiques 5 min | Agrégats sur 5 minutes | Comme les états | Non : elles doublonnent les états bruts |
| Statistiques externes | Données de fournisseurs, sans entité | Indéfinie | Hors V1 (02 § 5.2) |

**Seules les fonctions effectivement migrées** reçoivent un historique : le lien entité HA → feature Gladys vient de 03 § 3.4. Une fonction dont `keep_history` vaut faux dans Gladys n'en reçoit pas. L'outil le signale et propose de l'activer : c'est une modification journalisée, restaurée à l'annulation (01 § 5.4, règle 2).

## 3. Règles de conversion

### 3.1 Raccord entre les deux niveaux

Pour une fonction donnée :
- **avant** la date du plus ancien état brut conservé par HA, on importe les **statistiques horaires** ;
- **à partir** de cette date, on importe les **états bruts**.

On obtient une série continue, sans recouvrement : un point par heure pour le passé lointain, le détail réel pour les derniers jours.

### 3.2 Par type de capteur

| `state_class` HA | Valeur importée par heure | Remarque |
|---|---|---|
| `measurement` | `mean` | Les `min` et `max` ne sont pas repris : Gladys n'a qu'une valeur par point, et la moyenne suffit (H3). |
| `measurement_angle` | `mean` (moyenne circulaire calculée par HA) | Direction du vent, par exemple |
| `total`, `total_increasing` | `state` (index brut du compteur), voir § 3.3 | |
| Sans `state_class` | Pas de statistiques : seuls les états bruts (10 jours) | Typiquement les binaires, interrupteurs, ouvertures |

**Horodatage** : une statistique horaire couvre `[start, start + 1 h[`. Elle est datée au **début** de l'heure (`start`), comme dans HA.

### 3.3 Compteurs d'énergie

Côté HA, un compteur porte deux séries (R § 5.5) :
- `state` : l'index réel, qui peut revenir à zéro ;
- `sum` : un cumul qui traverse les remises à zéro, mais qui part de zéro à la première mesure de HA et ne correspond donc pas à l'index réel.

Côté Gladys, une fonction d'index alimente, par le service `energy-monitoring`, des fonctions **dérivées** : consommation par 30 minutes et coût.

**Règle (H2)** :
1. On importe l'**index réel** (`state`) dans la fonction d'index Gladys. Ainsi, la première valeur reçue en direct après la migration continue la série sans saut. Importer `sum` créerait une marche brutale entre la dernière valeur importée et la première valeur réelle.
2. Les remises à zéro sont déjà gérées par Gladys : un écart négatif n'est pas compté (`server/services/energy-monitoring/lib/energy-monitoring.calculateEnergyFromIndex.js`). On perd la consommation de la seule heure de la remise à zéro.
3. **On n'importe pas** directement les consommations ni les coûts. Après l'import, l'outil **déclenche le recalcul depuis le début**, que Gladys sait déjà faire (`calculateConsumptionFromIndexFromBeginning`, `calculateCostFromBeginning`). Gladys produit alors ses propres séries dérivées, cohérentes avec ses contrats d'énergie.
4. **Production** (solaire, T7) : même règle pour un index de **production**. On importe l'index réel dans la fonction d'index de production, puis l'outil déclenche `calculateProductionFromIndexFromBeginning` (`server/services/energy-monitoring/lib/energy-monitoring.calculateProductionFromIndexFromBeginning.js`, tâche `ENERGY_MONITORING_PRODUCTION_FROM_INDEX_BEGINNING` : `server/utils/constants.js:2549`). *Ajouté le 2026-11-19 : 08 § 2.1 suppose déjà ces compteurs.*
5. **Granularité** : avec un index horaire, la consommation à 30 minutes calculée par Gladys tombera sur une demi-heure sur deux. Le total par heure, par jour et par mois reste exact. *À valider sur une instance réelle (Q07-1).*

### 3.4 Unités

- Les valeurs sont lues dans l'**unité de stockage** de HA (`statistics_unit_of_measurement` en SQL, ou paramètre `units` explicite par l'API, 02 § 5.5), puis converties vers l'unité de la fonction Gladys.
- **Conversion inconnue** : l'historique de cette fonction n'est pas importé, et c'est signalé. On n'importe jamais de valeurs dans une mauvaise unité.
- Les transformations déclarées dans le catalogue s'appliquent aussi à l'historique (03 § 3.4 : `invert` pour les ouvertures, `scale`).

### 3.5 États binaires et textuels

- **Binaires** (`on`/`off`, `open`/`closed`…) : convertis en `1`/`0` selon la catégorie Gladys, avec l'inversion éventuelle déclarée. Ils n'ont pas de statistiques : seuls les 10 derniers jours (états bruts) sont disponibles.
- **Textuels** (modes, scènes de lecture…) : non importés. Gladys n'historise que des nombres.
- **`unknown`, `unavailable`** : écartés. Ce sont des trous dans la série, pas des valeurs.

## 4. Écriture dans Gladys

| Opération | Fonction du cœur | Remarque |
|---|---|---|
| Insertion par lots, **par ajout seulement** | `device.saveMultipleHistoricalStates(featureId, states)` | Écriture DuckDB groupée (`db.duckDbBatchInsertState`), découpée en paquets de 10 000 états, chacun écrit par **une seule** instruction `INSERT` (`server/models/index.js:400-407`). L'outil envoie donc des lots de **10 000 états au plus** : chaque lot est atomique. |
| Plus ancien état propre à Gladys | `device.getOldestStateFromDeviceFeatures` | Borne de l'import pour un appareil qui existait déjà (01 § 5.5) |
| ~~Relance sur une période déjà importée~~ | ~~`device.replaceHistoricalStatesFrom(featureId, from, states)`~~ | **Retiré le 2026-11-19 (F1)** : la fonction supprime **tous** les états à partir de `from`, sans borne de fin, avant de réinsérer (`server/models/index.js:424`). Sur un appareil que Gladys enregistre déjà, elle effacerait l'historique propre à Gladys. |

**Fonctions du cœur volontairement non utilisées** : `device.replaceHistoricalStatesFrom` (ci-dessus) et `device.destroyStatesFrom`, qui supprime elle aussi tout à partir d'une date (`server/lib/device/device.destroyStatesFrom.js:19-23`). Aucune ne permet de viser une plage bornée. L'absence de doublons repose donc sur des **bornes d'import strictes** (§ 6), et non sur un remplacement.

**Points d'attention vérifiés dans le code :**
- `saveMultipleHistoricalStates` appelle `saveHistoricalState` avec le **dernier** point du lot. Celui-ci ne met à jour la valeur courante de la fonction (`last_value`) **que s'il est plus récent** que la valeur actuelle. L'import d'un passé ancien n'écrase donc pas la valeur en direct (`server/lib/device/device.saveHistoricalState.js`).
- Les lots sont triés par date. On les envoie **du plus ancien au plus récent**, par fonction.

**Taille des lots et rythme** : 10 000 points par lot au plus (taille interne de Gladys, ci-dessus), avec une courte pause entre deux lots pour ne pas monopoliser DuckDB, que Gladys utilise en continu. **L'instance doit rester accessible en permanence pendant l'import.** Le rythme suit donc les pratiques du cœur pour ses propres calculs, imports et purges, que Pierre-Gilles réajuste régulièrement. Les paramètres (taille de lot, sous ce plafond, et pause) sont des constantes du service, faciles à revoir. *À calibrer par mesure (Q07-1).*

**Graphiques : aucun recalcul à déclencher (H5).** Depuis le passage à DuckDB, les graphiques sont calculés **à la lecture**, directement sur `t_device_feature_state` (`server/lib/device/device.getDeviceFeaturesAggregates.js`). Les états insérés dans le passé y apparaissent donc sans autre action. La table `t_device_feature_state_aggregate` et les types de tâches `HOURLY`, `DAILY` et `MONTHLY_DEVICE_STATE_AGGREGATE` existent encore dans le cœur, mais ne sont plus utilisés : ce sont des **vestiges** de l'ancien stockage (confirmé par le porteur du projet le 2026-11-19). L'outil ne les utilise pas.

## 5. Estimation du volume et de la durée (D24)

1. **Comptage** des points par fonction, selon la source (02 § 5.3) : `COUNT` SQL sur la base de la sauvegarde, `recorder/database_usage` (HA ≥ 2026.12) ou estimation par les bornes des statistiques.
2. **Volume disque** = nombre de points × coût moyen par point dans DuckDB. Ce coût n'est **pas supposé** : il est mesuré une fois, sur un jeu de test, puis gravé dans le code et revu à chaque évolution de DuckDB.
3. **Durée** = nombre de points ÷ débit d'insertion mesuré, affichée comme un ordre de grandeur (« environ 20 minutes »).
4. **Comparaison** à l'espace libre (`system.getDiskSpace`), avec une **marge** : 20 % de l'espace libre restant après l'import, et au moins 1 Go. Jauge verte, orange ou rouge (05, écran 8).

Le résultat est présenté **par appareil**, trié du plus volumineux au plus léger. L'utilisateur peut ainsi décocher les plus gros s'il manque de place.

## 6. Déroulement, reprise et annulation

- **Tâche de fond** `gladys.job`, avec progression (fonctions traitées / total, points écrits / total), **pause** et **reprise**. Elle est visible dans Paramètres → Tâches de fond. `gladys.job` n'ayant ni pause ni reprise, elles sont **gérées par le service** (01 § 3.5, F3) : une pause termine la tâche après le lot en cours ; chaque reprise démarre une **nouvelle tâche** à partir du curseur.
- **Bornes d'import** (F1), par fonction, enregistrées dans `t_migration_history_cursor` :
  - **borne de fin** : la **date de migration** de la fonction (création ou rapprochement de son appareil, d'après le journal). Après cette date, c'est Gladys qui enregistre ; on n'importe **jamais** d'historique HA postérieur ;
  - pour un appareil **préexistant**, la borne de fin est le **plus ancien état propre à Gladys** (01 § 5.5), s'il est antérieur.
- **Curseur par fonction** (`t_migration_history_cursor`, 01 § 5.2) : après chaque lot validé, on enregistre la date du dernier point écrit. Une interruption (pause, redémarrage de Gladys) reprend au point exact. À la reprise, le curseur est **recalé** sur l'état le plus récent présent avant la borne de fin, ce qui couvre une coupure entre l'écriture d'un lot et l'enregistrement du curseur : aucun lot n'est écrit deux fois.
- **Redémarrage de Gladys** : la tâche en cours est marquée échouée (`purged-when-restarted`) par le cœur ; le service la **relance** de lui-même au démarrage (01 § 3.5). Le dossier de travail, avec la base de la sauvegarde, est **conservé** tant que l'exécution est ouverte (01 § 7.2). En mode B, le jeton HA est **redemandé** avant de reprendre, puisque le coffre est vidé à l'arrêt.
- **Ordre de traitement** : les fonctions les plus utiles d'abord (énergie, température), les plus volumineuses ensuite, pour qu'une interruption laisse l'essentiel en place.
- **Mode A et C** : la base de la sauvegarde reste dans le dossier de travail **jusqu'à la fin de l'exécution**, puis elle est supprimée (01 § 7.2). Si l'import est mis en pause longtemps, la base reste sur le disque : l'écran Migration l'indique, avec sa taille.
- **Resynchronisation** (D13) : seules les fonctions **nouvellement migrées** reçoivent un import, avec les mêmes bornes. Une fonction déjà migrée n'en reçoit plus : depuis sa date de migration, Gladys enregistre lui-même. *Remplace le 2026-11-19 : ~~« seule la période postérieure au curseur est importée »~~, qui aurait mélangé valeurs HA et valeurs Gladys sur la même période.*
- **Annulation** (A9, amendée, 01 § 5.4) :
  - appareil **créé** par l'outil : son historique est supprimé avec lui ;
  - appareil **préexistant** : l'historique HA importé est **conservé**. L'écran de confirmation l'annonce (« L'historique importé sur N appareils existants sera conservé », 05 § 5). *Remplace le 2026-11-19 : ~~« on supprime exactement la plage importée »~~, impossible sans nouvelle fonction du cœur (01 § 5.5). Une fonction facultative « supprimer les états d'une fonction entre deux dates » lèverait cette exception (A7).*
  - on restaure la durée de conservation d'origine (§ 1).

## 7. Décisions

| # | Décision |
|---|---|
| H1 | Purge de Gladys : avant l'import, l'outil propose de passer la conservation de l'historique à **« illimitée »**, choix par défaut. L'utilisateur l'ajustera ensuite selon ses besoins. La valeur d'origine est restaurée en cas d'annulation (§ 1, ex-Q07-3). |
| H2 | Compteurs d'énergie : import de l'index réel, puis recalcul des consommations, des coûts et, pour un index de production, de la production par Gladys (§ 3.3). *Production ajoutée le 2026-11-19 (T7).* |
| H3 | Mesures : seule la **moyenne horaire** est importée, sans le minimum ni le maximum (ex-Q07-2). |
| H4 | Écriture **par ajout seulement** (`saveMultipleHistoricalStates`), en lots de 10 000 états au plus ; jamais d'historique HA après la date de migration d'une fonction ; à l'annulation, historique conservé sur les appareils préexistants (§ 4, § 6, F1). |
| H5 | Pas de recalcul d'agrégats après l'import : les graphiques sont calculés à la lecture dans DuckDB, et les agrégats sont des vestiges (§ 4, ex-Q07-4). |

## 8. Questions ouvertes

| # | Question |
|---|---|
| Q07-1 | **Mesures sur une instance réelle** : coût par point dans DuckDB, débit d'insertion, taille des lots et pause compatibles avec une instance qui doit rester réactive, comportement de la consommation à 30 minutes calculée à partir d'un index horaire. *Laissée ouverte. À traiter avec Pierre-Gilles et les sauvegardes réelles (Q02-3).* |
