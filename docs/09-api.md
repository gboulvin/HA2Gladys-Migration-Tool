# 09 — API du service

- **Statut** : 📝 Brouillon
- **Dernière mise à jour** : 2026-11-19
- **Prérequis** : [00 — Vision](00-vision.md) (§ 6 bis, § 8), [01 — Architecture](01-architecture.md) (§ 3.5, § 5.1 bis, § 7, § 9.2), [05 — Parcours](05-ux.md)
- **Référence Gladys étudiée** : commit `71edb17` du dépôt `GladysAssistant/Gladys`

Cette section définit le **contrat entre le front et le service** `migration-home-assistant` : routes REST, messages WebSocket, types de tâches et codes traduits par le front (T1). Les corps sont donnés sous forme abrégée ; le schéma exact (validation Joi) sera écrit avec le code.

---

## 1. Conventions de Gladys suivies

| Point | Convention | Code |
|---|---|---|
| Déclaration des routes | Le service expose un objet `controllers` dans son `index.js`. Chaque clé est `'<verbe> <chemin>'`. Le cœur fusionne les routes de tous les services avec les siennes. | `server/services/zigbee2mqtt/index.js:99`, `server/api/routes.js:67-75` |
| Préfixe | `/api/v1/service/<nom du service>/…`, soit **`/api/v1/service/migration-home-assistant/…`** | `server/services/zigbee2mqtt/api/zigbee2mqtt.controller.js:199-240` |
| Droits | Chaque route porte `authenticated: true` **et** `admin: true`. Le cœur ajoute alors `adminMiddleware` (A1, 01 § 7.1). | `server/api/setupRoutes.js:60-64`, exemple `zigbee2mqtt.controller.js:218-222` |
| Erreurs | Le service lève les erreurs du cœur (`BadParameters`, `NotFoundError`, `ConflictError`, `ForbiddenError`) ; le middleware les traduit en 400, 404, 409, 403. Le corps de la réponse porte `status`, `code` et `message`. Le service met dans `message` un **code de raison** (§ 4), jamais un texte. | `server/utils/coreErrors.js:8-58`, `server/api/middlewares/errorMiddleware.js:30-87`, `server/utils/httpErrors.js:9-14` |
| Messages WebSocket | Déclarés dans `WEBSOCKET_MESSAGE_TYPES`, émis par `EVENTS.WEBSOCKET.SEND` (un utilisateur, par `userId`) ou `SEND_ALL` (tous les utilisateurs). | `server/utils/constants.js:2194`, `:742-745` ; `server/api/websockets/index.js:10-11`, `:40-61` |
| Tâches de fond | `gladys.job.start(type, data)`, `updateProgress`, `finish(id, status, data)`. Le `type` doit figurer dans `JOB_TYPES`, et sa traduction dans `jobTypes` du front. | `server/lib/job/job.start.js:12`, `job.finish.js:15`, `server/models/job.js:79-84`, `front/src/config/i18n/fr.json:5577-5587` |
| Langue | Le serveur ne renvoie que des codes ; le front les traduit en en, fr, de et es (01 § 9.2). | — |

## 2. Routes REST

Toutes les routes ci-dessous sont **réservées aux administrateurs** (`authenticated: true`, `admin: true`). Le préfixe `/api/v1/service/migration-home-assistant` est noté `…`. Une route qui lance un traitement long répond **202** avec l'identifiant de la tâche (`job_id`) ; la suite arrive par WebSocket (§ 3).

### 2.1 Migration et exécutions (01 § 5.1 bis)

| Verbe et chemin | Rôle | Corps | Réponse |
|---|---|---|---|
| `get …/migration` | Migration ouverte, exécution courante, phase, résumé | — | `{ migration: { id, source, mode, status } \| null, run: { id, kind, phase, status, job_id } \| null, summary }` |
| `post …/migration` | Démarrer une migration (écran 1) | `{ mode: "coexistence" \| "switchover" }` | `201 { migration }` ; `409 MIGRATION_ALREADY_OPEN` |
| `post …/run/resume` | Reprendre l'exécution en pause ou interrompue | — | `202 { job_id }` ; `400 SECRET_REQUIRED` si un secret manque (§ 2.2) |
| `post …/run/pause` | Mettre en pause (fin du lot en cours, 01 § 3.5) | — | `200 { run }` |
| `post …/run/abandon` | Abandonner l'exécution en cours (purge du dossier de travail) | — | `200 { run }` |
| `post …/resync` | Relancer l'analyse (D13) : nouvelle exécution `resync` | — | `202 { run, job_id }` ; `409 MIGRATION_RUN_IN_PROGRESS` |
| `post …/switchover` | Passer en bascule (A11) : nouvelle exécution `switchover` | — | `202 { run, job_id }` ; `409 MIGRATION_RUN_IN_PROGRESS` |
| `get …/cancel/preview` | Ce que l'annulation supprimera, restaurera et conservera (05 § 5) | — | `{ delete: […], restore: […], keep_history: { devices: N }, user_dependents: […] }` |
| `post …/cancel` | Annuler toute la migration (A9) | `{ confirmation: "<mot saisi>", keep_user_items: bool }` | `202 { run, job_id }` ; `400 CONFIRMATION_MISMATCH` |
| `get …/report/export` | Rapport en Markdown, expurgé (U3) | — | `text/markdown` |

