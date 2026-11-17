# 07 — Import des historiques

- **Statut** : 📝 Brouillon
- **Dernière mise à jour** : 2026-11-17
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
2. Si elle la dépasse, l'utilisateur choisit sur l'écran 7 (05) :
   - **passer la conservation à « illimitée »** (choix proposé par défaut, puisque D24 demande de tout garder) ;
   - ou l'**allonger** jusqu'à couvrir l'historique importé ;
   - ou **garder son réglage**. Les points trop anciens ne sont alors **pas importés**, plutôt qu'importés puis purgés : on évite une écriture inutile.
3. C'est une **modification d'un réglage existant** : l'ancienne valeur est enregistrée dans le journal et **restaurée en cas d'annulation** (A9).
4. L'estimation d'espace disque (§ 5) tient compte du choix.

## 2. Ce qui est importé

| Source HA | Contenu | Rétention HA | Import |
|---|---|---|---|
| États bruts (`states`) | Chaque changement de valeur | 10 jours par défaut (R § 5.1) | Valeurs numériques uniquement |
| Statistiques horaires (`statistics`) | Moyenne, minimum, maximum (mesures) ; index et cumul (compteurs) | **Indéfinie**, pour les capteurs ayant une `state_class` (R § 5.5) | Une valeur par heure (§ 3) |
| Statistiques 5 min | Agrégats sur 5 minutes | Comme les états | Non : elles doublonnent les états bruts |
| Statistiques externes | Données de fournisseurs, sans entité | Indéfinie | Hors V1 (02 § 5.2) |

**Seules les fonctions effectivement migrées** reçoivent un historique : le lien entité HA → feature Gladys vient de 03 § 3.4. Une fonction dont `keep_history` vaut faux dans Gladys n'en reçoit pas. L'outil le signale et propose de l'activer.

## 3. Règles de conversion

### 3.1 Raccord entre les deux niveaux

Pour une fonction donnée :
- **avant** la date du plus ancien état brut conservé par HA, on importe les **statistiques horaires** ;
- **à partir** de cette date, on importe les **états bruts**.

On obtient une série continue, sans recouvrement : un point par heure pour le passé lointain, le détail réel pour les derniers jours.

### 3.2 Par type de capteur

| `state_class` HA | Valeur importée par heure | Remarque |
|---|---|---|
| `measurement` | `mean` | Les `min` et `max` sont perdus : Gladys n'a qu'une valeur par point. Voir § 7, Q07-2. |
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
4. **Granularité** : avec un index horaire, la consommation à 30 minutes calculée par Gladys tombera sur une demi-heure sur deux. Le total par heure, par jour et par mois reste exact. *À valider sur une instance réelle (Q07-1).*

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
| Insertion par lots | `device.saveMultipleHistoricalStates(featureId, states)` | Écriture DuckDB groupée (`db.duckDbBatchInsertState`) |
| Relance sur une période déjà importée | `device.replaceHistoricalStatesFrom(featureId, from, states)` | Remplacement atomique : les anciennes valeurs restent en place si l'écriture échoue |
| Plus ancien état propre à Gladys | `device.getOldestStateFromDeviceFeatures` | Borne de l'import pour un appareil qui existait déjà (01 § 5.5) |

**Points d'attention vérifiés dans le code :**
- `saveMultipleHistoricalStates` appelle `saveHistoricalState` avec le **dernier** point du lot. Celui-ci ne met à jour la valeur courante de la fonction (`last_value`) **que s'il est plus récent** que la valeur actuelle. L'import d'un passé ancien n'écrase donc pas la valeur en direct (`server/lib/device/device.saveHistoricalState.js`).
- Les lots sont triés par date. On les envoie **du plus ancien au plus récent**, par fonction.

**Taille des lots et rythme** : par exemple 10 000 points par lot, avec une courte pause entre deux lots pour ne pas monopoliser DuckDB, que Gladys utilise en continu. *À calibrer par mesure (Q07-1).*

## 5. Estimation du volume et de la durée (D24)

1. **Comptage** des points par fonction, selon la source (02 § 5.3) : `COUNT` SQL sur la base de la sauvegarde, `recorder/database_usage` (HA ≥ 2026.12) ou estimation par les bornes des statistiques.
2. **Volume disque** = nombre de points × coût moyen par point dans DuckDB. Ce coût n'est **pas supposé** : il est mesuré une fois, sur un jeu de test, puis gravé dans le code et revu à chaque évolution de DuckDB.
3. **Durée** = nombre de points ÷ débit d'insertion mesuré, affichée comme un ordre de grandeur (« environ 20 minutes »).
4. **Comparaison** à l'espace libre (`system.getDiskSpace`), avec une **marge** : 20 % de l'espace libre restant après l'import, et au moins 1 Go. Jauge verte, orange ou rouge (05, écran 7).

Le résultat est présenté **par appareil**, trié du plus volumineux au plus léger. L'utilisateur peut ainsi décocher les plus gros s'il manque de place.

## 6. Déroulement, reprise et annulation

- **Tâche de fond** `gladys.job`, avec progression (fonctions traitées / total, points écrits / total), **pause** et **reprise**. Elle est visible dans Paramètres → Tâches de fond.
- **Curseur par fonction** (`t_migration_history_cursor`, 01 § 5.2) : après chaque lot validé, on enregistre la date du dernier point écrit. Une interruption (redémarrage de Gladys, pause) reprend au point exact.
- **Ordre de traitement** : les fonctions les plus utiles d'abord (énergie, température), les plus volumineuses ensuite, pour qu'une interruption laisse l'essentiel en place.
- **Mode A et C** : la base de la sauvegarde reste dans le dossier de travail **jusqu'à la fin de l'import** des historiques, puis elle est supprimée (01 § 7.2). Si l'import est mis en pause longtemps, la base reste sur le disque : l'écran Migration l'indique, avec sa taille.
- **Resynchronisation** (D13) : seule la période postérieure au curseur est importée.
- **Annulation** (A9) : on supprime exactement la plage importée, fonction par fonction, d'après le curseur. Puis on relance le recalcul des fonctions dérivées d'énergie, et on restaure la durée de conservation d'origine (§ 1).

## 7. Questions ouvertes

| # | Question |
|---|---|
| Q07-1 | **Mesures sur une instance réelle** : coût par point dans DuckDB, débit d'insertion, taille des lots, comportement de la consommation à 30 minutes calculée à partir d'un index horaire. À faire avec les sauvegardes réelles attendues (Q02-3). |
| Q07-2 | **Minimum et maximum horaires** : HA les conserve pour les mesures, Gladys n'a qu'une valeur par point. Importer seulement la moyenne (proposé) suffit-il ? Ou faut-il, par exemple, trois points par heure (min, moyenne, max) au risque de fausser les moyennes calculées par Gladys ? |
| Q07-3 | **Mise en garde** : passer la conservation à « illimitée » vaut pour **tous** les appareils de Gladys, pas seulement ceux importés. Est-ce acceptable comme choix par défaut, ou faut-il plutôt proposer une durée calculée (par exemple l'âge de l'historique HA plus un an) ? |
