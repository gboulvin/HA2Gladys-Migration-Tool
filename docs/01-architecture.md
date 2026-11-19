# 01 — Architecture

- **Statut** : 📝 Brouillon
- **Dernière mise à jour** : 2026-11-19
- **Prérequis** : [00 — Vision](00-vision.md), [note de recherche sur le code Gladys](research/gladys-codebase.md). À lire avec [04 — Scènes](04-mapping-scenarios.md), [07 — Historiques](07-historiques.md), [08 — Tableaux de bord](08-tableaux-de-bord.md) et [09 — API du service](09-api.md).
- **Référence Gladys étudiée** : commit `71edb17` du dépôt `GladysAssistant/Gladys`

Les affirmations sur Gladys renvoient au code (chemins relatifs au dépôt Gladys). Celles sur HA sont marquées *à vérifier* tant que la section 02 ne les a pas confirmées ; celles que 02 a confirmées y renvoient.

---

## 1. Vue d'ensemble

L'outil est un **service du cœur de Gladys** (D1), nommé **`migration-home-assistant`** (A8). Il se compose d'un backend dans `server/services/migration-home-assistant/` et d'une interface dans `front/src/routes/settings/settings-migration/` (voir § 9).

**Ouverture à d'autres systèmes (A8).** Le principe pourra être réutilisé pour migrer depuis d'autres solutions domotiques. Le code est donc séparé en deux parties :
- une partie **générique** : inventaire, analyseur, orchestrateur, exécuteurs, journal, coffre des secrets ;
- une partie **propre à HA** : les sources (API et sauvegarde), la normalisation vers l'inventaire et la conversion des automatisations.

Les tables du journal sont nommées `t_migration_*` et portent une colonne `source` (`home-assistant`). Un futur outil, par exemple `migration-jeedom`, pourra ainsi les réutiliser sans renommage. On n'extrait pas pour autant un framework générique dès la V1 : on le fera quand un deuxième système le justifiera.

```
                    ┌─────────────────────── Gladys (cœur) ─────────────────────────────────────────┐
                    │                                                                               │
 ┌──────────────┐   │  ┌───────────────┐    ┌────────────┐    ┌──────────────┐    ┌──────────────┐  │
 │ Home         │──────▶ Sources       │───▶│ Inventaire │───▶│ Analyseur    │───▶│ Plan de      │  │
 │ Assistant    │ API │ • API HA      │    │ (modèle    │    │ (classement) │    │ migration    │  │
 │ (API WS/REST)│   │  │ • Sauvegarde  │    │  pivot)    │    └──────▲───────┘    └──────┬───────┘  │
 └──────────────┘   │  └───────▲───────┘    └────────────┘           │                   │          │
 ┌──────────────┐   │          │                          ┌──────────┴───────┐           ▼          │
 │ Sauvegarde   │──────────────┘                          │ Catalogue        │    ┌──────────────┐  │
 │ HA (.tar)    │ téléversement                           │ (client + cache) │    │ Orchestrateur│  │
 └──────────────┘   │                                     └──────────▲───────┘    │ de phases    │  │
                    │                                                │            └──────┬───────┘  │
                    │                                                │                   │          │
                    │      ┌─────────────────────────────────────────┼───────────────────┤          │
                    │      ▼                 ▼                 ▼     │           ▼       ▼          │
                    │ ┌──────────┐   ┌──────────────┐  ┌───────────┐ │   ┌───────────┐ ┌─────────┐  │
                    │ │Installeur│   │ Rapprocheur  │  │Importeur  │ │   │Convertis- │ │ Journal │  │
                    │ │d'intégr. │   │ et créateur  │  │d'historiq.│ │   │seur de    │ │ (tables │  │
                    │ │          │   │ d'appareils  │  │           │ │   │scènes (V1)│ │ dédiées)│  │
                    │ └────┬─────┘   └──────┬───────┘  └─────┬─────┘ │   └─────┬─────┘ └─────────┘  │
                    │      ▼                ▼                ▼       │         ▼                    │
                    │  API internes du cœur : externalIntegration.*, device.*, room.*, user.*, scene.*│
                    └────────────────────────────────────────────────┼──────────────────────────────┘
                                                                     │
                                                       ┌─────────────┴────────────┐
                                                       │ Catalogue distant (D32)  │
                                                       │ indexeur du store        │
                                                       └──────────────────────────┘
```

Le principe : **deux sources produisent un même inventaire normalisé**. Tout le reste de l'outil (analyse, rapprochement, import) ne connaît que cet inventaire, jamais la source.

## 2. Pourquoi un service du cœur (confirmation de D1)

Le cœur de Gladys tend à se délester de ses intégrations au profit d'intégrations externes. L'outil de migration doit pourtant rester dans le cœur, parce qu'une intégration externe n'a accès qu'à une API hôte restreinte (`docs/specs/external-integrations/contracts/host-api-endpoints.md`) :

| Besoin de l'outil | API hôte d'une intégration externe | Cœur |
|---|---|---|
| Installer d'autres intégrations | ❌ | ✅ `externalIntegration.install` |
| Lire les appareils découverts par **d'autres** intégrations | ❌ (seulement les siens) | ✅ `externalIntegration.getDiscoveredDevices`, services natifs |
| Créer des pièces et des utilisateurs | ❌ (lecture des maisons seulement) | ✅ |
| Créer des scènes | ❌ | ✅ |
| Écrire un historique sur les appareils d'autres intégrations | ❌ | ✅ `device.saveMultipleHistoricalStates` |

**Conclusion** : D1 est confirmée. L'outil est un service du cœur, réservé aux administrateurs.

## 3. Composants

### 3.1 Sources

Les deux sources implémentent la même interface et produisent l'**inventaire** (§ 4).