### 2.2 Source et secrets (écran 2, 02 § 2 et § 3)

| Verbe et chemin | Rôle | Corps | Réponse |
|---|---|---|---|
| `post …/source/connect` | Connexion à HA : contrôles de version et de statut administrateur | `{ url, token }` | `{ ha_version, backups: [{ id, agent_id, date, homeassistant_included, database_included }] }` |
| `post …/source/backup` | Mode A : télécharger et lire la sauvegarde choisie (extraction sélective, 01 § 7.2) | `{ backup_id, agent_id, password }` | `202 { job_id }` |
| `post …/source/no-backup` | Mode B : continuer sans sauvegarde | — | `200` |
| `post …/source/file/prepare` | Mode C : annoncer un fichier et son mot de passe **avant** l'envoi, pour que l'extraction sélective se fasse pendant le transfert | `{ password, size }` | `{ upload_id }` ; `400 DISK_SPACE_INSUFFICIENT` |
| `post …/source/file/<upload_id>` | Mode C, voie 2 (facultative, 01 § 7.3) : corps brut en flux | `application/octet-stream` | `202 { job_id }` |
| `post …/source/file/<upload_id>/deposited` | Mode C, voie 3 : lire le fichier déposé dans le dossier prévu | `{ file_name }` | `202 { job_id }` |
| `post …/secrets` | Redonner un secret après un redémarrage (01 § 7.2) | `{ token?, backup_password? }` | `200` |

Les secrets ne transitent que dans ces corps de requête. Ils ne sont **jamais** renvoyés, ni journalisés (01 § 7.2).

### 2.3 Analyse et choix (écrans 3 et 4)

| Verbe et chemin | Rôle | Corps | Réponse |
|---|---|---|---|
| `post …/analysis` | Lancer l'analyse (phase A) | — | `202 { job_id }` |
| `get …/report` | Rapport de compatibilité, filtrable | requête : `type`, `status`, `integration`, `room`, `page` | `{ counts, items: [{ type, key, name, status, reasons: [{ code, path?, params? }], target? }] }` |
| `patch …/choices` | Enregistrer les choix de l'écran 4 | `{ items: [{ type, key, selected }], rooms, floor_prefix, house_id, persons: [{ key, login, email?, password?, birthdate, language, role, track_presence }], zones, use_ha_coordinates, history, dashboards: { ha: [url_path] } }` | `200 { counts }` |

### 2.4 Phases B à F (écrans 5 à 9)

| Verbe et chemin | Rôle | Corps | Réponse |
|---|---|---|---|
| `get …/integrations` | Intégrations et modules à installer, réglages repris (secrets masqués) | — | `{ items: [{ target, state, config: [{ key, secret, masked_value }] }] }` |
| `post …/integrations/install` | Phase B | `{ items: [{ target, config_keys: [...] }] }` | `202 { job_id }` |
| `get …/devices/matches` | Candidats de rapprochement, appareils virtuels, présence | — | `{ devices: [...], helpers: [...], presence: [{ person, trackers: [{ key, candidate }], minutes }], substitutes: [...] }` |
| `post …/devices/import` | Phase C | `{ matches: [{ key, device_external_id }], substitutes: [{ key, device_feature }], presence: [{ person, device_features: [...], minutes }] }` | `202 { job_id }` |
| `post …/devices/reappair` | Ouvrir le parcours de réappairage (03 § 9) | `{ integration }` | `200` |
| `post …/scenes/import` | Phase D | `{ items: [key], rewrites: [key], notifications: [{ ha_service, user, service? }] }` | `202 { job_id }` |
| `post …/automations/<key>/draft` | Créer un brouillon de scène (04 § 2.5) | — | `201 { scene_selector }` |
| `get …/history/estimate` | Estimation du volume, de la durée et de l'espace (07 § 5) | requête : `retention` | `{ points, bytes, duration_s, free_bytes, gauge: "green" \| "orange" \| "red", features: [...] }` |
| `post …/history/import` | Phase E | `{ features: [selector], retention: "unlimited" \| "extend" \| "keep" }` | `202 { job_id }` ; `400 DISK_SPACE_INSUFFICIENT` |
| `get …/dashboards/preview` | Aperçu de l'Accueil et des reprises (08) | — | `{ home, ha: [{ url_path, widgets, skipped_cards }] }` |
| `post …/dashboards/create` | Phase F | `{ ha: [url_path], home_first: bool }` | `202 { job_id }` |

