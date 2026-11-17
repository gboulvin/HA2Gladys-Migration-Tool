# 01 — Architecture

- **Statut** : 📝 Brouillon
- **Dernière mise à jour** : 2026-11-17
- **Prérequis** : [00 — Vision](00-vision.md), [note de recherche sur le code Gladys](research/gladys-codebase.md)
- **Référence Gladys étudiée** : commit `71edb17` du dépôt `GladysAssistant/Gladys`

Les affirmations sur Gladys renvoient au code (chemins relatifs au dépôt Gladys). Celles sur HA sont marquées *à vérifier* tant que la section 02 ne les a pas confirmées.

---

## 1. Vue d'ensemble

L'outil est un **service du cœur de Gladys** (D1), nommé provisoirement **`home-assistant-migration`**. Il se compose d'un backend dans `server/services/home-assistant-migration/` et d'une interface dans `front/src/routes/integration/all/home-assistant-migration/`.

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
                    │ │          │   │ d'appareils  │  │           │ │   │scènes (L2)│ │ dédiées)│  │
                    │ └────┬─────┘   └──────┬───────┘  └─────┬─────┘ │   └─────┬─────┘ └─────────┘  │
                    │      ▼                ▼                ▼       │         ▼                    │
                    │  API internes du cœur : externalIntegration.*, device.*, room.*, user.*, scene.*│
                    └────────────────────────────────────────────────┼──────────────────────────────┘
                                                                     │
                                                       ┌─────────────┴────────────┐
                                                       │ Catalogue distant (D21)  │
                                                       │ base : tracker William   │
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
| Transport | WebSocket HA (registres, états, statistiques, historique) et REST (configuration des automatisations) *à vérifier* | Lecture du fichier : `.storage/core.*_registry`, `automations.yaml`, base du *recorder* *à vérifier* |
| Historiques | Via l'API (statistiques et historique), quel que soit le moteur de base de HA | Uniquement si la base est en SQLite **et** incluse dans la sauvegarde |
| Dépendances existantes | `ws` (déjà dans `server/package.json`) | `sqlite3` (déjà présent) pour lire la base du *recorder*, binaire `tar` (précédent : sauvegardes Gladys) |
| Dépendances à ajouter | Aucune | Un **analyseur YAML** (aucun dans le serveur) et le **déchiffrement des sauvegardes HA** (format SecureTar *à vérifier*) |

Le détail (commandes, formats, versions de HA) relève de la section 02.

### 3.2 Inventaire (modèle pivot)

Structure normalisée et indépendante de la source. Elle est décrite au § 4.

### 3.3 Client du catalogue

- Il télécharge le catalogue distant (D12, D21). Il garde un **cache local** et embarque un **instantané** dans le code du service, au cas où le catalogue serait injoignable lors de la première utilisation.
- Il reprend le schéma déjà éprouvé du **store des intégrations** (`docs/specs/external-integrations/core/store.md`) : index statique publié sur GitHub Pages, rafraîchi périodiquement, cache local et fonctionnement dégradé hors ligne.
- Le format du catalogue sera défini en section 03, avec William-De71.

### 3.4 Analyseur

L'analyseur prend en entrée l'inventaire, le catalogue et l'état actuel de Gladys (intégrations installées, appareils existants, pièces, utilisateurs). Il produit le **plan de migration** : un statut par élément (§ 8 de la vision), les intégrations à installer, les rapprochements prévus et l'estimation du volume d'historique (D24).

C'est une fonction **pure** : sans effet de bord, elle est entièrement testable avec des jeux de données.

### 3.5 Orchestrateur de phases

Il fait avancer la migration de phase en phase (§ 6 bis de la vision), en conservant son état (§ 5). Les traitements longs (installation, rapprochement, historiques, scènes) s'exécutent en **tâches de fond** avec progression via `gladys.job` (`server/lib/job/`), et les événements sont poussés au front par WebSocket.

### 3.6 Exécuteurs