| | **Source API** | **Source sauvegarde** |
|---|---|---|
| Entrée | URL de HA et jeton d'accès longue durée | Fichier de sauvegarde HA (`.tar`) et, s'il est chiffré, son mot de passe ou sa clé |
| Transport | WebSocket HA (registres, états, statistiques, historique, configuration des automatisations et des scripts) et REST (configuration des scènes) : 02 § 3.2 | Lecture en flux de l'archive : fichiers `.storage`, `automations.yaml`, `scripts.yaml`, `scenes.yaml`, base du *recorder* : 02 § 4.2 |
| Historiques | Via l'API (statistiques et historique), quel que soit le moteur de base de HA | Uniquement si la base est en SQLite **et** incluse dans la sauvegarde |
| Dépendances existantes | `ws` (déjà dans `server/package.json`) | `sqlite3` (déjà présent) pour lire la base du *recorder* |
| Dépendances à ajouter (propres au service, A7) | Un **lecteur tar en flux** en JavaScript, en mode A, pour la sauvegarde téléchargée | Le même **lecteur tar en flux** (le binaire `tar` ne suffit pas : 02 § 4.3 règle 3), un **analyseur YAML** (`automations.yaml`, `scripts.yaml`, `scenes.yaml`) et **libsodium** pour SecureTar v3 (02 § 4.4) |

*Corrigé le 2026-11-19 (C12) : la version précédente prévoyait le binaire `tar` et ne citait pas le lecteur tar ; les trois dépendances sont propres au service (A7, 02 § 9).*

Le détail (commandes, formats, versions de HA) relève de la section 02.

### 3.2 Inventaire (modèle pivot)

Structure normalisée et indépendante de la source. Elle est décrite au § 4.

### 3.3 Client du catalogue

- Il télécharge le catalogue distant (D12, D32), **publié par l'indexeur du store** de l'organisation GladysAssistant, à côté de `index.json`. Il garde un **cache local** et embarque un **instantané** dans le code du service, au cas où le catalogue serait injoignable lors de la première utilisation.
- Il reprend le schéma déjà éprouvé du **store des intégrations** (`docs/specs/external-integrations/core/store.md`) : index statique publié sur GitHub Pages, rafraîchi périodiquement, cache local et fonctionnement dégradé hors ligne.
- Les règles des **services natifs** (zigbee2mqtt, mqtt, matter) sont dans le dépôt Gladys, versionnées avec le cœur. Le format du catalogue est défini en section 03.

### 3.4 Analyseur

L'analyseur prend en entrée l'inventaire, le catalogue et l'état actuel de Gladys (intégrations installées, appareils existants, pièces, utilisateurs). Il produit le **plan de migration** : un statut par élément (table de référence : 00 § 8), les intégrations à installer, les rapprochements prévus et l'estimation du volume d'historique (D24).

C'est une fonction **pure** : sans effet de bord, elle est entièrement testable avec des jeux de données.

### 3.5 Orchestrateur de phases

Il fait avancer une exécution de phase en phase (phases A à F, 00 § 6 bis), en conservant son état (§ 5). Les traitements longs (installation, rapprochement, historiques, scènes, tableaux de bord, annulation) s'exécutent en **tâches de fond** avec progression via `gladys.job` (`server/lib/job/`), et les événements sont poussés au front par WebSocket (09 § 3).

**Pause et reprise, gérées par le service (F3).** `gladys.job` ne connaît ni pause ni reprise : ses seuls statuts sont `in-progress`, `success` et `failed` (`server/utils/constants.js:2559-2563`), et au démarrage toute tâche encore `in-progress` est marquée `failed` avec `purged-when-restarted` (`server/lib/job/job.init.js`, `server/utils/constants.js:2565-2567`). Le service gère donc lui-même la reprise :
- l'avancement est porté par un **curseur** en base (`t_migration_run`, `t_migration_item`, `t_migration_history_cursor`), jamais par la tâche ;
- une **pause** laisse finir le lot en cours, enregistre le curseur et termine la tâche (`job.finish`, statut `success`, avec la mention de la pause dans `data`) ;
- chaque **reprise** démarre une **nouvelle tâche** `gladys.job` à partir du curseur ;
- au démarrage de Gladys, le service repère son exécution ouverte dont la dernière tâche a été marquée `failed` (`purged-when-restarted`) et la **relance** lui-même, après avoir redemandé les secrets si la suite en a besoin (§ 7.2).

Le `type` d'une tâche est validé contre `JOB_TYPES` (`server/models/job.js:79-84`) : les types de tâches du service sont une **modification du cœur** (A7, 09 § 3).

### 3.6 Exécuteurs