## 3. Messages WebSocket et types de tâches

**Messages** : un nouveau groupe `MIGRATION` dans `WEBSOCKET_MESSAGE_TYPES` (modification du cœur, A7). Ils sont envoyés **à chaque administrateur connecté**, un message par `userId` avec `EVENTS.WEBSOCKET.SEND`, et **jamais** par `SEND_ALL`, qui atteint aussi les habitants (`server/api/websockets/index.js:40-61`).

| Type | Charge utile | Quand |
|---|---|---|
| `migration.run-updated` | `{ run_id, kind, phase, status }` | Changement de phase ou de statut d'une exécution |
| `migration.progress` | `{ run_id, phase, done, total, current? }` | Avancement détaillé, en plus de `job.updated` |
| `migration.device-matched` | `{ key, device_selector }` | Un appareil découvert vient d'être rapproché (réappairage, 03 § 9) |
| `migration.secret-required` | `{ run_id, secrets: ["token" \| "backup_password"] }` | Reprise après redémarrage qui a besoin d'un secret (01 § 7.2) |
| `migration.run-finished` | `{ run_id, status, summary }` | Fin d'une exécution |
| `migration.error` | `{ run_id, code, params? }` | Erreur bloquante d'une exécution (§ 4) |

**Types de tâches** : nouveaux `JOB_TYPES` (modification du cœur, A7), nommés comme ceux des services existants (`service-zigbee2mqtt-install`, `server/utils/constants.js:2539`), avec leur traduction dans `jobTypes` du front :

| Type | Phase |
|---|---|
| `service-migration-home-assistant-analysis` | A (lecture de la source et analyse) |
| `service-migration-home-assistant-integrations` | B |
| `service-migration-home-assistant-devices` | C |
| `service-migration-home-assistant-scenes` | D |
| `service-migration-home-assistant-history` | E |
| `service-migration-home-assistant-dashboards` | F |
| `service-migration-home-assistant-cancel` | Annulation |

Une pause ou un redémarrage termine la tâche ; la reprise en démarre une nouvelle du même type (01 § 3.5).

## 4. Codes traduits par le front