| Exécuteur | Rôle | API du cœur utilisées |
|---|---|---|
| **Installeur d'intégrations** | Installe les intégrations externes cochées et configure les natives. Préremplit leur configuration (avec l'accord de l'utilisateur pour les secrets, voir § 9 bis de la vision). | `externalIntegration.install`, `externalIntegration.setIntegrationConfig`, variables des services natifs |
| **Rapprocheur** | Associe chaque appareil HA à un appareil **découvert** dans Gladys (§ 6). | `externalIntegration.getDiscoveredDevices`, découverte des services natifs (ex. `zigbee2mqtt`) |
| **Créateur d'appareils** | Crée l'appareil Gladys **à partir du payload découvert**, puis applique le nom et la pièce HA. | `device.create` (voir l'avertissement du § 6.3) |
| **Créateur de pièces et d'utilisateurs** | Crée les pièces (avec renommage « Étage - Pièce » si l'utilisateur l'a choisi, D14) et les utilisateurs (D11). | `room.*`, `user.*` |
| **Importeur d'historiques** | Copie les états et statistiques HA vers les fonctions migrées, par lots. | `device.saveMultipleHistoricalStates`, `device.replaceHistoricalStatesFrom` |
| **Convertisseur de scènes** (lot 2) | Traduit les automatisations convertibles en scènes. | `scene.create` |

### 3.7 Journal de migration

Il conserve la correspondance entre chaque élément HA et l'objet Gladys créé. Il sert à l'idempotence, à la réversibilité, à la reprise et à la resynchronisation (§ 5).

## 4. Inventaire : le modèle pivot

| Entité | Champs principaux | Usage |
|---|---|---|
| `Area` | id HA, nom, étage (id) | Pièces Gladys |
| `Floor` | id HA, nom, niveau | Uniquement pour le renommage proposé (D14) |
| `IntegrationEntry` | domaine HA (ex. `hue`, `mqtt`), titre, identifiants **si présents** (marqués secrets) | Recherche dans le catalogue, préremplissage des configurations |
| `Device` | id HA, nom (y compris le nom défini par l'utilisateur), fabricant, modèle, `area_id`, **identifiants** (`identifiers`) et **connexions** (`connections` : adresse MAC, adresse Zigbee…) *à vérifier*, intégration d'origine, désactivé ou non | Classement et rapprochement |
| `Entity` | `entity_id`, appareil parent, domaine, `device_class`, `state_class`, unité, désactivée ou masquée | Correspondance des fonctions, historiques |
| `Person` | id HA, nom, utilisateur HA lié, appareils de suivi | Utilisateurs Gladys (D11) |
| `Automation` | id HA, alias, déclencheurs, conditions, actions, mode | Lot 2 |
| `HistorySource` | accès en flux aux états et statistiques d'une entité, avec une estimation du nombre de points | Historiques (D22 à D24) |

**Règle de confidentialité** : l'inventaire ne garde **aucun secret**. Les identifiants des intégrations restent en mémoire, dans un coffre éphémère (§ 7), et ne sont référencés dans l'inventaire que par un identifiant opaque.

## 5. Persistance et reprise

### 5.1 Ce qui est conservé

| Donnée | Conservée ? | Où | Pourquoi |
|---|---|---|---|
| Inventaire normalisé (sans secrets) | Oui | Table dédiée | Reprendre la migration après la phase B sans relire HA |
| Plan de migration et choix de l'utilisateur | Oui | Table dédiée | Reprise, rapport final |
| Correspondance élément HA → objet Gladys | Oui | Table dédiée | Idempotence, réversibilité, resynchronisation, lot 2 |
| Curseur d'import des historiques par fonction | Oui | Table dédiée | Reprise après interruption, resynchronisation incrémentale |
| Jeton HA, mot de passe de sauvegarde, identifiants | **Non** | Mémoire uniquement | Confidentialité (§ 9 bis de la vision) |
| Fichier de sauvegarde | **Temporairement** | Dossier de travail du service, supprimé à la fin de l'analyse **et** au démarrage de Gladys | Voir § 7.2 |

### 5.2 Tables proposées

Gladys n'a aucun marqueur d'origine sur ses objets (voir la note de recherche). Plutôt que de modifier les modèles existants, on ajoute des tables propres au service, avec une migration Sequelize :

- **`t_ha_migration_run`** : une exécution de l'outil (source, mode, phase courante, dates, résumé).
- **`t_ha_migration_item`** : un élément HA dans une exécution. Champs : type (`area`, `device`, `entity`, `person`, `integration`, `automation`), id HA, empreinte du contenu, statut, choix de l'utilisateur, **type et id de l'objet Gladys créé**, erreur éventuelle.
- **`t_ha_migration_history_cursor`** : par fonction Gladys, l'entité HA source et la dernière date importée (états et statistiques séparément).

Les variables de service (`t_variable`) ont été écartées. Elles sont prévues pour des paires clé-valeur et n'offrent ni requêtes ni intégrité référentielle sur des milliers d'éléments.

**Suppression d'un objet dans Gladys** : si l'utilisateur supprime à la main un appareil importé, le journal pointe vers un objet absent. À la relance, l'élément est considéré comme « non migré » et redevient proposable. *Une suppression en cascade côté Gladys n'est pas envisagée : elle toucherait au cœur des modèles.*

### 5.3 Idempotence et resynchronisation (D13)

À la relance, l'analyseur compare l'inventaire neuf au journal :

| Cas | Traitement |
|---|---|
| Élément HA déjà migré, objet Gladys toujours présent | Statut « déjà migré », pas d'action |
| Élément HA déjà migré mais modifié dans HA (empreinte différente) | Signalé. L'utilisateur choisit de mettre à jour ou d'ignorer. On n'écrase jamais en silence. |
| Nouvel élément HA | Proposé à l'import |
| Élément HA supprimé depuis | Signalé. L'objet Gladys n'est pas supprimé automatiquement. |
| Historique | Import incrémental à partir du curseur |

### 5.4 Réversibilité

« Annuler la migration » supprime les objets Gladys listés dans le journal, en ordre inverse (scènes, puis historiques, appareils, utilisateurs, pièces vides), après une confirmation qui récapitule ce qui sera supprimé. **Les intégrations installées ne sont pas désinstallées automatiquement** : l'utilisateur peut avoir créé d'autres appareils avec elles entre-temps. La liste lui est proposée.

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

| Type de règle | Exemple | Clé HA *à vérifier* | Clé Gladys |
|---|---|---|---|
| Adresse IEEE | Zigbee (ZHA, Zigbee2MQTT) | connexion `zigbee` ou identifiant de l'appareil | `ieee_address` (`server/services/zigbee2mqtt/utils/convertDevice.js:26`) |
| Adresse MAC | Shelly, Tasmota, ESPHome, Google Cast | connexion `mac` | paramètre ou `external_id` de l'intégration |
| Numéro de série ou identifiant constructeur | Hue, Tuya | `identifiers` | paramètre ou `external_id` |
| Motif sur l'`external_id` | Intégrations dont l'identifiant est prévisible | `identifiers` | gabarit déclaré dans le catalogue |
| Manuel | Matter (l'identifiant n'existe qu'après l'appairage) | — | choix de l'utilisateur, aidé par le nom, le fabricant et le modèle |

### 6.3 Création d'un appareil rapproché

- L'appareil est créé **à partir du payload découvert**, tel que l'intégration l'a publié. Seuls le **nom**, la **pièce** et le réglage `keep_history` sont modifiés.
- ⚠️ `device.create` remplace l'appareil existant de même `external_id` et **supprime les fonctions et paramètres absents** du payload (voir la note de recherche). L'exécuteur ne doit donc jamais envoyer un payload partiel. Pour un appareil déjà présent dans Gladys, il n'utilise que les mises à jour ciblées (nom, pièce).
- Une fonction HA dont l'équivalent n'apparaît pas dans le payload découvert est marquée « perdue » dans le rapport (statut ⚠️ « migrable partiellement »).

### 6.4 Correspondance des entités et des fonctions

Pour les historiques et les scènes, il faut savoir quelle **entité HA** correspond à quelle **fonction Gladys**. La règle est également déclarée dans le catalogue : `device_class`, unité, suffixe d'identifiant, etc. Le service MQTT de Gladys contient déjà une table `device_class` HA → catégorie et type Gladys (`server/services/mqtt/lib/homeAssistant/constants.js`), qui servira de base. Le détail relève de la section 03.

## 7. Sécurité

### 7.1 Contrôle d'accès

- Toutes les routes du service sont **réservées aux administrateurs** (même mécanisme que les routes d'installation des intégrations).
- L'URL de HA saisie est validée : protocole `http` ou `https`, sans redirection suivie automatiquement. L'appel part du serveur Gladys vers une adresse choisie par l'administrateur. C'est un usage légitime, mais il ne doit pas devenir un relais générique.

### 7.2 Secrets et fichiers

- **Coffre éphémère** : les secrets vivent en mémoire, sont associés à l'exécution en cours et sont effacés à la fin de chaque phase qui les utilise, à l'abandon de l'exécution et à l'arrêt de Gladys.
- **Journaux** : aucun secret n'est journalisé. Les URL contenant un jeton sont expurgées.
- **Sauvegarde téléversée** : une sauvegarde HA peut peser plusieurs gigaoctets lorsqu'elle contient la base du *recorder*. Elle ne tient donc pas en mémoire. Elle est écrite en **flux** dans un dossier de travail du service, puis **supprimée** dès que l'analyse et l'import des historiques n'en ont plus besoin, et purgée à chaque démarrage de Gladys. Cela nuance le principe 7 de la vision : la sauvegarde ne reste pas « en mémoire », mais elle ne survit jamais à la migration.
- **Rapport exporté** : il est expurgé de tout identifiant ou secret.

### 7.3 Modification du cœur pour le téléversement (D18)

Le cœur limite le corps des requêtes JSON à 100 ko et n'accepte pas l'envoi de fichiers (`server/api/middlewares/jsonBodyMiddleware.js`). Proposition, à argumenter sur le forum :

- une route dédiée, réservée aux administrateurs, qui accepte un corps **brut en flux** (`application/octet-stream`) et l'écrit directement sur disque, sans le charger en mémoire ;
- un précédent existe avec le corps brut audio de la reconnaissance vocale (`audioRawBodyMiddleware.js`), mais il est limité à 5 Mo et lu en mémoire. Il faut ici un **flux vers disque** ;
- une taille maximale configurable, et un contrôle de l'espace disque **avant** l'écriture (`system.getDiskSpace`).

## 8. Estimation du volume des historiques (D24)

1. **Compter** : pour chaque entité retenue, on obtient le nombre de points d'états et de statistiques. La méthode dépend de la source et sera définie en section 02.
2. **Estimer** : on multiplie par un coût moyen par point dans DuckDB. Ce coût sera mesuré sur une instance réelle, en s'aidant de `device.getDeviceStatesSize` (`server/lib/device/device.getDeviceStatesSize.js`), et non supposé.
3. **Comparer** à l'espace libre (`system.getDiskSpace`, `server/lib/system/system.getDiskSpace.js`), avec une marge de sécurité.
4. **Décider** : si l'espace suffit avec la marge, l'import est proposé. S'il suffit sans la marge, un avertissement est affiché. S'il ne suffit pas, l'import est bloqué, avec la liste des entités les plus volumineuses pour que l'utilisateur puisse en écarter.

## 9. Interface (aperçu)

L'interface est un **assistant** qui suit les phases (§ 6 et 6 bis de la vision), dans une page d'intégration du front. Elle reprend les conventions de Gladys : Preact, actions par page, traductions **en, fr, de et es** contrôlées en CI, et messages WebSocket déclarés dans `WEBSOCKET_MESSAGE_TYPES`. Le serveur ne renvoie que des **codes**, que le front traduit (pas d'i18n côté serveur). L'écran par écran relève de la section 05.

## 10. Testabilité

Le dépôt Gladys exige une **couverture de 100 % des lignes modifiées**. L'architecture en tient compte :

- les **sources** (client WebSocket HA, lecture de la sauvegarde) sont **injectées** et remplaçables par des simulations ;
- l'**analyseur** et le **convertisseur de scènes** sont des fonctions pures, testées avec des jeux de données d'inventaire ;
- les **exécuteurs** n'appellent que les API du cœur, simulées avec Sinon comme dans les autres services ;
- un **jeu de données de référence** (installations HA anonymisées : API et sauvegarde) sera constitué. Voir section 06.

## 11. Arborescence prévisionnelle

```
server/services/home-assistant-migration/
├── index.js                     # start/stop, enregistrement du service
├── api/                         # contrôleur et routes (admin)
├── lib/
│   ├── sources/
│   │   ├── api/                 # client WebSocket/REST HA
│   │   └── backup/              # extraction, déchiffrement, lecture
│   ├── inventory/               # modèle pivot et normalisation
│   ├── catalog/                 # client du catalogue distant et cache
│   ├── analyzer/                # classement → plan
│   ├── orchestrator/            # phases, reprise, tâches de fond
│   ├── executors/               # intégrations, rapprochement, appareils, pièces, utilisateurs, historiques
│   ├── scenes/                  # convertisseur (lot 2)
│   └── vault/                   # coffre éphémère des secrets
├── catalog-snapshot.json        # instantané embarqué du catalogue
└── utils/
server/migrations/               # création des tables t_ha_migration_*
server/models/                   # modèles des tables t_ha_migration_*
front/src/routes/integration/all/home-assistant-migration/
```

## 12. Décisions d'architecture proposées

Elles seront formalisées en ADR une fois cette section validée.

| # | Proposition |
|---|---|
| A1 | Service du cœur, réservé aux administrateurs (confirmation de D1, § 2) |
| A2 | Inventaire normalisé (modèle pivot) commun aux deux sources (§ 4) |
| A3 | Persistance dans des tables dédiées `t_ha_migration_*`, sans modifier les modèles existants (§ 5.2) |
| A4 | Règles de rapprochement déclarées dans le catalogue, et non codées par intégration (§ 6.2) |
| A5 | Création d'appareil uniquement à partir du payload découvert (§ 6.3) |
| A6 | Coffre éphémère des secrets, sauvegarde en fichier temporaire purgé (§ 7.2) |
| A7 | Modifications du cœur requises : route de téléversement en flux (§ 7.3), dépendance YAML, déchiffrement des sauvegardes HA (§ 3.1). À présenter sur le forum (D18). |

## 13. Questions ouvertes

| # | Question |
|---|---|
| QA1 | Nom définitif du service : `home-assistant-migration`, `ha-import`… ? |
| QA2 | Annulation (§ 5.4) : faut-il aussi proposer de supprimer **l'historique importé** sur les appareils qui existaient déjà dans Gladys avant la migration ? |
| QA3 | Mode coexistence avec Zigbee2MQTT partagé : Gladys se connecte au Zigbee2MQTT de HA. Si l'utilisateur arrête HA plus tard (et avec lui son Zigbee2MQTT), l'outil doit-il **aider à basculer** vers un Zigbee2MQTT géré par Gladys ? |
| QA4 | Où apparaît l'outil dans le catalogue des intégrations du front (catégorie, voir `docs/specs/integration-catalog-categories.md`) ? Faut-il en plus une **mise en avant à l'accueil** pour les nouveaux utilisateurs ? |