| Exécuteur | Rôle | API du cœur utilisées |
|---|---|---|
| **Installeur d'intégrations** | Installe les intégrations externes cochées et configure les natives. Préremplit leur configuration (avec l'accord de l'utilisateur pour les secrets, voir § 9 bis de la vision). | `externalIntegration.install`, `externalIntegration.setIntegrationConfig`, variables des services natifs |
| **Rapprocheur** | Associe chaque appareil HA à un appareil **découvert** dans Gladys (§ 6). | `externalIntegration.getDiscoveredDevices`, découverte des services natifs (ex. `zigbee2mqtt`) |
| **Créateur d'appareils** | Crée l'appareil Gladys **à partir du payload découvert**, puis applique le nom et la pièce HA. | `device.create` (voir l'avertissement du § 6.3) |
| **Créateur d'appareils virtuels** | Crée les appareils virtuels MQTT issus des entrées HA (D27, 04 § 6.2). | `device.create` (*création programmatique à vérifier*, 04 § 6.2) |
| **Créateur de pièces et d'utilisateurs** | Crée les pièces (avec renommage « Étage - Pièce » si l'utilisateur l'a choisi, D14) et les utilisateurs (D11). | `room.*`, `user.*` |
| **Exécuteur de présence et de zones** | Crée les zones (`area.create`), met à jour les coordonnées de la maison (`house.update`, valeur d'avant journalisée), génère les scènes de présence par utilisateur (04 § 6.3). | `area.create`, `house.update`, `scene.create` |
| **Importeur d'historiques** | Copie les états et statistiques HA vers les fonctions migrées, par **ajout** seulement, en lots de 10 000 états au plus (07 § 4). Règle la conservation de l'historique et `keep_history` (valeurs d'avant journalisées). | `device.saveMultipleHistoricalStates`, `variable.setValue`. *Ni `device.replaceHistoricalStatesFrom` ni `device.destroyStatesFrom` : tous deux suppriment tout à partir d'une date, sans borne de fin (F1, 07 § 4).* |
| **Convertisseur et créateur de scènes** (V1) | Traduit les scènes HA, les scripts HA simples et les automatisations convertibles (socle 04 § 2.4) en scènes ; crée les brouillons à la demande (04 § 2.5) ; à la bascule, active les scènes et ajoute les appels `scene.start` différés (04 § 7). | `scene.create`, `scene.update` |
| **Créateur de tableaux de bord** | Génère l'Accueil et reprend les tableaux de bord HA (08) ; modifie l'ordre si l'utilisateur l'accepte (valeur d'avant journalisée). | `dashboard.create`, `dashboard.updateOrder` |
| **Exécuteur d'annulation** | Défait la migration d'après le journal, en ordre inverse (§ 5.4). | `scene.destroy`, `dashboard.destroy`, `device.destroy`, `area.destroy`, `room.destroy`, `user.destroy`, `externalIntegration.uninstall`, et les fonctions de mise à jour pour restaurer les valeurs d'avant |

### 3.7 Journal de migration

Il conserve la correspondance entre chaque élément HA et l'objet Gladys créé. Il sert à l'idempotence, à la réversibilité, à la reprise et à la resynchronisation (§ 5).

## 4. Inventaire : le modèle pivot

| Entité | Champs principaux | Usage |
|---|---|---|
| `Area` | id HA, nom, étage (id) | Pièces Gladys |
| `Floor` | id HA, nom, niveau | Uniquement pour le renommage proposé (D14) |
| `IntegrationEntry` | domaine HA (ex. `hue`, `mqtt`), titre, identifiants **si présents** (marqués secrets) | Recherche dans le catalogue, préremplissage des configurations |
| `Device` | id HA, nom (y compris le nom défini par l'utilisateur), fabricant, modèle, `area_id`, **identifiants** (`identifiers`) et **connexions** (`connections` : adresse MAC, adresse Zigbee…, confirmé en 02 § 6.1), intégration d'origine, désactivé ou non | Classement et rapprochement |
| `Entity` | `entity_id`, appareil parent, domaine, `device_class`, `state_class`, unité, désactivée ou masquée | Correspondance des fonctions, historiques |
| `Person` | id HA, nom, utilisateur HA lié, appareils de suivi, photo (référence) | Utilisateurs Gladys (D11), présence (D30) |
| `Helper` | id HA, `entity_id`, type (`input_boolean`, `input_number`, `counter`…), nom, pièce, min, max, pas, unité, état actuel | Appareils virtuels (D27, 03 § 7 bis) |
| `Zone` | id HA, nom, centre, rayon, passive ou non ; `zone.home` à part | Zones Gladys et coordonnées de la maison (D30, 04 § 6.3) |
| `Scene` | id HA, nom, états et attributs par entité | Scènes Gladys sans déclencheur (D25, 04 § 8 bis) |
| `Script` | clé du script, alias, `fields`, séquence d'actions, mode | Scènes Gladys sans déclencheur (D31, 04 § 8 ter) |
| `Automation` | id HA, alias, déclencheurs, conditions, actions, mode, `max`, activée ou non, origine blueprint, scripts et scènes appelés (04 § 4) | Convertisseur de scènes V1 (socle 04 § 2.4), fiches lisibles (04 § 2.5) |
| `Dashboard` | `url_path`, titre, vues, cartes | Tableaux de bord (D28, 08 § 3) |
| `Addon` | slug, nom, version, en marche ou non | Modules (D29, 03 § 7 ter) |
| `HaConfig` | fuseau horaire, coordonnées, altitude, système d'unités | Contrôles de conversion (04 § 4), coordonnées de la maison (04 § 6.3) |
| `HistorySource` | accès en flux aux états et statistiques d'une entité, avec une estimation du nombre de points | Historiques (D22 à D24) |

**Règle de confidentialité** : l'inventaire ne garde **aucun secret**. Les identifiants des intégrations restent en mémoire, dans un coffre éphémère (§ 7), et ne sont référencés dans l'inventaire que par un identifiant opaque.

## 5. Persistance et reprise

### 5.1 Ce qui est conservé

| Donnée | Conservée ? | Où | Pourquoi |
|---|---|---|---|
| Inventaire normalisé (sans secrets) | Oui | Table dédiée | Reprendre la migration après la phase B sans relire HA |
| Plan de migration et choix de l'utilisateur | Oui | Table dédiée | Reprise, rapport final |
| Correspondance élément HA → objet Gladys | Oui | Table dédiée | Idempotence, réversibilité, resynchronisation |
| Curseur d'import des historiques par fonction | Oui | Table dédiée | Reprise après interruption, resynchronisation incrémentale |
| Jeton HA, mot de passe de sauvegarde, identifiants | **Non** | Mémoire uniquement (coffre éphémère, vidé à l'arrêt). Redemandés après un redémarrage si la suite en a besoin. | Confidentialité (§ 9 bis de la vision) |
| Extrait de la sauvegarde : base du *recorder* et son WAL, archive du module Zigbee2MQTT si besoin | **Temporairement** | Dossier de travail du service, **conservé entre deux redémarrages** tant qu'une exécution est ouverte, purgé à la fin de l'exécution, à son abandon ou à l'annulation | Voir § 7.2. *Corrigé le 2026-11-19 (C10) : la version précédente le purgeait aussi au démarrage, ce qui empêchait la reprise promise en 07 § 6.* |

### 5.1 bis Migration, exécutions et identité des éléments (T2, T9)

**Migration.** Une migration regroupe tout le travail mené depuis **une source** : une installation HA, reconnue par son identifiant d'instance (`core.uuid` : [core] `helpers/instance_id.py:10` ; lecture par l'API *à vérifier*). **Une seule migration est ouverte à la fois** sur l'instance Gladys. Son statut est `open`, `finished` (bilan validé après la bascule, ou clôture par l'utilisateur) ou `cancelled`.

**Exécutions.** Une migration contient plusieurs exécutions, chacune parcourant les phases utiles (00 § 6 bis) :

| Nature (`kind`) | Déclenchée par | Phases |
|---|---|---|
| `initial` | « Démarrer une migration » | A à F |
| `resync` | « Relancer l'analyse » (D13) | A, puis les phases utiles aux éléments nouveaux ou mis à jour |
| `switchover` | « Passer en bascule » (A11) | A, puis activation des scènes, appels différés, dépendances à HA (§ 9.3) |
| `cancel` | « Annuler la migration » (A9) | Annulation (§ 5.4) |

- **Verrou** : **deux exécutions simultanées sont interdites**. Le service tient un verrou (une exécution `running` ou `paused` au plus par instance). Une seconde demande est refusée avec le code `MIGRATION_RUN_IN_PROGRESS` (09 § 4). Une exécution `paused` doit être reprise ou abandonnée avant d'en lancer une autre.
- **Statuts d'une exécution** : `running`, `paused`, `interrupted` (redémarrage de Gladys, reprise automatique, § 3.5), `finished`, `abandoned`.
- **Annuler la migration défait toute la migration**, toutes exécutions confondues, d'après le journal et en ordre inverse (§ 5.4). La migration passe alors à `cancelled`, et une nouvelle peut commencer.

**Identité stable des éléments.** Un élément source n'est journalisé qu'**une fois par migration**, sous une clé stable, quel que soit le nombre d'exécutions. Une relance ou la bascule retrouve ainsi l'objet Gladys créé par une exécution précédente (D13).

| Type (`t_migration_item.type`) | Clé source |
|---|---|
| `area`, `floor` | `area_id`, `floor_id` HA |
| `device` | `id` du registre des appareils HA ; pour des appareils scindés regroupés (02 § 6.1), la connexion commune (ex. `mac:aa:bb:…`) |
| `entity` | `unique_id` du registre (avec la plateforme) ; `entity_id` pour une entité sans registre |
| `person` | `id` HA de la personne |
| `presence_scene` | `id` de la personne et rôle de la scène (`seen` ou `check`, 04 § 6.3) |
| `integration` | `entry_id` de l'entrée de configuration |
| `helper` | `id` de l'entrée (`.storage/input_*`, `.storage/counter`) ; `entity_id` pour une entrée YAML |
| `zone` | `id` de la zone (`.storage/zone`) ; `entity_id` pour une zone YAML |
| `house` | clé fixe `home` (`zone.home` et configuration HA) |
| `scene` | `id` HA de la scène (les scènes sans `id` sont hors périmètre, 04 § 8 bis) |
| `script` | clé du script dans `scripts.yaml` (`script.<clé>`) *à vérifier : identique à son `unique_id`* |
| `automation` | `id` HA ; sans `id`, **empreinte du contenu** (02 § 6.3) |
| `dashboard` | `url_path` (tableau de bord par défaut : `lovelace`) ; clé fixe `home` pour l'Accueil généré |
| `addon` | slug du module |
| `setting` | nom de la variable Gladys modifiée (ex. `DEVICE_STATE_HISTORY_IN_DAYS`) |

### 5.2 Tables proposées

Gladys n'a aucun marqueur d'origine sur ses objets (voir la note de recherche). Plutôt que de modifier les modèles existants, on ajoute des tables propres au service, avec une migration Sequelize :

- **`t_migration`** : une migration (`source`, identifiant de l'instance source, mode courant, statut, dates). Voir § 5.1 bis.
- **`t_migration_run`** : une exécution (migration, nature `kind`, mode, **phase courante** parmi `analysis`, `integrations`, `devices`, `scenes`, `history`, `dashboards` (00 § 6 bis), statut, tâche `gladys.job` courante, dates, résumé).
- **`t_migration_item`** : un élément source **dans une migration**, unique par (migration, type, clé source). Champs : type (`area`, `floor`, `device`, `entity`, `person`, `presence_scene`, `integration`, `helper`, `zone`, `house`, `scene`, `script`, `automation`, `dashboard`, `addon`, `setting`), clé source (§ 5.1 bis), empreinte du contenu, statut (00 § 8), choix de l'utilisateur, dernière exécution qui l'a traité, erreur éventuelle.
- **`t_migration_operation`** : le **journal des opérations**, une ligne par objet Gladys touché : élément, exécution, horodatage, **type et id de l'objet Gladys créé ou modifié**, **nature de l'opération** (création, modification), **champ modifié et valeur d'avant** pour une modification (§ 5.4). Un élément peut produire plusieurs opérations : une automatisation réécrite donne deux scènes, une personne un utilisateur et deux scènes de présence. L'annulation rejoue ce journal en ordre inverse.
- **`t_migration_history_cursor`** : par fonction Gladys, l'entité HA source, l'appareil **créé ou préexistant**, la **borne de fin** de l'import (date de migration de la fonction, ou plus ancien état propre à Gladys, § 5.5), la **plage de dates importée** (états et statistiques séparément) et le dernier point écrit (07 § 6).

Les variables de service (`t_variable`) ont été écartées. Elles sont prévues pour des paires clé-valeur et n'offrent ni requêtes ni intégrité référentielle sur des milliers d'éléments.

**Suppression d'un objet dans Gladys** : si l'utilisateur supprime à la main un appareil importé, le journal pointe vers un objet absent. À la relance, l'élément est considéré comme « non migré » et redevient proposable. *Une suppression en cascade côté Gladys n'est pas envisagée : elle toucherait au cœur des modèles.*

### 5.3 Idempotence et resynchronisation (D13)

À la relance, l'analyseur compare l'inventaire neuf au journal :

| Cas | Traitement |
|---|---|
| Élément HA déjà migré, objet Gladys toujours présent | Statut ☑️ « déjà migré » (00 § 8), pas d'action |
| Élément HA déjà migré mais modifié dans HA (empreinte différente) | Signalé. L'utilisateur choisit de mettre à jour ou d'ignorer. On n'écrase jamais en silence. |
| Nouvel élément HA | Proposé à l'import |
| Élément HA supprimé depuis | Signalé. L'objet Gladys n'est pas supprimé automatiquement. |
| Historique | Import seulement pour les fonctions **nouvellement migrées**. Jamais d'historique HA postérieur à la date de migration d'une fonction : c'est Gladys qui l'enregistre depuis (07 § 6). |

### 5.4 Réversibilité : la garantie de retour à l'état initial (A9)

**Engagement envers l'utilisateur** : si la migration se passe mal, il peut l'annuler et retrouver **ses deux systèmes comme avant**, à **une exception près, affichée avant confirmation** : l'historique HA importé sur des appareils qui **existaient déjà** dans Gladys est **conservé** (§ 5.5). L'appareil est simplement plus riche d'un historique. L'écran de confirmation le dit : « L'historique importé sur N appareils existants sera conservé » (05 § 5). *Amendé le 2026-11-19 (F1).*

- **Côté HA**, rien n'est à faire : l'outil n'y écrit jamais (principe 2 de la vision).
- **Côté Gladys**, l'annulation défait **uniquement ce que l'outil a fait**, d'après le journal (`t_migration_operation`), pour **toutes les exécutions** de la migration (§ 5.1 bis), en ordre inverse :

| Ce que l'outil a fait | Ce que l'annulation fait |
|---|---|
| Créer une scène : issue d'une automatisation, d'une scène ou d'un script HA, **brouillon** (04 § 2.5), **scène de présence générée** (04 § 6.3) | La supprimer |
| Activer une scène, ou y **ajouter un appel `scene.start` différé**, à la bascule (04 § 7) | Restaurer l'état et les actions d'avant (inutile si la scène est elle-même supprimée) |
| Créer un tableau de bord (Accueil, reprise HA) | Le supprimer |
| Modifier l'**ordre des tableaux de bord** (08 § 4) | Restaurer l'ordre d'avant |
| Importer de l'historique sur un appareil **créé** par l'outil | Supprimé avec l'appareil (ligne suivante) |
| Importer de l'historique sur un appareil **préexistant** | **Conservé** : exception annoncée (§ 5.5) |
| Créer un appareil, y compris un **appareil virtuel** (D27) | Le supprimer, ainsi que son historique. Au-delà de 5 000 états, `device.destroy` refuse et lance la purge des états de chaque fonction (`server/lib/device/device.destroy.js:76-82`, seuil `server/lib/device/index.js:79`) : l'exécuteur attend la fin de cette purge, puis supprime l'appareil. |
| **Modifier** un appareil qui existait déjà (nom, pièce, `keep_history`) | Restaurer les **valeurs d'avant**, gardées dans le journal |
| Créer un utilisateur | Le supprimer |
| Créer une pièce | La supprimer si elle est vide |
| Créer une **zone** | La supprimer |
| Modifier les **coordonnées de la maison** (`zone.home`, 04 § 6.3) | Restaurer les valeurs d'avant |
| Modifier un **réglage système** : conservation de l'historique (07 § 1, H1) | Restaurer la valeur d'avant |
| Installer une intégration externe | La désinstaller |
| Configurer une intégration qui ne l'était pas | Effacer cette configuration |

**Règles qui rendent cette garantie tenable :**
1. **L'outil ne remplace jamais une configuration existante.** Il ne préremplit que des intégrations non configurées. On n'a ainsi jamais à garder une copie d'un ancien secret pour le restaurer.
2. ~~Une modification n'est permise que sur des champs dont la valeur d'avant peut être enregistrée et restaurée (nom, pièce).~~ **Étendue le 2026-11-19 (C7)** : une modification d'un objet existant n'est permise que sur **un champ dont la valeur d'avant est journalisée** (`t_migration_operation`) et restaurable. Cas prévus :
   - nom et pièce d'un appareil (§ 6.3) ;
   - `keep_history` d'une fonction (§ 6.3, 07 § 2) ;
   - conservation de l'historique, variable système `DEVICE_STATE_HISTORY_IN_DAYS` (07 § 1, H1) ;
   - coordonnées de la maison (04 § 6.3) ;
   - ordre des tableaux de bord (08 § 4) ;
   - activation d'une scène et ajout différé d'appels `scene.start`, à la bascule (04 § 7).

   Toute autre modification d'un objet existant est hors du périmètre de l'outil.
3. **Ce que l'utilisateur a fait lui-même depuis la migration est préservé et signalé.** Si une scène créée à la main utilise un appareil importé, ou si une intégration installée par l'outil porte aussi des appareils ajoutés à la main, l'annulation l'affiche avant confirmation. L'utilisateur choisit alors de garder ces éléments (et ce dont ils dépendent) ou de tout annuler.
4. L'annulation est elle-même une tâche de fond, avec progression et rapport.

**Pour un retour complet, plus tard.** Une fonction **facultative** du cœur, « supprimer les états d'une fonction entre deux dates », permettrait de supprimer aussi l'historique importé sur les appareils préexistants, et donc de lever l'exception. Elle sera proposée à Pierre-Gilles (A7, D18). Elle n'est **pas bloquante** : la V1 n'en dépend pas.

### 5.5 Historique sur un appareil qui existait déjà

En coexistence, un appareil peut déjà exister dans Gladys et avoir son propre historique, par exemple un capteur Zigbee2MQTT partagé. Pour ne jamais mélanger les deux historiques :
- l'import HA s'arrête **à la date du plus ancien état propre à Gladys** pour cette fonction (`device.getOldestStateFromDeviceFeatures`, `server/lib/device/`) ;
- cette date et la plage importée sont enregistrées dans `t_migration_history_cursor` ;
- à l'annulation, cette plage est **conservée** (§ 5.4). ~~L'annulation supprime exactement cette plage.~~ *Remplacé le 2026-11-19 (F1)* : le cœur n'a pas de fonction qui supprime une plage bornée. `device.destroyStatesFrom` supprime tout à partir d'une date (`server/lib/device/device.destroyStatesFrom.js:19-23`), et `device.replaceHistoricalStatesFrom` aussi, avant de réinsérer (`server/models/index.js:424`). Les utiliser effacerait l'historique propre à Gladys.

## 6. Rapprochement des appareils (D16)

### 6.1 Principe

1. Le catalogue indique, pour l'intégration HA d'origine, l'**intégration Gladys cible** et une **règle de rapprochement**.
2. L'intégration cible est installée (phase B) et découvre les appareils.
3. Le rapprocheur compare chaque appareil HA à la liste des appareils découverts, selon la règle.
4. Pour chaque appareil, le résultat est l'un des suivants :
   - **un** candidat sûr → rapprochement proposé ;
   - **aucun** candidat → l'appareil reste « en attente de découverte » (appairage à faire, intégration à configurer…) ;
   - **plusieurs** candidats → l'utilisateur choisit. On ne devine jamais (principe « fiabilité d'abord »).

### 6.2 Règles de rapprochement

La règle est **déclarée dans le catalogue**, jamais codée en dur par intégration. On ajoute ainsi une intégration au catalogue sans modifier le cœur, ce qui sert l'objectif de couverture maximale (D20).

Les types de règles ci-dessous sont illustrés par des exemples ; **la référence, intégration par intégration, est 03 § 4** (et le détail des règles, 03 § 3.3).

| Type de règle | Exemple | Clé HA (02 § 6.1) | Clé Gladys |
|---|---|---|---|
| Adresse IEEE | Zigbee (ZHA, Zigbee2MQTT) | connexion `zigbee` ou identifiant de l'appareil | param `IEEE_ADDRESS` (`server/services/zigbee2mqtt/utils/convertDevice.js:26-30` ; `ieee_address` n'est qu'une propriété transitoire du payload découvert) |
| Adresse MAC | Shelly, Tasmota, ESPHome | connexion `mac` | paramètre ou `external_id` de l'intégration |
| Numéro de série, UUID ou identifiant constructeur | Hue, Tuya, Google Cast (UUID, 03 § 4) | `identifiers` | paramètre ou `external_id` |
| Motif sur l'`external_id` | Intégrations dont l'identifiant est prévisible | `identifiers` | gabarit déclaré dans le catalogue |
| Probable, sinon manuel | Matter : numéro de série s'il est exploitable, sinon choix de l'utilisateur (03 § 4, § 9) | `serial_number` | param `SERIAL_NUMBER`, sinon choix aidé par le nom, le fabricant et le modèle |

*Corrigé le 2026-11-19 (C19, F4) : Google Cast est rapproché par UUID et Matter par numéro de série quand il existe, comme en 03 § 4.*

### 6.3 Création d'un appareil rapproché

- L'appareil est créé **à partir du payload découvert**, tel que l'intégration l'a publié. Seuls le **nom**, la **pièce** et le réglage `keep_history` sont modifiés.
- ⚠️ `device.create` remplace l'appareil existant de même `external_id` et **supprime les fonctions et paramètres absents** du payload (voir la note de recherche). L'exécuteur ne doit donc jamais envoyer un payload partiel. Pour un appareil déjà présent dans Gladys, il n'utilise que les mises à jour ciblées (nom, pièce, `keep_history`), dont les valeurs d'avant sont journalisées (§ 5.4, règle 2).
- Une fonction HA dont l'équivalent n'apparaît pas dans le payload découvert est marquée « perdue » dans le rapport (statut ⚠️ « migrable partiellement »).

### 6.4 Correspondance des entités et des fonctions

Pour les historiques et les scènes, il faut savoir quelle **entité HA** correspond à quelle **fonction Gladys**. La règle est également déclarée dans le catalogue : `device_class`, unité, suffixe d'identifiant, etc. Le service MQTT de Gladys contient déjà une table `device_class` HA → catégorie et type Gladys (`server/services/mqtt/lib/homeAssistant/constants.js`), qui servira de base. Le détail relève de la section 03.

## 7. Sécurité

### 7.1 Contrôle d'accès

- Toutes les routes du service sont **réservées aux administrateurs** (même mécanisme que les routes d'installation des intégrations). Elles sont listées en 09 § 2.
- L'URL de HA saisie est validée : protocole `http` ou `https`, sans redirection suivie automatiquement. L'appel part du serveur Gladys vers une adresse choisie par l'administrateur. C'est un usage légitime, mais il ne doit pas devenir un relais générique.

### 7.2 Secrets et fichiers

- **Coffre éphémère** : les secrets vivent en mémoire, sont associés à l'exécution en cours et sont effacés à la fin de chaque phase qui les utilise, à l'abandon de l'exécution et à l'arrêt de Gladys.
- **Reprise après redémarrage** (C10) : le coffre est alors vide. Si la suite de l'exécution a besoin d'un secret, l'outil le **redemande** avant de reprendre (message `migration.secret-required`, 09 § 3) : le **jeton HA** (mode B, ou relecture par l'API) et, si les fichiers sensibles de la sauvegarde doivent être relus (préremplissage pas encore fait), le **mot de passe de la sauvegarde** avec un nouveau téléchargement (mode A) ou un nouvel envoi du fichier (mode C).
- **Journaux** : aucun secret n'est journalisé. Les URL contenant un jeton sont expurgées.
- **Sauvegarde : extraction sélective pendant le transfert** (C9). Une sauvegarde HA peut peser plusieurs gigaoctets lorsqu'elle contient la base du *recorder*, et, en mode A, HA la renvoie **déchiffrée** ([core] `components/backup/http.py:120-160`) : elle contient alors en clair `secrets.yaml`, `core.config_entries` et `auth`. Elle n'est donc **jamais écrite telle quelle sur disque**. Pendant le téléchargement (mode A) ou le téléversement (mode C), elle est lue **en flux** :
  - les fichiers sensibles (02 § 4.2) sont lus **en mémoire** et jamais écrits sur disque ;
  - seuls la **base du *recorder* et son WAL** et, si besoin (bascule, § 9.3), l'**archive du module Zigbee2MQTT** sont écrits dans le dossier de travail du service ;
  - le reste est ignoré.
- **Durée de vie du dossier de travail** (C10) : il est **conservé entre deux redémarrages** tant qu'une exécution est ouverte (`running`, `paused` ou `interrupted`), pour que l'import des historiques reprenne au point exact (07 § 6). Il est **purgé** à la fin de l'exécution, à son abandon ou à l'annulation. Au démarrage, Gladys ne purge que les dossiers qui ne correspondent à aucune exécution ouverte. Cela nuance le principe 7 de la vision : la base d'historique ne reste pas « en mémoire », mais elle ne survit jamais à l'exécution.
- *Remplacé le 2026-11-19 : ~~la sauvegarde est écrite en flux dans un dossier de travail, supprimée dès que l'analyse et l'import des historiques n'en ont plus besoin, et purgée à chaque démarrage de Gladys~~.*
- **Rapport exporté** : il est expurgé de tout identifiant ou secret.

### 7.3 Obtenir le fichier de sauvegarde (D18)

**Ce qu'on peut reprendre de la restauration Gladys Plus.** La restauration d'une sauvegarde Gladys Plus ne reçoit pas de fichier depuis le navigateur : c'est le **serveur Gladys qui télécharge** l'archive depuis Gladys Plus (`server/lib/gateway/gateway.downloadBackup.js`). Sa route (`post /api/v1/gateway/backup/restore`) ne peut donc pas servir de route de téléversement. En revanche, une bonne partie de sa mécanique est réutilisable telle quelle :
- l'écriture **en flux sur disque**, dans un dossier de travail vidé avant chaque usage (`backupsFolder/restore`) ;
- les **contrôles de sécurité des archives** : noms de fichiers sûrs et entrées tar sans traversée de chemin (`server/utils/backupSafety.js` : `assertSafeBackupName`, `isSafeArchiveEntry`) ;
- l'appel aux binaires `tar` et `gzip` et la remontée de progression au front.

**Trois façons d'obtenir la sauvegarde HA, par ordre de préférence :**

| # | Voie | Modification du cœur | Quand |
|---|---|---|---|
| 1 | **Gladys télécharge la sauvegarde depuis HA**, par l'API, avec le jeton (même schéma que Gladys Plus : le serveur tire le fichier) | **Aucune** | HA est en marche. C'est le cas courant, et cela donne en plus les données des modules complémentaires (Zigbee2MQTT, § 9.3) et la base du *recorder* en un seul fichier. Point d'API confirmé en 02 § 3.3. |
| 2 | **Téléversement depuis le navigateur** : route réservée aux administrateurs, corps brut en flux (`application/octet-stream`) lu avec extraction sélective (§ 7.2), taille maximale configurable, contrôle d'espace disque préalable (`system.getDiskSpace`) | **Oui** : nouvelle route et nouveau middleware (le précédent `audioRawBodyMiddleware.js` est limité à 5 Mo et lu en mémoire) | HA est arrêté ou hors d'usage, et l'utilisateur n'a que son fichier. |
| 3 | **Dépôt manuel du fichier** dans un dossier surveillé par Gladys (volume Docker) | **Aucune** | Solution de secours pour les très gros fichiers, ou si la voie 2 n'est pas acceptée. Moins conviviale. |

La voie 1 combinée à la voie 3 suffit à une première version **sans aucune modification du cœur** pour le téléversement. La voie 2 reste souhaitable pour le confort de l'utilisateur et sera défendue sur le forum.

## 8. Estimation du volume des historiques (D24)

1. **Compter** : pour chaque entité retenue, on obtient le nombre de points d'états et de statistiques. La méthode dépend de la source et sera définie en section 02.
2. **Estimer** : on multiplie par un coût moyen par point dans DuckDB. Ce coût sera mesuré sur une instance réelle, en s'aidant de `device.getDeviceStatesSize` (`server/lib/device/device.getDeviceStatesSize.js`), et non supposé.
3. **Comparer** à l'espace libre (`system.getDiskSpace`, `server/lib/system/system.getDiskSpace.js`), avec une marge de sécurité.
4. **Décider** : si l'espace suffit avec la marge, l'import est proposé. S'il suffit sans la marge, un avertissement est affiché. S'il ne suffit pas, l'import est bloqué, avec la liste des entités les plus volumineuses pour que l'utilisateur puisse en écarter.

## 9. Interface et points d'entrée (A10)

### 9.1 Points d'entrée

| # | Où | Comportement |
|---|---|---|
| 1 | **Premier lancement de Gladys** : écran d'accueil (`front/src/routes/signup/1-welcome/WelcomeTab.jsx`), à côté de « Restaurer une sauvegarde » | Une ligne secondaire « Je viens de Home Assistant ». La migration exige un administrateur : l'utilisateur crée donc d'abord son compte et sa maison (étapes existantes), puis l'assistant de migration s'ouvre au lieu du tableau de bord. |
| 2 | **Paramètres → Migration** : nouvelle page `/dashboard/settings/migration`, à côté de « Sauvegardes » et « Tâches de fond » | Page d'accueil de la migration. Elle liste les systèmes sources (Home Assistant pour l'instant, d'autres plus tard, A8) et l'historique des exécutions, et c'est là qu'on relance, resynchronise, passe en bascule ou annule. |
| 3 | **Catalogue des intégrations** | Une carte « Home Assistant » qui renvoie vers la page Migration. C'est là qu'un utilisateur cherchera naturellement « Home Assistant ». |
| 4 | **Tâches de fond** (page existante) | Les tâches longues de l'outil (installation, historiques, annulation) y apparaissent automatiquement via `gladys.job`. |

Une page dans **Paramètres** est préférable à la page **Système**, qui regroupe des informations techniques (version, disque, conteneurs). Une page dédiée reste lisible quand d'autres sources s'y ajouteront.

### 9.2 Conventions

L'interface est un **assistant** qui suit les phases (§ 6 et 6 bis de la vision). Elle reprend les conventions de Gladys : Preact, actions par page, traductions **en, fr, de et es** contrôlées en CI, et messages WebSocket déclarés dans `WEBSOCKET_MESSAGE_TYPES` (`server/utils/constants.js:2194`), ce qui est une modification du cœur (A7). Le serveur ne renvoie que des **codes**, que le front traduit (pas d'i18n côté serveur). Le contrat entre le front et le service (routes, messages WebSocket, codes) est défini en **09**. L'écran par écran relève de la section 05.

### 9.3 De la coexistence à la bascule (A11)

À la fin d'une migration en coexistence, l'outil explique que, pour se séparer définitivement de HA, il suffit de **relancer l'outil en mode bascule**. La relance :
1. resynchronise ce qui a changé dans HA depuis (D13) ;
2. **active** les scènes issues d'automatisations, créées désactivées (choix par scène), et **ajoute** les appels `scene.start` différés (04 § 7), modifications journalisées (§ 5.4) ;
3. traite les **dépendances à HA** restées en place, au premier rang desquelles le Zigbee2MQTT partagé (cas QA3). Si Zigbee2MQTT tournait comme module complémentaire de HA, il disparaîtra avec lui. L'outil propose alors de le faire gérer par Gladys **en reprenant ses données** (configuration, base des appareils appairés, sauvegarde du coordinateur), afin d'éviter tout réappairage. Ces données se trouvent dans la sauvegarde HA, dans l'archive du module complémentaire ; c'est le seul fichier sensible écrit sur disque (§ 7.2), le temps de l'exécution de bascule. *Format et faisabilité à vérifier en section 02.* Le passage de ZHA à Zigbee2MQTT ne permet pas cette reprise : il demande un réappairage, et l'outil le dit clairement.

## 10. Testabilité

Le dépôt Gladys exige une **couverture de 100 % des lignes modifiées**. L'architecture en tient compte :

- les **sources** (client WebSocket HA, lecture de la sauvegarde) sont **injectées** et remplaçables par des simulations ;
- l'**analyseur** et le **convertisseur de scènes** sont des fonctions pures, testées avec des jeux de données d'inventaire ;
- les **exécuteurs** n'appellent que les API du cœur, simulées avec Sinon comme dans les autres services ;
- un **jeu de données de référence** (installations HA anonymisées : API et sauvegarde) sera constitué. Voir section 06.

## 11. Arborescence prévisionnelle

```
server/services/migration-home-assistant/
├── index.js                     # start/stop, enregistrement du service
├── api/                         # contrôleur et routes (admin, 09)
├── lib/
│   ├── sources/
│   │   ├── api/                 # client WebSocket/REST HA
│   │   └── backup/              # extraction, déchiffrement, lecture
│   ├── inventory/               # modèle pivot et normalisation
│   ├── catalog/                 # client du catalogue distant et cache
│   ├── analyzer/                # classement → plan
│   ├── orchestrator/            # phases, reprise, tâches de fond
│   ├── executors/               # intégrations, rapprochement, appareils, appareils virtuels, pièces,
│   │                            # utilisateurs, zones et présence, scènes, historiques, tableaux de bord,
│   │                            # réglages, annulation
│   ├── scenes/                  # convertisseur V1 (socle 04 § 2.4), fiches lisibles, scènes de présence
│   ├── dashboards/              # Accueil généré et reprise des tableaux de bord HA (08)
│   └── vault/                   # coffre éphémère des secrets
├── catalog-snapshot.json        # instantané embarqué du catalogue
├── rules/                       # règles des services natifs et tables par défaut (03 § 3.2, D32)
├── package.json                 # dépendances propres au service (A7)
└── utils/
server/migrations/               # création des tables t_migration_*
server/models/                   # modèles des tables t_migration_*
front/src/routes/settings/settings-migration/
```

## 12. Décisions d'architecture proposées

Elles seront formalisées en ADR une fois cette section validée.

| # | Proposition |
|---|---|
| A1 | Service du cœur, réservé aux administrateurs (confirmation de D1, § 2) |
| A2 | Inventaire normalisé (modèle pivot) commun aux deux sources (§ 4) |
| A3 | Persistance dans des tables dédiées `t_migration_*`, sans modifier les modèles existants (§ 5.2) |
| A4 | Règles de rapprochement déclarées dans le catalogue, et non codées par intégration (§ 6.2) |
| A5 | Création d'appareil uniquement à partir du payload découvert (§ 6.3) |
| A6 | Coffre éphémère des secrets. ~~Sauvegarde en fichier temporaire purgé.~~ **Amendée le 2026-11-19 (C9, C10)** : extraction sélective pendant le transfert, seuls la base du *recorder* et l'archive Zigbee2MQTT vont sur disque, dans un dossier conservé tant que l'exécution est ouverte (§ 7.2). |
| A7 | Les nouvelles dépendances (lecteur tar en flux, YAML, libsodium) sont **propres au service**, dans son `package.json`, comme `yaml` pour `zigbee2mqtt` : aucune dépendance ajoutée au cœur. **Modifications du cœur** (liste complétée le 2026-11-19, F1, F3) : tables `t_migration_*` ; page de paramètres ; params réservés `GLADYS_HW_*` (03 § 2.4) ; nouveaux **`JOB_TYPES`** pour les tâches du service et leurs traductions dans le front (§ 3.5, 09 § 3) ; nouveaux **`WEBSOCKET_MESSAGE_TYPES`** (§ 9.2, 09 § 3). **Facultatives, non bloquantes** : la route de téléversement en flux (§ 7.3, voie 2) ; un statut « en pause » pour `gladys.job` (§ 3.5) ; une fonction « **supprimer les états d'une fonction entre deux dates** », qui permettrait une annulation complète de l'historique (§ 5.4). À présenter sur le forum et à Pierre-Gilles (D18). |
| A8 | Nom du service : **`migration-home-assistant`**. Code séparé entre une partie générique et une partie propre à HA, tables `t_migration_*` avec colonne `source`, pour d'autres systèmes à venir (ex-QA1). |
| A9 | **Garantie de retour à l'état initial** : l'annulation ne défait que ce que l'outil a fait, y compris la restauration des valeurs modifiées. L'outil ne remplace jamais une configuration existante (§ 5.4 et 5.5, ex-QA2). **Amendée le 2026-11-19 (F1)** : l'annulation porte sur toute la migration ; l'historique HA importé sur un appareil **préexistant** est conservé, et l'écran de confirmation le dit. |
| A10 | Points d'entrée : premier lancement (à côté de « Restaurer une sauvegarde »), **Paramètres → Migration**, carte dans le catalogue des intégrations (§ 9.1, ex-QA4). |
| A11 | Passage de la coexistence à la bascule en **relançant l'outil en mode bascule**, avec reprise des données de Zigbee2MQTT si possible (§ 9.3, ex-QA3). |
| A12 | **Une migration par source**, une seule ouverte à la fois, regroupant plusieurs exécutions (initiale, relances, bascule, annulation) ; identité stable de chaque élément ; deux exécutions simultanées interdites (§ 5.1 bis, T2, T9). |
| A13 | **Pause et reprise gérées par le service** : curseur en base, nouvelle tâche `gladys.job` à chaque reprise, relance automatique d'une tâche marquée échouée au redémarrage (§ 3.5, F3). |

## 13. Questions ouvertes

| # | Question |
|---|---|
| — | *QA1 à QA4 sont tranchées : voir A8 à A11.* |