Chaque code est une clé de traduction du front, en en, fr, de et es. Une **raison** d'analyse s'accompagne, quand c'est utile, du **chemin** de l'élément fautif (`actions[2].choose[0]`, 04 § 2.2) et de **paramètres** (nom d'appareil, version…). Les **statuts** eux-mêmes sont ceux de 00 § 8.

### 4.1 Erreurs des routes

| Code | HTTP | Sens |
|---|---|---|
| `HA_URL_INVALID` | 400 | URL non `http` ou `https`, ou redirection (01 § 7.1) |
| `HA_UNREACHABLE` | 400 | HA injoignable depuis Gladys |
| `HA_INVALID_TOKEN` | 400 | Jeton refusé par HA |
| `HA_NOT_ADMIN` | 400 | Jeton d'un compte non administrateur (02 § 3.1) |
| `HA_VERSION_TOO_OLD` | 400 | HA antérieur à 2026.8 (E2) |
| `BACKUP_NOT_FOUND` | 404 | Sauvegarde absente |
| `BACKUP_WRONG_PASSWORD` | 400 | Mot de passe ou clé de chiffrement incorrect |
| `BACKUP_NO_HOMEASSISTANT` | 400 | Sauvegarde partielle sans HA (02 § 4.1) |
| `BACKUP_FORMAT_UNSUPPORTED` | 400 | Format de fichier ou version non pris en charge (02 § 7) |
| `DISK_SPACE_INSUFFICIENT` | 400 | Espace disque insuffisant (01 § 8) |
| `MIGRATION_ALREADY_OPEN` | 409 | Une migration est déjà ouverte (01 § 5.1 bis) |
| `MIGRATION_RUN_IN_PROGRESS` | 409 | Une exécution est déjà en cours ou en pause |
| `MIGRATION_NOT_FOUND` | 404 | Aucune migration ouverte |
| `SECRET_REQUIRED` | 400 | Un secret doit être redonné avant de reprendre (01 § 7.2) |
| `CONFIRMATION_MISMATCH` | 400 | Mot de confirmation de l'annulation incorrect (05 § 5) |
| `MQTT_NOT_CONFIGURED` | 400 | Appareils virtuels demandés sans service MQTT configuré (04 § 6.2) |

### 4.2 Avertissements (non bloquants)

| Code | Sens |
|---|---|
| `HA_VERSION_NEWER_THAN_TESTED` | HA plus récent que la dernière version testée (02 § 7) |
| `HA_NOT_HTTPS` | Mot de passe de sauvegarde transmis sans HTTPS (02 § 3.3) |
| `CATALOG_OFFLINE` | Catalogue injoignable : cache ou instantané embarqué utilisé (01 § 3.3) |
| `DISK_SPACE_LOW` | Espace suffisant sans la marge (jauge orange, 07 § 5) |
| `HISTORY_KEPT_ON_CANCEL` | Historique conservé sur les appareils préexistants (01 § 5.4) |

### 4.3 Raisons d'analyse

| Domaine | Codes |
|---|---|
| Appareils (03 § 8) | `DOMAIN_NOT_IN_CATALOG`, `TARGET_IN_PROGRESS`, `TARGET_FREE`, `TARGET_SUGGESTED_ONLY` (CT1), `DEVICE_UNSUPPORTED` (règle `unsupported`), `DEVICE_EXCLUDED`, `ENTITY_DISABLED`, `ENTITY_HIDDEN`, `ENTITY_DIAGNOSTIC`, `FEATURE_LOST`, `REPAIR_REQUIRED` (ZHA, Matter), `ALTERNATIVE_VIA` (🔀) |
| Entrées et présence (03 § 7 bis, 04 § 6.3) | `HELPER_TYPE_UNSUPPORTED`, `TRACKER_GPS_GUIDED`, `ZONE_NEEDS_USER_LOCATION`, `USER_PRESENCE_CONDITION` |
| Scènes, scripts, automatisations (04 § 2 à § 8 ter) | `TEMPLATE`, `TRIGGER_UNSUPPORTED`, `CONDITION_UNSUPPORTED`, `ACTION_UNSUPPORTED`, `CONDITION_IN_BRANCH`, `OR_WITH_AND`, `STATE_WITHOUT_FROM_TO`, `WAIT_FOR_EVENT`, `MODE_WITH_WAIT`, `TIME_IN_DST_GAP`, `SUN_OFFSET_OUTSIDE_DAY`, `SUN_CONDITION_OFFSET`, `MQTT_WILDCARD_UNSUPPORTED`, `CONFIG_MISMATCH` (fuseau, coordonnées, unités), `DEVICE_NOT_MIGRATED`, `SUBSTITUTE_REQUIRED`, `NOTIFY_MAPPING_REQUIRED`, `SCRIPT_WITH_FIELDS`, `SCRIPT_NOT_SIMPLE`, `SCENE_CALL_NOT_LAST`, `SCENE_START_DEFERRED`, `ATTRIBUTE_UNSUPPORTED`, `SCENE_FROM_INTEGRATION`, `CONVERTER_ERROR` (échec de validation Joi, 04 § 7) |
| Historiques (07) | `UNIT_CONVERSION_UNKNOWN`, `TEXT_STATE`, `KEEP_HISTORY_DISABLED`, `MEAN_TYPE_MISSING`, `OLDER_THAN_RETENTION` |
| Modules (03 § 7 ter) | `ADDON_NOT_NEEDED`, `ADDON_NO_EQUIVALENT`, `ADDON_DATA_NOT_IMPORTED` |
| Tableaux de bord (08) | `CARD_UNSUPPORTED`, `CARD_ENTITY_NOT_MIGRATED`, `DASHBOARD_AUTO_GENERATED` |
| Relance (01 § 5.3) | `ALREADY_MIGRATED`, `CHANGED_IN_HA`, `DELETED_IN_HA`, `GLADYS_OBJECT_DELETED` |

Cette liste est fermée : ajouter une raison, c'est ajouter un code ici, sa traduction dans les quatre langues et son test (06 § 3).
