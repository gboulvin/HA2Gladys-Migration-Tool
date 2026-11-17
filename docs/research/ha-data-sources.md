# Recherche — Sources de données Home Assistant (API et sauvegarde)

- **Date** : 2026-11-17
- **Statut** : note de recherche factuelle (entrée de la section 02 `02-extraction-ha.md`, et en partie des sections 04 et 07)
- **Commits étudiés** :
  - `home-assistant/core` : `9b5a396` (2026-11-17, branche `dev`, version `2026.12.0.dev0` d'après `homeassistant/const.py:24-26`), clone sparse dans `/home/user/ref/ha-core`. Les constantes de format les plus importantes ont été **comparées au tag publié `2026.11.2`** (fichiers bruts de GitHub) : voir § 0.
  - `home-assistant-libs/securetar` : `08db90e` (2026-11-02, version de paquet `2026.10.0`, `pyproject.toml:7`), dans `/home/user/ref/securetar`.
  - `home-assistant/supervisor` : `e610ca8` (2026-11-16), dossier `supervisor/backups`, dans `/home/user/ref/supervisor`.

Conventions :

- Préfixes de chemins : **[core]** = dépôt `home-assistant/core`, **[st]** = `securetar`, **[sup]** = `supervisor`. `fichier:NN` renvoie à la ligne NN au commit étudié.
- « **non vérifié** » : affirmation qui ne vient pas de la lecture du code étudié (documentation, mémoire, bibliothèque absente du clone).
- Les paragraphes **« Analyse »** sont des déductions pour notre service, pas des faits sur HA.
- HA a remplacé `voluptuous` par une bibliothèque de schémas nommée `probatio` (ex. [core] `homeassistant/components/websocket_api/auth.py:32`). C'est sans effet pour un client, mais explique la syntaxe des extraits cités.

---

## 0. Écart entre le commit étudié (dev 2026.12) et la version publiée 2026.11.2

| Élément | dev `9b5a396` | `2026.11.2` publié | Conséquence |
|---|---|---|---|
| `SCHEMA_VERSION` du recorder | 53 | 53 | Identique |
| `core.device_registry` | 3.4 | 3.4 | Identique |
| `core.entity_registry` | 1.24 | 1.24 | Identique |
| `core.area_registry` | 1.9 | 1.9 | Identique |
| `core.config_entries` | 1.5 | 1.5 | Identique |
| `SECURETAR_CREATE_VERSION` (intégration `backup`) | 3 | 3 | Identique |
| `history/history_during_period` | — | — | Fichier identique (0 ligne de diff) |
| `recorder/*` (statistiques) | — | — | Diff cosmétique (`UNDEFINED` → `Undefined`) |
| `config/entity_registry/get_automatic_entity_ids` | paramètre `overrides` en plus | absent | Sans effet pour nous |
| **`recorder/database_usage`** ([core] `components/recorder/database_usage.py`) | **présent** | **absent (HTTP 404 sur le tag)** | Comptage des points d'historique disponible seulement à partir de 2026.12 (probable, non vérifié) |

**Analyse** : pour tout ce qui suit, le commit étudié est représentatif de 2026.11, sauf la commande `recorder/database_usage` (§ 1.9).

---

## 1. API WebSocket

### 1.1 Connexion et authentification

- Point d'accès : `/api/websocket` ([core] `components/websocket_api/const.py:20`).
- Flux ([core] `components/websocket_api/auth.py:25-46`, `:85-133`) :
  1. le serveur envoie `{"type": "auth_required", "ha_version": "2026.11.x"}` ;
  2. le client répond `{"type": "auth", "access_token": "<jeton>"}` (le champ `api_password`, exclusif, est un reliquat) ;
  3. le serveur répond `{"type": "auth_ok", "ha_version": "…"}` ou `{"type": "auth_invalid", "message": "…"}` puis ferme.
- Pendant la phase d'authentification, un message trop gros, binaire ou non JSON coupe la connexion ([core] `components/websocket_api/http.py:420-438`). La valeur de `MAX_UNAUTHENTICATED_BODY_SIZE` est dans le composant `http`, absent du clone : **non vérifiée**.
- Un jeton longue durée est un *refresh token* de type `long_lived_access_token` ([core] `homeassistant/auth/models.py:23`). Il porte l'identité et les droits de l'utilisateur qui l'a créé.
- Le refus pour les utilisateurs `local_only` connectés depuis une IP distante passe par `async_user_not_allowed_do_auth` du composant `http` ([core] `components/websocket_api/auth.py:10,100`) : comportement exact **non vérifié**.
- Après authentification, chaque commande porte un `id` entier strictement croissant ([core] `components/websocket_api/connection.py:272-300`, erreur `id_reuse`).
- Réponse : `{"id", "type": "result", "success": true, "result": …}` ou `{"id", "type": "result", "success": false, "error": {"code", "message"}}` ([core] `components/websocket_api/messages.py:63-103`). Codes : `unauthorized`, `not_found`, `invalid_format`, `unknown_command`, `home_assistant_error`… ([core] `components/websocket_api/const.py:35-47`).
- Le serveur envoie des *heartbeats* toutes les 55 s ([core] `components/websocket_api/http.py:103`).
- Si le client annonce `{"type": "supported_features", "features": {"coalesce_messages": 1}}`, le serveur peut regrouper plusieurs messages dans un **tableau JSON** ([core] `components/websocket_api/const.py:58`, `http.py:157-184`). Sans cette annonce, un message = un objet.

### 1.2 Droits administrateur

Un utilisateur est admin s'il est propriétaire ou membre actif du groupe `system-admin` ([core] `homeassistant/auth/models.py:92-96`, `auth/const.py:9-11`). Le décorateur `require_admin` lève `Unauthorized` sinon ([core] `components/websocket_api/decorators.py:54-69`).

| Commande / route | Admin requis ? | Source |
|---|---|---|
| `config/device_registry/list`, `config/entity_registry/list`, `list_for_display`, `get`, `get_entries` | Non | [core] `components/config/device_registry.py:75-102`, `entity_registry.py:39-151` |
| `config/area_registry/list`, `config/floor_registry/list`, `config/label_registry/list` | Non | [core] `components/config/area_registry.py:23-37`, `floor_registry.py:25-40`, `label_registry.py:53-68` |
| `config_entries/get` | Non | [core] `components/config/config_entries.py:674-691` |
| `config_entries/get_single`, `config_entries/subentries/list` | **Oui** | [core] `components/config/config_entries.py:515,790` |
| `person/list` | Non (`admin_only` non activé) | [core] `components/person/__init__.py:450-452`, `helpers/collection.py:568-598` |
| `config/auth/list` (utilisateurs HA) | **Oui** | [core] `components/config/auth.py:22-23` |
| `get_states`, `get_config`, `entity/source` | Non, mais **filtré par permissions** de l'utilisateur | [core] `components/websocket_api/commands.py:430-473,734-745,938-956` |
| `automation/config`, `script/config` | **Oui** | [core] `components/automation/__init__.py:212-213`, `components/script/__init__.py:192-193` |
| REST `GET /api/config/automation/config/<id>`, `/api/config/script/config/<clé>` | **Oui** | [core] `components/config/view.py:85-96` |
| `blueprint/list`, `blueprint/substitute` | **Oui** | [core] `components/blueprint/websocket_api.py:66-69,240-243` |
| `history/history_during_period` | Non (entités filtrées par permission) | [core] `components/history/websocket_api.py:96-150` |
| `recorder/statistics_during_period`, `list_statistic_ids`, `get_statistics_metadata`, `recorder/info` | Non | [core] `components/recorder/websocket_api.py:286-355,455-470`, `basic_websocket_api.py:21-27` |
| `recorder/database_usage` (2026.12 dev) | **Oui** | [core] `components/recorder/database_usage.py:459-466` |
| REST `GET /api/backup/download/<id>` | **Oui** | [core] `components/backup/http.py:48-49` |

**Analyse** : un jeton d'utilisateur non admin donne les registres et les états, mais **pas** la configuration des automatisations, des scripts ni des blueprints. L'outil doit vérifier tôt le statut admin (par exemple en appelant `config/auth/list` ou `automation/config` et en interprétant `unauthorized`) et l'expliquer.

### 1.3 Registres

#### Appareils — `config/device_registry/list`

- Sans paramètre. La réponse concatène le JSON en cache de **tous les appareils, y compris les « appareils enfants »** (`chain(registry.devices, registry.child_devices)`) ([core] `components/config/device_registry.py:75-102`).
- Champs d'un appareil ([core] `helpers/device_registry.py:820-860`, `DeviceEntry.dict_repr`) : `area_id`, `configuration_url`, `config_entries` (déprécié), `config_entries_subentries` (déprécié), **`config_entry_id`**, `config_subentry_id`, `connections` (liste de paires), `created_at`/`modified_at` (secondes epoch), `disabled_by`, `entry_type`, `hw_version`, `id`, `identifiers` (liste de paires), `labels`, `manufacturer`, `model`, `model_id`, `name`, `name_by_user`, `next_name_part`, `parent_device_id` (toujours `null` pour un appareil principal), `primary_config_entry` (déprécié), `serial_number`, `sw_version`, `via_device_id`.
- Champs d'un **appareil enfant** (`ChildDeviceEntry.dict_repr`, [core] `helpers/device_registry.py:915-985`) : seulement `area_id`, `config_entry_id`, `config_subentry_id`, dates, `disabled_by`, `id`, `identifiers`, `labels`, `name`, `name_by_user`, `next_name_part`, **`parent_device_id`**. Pas de `connections`, `manufacturer`, `model`… Introduits en 2026.9 (storage 3.4, [core] `helpers/device_registry.py:1468-1470`).
- `config/device_registry/list_composite_splits` donne, pour chaque ancien identifiant d'appareil « composite » scindé, la liste des appareils qui l'ont remplacé ([core] `components/config/device_registry.py:36-72`). Voir § 2.1.

#### Entités — `config/entity_registry/list`, `list_for_display`, `get`, `get_entries`

| Commande | Contenu | Source |
|---|---|---|
| `config/entity_registry/list` | Toutes les entités (y compris désactivées), forme « partielle » : `area_id`, `categories`, `config_entry_id`, `config_subentry_id`, dates, `device_id`, `disabled_by`, `entity_category`, `entity_id`, `hidden_by`, `icon`, `id`, `labels`, `name`, `next_name_part`, `options`, `original_name`, `platform`, `translation_key`, `unique_id` | [core] `helpers/entity_registry.py:393-425` |
| `config/entity_registry/list_for_display` | Entités **non désactivées** seulement, clés abrégées (`ei`, `pl`, `ai`, `di`, `ec` en index, `hb`, `en`, `dp`…) | [core] `components/config/entity_registry.py:70-95`, `helpers/entity_registry.py:221-228,344-369` |
| `config/entity_registry/get` (`entity_id`) | Forme « étendue » d'une entité | [core] `components/config/entity_registry.py:98-124` |
| `config/entity_registry/get_entries` (`entity_ids`: liste) | Dictionnaire `entity_id → forme étendue ou null` | [core] `components/config/entity_registry.py:127-151` |

La forme étendue ajoute `aliases`, **`capabilities`**, **`device_class`**, **`original_device_class`**, `original_icon` ([core] `helpers/entity_registry.py:428-440`).

⚠️ **Aucune forme API n'expose `unit_of_measurement` ni `supported_features`** du registre : ils ne sont que dans le fichier de stockage ([core] `helpers/entity_registry.py:459-497`). Via l'API, l'unité vient des attributs d'état (`get_states`). Le `state_class` d'un capteur est dans `capabilities` (`SensorEntityCapabilityAttribute.STATE_CLASS = "state_class"`, [core] `components/sensor/const.py:92-96`) et dans les attributs d'état.

#### Pièces, étages, étiquettes

| Commande | Champs | Source |
|---|---|---|
| `config/area_registry/list` | `aliases`, `area_id`, `floor_id`, `humidity_entity_id`, `icon`, `labels`, `name`, `picture`, `temperature_entity_id`, `created_at`, `modified_at` | [core] `helpers/area_registry.py:88-104` |
| `config/floor_registry/list` | `aliases`, `created_at`, `floor_id`, `icon`, **`level`** (entier ou `null`), `name`, `modified_at` | [core] `components/config/floor_registry.py:158-169` |
| `config/label_registry/list` | `color`, `created_at`, `description`, `icon`, `label_id`, `name`, `modified_at` | [core] `components/config/label_registry.py:156-166` |

#### Entrées de configuration (intégrations) — `config_entries/get`

- Paramètres facultatifs `type_filter` (liste) et `domain` ([core] `components/config/config_entries.py:674-691`).
- Champs (`as_json_fragment`, [core] `homeassistant/config_entries.py:702-728`) : `created_at`, `entry_id`, `domain`, `modified_at`, `title`, `source`, `state`, `supports_*`, `pref_disable_new_entities`, `pref_disable_polling`, `disabled_by`, `reason`, `error_reason_*`, `num_subentries`.
- ⚠️ **Ni `data`, ni `options`, ni `unique_id`** ne sont exposés. Les identifiants des intégrations (broker MQTT, comptes…) **ne sont pas accessibles par l'API**. Ils ne le sont que dans la sauvegarde (§ 3.2).

#### Personnes — `person/list`

Réponse `{"storage": [...], "config": [...]}` ([core] `components/person/__init__.py:305-324`) : `storage` = personnes créées dans l'interface (fichier `.storage/person`), `config` = personnes déclarées en YAML (`person:` dans `configuration.yaml`). Champs d'une personne : `id`, `name`, `user_id`, `device_trackers`, `picture` ([core] `components/person/__init__.py:55-63,160-177`). `config/auth/list` (admin) donne les utilisateurs : `id`, `username`, `name`, `is_owner`, `is_active`, `local_only`, `system_generated`, `group_ids`, `credentials` ([core] `components/config/auth.py:169-191`).

### 1.4 États — `get_states`

`get_states` renvoie la liste des états autorisés pour l'utilisateur ([core] `components/websocket_api/commands.py:445-473`) : `entity_id`, `state`, `attributes` (dont `unit_of_measurement`, `device_class`, `state_class`, `friendly_name`), `last_changed`, `last_updated`, `context`. Il y a aussi `entity/source` (domaine d'origine de chaque entité, y compris celles sans registre) et `get_config` (version, fuseau horaire, système d'unités) ([core] `components/websocket_api/commands.py:734-745,938-956`).

### 1.5 Automatisations

| Moyen | Ce qu'il renvoie | Limites | Source |
|---|---|---|---|
| `get_states` sur `automation.*` | État, `friendly_name`, `last_triggered`, `mode`… et l'attribut **`id`** quand l'automatisation a un identifiant | Pas la configuration | [core] `components/automation/entity.py:93-96` |
| WS `automation/config` (`entity_id`) | `{"config": raw_config}` | Admin. Fonctionne pour **toutes** les automatisations chargées : `automations.yaml`, YAML **sans `id`**, **packages**, fichiers `!include` | [core] `components/automation/__init__.py:212-232` |
| REST `GET /api/config/automation/config/<id>` | L'élément de `automations.yaml` portant cet `id`, tel qu'écrit | Admin. **Lit uniquement `automations.yaml`** ([core] `homeassistant/config.py:45`). 404 pour une automatisation sans `id`, dans un package ou dans un autre fichier | [core] `components/config/automation.py:41-48`, `config/view.py:85-96` |

Nature de `raw_config` ([core] `components/automation/config.py:90-216`, `util.py:52-80`) :

- C'est la configuration **avant normalisation** par le schéma : on peut y trouver l'ancienne syntaxe (`trigger`/`condition`/`action`, `platform:` dans un déclencheur, `service:` dans une action) comme la nouvelle (`triggers`/`conditions`/`actions`, `trigger:`, `action:`).
- Les balises YAML (`!secret`, `!include`…) sont **déjà résolues** (le fichier a été chargé). Les templates Jinja ne sont **pas** rendus.
- **Automatisation issue d'un blueprint** : `raw_config` contient la configuration **après substitution** des entrées (`blueprint_inputs.async_substitute()`, [core] `components/automation/config.py:181-185`). La forme `use_blueprint: {path, input}` est conservée à part (`raw_blueprint_inputs`) mais **n'est pas exposée** par la commande WS ([core] `components/automation/entity.py:150-161`). Si la substitution échoue, `raw_config` reste la forme `use_blueprint` et l'entité est indisponible ([core] `components/automation/config.py:148-158,166-196`).
- Les automatisations invalides deviennent des `UnavailableAutomationEntity` avec leur `raw_config` ([core] `components/automation/util.py:100-114`).

Blueprints ([core] `components/blueprint/websocket_api.py`) : `blueprint/list` (`domain`: `automation` ou `script`) renvoie `chemin → {metadata}` (nom, domaine, entrées, source_url), **pas le YAML du blueprint** ([core] `:64-96`). `blueprint/substitute` (`domain`, `path`, `input`) renvoie `{"substituted_config": …}` ([core] `:238-275`). Le YAML source n'est lisible que dans la sauvegarde (`blueprints/<domaine>/<espace>/<fichier>.yaml`, [core] `components/blueprint/models.py:244-283`, `const.py:3`).

**Analyse** : par l'API, `automation/config` suffit pour obtenir une automatisation « résolue », y compris depuis un blueprint. Pour savoir qu'elle vient d'un blueprint, l'attribut `referenced_blueprint` n'est pas exposé en WS. On peut utiliser la REST `GET /api/config/automation/config/<id>` (présence de `use_blueprint`), mais seulement pour `automations.yaml`.

### 1.6 Scripts

`script/config` (`entity_id`, admin) renvoie `{"config": raw_config}` ([core] `components/script/__init__.py:192-214`). La REST `/api/config/script/config/<clé>` lit `scripts.yaml`, qui est un dictionnaire indexé par la clé du script ([core] `components/config/script.py:13,40-41`, `homeassistant/config.py:46`).

### 1.7 Historique des états — `history/history_during_period`

Paramètres ([core] `components/history/websocket_api.py:96-107`) : `start_time` (requis, ISO), `end_time`, `entity_ids` (**requis, liste non vide d'entités explicites**), `include_start_time_state` (vrai par défaut), `significant_changes_only` (vrai par défaut), `minimal_response` (faux), `no_attributes` (faux).

- Format de réponse compressé : `{entity_id: [ {"s": état, "a": attributs, "lc": last_changed, "lu": last_updated?}, … ]}` avec des horodatages en **secondes epoch (float)** ([core] `components/history/websocket_api.py:64-93` qui passe `compressed_state_format=True` ; clés : `homeassistant/const.py:933-937`, `core.py:1912-1919`).
- `minimal_response` : seuls le premier et le dernier élément sont complets. Les intermédiaires sont `{"s", "lu"}`, et les doublons d'état consécutifs sont **supprimés** ([core] `components/recorder/history/__init__.py:925-970`).
- `significant_changes_only=true` : hors des domaines `climate`, `device_tracker`, `humidifier`, `thermostat` et `water_heater`, seuls les **changements de valeur** sont rendus, pas les mises à jour d'attributs seules ([core] `components/recorder/history/__init__.py:281-287`, `history/const.py:6-12`).
- **Aucune pagination ni limite** : toute la période demandée part dans **un seul message**. Il faut découper par fenêtres de temps côté client. `history/stream` existe aussi, mais il est prévu pour suivre le direct (file de 2048 états, [core] `components/history/const.py:7`).
- Équivalent REST : `GET /api/history/period/<début>?filter_entity_id=…&end_time=…&minimal_response&no_attributes&significant_changes_only=0` ([core] `components/history/__init__.py:57-113`).

### 1.8 Statistiques long terme

| Commande | Paramètres | Réponse | Source |
|---|---|---|---|
| `recorder/list_statistic_ids` | `statistic_type` facultatif (`mean` ou `sum`) | Liste de `{statistic_id, display_unit_of_measurement, has_mean (déprécié), mean_type, has_sum, name, source, statistics_unit_of_measurement, unit_class}` | [core] `components/recorder/websocket_api.py:342-355`, `statistics.py:1162-1180` |
| `recorder/get_statistics_metadata` | `statistic_ids` facultatif | Même format | [core] `components/recorder/websocket_api.py:455-469` |
| `recorder/statistics_during_period` | `start_time`, `end_time`, `statistic_ids` (non vide), **`period`** ∈ `5minute`, `hour`, `day`, `week`, `month`, `year`, `units`, `types` ⊂ {`change`, `last_reset`, `max`, `mean`, `min`, `state`, `sum`} | `{statistic_id: [{start, end, mean, min, max, state, sum, last_reset, change}]}`. `start`, `end` et `last_reset` en **millisecondes epoch** | [core] `components/recorder/websocket_api.py:217-283,286-318` |
| `recorder/statistic_during_period` | `statistic_id`, `types` ⊂ {max, mean, min, change}, période | **Un seul agrégat** sur la période | [core] `components/recorder/websocket_api.py:180-216` |

- `period: "5minute"` lit les statistiques court terme, les autres périodes lisent les horaires et les agrègent ([core] `components/recorder/statistics.py:1238-1260`, `_reduce_statistic`).
- **Unités** : sans paramètre `units`, les valeurs sont converties dans l'unité d'**affichage**, c'est-à-dire l'unité actuelle de l'état si elle est convertible ([core] `components/recorder/statistics.py:365-384`). La base stocke `statistics_unit_of_measurement`.
- `mean_type` : 0 = aucune moyenne, 1 = arithmétique, 2 = **circulaire** (angles) ([core] `components/recorder/models/statistics.py:53-58`). `has_mean` est annoncé comme supprimable « avec 2026.4 » ([core] `statistics.py:1170-1171`) mais est encore renvoyé.

### 1.9 Compter les points d'historique sans tout télécharger

| Méthode | Disponibilité | Précision | Coût |
|---|---|---|---|
| **`recorder/database_usage`** (`refresh: true`) : par entité `{entity_id, states, states_bytes, attributes_bytes}` et par statistique `{statistic_id, source, statistics, statistics_short_term, …_bytes}`, plus taille totale, WAL et tables | **dev 2026.12 seulement** (absent de 2026.11.2), **admin** | Comptes exacts (un instantané). Octets estimés | Un `COUNT … GROUP BY metadata_id` par table côté HA. Résultat en cache 1 h, rafraîchi au plus toutes les minutes ([core] `components/recorder/database_usage.py:42-43,306-395,459-490`) |
| Statistiques : `statistics_during_period` avec `period: "month"` et `types: ["state"]` (ou `mean`) | Toutes versions | **Estimation** : on borne la première et la dernière période, puis nombre d'heures ≈ durée × 1. Les trous (HA arrêté) surestiment | Faible (≤ 12 lignes par an et par statistique) |
| États : `history_during_period` avec `minimal_response`, `no_attributes`, `significant_changes_only: false`, par fenêtres | Toutes versions | Exact, mais **c'est un téléchargement** | Volume borné par la rétention (10 jours par défaut, § 5.1) |
| Sauvegarde : requêtes SQL (§ 5.4) | Si la base SQLite est dans la sauvegarde | Exact | Local |

**Analyse** : pour 2026.11, il n'existe **aucune commande de comptage**. L'estimation de D24 doit combiner une estimation par les statistiques (bornes mensuelles) et un comptage réel des états, acceptable parce que la rétention des états est courte. À partir de 2026.12 (à confirmer à sa sortie), `recorder/database_usage` donne les comptes exacts.

### 1.10 Taille des messages et flux

- Les listes de registres sont envoyées en **un seul message** construit à partir de fragments JSON en cache ([core] `components/config/device_registry.py:86-102`, `entity_registry.py:49-67`). Pour une grosse installation, ce message peut peser plusieurs Mo.
- Côté serveur, la file d'envoi est limitée à 4096 messages : au-delà, le client lent est déconnecté ([core] `components/websocket_api/const.py:21-33`, `http.py:223-235`).
- La limite de taille des messages **entrants** (client vers HA) vient de la configuration aiohttp du composant `http` : **non vérifiée**. Nos requêtes sont petites.
- Client Node : la bibliothèque `ws` limite par défaut les messages reçus (`maxPayload`, 100 Mio d'après sa documentation, **non vérifié**). Une réponse d'historique sur une longue période peut s'en approcher, d'où le découpage par fenêtres.

---

## 2. Registres : champs utiles au rapprochement

### 2.1 Appareils (`core.device_registry`)

| Champ | Sens | Source |
|---|---|---|
| `identifiers` | Ensemble de paires `(domaine, identifiant)` propres à l'intégration | [core] `helpers/device_registry.py:642` |
| `connections` | Paires `(type, valeur)`. Types définis par le cœur : `mac` (`CONNECTION_NETWORK_MAC`), `zigbee` (`CONNECTION_ZIGBEE`), `bluetooth` (`CONNECTION_BLUETOOTH`), `upnp` (`CONNECTION_UPNP`). Les intégrations peuvent en publier d'autres (MQTT recopie ce que publie l'appareil, [core] `components/mqtt/entity.py:1501-1506`) | [core] `helpers/device_registry.py:88-91` |
| Adresse MAC | Normalisée en minuscules `aa:bb:cc:dd:ee:ff` (depuis les formes `aabbccddeeff`, `aa-bb-…`, `aabb.ccdd.eeff`), normalisation des anciennes données en 2025.8 | [core] `helpers/device_registry.py:573-612,1268` |
| `manufacturer`, `model`, `model_id`, `serial_number`, `hw_version`, `sw_version` | Chaînes libres fournies par l'intégration | [core] `helpers/device_registry.py:562-570,749-769` |
| `name` / `name_by_user` | Nom de l'intégration / nom donné par l'utilisateur (prioritaire pour le nom calculé, [core] `helpers/entity_registry.py:795`) | [core] `helpers/device_registry.py:645-646` |
| `area_id` | Pièce de l'appareil (les entités peuvent la surcharger avec leur propre `area_id`) | [core] `helpers/device_registry.py:637` |
| `config_entry_id` | **Une seule** entrée de configuration propriétaire depuis 2026.8. `config_entries` est un alias déprécié (suppression prévue en 2027.10) | [core] `helpers/device_registry.py:615-700` (champ `:635`), `:1282-1290` |
| `via_device_id` | Passerelle ou coordinateur par lequel passe l'appareil | [core] `helpers/device_registry.py:770` |
| `disabled_by` | `config_entry`, `device` (enfant d'un parent désactivé), `integration`, `user` | [core] `helpers/device_registry.py:124-132` |
| `entry_type` | `service` pour un « appareil » logiciel (météo, service cloud…), sinon `null` | [core] `helpers/device_registry.py:432-435` |
| `composite_device_id`, `split_at`, `has_composite_identifiers` | **Stockage seulement** : trace de la scission des anciens appareils multi-intégrations | [core] `helpers/device_registry.py:750-767,862-895` |

**Scission des appareils composites (2026.8)** : avant 2026.8, un appareil pouvait appartenir à plusieurs entrées de configuration (par exemple une prise vue à la fois par ZHA et par une autre intégration). La migration 3.0 crée **un appareil par entrée de configuration**. Chacun garde une **copie des `identifiers` et `connections`** et un lien `composite_device_id` vers l'ancien identifiant, qui reste résolu pour les actions existantes ([core] `helpers/device_registry.py:1282-1290`).

**Analyse** : (1) une même adresse MAC ou IEEE peut apparaître sur **plusieurs** appareils HA. Le rapprochement doit regrouper par connexion avant de chercher le candidat Gladys, sinon il trouvera « plusieurs candidats ». (2) Une automatisation peut référencer un `device_id` **qui n'existe plus** dans la liste (ancien composite). Il faut le résoudre avec `config/device_registry/list_composite_splits` (API) ou `composite_device_id` (sauvegarde). (3) Les appareils enfants n'ont ni connexion ni fabricant : on les rattache par `parent_device_id`.

### 2.2 Entités (`core.entity_registry`)

| Champ | Sens | Source |
|---|---|---|
| `entity_id`, `unique_id`, `platform` | `platform` = domaine de l'intégration qui fournit l'entité. `unique_id` est unique par `(domaine, platform)` | [core] `helpers/entity_registry.py:264-269` |
| `device_id`, `config_entry_id`, `area_id` | Rattachements. `area_id` de l'entité, s'il existe, prime sur celui de l'appareil | [core] `helpers/entity_registry.py:272,275,279` |
| `device_class` / `original_device_class` | Surcharge utilisateur / valeur de l'intégration. La valeur effective est `device_class or original_device_class` | [core] `helpers/entity_registry.py:278,295,507` |
| `unit_of_measurement` | Stockage seulement (voir § 1.3) | [core] `helpers/entity_registry.py:301,495` |
| `capabilities` | Dont `state_class` pour les capteurs | [core] `helpers/entity_registry.py:274`, `components/sensor/const.py:92-96` |
| `entity_category` | `config` ou `diagnostic` (sinon `null`) | [core] `helpers/entity_registry.py:282` |
| `disabled_by` | `config_entry`, `device`, `hass`, `integration`, `user` | [core] `helpers/entity_registry.py:140-147` |
| `hidden_by` | `integration`, `user` | [core] `helpers/entity_registry.py:172-176` |
| `name` / `original_name` / `has_entity_name` | Depuis le stockage 1.24, `name` ne contient plus le préfixe du nom de l'appareil. Le nom complet est calculé (appareil + entité) | [core] `helpers/entity_registry.py:1053-1070` |
| `options` | Réglages par domaine (ex. `sensor.display_precision`, unité choisie par l'utilisateur) | [core] `helpers/entity_registry.py:293,363-367` |

Les entités YAML sans `unique_id` ne sont **pas** dans le registre. Elles n'apparaissent que dans `get_states`.

### 2.3 Pièces et étages

- Une pièce a `floor_id`, des `aliases` et `labels` ([core] `helpers/area_registry.py:77-104`). Un étage a `level` (entier, `null` possible) et des `aliases` ([core] `helpers/floor_registry.py:332-347`).

### 2.4 ZHA : identifiant IEEE

Chaque appareil ZHA est enregistré avec ([core] `components/zha/helpers.py:1106-1115`, `entity.py:141-148`) :

```python
connections={(dr.CONNECTION_ZIGBEE, str(zha_device.ieee))},   # ("zigbee", ieee)
identifiers={(DOMAIN, str(zha_device.ieee))},                 # ("zha", ieee)
```

- L'IEEE est donc **à la fois** dans `identifiers` (`["zha", …]`) et dans `connections` (`["zigbee", …]`). Le coordinateur est enregistré de la même façon ([core] `components/zha/__init__.py:264-265`).
- La valeur est `str(EUI64)` de zigpy. Format attendu : 8 octets hexadécimaux en minuscules séparés par `:` (`00:12:4b:00:01:02:03:04`). Bibliothèque absente du clone : **non vérifié**. Zigbee2MQTT et Gladys utilisent `0x00124b0001020304` (**non vérifié** ici, voir la note Gladys).
- Les groupes ZHA sont des appareils de type `service`, modèle `Group`, sans IEEE ([core] `components/zha/helpers.py:365-382`).

**Analyse** : la règle de rapprochement « adresse IEEE » doit normaliser les deux formes (suppression des `:` et du préfixe `0x`, minuscules). Pour Zigbee2MQTT intégré à HA via MQTT, les identifiants sont ceux publiés par Zigbee2MQTT (`("mqtt", …)`), dont le contenu exact (par exemple `zigbee2mqtt_0x…`) est **non vérifié**.

---

## 3. Fichiers du dossier de configuration et `.storage`

### 3.1 Format commun des fichiers `.storage`

Dossier `.storage` dans le dossier de configuration ([core] `helpers/storage.py:41`). Chaque fichier est un JSON :

```json
{ "version": 1, "minor_version": 24, "key": "core.entity_registry", "data": { … } }
```

([core] `helpers/storage.py:519-523`.) Les données sont migrées **par HA au chargement** quand la version change (`_async_migrate_func`, [core] `helpers/storage.py:481-512`). Un fichier lu dans une sauvegarde ancienne est donc dans l'**ancien** format.

| Clé (nom du fichier) | Version (2026.11) | Contenu `data` | Source |
|---|---|---|---|
| `core.device_registry` | 3.4 | `devices`, `child_devices`, `deleted_devices` | [core] `helpers/device_registry.py:82-84,4699-4715` |
| `core.entity_registry` | 1.24 | `entities`, `deleted_entities`, `settings` | [core] `helpers/entity_registry.py:92-94,2658-2671` |
| `core.area_registry` | 1.9 | `areas` (`id` au lieu de `area_id`, dates ISO) | [core] `helpers/area_registry.py:39-41,489-508` |
| `core.floor_registry` | 1.3 | `floors` | [core] `helpers/floor_registry.py:30-32,332-347` |
| `core.label_registry` | 1.2 | `labels` | [core] `helpers/label_registry.py:27-29` |
| `core.category_registry` | 1.2 | catégories | [core] `helpers/category_registry.py:24-26` |
| `core.config_entries` | 1.5 | `entries` : `data`, `options`, `subentries[].data`, `domain`, `title`, `unique_id`, `source`, `disabled_by`, `version`… | [core] `homeassistant/config_entries.py:138-140,1344-1363,3131-3135` |
| `person` | 2 | `items` : `id`, `name`, `user_id`, `device_trackers`, `picture` | [core] `components/person/__init__.py:45-46`, `helpers/collection.py:269-277,375` |
| `auth` | 1 | `users`, `groups`, `credentials`, `refresh_tokens` (**avec `token` et `jwt_key`**) | [core] `homeassistant/auth/auth_store.py:28-29,565-622` |
| `auth_provider.homeassistant` | 1 | identifiants et mots de passe hachés des comptes locaux | [core] `homeassistant/auth/providers/homeassistant.py:25-26,89` |
| `core.restore_state` | 1 | derniers états connus (`state`, `extra_data`, `last_seen`) | [core] `helpers/restore_state.py:26-27,80-86` |
| `backup` | — | configuration des sauvegardes, **dont le mot de passe des sauvegardes automatiques en clair** | [core] `components/backup/store.py:15,40-43` |

Les fichiers `auth`, `auth_provider.homeassistant`, `backup` et les modules MFA sont écrits en mode **privé** (permissions restreintes) ([core] `helpers/storage.py:234-278`, `auth/auth_store.py:66`, `components/backup/store.py:43`). **Aucun chiffrement** n'est appliqué au contenu.

### 3.2 Secrets des intégrations

`data` et `options` des entrées de configuration sont écrits tels quels (`as_dict` → `as_storage_fragment`, [core] `homeassistant/config_entries.py:735-737,1344-1363`) dans un fichier **non privé**. **Confirmé : les identifiants des intégrations sont en clair** dans `.storage/core.config_entries`, donc dans toute sauvegarde déchiffrée. Ils ne sont **pas** exposés par l'API (§ 1.3).

### 3.3 Fichiers YAML

| Fichier | Rôle | Source |
|---|---|---|
| `configuration.yaml` | Configuration principale. Par défaut : `automation: !include automations.yaml`, `script: !include scripts.yaml`, `scene: !include scenes.yaml` | [core] `homeassistant/config.py:39,54-65` |
| `automations.yaml` | **Liste** d'automatisations (créé avec `[]`), seul fichier écrit par l'éditeur | [core] `homeassistant/config.py:45,185-187` |
| `scripts.yaml` | **Dictionnaire** `clé → script` | [core] `homeassistant/config.py:46` |
| `secrets.yaml` | Valeurs référencées par `!secret` | [core] `homeassistant/config.py:34,168,177-179` (constante `SECRET_YAML` de la bibliothèque `annotatedyaml`) |
| `blueprints/<automation|script>/<espace>/<nom>.yaml` | Sources des blueprints | [core] `components/blueprint/models.py:244-283` |
| `.HA_VERSION` | Version de HA qui a écrit la configuration | [core] `homeassistant/config.py:40,181-182` |

- Le chargeur YAML vient de la bibliothèque externe **`annotatedyaml`** ([core] `homeassistant/util/yaml/loader.py:7-18`). Balises connues (documentation HA, **non vérifiées** dans le code) : `!include`, `!include_dir_list`, `!include_dir_named`, `!include_dir_merge_list`, `!include_dir_merge_named`, `!secret`, `!env_var`, et `!input` dans les blueprints.
- **Packages** : `homeassistant: packages:` fusionne des blocs de configuration ([core] `homeassistant/config.py:248-258,667-681`). Les automatisations des packages sont fusionnées en liste (`PACKAGE_MERGE_HINT = "list"`, [core] `components/automation/config.py:44`).
- Une configuration peut aussi déclarer plusieurs clés du même domaine, comme `automation manual:` ou `automation ui:` ([core] `homeassistant/config.py:1051-1062`, `cv.domain_key`).

**Analyse** : côté sauvegarde, reconstituer la liste **complète** des automatisations exige de réimplémenter le chargement YAML de HA (inclusions, répertoires, packages, clés suffixées, `!secret`), puis la **substitution des blueprints** (`!input`). L'API donne tout cela déjà résolu (§ 1.5). C'est la principale asymétrie entre les deux sources.

---

## 4. Sauvegardes

### 4.1 Structure de l'archive

Une sauvegarde est un **tar externe non compressé** (format PAX). Il contient `./backup.json` en clair et une **archive interne par composant**, éventuellement compressée en gzip et chiffrée ([st] `README.md`, « Archive layout » ; [core] `components/backup/manager.py:1990-2012`).

```
<slug>.tar
├── ./backup.json                 (toujours en clair)
├── ./homeassistant.tar.gz        (ou .tar si "compressed": false ; chiffré si "protected")
│     └── data/                   (= dossier de configuration de HA)
│         ├── .storage/…
│         ├── configuration.yaml, automations.yaml, scripts.yaml, secrets.yaml, blueprints/…
│         └── home-assistant_v2.db (+ home-assistant_v2.db-wal) si la base est incluse
├── ./<slug_addon>.tar.gz         (Supervisor : une archive par module complémentaire)
├── ./share.tar.gz, ssl.tar.gz, media.tar.gz, apps_local.tar.gz   (Supervisor : dossiers)
└── ./supervisor.tar.gz           (Supervisor : configuration du Supervisor)
```

- Le préfixe `data/` est commun aux sauvegardes du cœur et du Supervisor : la restauration lit `homeassistant/data` dans les deux cas ([core] `components/backup/manager.py:2004-2011` avec `arcname="data"`, `homeassistant/backup_restore.py:124-160`).
- Noms des archives Supervisor : `homeassistant.tar[.gz]`, `<slug>.tar[.gz]` pour les modules et les dossiers (`/` remplacé par `_`), `supervisor.tar[.gz]` ([sup] `supervisor/backups/backup.py:700,813,1003,1088`). Dossiers : `share`, `apps/local`, `ssl`, `media` ([sup] `supervisor/const.py:459-466` selon l'extrait lu ; fichier hors du dossier `backups`).
- Une archive interne chiffrée est un tar **compressé en gzip puis chiffré** (mode `w|gz` sous le chiffrement, [st] `securetar/__init__.py:971-980`). On ne peut la lire qu'en **séquentiel** : pas d'accès direct à un membre ([st] `README.md`, « Reading an archive »).

### 4.2 `backup.json`

| Champ | Sauvegarde du cœur (intégration `backup`) | Sauvegarde Supervisor |
|---|---|---|
| `version` | `2` | 1 ou 2 |
| `slug`, `name`, `date` | oui | oui |
| `type` | toujours `"partial"` | `full` ou `partial` |
| `compressed` | `true` | `true` par défaut |
| `protected` | `password is not None` | oui |
| `homeassistant` | `{exclude_database, version}` | `{version, size, exclude_database}` ou **`null`** si HA n'est pas inclus |
| `folders`, `addons`, `repositories`, `supervisor_version` | absents | présents |
| `extra` | métadonnées (ex. `supervisor.backup_request_date`) | idem |
| `crypto` (`aes128`) | — | ancien champ, supprimé à la lecture |

Sources : [core] `components/backup/manager.py:1871-1885`, `components/backup/util.py:75-130` ; [sup] `supervisor/backups/validate.py:87-137`.

**Différences cœur / Supervisor** :

- **Cœur** (installations Container et Core, mais aussi HA OS via l'intégration `backup`) : seule `homeassistant.tar.gz` est produite. Exclusions permanentes : `*.log*`, `*.db-shm`, `backups/*.tar`, `tts/*`, `.cache/*`, `__pycache__`… Si la base est exclue : `home-assistant_v2.db` et `home-assistant_v2.db-wal` ([core] `components/backup/const.py:16-32`, `manager.py:1973-1988`).
- **Supervisor** : sauvegarde complète ou partielle avec modules et dossiers. Une sauvegarde **partielle peut ne pas contenir HA** (`homeassistant: null`) ([sup] `supervisor/backups/validate.py:102-115`).
- Pendant la sauvegarde, le recorder **verrouille la base en écriture** ([core] `components/recorder/backup.py:14-30`). La base et son fichier WAL sont donc cohérents.

### 4.3 Base du recorder dans l'archive

- Chemin : `data/home-assistant_v2.db`, accompagné de `data/home-assistant_v2.db-wal` (le `-shm` est toujours exclu) ([core] `components/backup/const.py:20,29-32`, `components/recorder/__init__.py:50`).
- **Exclue** quand `homeassistant.exclude_database` vaut `true`. Seuls ces deux noms par défaut sont exclus.
- Une base SQLite configurée ailleurs via `db_url` dans le dossier de configuration serait sauvegardée sous son nom. Une base MariaDB ou PostgreSQL n'est **pas** dans `homeassistant.tar.gz` ([core] `components/recorder/__init__.py:169,222-226`). Le module MariaDB du Supervisor a sa propre archive, dont le contenu est **non vérifié** et hors périmètre.

### 4.4 Chiffrement SecureTar

Seules les **archives internes** sont chiffrées. `backup.json` reste lisible ([st] `README.md` ; [core] `components/backup/manager.py:1999-2002`). En-têtes ([st] `securetar/__init__.py:54-114,200-246`) :

| Version | Reconnaissance | En-tête | Chiffrement | Dérivation de clé |
|---|---|---|---|---|
| **v1** (lecture seule) | pas de magique | 16 octets = sel | AES-128-CBC, PKCS7 | comme v2 |
| **v2** | `"SecureTar"` + octet `2` + 6 octets nuls | 16 (ID) + 16 (taille en clair `uint64` big-endian + 8 réservés) + 16 (sel) = **48 octets** | **AES-128-CBC**, remplissage PKCS7 | clé = `SHA-256` itéré **100 fois** sur le mot de passe UTF-8, **16 premiers octets**. IV = `SHA-256` itéré 100 fois sur `clé ‖ sel`, 16 premiers octets ([st] `:1569-1584,1706-1721`) |
| **v3** (défaut) | `"SecureTar"` + octet `3` | 16 + 16 + 104 = **136 octets** : sel racine (16), sel de validation (16), **clé de validation (32)**, sel de dérivation (16), en-tête *secretstream* (24) | **XChaCha20-Poly1305 *secretstream*** (libsodium), blocs de **1 Mio** de clair + 17 octets d'authentification, étiquette `FINAL` sur le dernier bloc | clé racine = **Argon2id** (libsodium `crypto_pwhash`, `opslimit = 8`, `memlimit = 16 Mio`, 32 octets, sel racine). Clé de validation = **BLAKE2b**(message vide, clé = clé racine, sel = sel de validation, personnalisation `"SecureTarv3"`). Clé du fichier = BLAKE2b(vide, clé racine, sel de dérivation, `"SecureTarv3"`) ([st] `:42-50,1597-1700,1729-1767`) |

- La v3 permet de **vérifier le mot de passe** sans déchiffrer : on compare la clé de validation recalculée à celle de l'en-tête ([st] `:1627-1644`). La v1 et la v2 ne le permettent pas : un mauvais mot de passe se détecte quand le premier bloc déchiffré n'est ni un gzip ni un tar ([st] `:420-433`, `README.md`).
- **Versions produites** : le cœur écrit en **v3** (`SECURETAR_CREATE_VERSION = 3`, [core] `components/backup/const.py:35`, identique en 2026.11.2). Le Supervisor écrit en v3 si HA ≥ **2026.3.0**, sinon en v2 ([sup] `supervisor/backups/const.py:16-18`, `backup.py:483-491`). Les sauvegardes anciennes sont donc en v2, voire en v1.
- **« Clé de chiffrement » des sauvegardes automatiques** : pour le cœur, c'est une **chaîne de mot de passe comme une autre**, stockée en clair dans `.storage/backup` (`create_backup.password`, [core] `components/backup/config.py:119,652-663`, `store.py:40-43`). Elle passe par la même dérivation. Elle est générée par l'interface (dépôt `home-assistant/frontend`, absent). Format « groupes de caractères séparés par des tirets » (par exemple 7 × 4) : **non vérifié**. L'outil doit donc transmettre la saisie **telle quelle**, sans normalisation non vérifiée.
- **Contournement possible** : HA sait servir une sauvegarde **déchiffrée** : `GET /api/backup/download/<backup_id>?agent_id=<agent>&password=<mdp>` (admin) produit un tar en clair ([core] `components/backup/http.py:40-92,120-155`, `util.py:227-252`).

**Faisabilité en Node.js**

| Besoin | `crypto` natif de Node | Remarque |
|---|---|---|
| v1/v2 : SHA-256 itéré, AES-128-CBC, PKCS7 | **Oui** (`createHash('sha256')`, `createDecipheriv('aes-128-cbc')`) | Simple, en flux |
| v3 : Argon2id | Ajouté récemment à Node (`crypto.argon2`, version 24.x) : **non vérifié** | Dépend de la version de Node de Gladys |
| v3 : BLAKE2b avec clé, sel et personnalisation | **Non** (`blake2b512` sans clé, sel ni personnalisation) | libsodium : `crypto_generichash_blake2b_salt_personal` |
| v3 : XChaCha20-Poly1305 *secretstream* | **Non** (Node offre ChaCha20-Poly1305 IETF, pas XChaCha ni *secretstream*) | libsodium : `crypto_secretstream_xchacha20poly1305_init_pull` / `_pull` |

**Analyse** : la v2 se fait en `crypto` natif. La v3, qui est le format par défaut depuis 2026.3, exige **libsodium**, par exemple le paquet `libsodium-wrappers-sumo` (WebAssembly, sans compilation native). La variante *sumo* serait nécessaire pour `crypto_pwhash` et `blake2b_salt_personal` : **à confirmer**. La personnalisation `"SecureTarv3"` (11 octets) doit probablement être complétée par des zéros jusqu'à 16 octets, comme PyNaCl le fait vraisemblablement : **non vérifié**. Un test de bout en bout sur une vraie sauvegarde v3 est indispensable. Le déchiffrement doit se faire en **flux** (tar externe → en-tête → *secretstream* → gunzip → tar interne), sans fichier temporaire en clair pour les membres non nécessaires. La dépendance au binaire `tar` prévue dans l'architecture (§ 3.1 de 01) ne suffit pas : il faut un lecteur tar en JavaScript (par exemple `tar-stream`, **non vérifié**) pour les archives internes chiffrées.

---

## 5. Recorder et historique

### 5.1 Moteur et rétention

- Base par défaut : SQLite `home-assistant_v2.db` dans le dossier de configuration (`sqlite:///<config>/home-assistant_v2.db`) ([core] `components/recorder/__init__.py:50,222-226`). Alternatives via `recorder: db_url:` : MySQL/MariaDB, PostgreSQL ([core] `components/recorder/__init__.py:58,135,169` ; dialectes `SupportedDialect` dans `recorder/const.py`).
- `purge_keep_days` : **10 jours** par défaut ; `auto_purge` et `auto_repack` vrais par défaut ; `commit_interval` = 5 s ([core] `components/recorder/__init__.py:54,126-150`).
- La purge supprime les états, les événements et les **statistiques court terme** (5 min) plus vieux que `keep_days` ([core] `components/recorder/purge.py:97-111`, `queries.py:308-318`). Elle **ne touche pas** la table `statistics` (horaire), qui est donc conservée indéfiniment.
- Les entités exclues par la configuration `recorder: exclude/include` n'ont aucun historique ([core] `components/recorder/filters.py`, **non lu en détail**).

### 5.2 Schéma (version 53)

`SCHEMA_VERSION = 53` ([core] `components/recorder/db_schema.py:70`). Table de suivi : `schema_changes(change_id, schema_version, changed)` ([core] `db_schema.py:836-845`).

| Table | Colonnes utiles | Source |
|---|---|---|
| `states_meta` | `metadata_id`, `entity_id` (unique) | [core] `db_schema.py:607-617` |
| `states` | `state_id`, `metadata_id` → `states_meta`, `state` (texte ≤ 255), `last_updated_ts`, `last_changed_ts`, `last_reported_ts` (float epoch, secondes), `old_state_id`, `attributes_id` → `state_attributes`, `origin_idx`, `context_*_bin`. Colonnes héritées `entity_id`, `last_updated`, `attributes`… **inutilisées (NULL)** | [core] `db_schema.py:404-450`, `homeassistant/const.py` (`MAX_LENGTH_STATE_STATE = 255`) |
| `state_attributes` | `attributes_id`, `hash`, `shared_attrs` (JSON texte, partagé entre états) | [core] `db_schema.py:539-551` |
| `statistics_meta` | `id`, `statistic_id` (unique), `source` (`recorder` ou domaine d'une intégration), `unit_of_measurement`, `unit_class`, `has_mean` (**mis à NULL depuis la v50**), `has_sum`, `name`, **`mean_type`** (0, 1 ou 2) | [core] `db_schema.py:758-781`, `migration.py:2101-2107` |
| `statistics` (horaire) | `id`, `metadata_id`, `created_ts`, **`start_ts`** (début de l'heure), `mean`, `mean_weight`, `min`, `max`, `last_reset_ts`, `state`, `sum`. Unicité `(metadata_id, start_ts)` | [core] `db_schema.py:625-648,696-711` |
| `statistics_short_term` | mêmes colonnes, période 5 min | [core] `db_schema.py:714-735` |

Jalons de schéma utiles : `states_meta` en v38, `context_id_bin` en v36, `last_reported_ts` en v43, `mean_type` et `mean_weight` en v49, `has_mean` vidé en v50, `unit_class` en v52 ([core] `components/recorder/const.py:44-51`, `migration.py:1791-1930,2070-2186`). La v53 change le jeu de caractères sous MySQL. Avant la v31/v32, les horodatages étaient en `DATETIME` et l'`entity_id` dans `states` ([core] `migration.py:1594-1655`).

**Analyse** : une base issue d'une sauvegarde est au schéma de la version de HA qui l'a produite. La v48 force la migration des anciennes lignes vers les colonnes `*_ts` ([core] `migration.py:2058-2067`). Le lecteur doit lire `schema_changes`, accepter une plage connue (par exemple ≥ 48) et refuser au-delà de ce qu'il connaît. Il ne doit **jamais** migrer la base lui-même.

### 5.3 Ce que contient une ligne d'état

- **Une ligne n'est écrite qu'au changement** d'état ou d'attributs. Un état identique **re-signalé** met à jour `last_reported_ts` de la ligne existante, sans nouvelle ligne (colonne ajoutée en v43, [core] `migration.py:1923-1930`). Mécanisme d'écriture **non lu en détail**.
- `state` est du **texte** : valeurs numériques en chaîne, mais aussi `unknown` et `unavailable`. Ces deux valeurs sont celles de HA, **non vérifiées** dans ce clone.

### 5.4 Requêtes SQL de comptage (SQLite)

```sql
-- Nombre d'états par entité, et plage couverte
SELECT sm.entity_id, COUNT(*) AS n, MIN(s.last_updated_ts) AS first_ts, MAX(s.last_updated_ts) AS last_ts
FROM states s JOIN states_meta sm ON sm.metadata_id = s.metadata_id
GROUP BY sm.entity_id;

-- Nombre de statistiques horaires et 5 min par statistic_id
SELECT m.statistic_id, m.source, m.unit_of_measurement, m.mean_type, m.has_sum,
       (SELECT COUNT(*) FROM statistics s WHERE s.metadata_id = m.id) AS n_hour,
       (SELECT MIN(start_ts) FROM statistics s WHERE s.metadata_id = m.id) AS first_hour_ts,
       (SELECT COUNT(*) FROM statistics_short_term t WHERE t.metadata_id = m.id) AS n_5min
FROM statistics_meta m;
```

Ces requêtes reproduisent ce que fait `recorder/database_usage` (`COUNT` groupé par `metadata_id`, [core] `components/recorder/database_usage.py:202,306-330`). Les index `(metadata_id, last_updated_ts)` et `(metadata_id, start_ts)` les rendent rapides ([core] `db_schema.py:407-410,699-708`). Pour trouver la taille d'une table, SQLite peut utiliser `dbstat` si l'extension est compilée ([core] `database_usage.py:47-51,98-112`).

### 5.5 Statistiques d'un compteur (énergie)

Seuls les capteurs (`sensor`) dotés d'un `state_class` produisent des statistiques à partir de leurs états ([core] `components/sensor/recorder.py:66-75,545-551`) :

| `state_class` | Statistiques | `mean_type` |
|---|---|---|
| `measurement` | `mean`, `min`, `max` | arithmétique (1) |
| `measurement_angle` | `mean` | **circulaire** (2) |
| `total` | `sum` (+ `state`, `last_reset`) | aucune (0) |
| `total_increasing` | `sum` (+ `state`) | aucune (0) |

Pour `total` et `total_increasing` ([core] `components/sensor/recorder.py:767-870`) :

- **`state`** = dernière valeur brute du capteur dans l'heure (l'index du compteur).
- **`sum`** = cumul **depuis la première compilation**, qui « traverse » les remises à zéro : à chaque cycle, la différence est ajoutée et le point de départ est rebasé.
- **Remise à zéro détectée** : `total_increasing` quand la nouvelle valeur est < 90 % de la précédente ([core] `components/sensor/recorder.py:524-542`). `total` quand l'attribut `last_reset` change ([core] `:790-820`).
- `last_reset` est conservé dans `last_reset_ts` ([core] `:868-870`).
- La consommation d'une heure = `sum(h) − sum(h−1)`. C'est aussi ce que renvoie `types: ["change"]` en WS ([core] `components/recorder/websocket_api.py:271`).

Il existe aussi des **statistiques externes** importées par des intégrations (fournisseurs d'énergie…), d'identifiant `domaine:objet` et de `source` égale au domaine. Elles ne sont rattachées à aucune entité ([core] `components/recorder/statistics.py:482-490,2881`).

---

## 6. Automatisations (survol pour le lot 2)

- **Clés de premier niveau** ([core] `components/automation/config.py:64-84`) : `id` (chaîne), `alias`, `description`, `trace`, `initial_state`, **`triggers`** (ancien nom `trigger`), **`conditions`** (`condition`), **`actions`** (`action`), `variables`, `trigger_variables`, et `mode` / `max` / `max_exceeded` (via `make_script_schema`, mode par défaut `single`). Les anciennes clés sont renommées par `_backward_compat_schema` ([core] `:56-61`).
- **Dans un déclencheur** : `trigger: state` (nouveau) ou `platform: state` (ancien). Les deux ensemble sont refusés ([core] `helpers/config_validation.py:1910-1929`).
- **Dans une action d'appel de service** : `action: light.turn_on` (nouveau) ou `service:` (ancien) ([core] `helpers/config_validation.py:1561-1575`).
- **Modes** : `single`, `restart`, `queued`, `parallel` ([core] `helpers/script.py:117-126`).
- **Déclencheurs** : alias du cœur `state`, `numeric_state`, `event`, `time`, `time_pattern` (→ `homeassistant`), `device` (→ `device_automation`) ([core] `helpers/trigger/__init__.py:164-171`, `components/homeassistant/triggers/*.py`). Les autres viennent du fichier `trigger.py` de chaque intégration (`sun`, `zone`, `template`, `mqtt`, `webhook`, `tag`, `calendar`…, **dossiers absents du clone**). WS `trigger_platforms/subscribe` liste les déclencheurs disponibles sur l'instance ([core] `components/websocket_api/commands.py:706`).
- **Conditions** : `and`, `or`, `not`, `state`, `numeric_state`, `template`, `time`, `trigger`, `device` dans le cœur ([core] `helpers/condition/__init__.py:171-200`), `zone` et `sun` dans leurs intégrations. WS `condition_platforms/subscribe` ([core] `commands.py:632`).
- **Types d'actions** : `call_service`, `scene`, `condition`, `choose`, `delay`, `device`, `event`, `if`, `parallel`, `repeat`, `sequence`, `set_conversation_response`, `stop`, `variables`, `wait_for_trigger`, `wait_template` ([core] `helpers/config_validation.py:2128-2143,2173-2200`).

---

## 7. Synthèse comparative API / sauvegarde

| Donnée | API (WS/REST) | Sauvegarde | Fidélité et limites |
|---|---|---|---|
| Appareils | ✅ `config/device_registry/list` | ✅ `.storage/core.device_registry` | Équivalent. La sauvegarde ajoute `composite_device_id` et `deleted_devices` |
| Appareils enfants | ✅ (même liste, `parent_device_id`) | ✅ `child_devices` | Depuis 2026.9 |
| Entités : `unique_id`, `platform`, `device_class`, catégorie, `disabled_by`/`hidden_by` | ✅ `list` + `get_entries` | ✅ | Équivalent |
| Unité d'une entité | ⚠️ via `get_states` (unité **actuelle** de l'état) | ✅ `unit_of_measurement` du registre | L'API n'a pas le champ du registre |
| `state_class` | ✅ `capabilities` (`get_entries`) ou attributs d'état | ✅ `capabilities` | Équivalent |
| Entités YAML sans `unique_id` | ✅ `get_states`, `entity/source` | ❌ (pas de registre ; il faudrait interpréter le YAML) | |
| Pièces, étages, étiquettes | ✅ | ✅ | Équivalent |
| Intégrations (entrées) | ✅ sans `data`/`options` | ✅ **avec `data`/`options` en clair** | Identifiants seulement via la sauvegarde |
| Personnes UI / YAML | ✅ `storage` + `config` | ⚠️ `.storage/person` + YAML à interpréter | |
| Utilisateurs HA | ✅ admin (`config/auth/list`) | ✅ `.storage/auth` (avec jetons : sensible) | |
| États courants | ✅ `get_states` | ⚠️ `.storage/core.restore_state` (dernier état mémorisé, partiel) | |
| Automatisations (`automations.yaml`) | ✅ `automation/config` (admin) | ✅ YAML brut | |
| Automatisations YAML sans `id`, packages, `!include` | ✅ `automation/config` | ⚠️ chargeur YAML complet à réécrire | Asymétrie forte |
| Automatisations de blueprint | ✅ **résolues** (`raw_config` substitué) | ⚠️ forme `use_blueprint` + blueprint source : substitution `!input` à réimplémenter | |
| Source d'un blueprint | ❌ (métadonnées seulement) | ✅ `blueprints/…` | |
| `!secret` | résolus (invisibles) | ✅ `secrets.yaml` en clair | |
| Historique des états | ✅ `history_during_period`, quel que soit le moteur | ✅ SQL, si SQLite **et** base incluse | Rétention 10 jours par défaut dans les deux cas |
| Statistiques horaires | ✅ `statistics_during_period` (converties en unité d'affichage par défaut) | ✅ tables `statistics*` (unité de stockage) | Attention aux unités |
| Comptage des points | ❌ en 2026.11 (estimation) ; ✅ `recorder/database_usage` en 2026.12 dev (admin) | ✅ `COUNT(*)` | |
| Version de HA | ✅ `auth_required.ha_version` | ✅ `backup.json` → `homeassistant.version`, `.HA_VERSION` | |
| Modules complémentaires (Zigbee2MQTT, Mosquitto…) | ❌ (API du Supervisor non étudiée) | ⚠️ archives `<slug>.tar.gz` (Supervisor) | Hors périmètre (vision § 4.2), mais utile pour détecter Zigbee2MQTT |

---

## 8. Pièges et points d'attention pour l'implémentation Node.js

1. **Jeton non admin** : registres et états lisibles, mais pas `automation/config`, `script/config`, les blueprints, `config/auth/list` ni `recorder/database_usage`. Détecter tôt (erreur `unauthorized`) et l'expliquer (§ 1.2).
2. **Permissions** : pour un non-admin, `get_states`, `entity/source` et l'historique sont **filtrés** silencieusement. Un inventaire incomplet ne produit pas d'erreur ([core] `components/websocket_api/commands.py:430-444`, `history/websocket_api.py:139-144`).
3. **`id` des messages** strictement croissants. Gérer les réponses en tableau seulement si `coalesce_messages` est activé (§ 1.1).
4. **Gros messages uniques** : registres et historique arrivent en un bloc. Découper l'historique par fenêtres de temps, régler `maxPayload` côté `ws` et surveiller la mémoire.
5. **Versions des fichiers `.storage`** : lire `version`/`minor_version`. Le format dépend de la version de HA qui a fait la sauvegarde (par exemple `config_entries` → `config_entry_id` unique en 3.x). D9 vise « HA courant » : refuser ou avertir pour une version inconnue.
6. **Appareils scindés et enfants** (2026.8 et 2026.9) : doublons de `connections`/`identifiers`, `device_id` historiques à résoudre, enfants sans fabricant (§ 2.1).
7. **IEEE** : formes `xx:xx:…` (ZHA) et `0x…` (Zigbee2MQTT) à normaliser (§ 2.4).
8. **Unités** : `statistics_during_period` convertit en unité d'affichage par défaut. Pour un import fidèle, passer `units` explicitement ou lire `statistics_unit_of_measurement`.
9. **Horodatages** : secondes epoch (float) dans `history` et en SQL, **millisecondes** dans les statistiques WS.
10. **Énergie** : importer `sum` (cumul continu) et/ou `state` (index) selon la fonction Gladys cible. `last_reset` et les remises à zéro rendent `state` non monotone (§ 5.5).
11. **États non numériques** (`unknown`, `unavailable`, textes) à filtrer : Gladys n'historise que du numérique.
12. **Chiffrement** : v1, v2 et v3 coexistent. La v3 (défaut depuis 2026.3) exige libsodium (Argon2id, BLAKE2b avec sel et personnalisation, *secretstream*). Coût Argon2id : 16 Mio par dérivation, une dérivation par archive interne (racine) plus BLAKE2b par fichier. Les archives chiffrées ne se lisent qu'en flux séquentiel.
13. **Sauvegarde sans base** (`exclude_database`), base externe (`db_url`), ou sauvegarde Supervisor partielle sans HA (`homeassistant: null`) : prévoir chaque cas dans le rapport.
14. **WAL** : extraire `home-assistant_v2.db` **et** `home-assistant_v2.db-wal` côte à côte avant d'ouvrir la base, sinon les écritures récentes manquent. Ouvrir en lecture seule sur une copie temporaire purgée ensuite.
15. **Schéma du recorder** : vérifier `schema_changes` ; utiliser `mean_type` et non `has_mean` (NULL depuis la v50).
16. **Secrets en clair dans la sauvegarde déchiffrée** : `core.config_entries`, `secrets.yaml`, `auth` (jetons de rafraîchissement, y compris le jeton longue durée éventuellement fourni à l'outil), `auth_provider.homeassistant`, `backup` (mot de passe des sauvegardes). Extraire en mémoire uniquement les fichiers nécessaires et ne jamais écrire le reste sur disque.
17. **YAML côté sauvegarde** : balises `!include*`, `!secret`, `!env_var` (la valeur n'existe pas hors de la machine HA), `!input`, packages et clés `automation xxx:`. Il faut un analyseur YAML capable de balises personnalisées (par exemple le paquet `yaml` en JavaScript, **non vérifié**).
18. **Automatisations sans `id`** : pas d'entrée de registre ni de `unique_id`. L'idempotence (D13) devra s'appuyer sur l'`entity_id` (dérivé de l'alias, donc instable) ou sur une empreinte du contenu.
19. **Compatibilité de syntaxe** : accepter les deux syntaxes (`trigger`/`triggers`, `platform`/`trigger`, `service`/`action`), comme le fait HA.
20. **Évolution rapide** : plusieurs champs sont annoncés dépréciés (`config_entries` et `primary_config_entry` des appareils jusqu'en 2027.10, `has_mean`). S'appuyer sur les nouveaux champs (`config_entry_id`, `mean_type`).

---

## 9. Écarts avec la vision (00) et l'architecture (01)

| Référence | Affirmation | Constat |
|---|---|---|
| 00 § 9 bis | Identifiants des intégrations en clair dans `.storage` « à confirmer » | **Confirmé** (§ 3.2). Mais **inaccessibles par l'API** : le préremplissage n'est possible qu'avec la source sauvegarde |
| 00 § 9 ter | États 10 jours par défaut, statistiques indéfiniment, seulement avec `state_class` | **Confirmé** (§ 5.1, § 5.5). Nuance : les statistiques 5 min sont purgées comme les états, et il existe des statistiques externes sans entité |
| 00 § 9 ter | Base MariaDB/PostgreSQL absente de la sauvegarde | **Confirmé** pour `homeassistant.tar.gz` (§ 4.3) |
| 00 § 9 bis | « Mot de passe ou clé » | Même mécanisme : la clé est une chaîne de mot de passe (§ 4.4) |
| 01 § 3.1 | Transport WS + REST pour les automatisations | REST `/api/config/automation/config/<id>` ne couvre que `automations.yaml`. Utiliser plutôt **WS `automation/config`** (admin), qui couvre tout et résout les blueprints |
| 01 § 3.1 | Lecture de la sauvegarde avec le binaire `tar` | Insuffisant pour les archives internes chiffrées (flux à déchiffrer avant `gunzip`/tar) ; v3 = libsodium |
| 01 § 3.1 / § 1 | Deux sources → même inventaire | Vrai pour les registres. **Faux en fidélité pour les automatisations** : la source sauvegarde doit réimplémenter YAML, packages et blueprints |
| 01 § 4 | `Entity` : unité, `state_class` | Par l'API, l'unité vient de `get_states`, pas du registre |
| 01 § 6.2 | Clé IEEE « connexion `zigbee` ou identifiant » | Les deux existent pour ZHA, format `xx:xx:…` à normaliser ; doublons possibles depuis la scission de 2026.8 |
| 01 § 8 | « Compter » les points selon la source | Pas de commande de comptage en 2026.11 ; `recorder/database_usage` (admin) seulement à partir de 2026.12 (probable) |
