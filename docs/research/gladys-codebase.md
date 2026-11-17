# Recherche — Code source de Gladys Assistant

- **Commit Gladys étudié** : `71edb17` (« feat(nuki): deprecate the native Nuki integration (#3391) »), clone dans `/home/user/ref/gladys`
- **Date** : 2026-11-17
- **Statut** : note de recherche factuelle (entrée des sections 01 à 05 de la spec)

Conventions de cette note :

- Les chemins sont relatifs à la racine du dépôt `GladysAssistant/Gladys`. `fichier:NN` renvoie à la ligne NN.
- « **non vérifié** » signale une affirmation non confirmée par la lecture du code.
- Les paragraphes **« Analyse »** sont des déductions pour notre service, et pas des faits sur Gladys.

---

## 1. Conventions de contribution

### 1.1 Documents de référence

| Document | Contenu utile |
|---|---|
| `AGENTS.md` | La référence opérationnelle : lancement en dev (server sur le port 1443, front Vite sur le port 1444), processus *spec-first*, flux git, CI et commandes. Pointé par `.github/CONTRIBUTING.md` (« Point your agent to AGENTS.md »). |
| `.github/CONTRIBUTING.md` | Recommande d'**ouvrir d'abord une discussion sur le forum**, puis de suivre le guide « How to contribute a new service » (site gladysassistant.com, **non vérifié**). Il encourage les PR *draft* précoces pour les humains et l'usage de l'IA. |
| `.github/PULL_REQUEST_TEMPLATE.md` | Sections : *Description*, *Related request* (`Forum: https://community.gladysassistant.com/t/...` ou `Closes #1234`), *Checklist* (coverage, Cypress si UI, eslint/prettier, pas de breaking change non documenté). |
| `docs/specs/` | Les **specs vivantes**, toutes rédigées **en anglais** : `device-migration.md`, `device-feature-categories.md`, `integration-catalog-categories.md`, `scene-editor-mobile.md`, `scene-time-ranges.md`, `zigbee2mqtt-installation.md`, `external-integrations/` (dossier)… |
| `CHANGELOG.md` | Pas à jour pour les versions récentes : aucune mention des dépréciations récentes. Il est donc peu fiable comme source. |

**Processus spec-first** (`AGENTS.md`, section « Feature specs ») : il est exigé pour les « large projects and anything touching the data model ». Une spec dans `docs/specs/` devient la source de vérité. Toute PR qui modifie un comportement couvert doit mettre la spec à jour **dans le même diff**.

**Flux git pour les agents** (`AGENTS.md`, « Git workflow ») :

- jamais de commit sur `master`, une branche par changement ;
- titre et description de PR **en anglais** ;
- PR créée **prête pour revue** (pas en *draft*) ;
- lien `Forum:` ou `Closes #` recommandé, car les automatisations de release (annonce sur le forum, fermeture du sujet) en dépendent.

### 1.2 Outillage, lint et format

| Élément | Serveur (`server/`) | Front (`front/`) |
|---|---|---|
| Node / npm | `"node": "24.x"`, `"npm": "11.x"` (`server/package.json:5-8`) | idem (`front/package.json:3-6`). Les images Docker partent de `node:24-slim` (`docker/Dockerfile:5`). |
| Format | Prettier : `singleQuote`, `printWidth: 120`, `trailingComma: all` (`server/.prettierrc.json`) | Prettier, `printWidth: 120` (`front/.prettierrc*`) |
| Lint | ESLint `airbnb-base` + `plugin:jsdoc/recommended` + `prettier`, plugins `require-jsdoc`, `mocha`, `promise` (`server/.eslintrc.json`). Une **JSDoc est exigée** sur les fonctions exportées (`@description`, `@param`, `@returns`, `@example`). | `eslint src cypress` avec `eslint-config-preact` (`front/package.json:17`) |
| Traductions | — | `npm run compare-translations` : toutes les langues doivent avoir les mêmes clés, et les constantes de features du serveur doivent avoir leurs libellés dans `en.json` (`front/package.json:18`, `front/cli/check_translations.js`). |
| Build | — | `vite build` (`front/package.json:11`) |

### 1.3 Tests et couverture

- **Mocha 8 + Chai 4 (+ chai-as-promised) + Sinon 8**, avec `proxyquire` et `nock` pour les mocks (`server/package.json:55-76`, `server/test/hooks.js`).
- `npm test` lance Mocha en `--parallel` sur `./test/**/*.test.js`, avec une base SQLite de test seedée (`server/package.json:11`, `server/test/hooks.js`, `server/seeders/`, 19 fichiers).
- `npm run test-service --service=<nom>` teste un seul service (`server/package.json:13`).
- La CI exécute `npm run coverage` (c8 + lcov) (`server/package.json:14`).
- **Codecov** (`codecov.yml`) :
  - *project* : 90 % sur le flag `server` ;
  - ***patch* : 100 %** : chaque ligne ajoutée ou modifiée doit être exécutée par un test. `AGENTS.md` décrit cette règle comme la cause d'échec la plus fréquente.
- Les tests sont rangés en miroir du code source : `server/lib/foo/bar.js` → `server/test/lib/foo/bar.test.js` (`AGENTS.md`). Pour un service : `server/test/services/<service>/...` (ex. `server/test/services/zigbee2mqtt/{api,lib,utils,exposes}`).
- **Le front n'a pas de tests unitaires.** Il est validé par eslint, `compare-translations`, le build et **Cypress** (`front/cypress/e2e/`, lancé avec `npm run cypress:run`).
- Jobs CI (`.github/workflows/docker-pr-build.yml`, résumé dans `AGENTS.md`) : Front test, Server test, Cypress, puis Front build et Docker build AMD64 quand la PR n'est pas en *draft*.

### 1.4 Dépendances d'un service

Chaque service a **son propre `package.json`**, avec `os` et `cpu` déclarés (ex. `server/services/example/package.json`). Ses dépendances sont installées par le `postinstall` du serveur (`server/cli/install_service_dependencies.js`, cité par `AGENTS.md`).

Dépendances déjà disponibles au niveau du serveur : `axios`, `ws` 6, `sequelize` 6, `express` 4, `queue`, `@hapi/joi` (`server/package.json`).

Aucune bibliothèque d'archive (tar/zip) ni de YAML n'est présente au niveau du serveur. Seul le service zigbee2mqtt embarque `yaml` (`server/services/zigbee2mqtt/package.json:22`). Les sauvegardes Gladys utilisent le binaire système `tar` via `execFile` (`server/lib/gateway/gateway.backup.js:137`, `gateway.downloadBackup.js:67-80`).

---

## 2. Anatomie d'un service (backend)

### 2.1 Gabarit : le service `example`

`server/services/example/index.js` est une fonction fabrique `function ExampleService(gladys)`. Elle renvoie `Object.freeze({ start, stop, light, device })`. Le module `lib/light/` montre l'organisation habituelle : **un fichier par méthode**, rattachées ensuite au prototype de la classe (`light.turnOn.js`, `light.setValue.js`…).

### 2.2 Enregistrement et cycle de vie

| Étape | Fichier | Comportement |
|---|---|---|
| Déclaration | `server/services/index.js` | `module.exports['zwavejs-ui'] = require('./zwavejs-ui');` : la clé devient le nom du service. |
| Chargement | `server/lib/service/service.load.js:13-55` | Pour chaque service présent dans les fichiers : `db.Service.findOne({ pod_id: null, name })`, et création de la ligne `t_service` si elle manque. La fabrique est ensuite appelée avec `(gladys, serviceInDb.id)` : **le 2ᵉ argument est l'UUID du service**. Le résultat est stocké dans `stateManager` (`service` et `serviceById`). Si le constructeur lève une exception, le statut passe à `DISABLED`. |
| Démarrage | `server/lib/service/service.startAll.js` | Démarre en séquence (`concurrency: 1`) tous les services, sauf ceux en `STOPPED`. |
| Démarrage d'un service | `server/lib/service/service.start.js:15-60` | Statut `LOADING` → `await service.start()` → `RUNNING`. Une `ServiceNotConfiguredError` laisse le statut à `RUNNING` (« not configured ») ; toute autre exception donne `ERROR`. |
| Statuts | `server/utils/constants.js:488-497` | `UNKNOWN, ENABLED, DISABLED, LOADING, RUNNING, DEGRADED, STOPPED, ERROR` |

**Exemple réel : `server/services/zigbee2mqtt/index.js`.**

- La fabrique `Zigbee2mqttService(gladys, serviceId)` fait un `require('mqtt')` dans la fabrique, pour que les dépendances du service restent locales.
- Elle instancie un *manager* (`lib/index.js`, une classe avec une méthode par fichier).
- Elle renvoie `{ start, stop, isUsed, device: manager, controllers: Controller(gladys, manager) }`.
- `stop()` attend la fin des opérations en cours (installation, etc.) avant de couper.

### 2.3 Objet `gladys` injecté

L'objet est construit dans `server/lib/index.js`. Ses clés : `version, area, brain, calendar, config, dashboard, event, house, http, job, gateway, location, mdns, message, user, service, scene, scheduler, session, cache, device, room, stateManager, system, variable, weather, energyContract, energyPrice, externalIntegration, tts`.

Les services appellent directement le cœur, par exemple :

- `gladys.device.create(...)` : `services/zigbee2mqtt/lib/syncRenamedDevices.js:73`, `services/philips-hue/lib/light/light.configureBridge.js:37`, `services/thermostat/lib/thermostat.createDevice.js:276` ;
- `gladys.scene.create(...)` : `services/mcp/lib/buildSchemas.js:581`.

### 2.4 Configuration : `gladys.variable`

- API : `setValue(key, value, serviceId = null, userId = null)` (`server/lib/variable/variable.setValue.js:15`) et `getValue(key, serviceId)`. Exemple : `server/services/zigbee2mqtt/lib/getConfiguration.js:15-19`.
- Modèle `t_variable` (`server/models/variable.js`) : `name` doit être en **MAJUSCULES** (`validate.isUppercase`), `value` est de type `TEXT`, et les champs `service_id` et `user_id` sont optionnels.
- Le front lit une variable de service via `GET /api/v1/service/:service_name/variable/:variable_key` (`server/api/routes.js:1040`). Les variables globales (`/api/v1/variable/:key`) sont réservées aux admins (`server/api/routes.js:1044-1056`).

### 2.5 Contrôleurs et routes API

**Déclaration.** Le service expose `controllers`, un objet de la forme `{ 'get /api/v1/service/<nom>/<route>': { authenticated, admin?, controller } }` (ex. `services/zigbee2mqtt/api/zigbee2mqtt.controller.js:200-260`). Les handlers asynchrones sont enveloppés par `asyncMiddleware` (`server/api/middlewares/asyncMiddleware.js`).

**Fusion.** `server/api/routes.js:67-75` fusionne les `controllers` de tous les services dans la table des routes.

**Montage.** `server/api/setupRoutes.js` monte, selon les drapeaux de la route :

| Drapeau | Middleware monté |
|---|---|
| `authenticated: true` | `AuthMiddleware('dashboard:write')`, ou un `scope` spécifique |
| `admin: true` | `adminMiddleware` |
| `rateLimit` | limitation de débit |
| `largeJsonBody` | parseur JSON de 6 Mo |
| `audioRawBody` | corps brut `audio/*` ou `application/octet-stream`, 5 Mo (`api/middlewares/audioRawBodyMiddleware.js`) |
| `externalIntegrationAuth` | authentification des intégrations externes |

**Convention observée.** Les routes de configuration ou destructrices sont `admin: true` (ex. `post /api/v1/service/zigbee2mqtt/setup`, `.../reset`). Les lectures sont seulement `authenticated`.

**Corps JSON limité à 100 ko.**

- Le middleware global `jsonBodyMiddleware` parse tout corps JSON avec `limit: '100kb'`. Seuls deux préfixes y échappent : `/api/integration/v1/` et `/api/v1/dashboard_asset/` (`server/api/middlewares/jsonBodyMiddleware.js:22-58`).
- **Conséquence** : un drapeau `largeJsonBody` posé sur une route de service ne suffit pas. Le parseur global de 100 ko passe avant lui, car son exemption est codée en dur sur le préfixe des assets de dashboard. *Analyse* : il faudra modifier le cœur pour accepter un téléversement volumineux (voir § 8).

**Gestion des erreurs** : `server/api/middlewares/errorMiddleware.js`.

| Erreur levée | Réponse HTTP |
|---|---|
| Joi | 422 |
| `SequelizeValidationError` | 422, détails par attribut, avec `context` si `gladysContext` est présent |
| `SequelizeUniqueConstraintError` | 409 |
| `entity.too.large` | 413 |
| `NotFoundError` | 404 |
| `BadParameters` | 400 (`errorMiddleware.js:62`) |
| `ConflictError` | 409 |

Les classes d'erreur sont dans `server/utils/coreErrors.js` (`NotFoundError`, `ServiceNotConfiguredError`, `BadParameters`, `ConflictError`, `ForbiddenError`, `AbortScene`, `SceneStopped`…).

### 2.6 Événements et WebSocket vers le front

- **Émission** : `gladys.event.emit(EVENTS.WEBSOCKET.SEND_ALL, { type, payload })`. Exemple : `services/zigbee2mqtt/lib/events/emitStatusEvent.js:8-15`. Il existe aussi `EVENTS.WEBSOCKET.SEND`, pour un seul utilisateur (`server/utils/constants.js`, bloc `WEBSOCKET`).
- **Types de messages** : ils sont déclarés dans `WEBSOCKET_MESSAGE_TYPES` (`server/utils/constants.js:2194`), avec un sous-objet par service (ex. `ZIGBEE2MQTT.DISCOVER/STATUS_CHANGE/PERMIT_JOIN/INSTALL_PROGRESS`, `:2276`).
- **Réception côté front** : `this.props.session.dispatcher.addListener(WEBSOCKET_MESSAGE_TYPES.X.Y, handler)`. Le front **importe directement** `server/utils/constants.js` (`front/src/routes/integration/all/zigbee2mqtt/discover-page/index.js:6-18`).
- **Hooks d'appareil** : `device.notify` appelle `service.device.postCreate`, `postUpdate` et `postDelete` du service propriétaire après création, mise à jour ou suppression (`server/lib/device/device.notify.js:6-40`). Exemple : `services/zigbee2mqtt/lib/postCreate.js`.

### 2.7 i18n côté serveur

**Le serveur n'a pas de système i18n.** Le commentaire `server/lib/scene/scene.actions.js:804-805` dit : « the server has no i18n system. Falls back to english ». On y trouve seulement de petites tables locales `en/fr/de`.

La convention est de renvoyer des **codes ou des faits bruts** que le front traduit :

- `job.updateProgress` : « only put raw values here (counts, names), never sentences » (`server/lib/job/job.updateProgress.js:9-12`) ;
- codes d'erreur MQTT dans `services/zigbee2mqtt/lib/constants.js`.

Seuls les fichiers du *brain* (NLP) ont des réponses localisées : `server/config/brain/*/answers.{en,fr,es}.json`.

---

## 3. Front (Preact)

### 3.1 Pile technique

`front/package.json` : Preact 10, `preact-router`, `preact-async-route`, `unistore` (store global), `preact-i18n`, `axios`, Vite 8. Bootstrap/Tabler (classes `fe fe-*`) d'après le code observé.

### 3.2 UI d'une intégration

**Emplacement** : `front/src/routes/integration/all/<service>/`. Pour zigbee2mqtt :

| Sous-dossier | Rôle |
|---|---|
| `device-page/` | appareils créés |
| `discover-page/` | découverte |
| `setup-page/` | configuration |
| `edit-page/` | édition d'un appareil |
| `commons/` | composants partagés (statut, erreurs) |
| `Zigbee2mqttPage.js` | gabarit commun avec onglets |

**Patron d'une page** :

- composant classe qui s'abonne aux WebSocket dans `componentWillMount` et se désabonne dans `componentWillUnmount` ;
- `connect('clé1,clé2,…', actions)(Composant)` d'unistore ;
- `withIntlAsProp(...)` pour accéder au dictionnaire (`discover-page/index.js`).

**Actions** : un fichier `actions.js` par page, `createActions(store)`, qui compose `createActionsIntegration` et `createActionsHouse` (`front/src/actions/...`). Le client HTTP est dans l'état (`state.httpClient.get('/api/v1/service/zigbee2mqtt/discovered', {...})`, `discover-page/actions.js:30-35`).

**Routing** : déclaratif dans `front/src/components/app.jsx`, par exemple :

```jsx
<Zigbee2mqttDiscoverPage path="/dashboard/integration/device/zigbee2mqtt/discover" />
```

(`app.jsx:347-350`). Le motif d'URL est `/dashboard/integration/{type}/{key}` (`docs/specs/integration-catalog-categories.md` § 2).

### 3.3 Catalogue des intégrations

**Liste.** `front/src/config/integrations/{devices,communications,calendars,weathers}.json`, agrégés par `front/src/config/integrations/index.js`, qui ajoute le `type` technique (`device`, `communication`…). Une entrée a la forme :

```json
{ "key", "link"?, "img": "/assets/integrations/cover/x.jpg", "local"|"cloud"|"gladysPlus", "deprecated"?, "categories": [...] }
```

(ex. `devices.json:22-28`).

**Images** : `front/src/assets/integrations/cover/*.jpg` (couvertures) et `front/src/assets/integrations/logos/`.

**Catégories de navigation.** Les clés viennent de `INTEGRATION_CATALOG_CATEGORIES` (`server/utils/constants.js:526`) : `climate, lighting, energy, security, multimedia, appliances, environment, protocols, network, notifications, assistants, services, transport, vehicles, servers, leisure`. Les icônes sont dans `front/src/config/integrations/index.js:11-28`.

**Visibilité.**

- Le `type` reste une clé technique. Les types `device`, `weather`, `provider`, `ai` et `tts` sont **masqués aux non-admins** (`docs/specs/integration-catalog-categories.md` § 2.2).
- Le catalogue mélange intégrations natives et **intégrations externes du store**. Côté communautaire : 45 intégrations sur 81 en 2026-08 (`integration-catalog-categories.md` § 1).

### 3.4 i18n du front

- Fichiers : `front/src/config/i18n/{en,fr,de,es}.json`. **Quatre langues, espagnol compris**, plus `icon-keywords/`.
- Clés d'une intégration : `integration.<key>.title`, `integration.<key>.description`, puis des sous-clés par page (ex. `en.json:1892`).
- Les libellés des catégories, types et unités de features sont aussi dans `en.json`, et contrôlés par `check_translations.js`.
- Composants : `<Text id="..."/>`, `<MarkupText/>` (`preact-i18n`).

### 3.5 Bannière de dépréciation

`front/src/components/integration/DeprecationWarning.jsx` affiche `integration.deprecationWarning` sur les pages d'intégrations natives en cours de remplacement par une intégration externe.

---

## 4. Modèle de données

Modèles Sequelize dans `server/models/`, migrations dans `server/migrations/` (81 fichiers, la dernière étant `20261031000000-matter-onoff-light-category.js`). Le moteur est SQLite. L'historique des états est dans DuckDB (`t_device_feature_state`). Les noms de tables sont préfixés `t_`.

### 4.1 Tables clés

| Table (modèle) | Champs clés | Contraintes notables |
|---|---|---|
| `t_house` (`house.js`) | `name`, `selector`, `latitude`, `longitude`, `alarm_mode`, `alarm_code`… | `name` et `selector` uniques |
| `t_room` (`room.js:12-31`) | `house_id` (FK, requis), `name`, `selector` | **`name` unique sur toute l'instance** (pas par maison) et **longueur 1 à 40** (`validate.len`). `selector` unique. |
| `t_device` (`device.js:8-55`) | `service_id` (FK, requis), `room_id` (FK, null possible), `name`, `selector`, `model`, `external_id`, `should_poll`, `poll_frequency` (enum) | `selector` **unique** ; `external_id` **unique et requis** |
| `t_device_feature` (`device_feature.js:12-120`) | `device_id`, `energy_parent_id`, `name`, `selector`, `external_id`, `category` (enum), `type` (enum), `read_only`, `keep_history`, `has_feedback`, `unit` (enum, null possible), `min`, `max` (**requis**, flottants), `step` (> 0 ou null), `last_value`, `last_value_string`, `last_value_changed`… | `selector` et `external_id` **uniques sur toute l'instance** |
| `t_device_param` (`device_param.js`) | `device_id`, `name`, `value` (TEXT) | pas d'unicité déclarée |
| `t_device_feature_supported_option` | options étiquetées d'une feature (modes, presets) | gérées par `device.syncFeatureSupportedOptions` |
| `t_scene` (`scene.js:354-415`) | `name`, `description`, `icon` (enum de `server/config/icons.json`, requis), `active` (booléen, par défaut **true**), `selector` (unique, regex kebab-case), `actions` (JSON), `triggers` (JSON), `last_executed` | validation Joi des JSON (§ 6) |
| `t_tag_scene` (`tag_scene.js`) | `scene_id`, `name` | étiquettes de scènes |
| `t_service` (`service.js`) | `name`, `selector`, `version`, `status`, `pod_id`, `has_message_feature`, `type` (`internal` ou intégration externe, `SERVICE_TYPES` `constants.js:499`) | une intégration externe est une ligne de `t_service` (`docs/specs/external-integrations/README.md`) |
| `t_variable` | voir § 2.4 | — |
| `t_job` (`job.js`) | `type` (`isIn: JOB_TYPES_LIST`), `status`, `progress`, `data` (clés validées **par type de job**) | `job.js:16-60` |

**Sélecteurs.**

- Un sélecteur est dérivé du nom par `slugify` dans un hook `beforeValidate`, à la création seulement (`server/utils/addSelector.js`).
- `device.create` calcule un sélecteur **libre** (`buildUniqueSelector` : suffixes `-2`, `-3`… puis aléatoires) pour l'appareil et pour chaque feature (`server/lib/device/device.create.js:108-114`, `:166-180`).
- Pour une scène sans sélecteur, `scene.create` utilise `slugify(name, true)`, qui ajoute des caractères aléatoires (`server/lib/scene/scene.create.js:14-20`).

### 4.2 Créer ou mettre à jour un appareil : `gladys.device.create(device)`

Source : `server/lib/device/device.create.js`.

1. **`external_id` obligatoire.** Sinon `BadParameters` (`:94-96`).
2. **Transaction SQLite `IMMEDIATE`** (`:102`).
3. **Recherche de l'existant** par `id`, puis par `external_id` (`:33-41`).
   - S'il n'existe pas : création avec un sélecteur unique.
   - S'il existe : c'est une **mise à jour (upsert)**. Le sélecteur n'est jamais modifié.
4. **Suppression des features et des params absents du nouveau payload.**
   - Features : le *match* se fait par `id` ou `external_id` (`:44-49`, `:131-137`).
   - Params : le *match* se fait par `id` ou `name` (`:51-54`, `:140-145`).
   - *Analyse* : un upsert avec un payload partiel **détruit** les features et params manquants.
5. Création ou mise à jour des features. Les colonnes d'état (`last_value`…) ne sont jamais écrites par une sauvegarde (`DEVICE_FEATURE_STATE_COLUMNS`, `:209-212`).
6. Résolution de `energy_parent_id` et `enforceEnergyHierarchy`.
7. Synchronisation de `supported_options`.
8. Création ou mise à jour des params.
9. Ajout de l'appareil au cache mémoire : `stateManager` `device`, `deviceByExternalId`, etc. (`device.add.js:30`).
10. Démarrage du *poll* si `should_poll`.
11. `notify` : émet `EVENTS.DEVICE.CREATE` ou `UPDATE` et appelle `postCreate`/`postUpdate` du service propriétaire (`:333-336`).

**Route HTTP** : `POST /api/v1/device` (`authenticated`, **pas** `admin`) (`server/api/routes.js:255-258`). C'est la route qu'utilisent les pages de découverte.

**Pièces** :

- `gladys.room.create(houseSelector, { name })` (`server/lib/room/room.create.js`) ;
- `POST /api/v1/house/:house_selector/room` (`admin: true`, `routes.js:583-587`) ;
- maisons : `POST /api/v1/house` (admin).

**Idempotence.** L'idempotence native de Gladys repose sur l'**`external_id`**, unique et clé d'upsert. Aucune colonne ne trace l'origine d'un objet : il n'y a pas de champ « créé par » ni de métadonnée libre sur device, room ou scene.

---

## 5. Catalogue des catégories, types et unités de features

**Source unique** : `server/utils/constants.js`.

| Constante | Lignes | Contenu |
|---|---|---|
| `DEVICE_FEATURE_CATEGORIES` | `:1004` | environ 90 catégories (clé MAJUSCULE → valeur kebab-case) |
| `DEVICE_FEATURE_TYPES` | `:1126` | types par catégorie, environ 300 valeurs |
| `DEVICE_FEATURE_UNITS` | `:1705` | environ 100 unités |
| `DEVICE_FEATURE_UNITS_BY_CATEGORY` | `:1835` | unités admises par catégorie |
| `DEVICE_FEATURE_UNITS_BY_CATEGORY_AND_TYPE` | `:2099` | unités admises par catégorie et type |
| `COVER_STATE`, `AC_MODE`, `BUTTON_STATUS` | `:117`, `:156`, `:6` | énumérations de valeurs |
| `DEVICE_POLL_FREQUENCIES` | `:2178` | fréquences de *poll* |

Les colonnes `category`, `type` et `unit` sont des **ENUM Sequelize** construits sur ces listes (`device_feature.js:47-70`).

**Catégories principales**, par familles :

| Famille | Catégories |
|---|---|
| Actionneurs | `light`, `switch`, `shutter`, `curtain`, `lock`, `thermostat`, `air-conditioning`, `heater`, `fan`, `water-valve`, `water-heater`, `siren`, `vacuum-cleaner`, `television`, `music`, `camera`, `input`, `text`, `button` |
| Capteurs | `temperature-sensor`, `humidity-sensor`, `motion-sensor`, `opening-sensor`, `presence-sensor`, `light-sensor`, `leak-sensor`, `smoke-sensor`, `co-sensor`, `co2-sensor`, `pm25-sensor`, `voc-sensor`, `pressure-sensor`, `noise-sensor`, `vibration-sensor`, `uv-sensor`, `distance-sensor`, `soil-moisture-sensor`, `precipitation-sensor`… |
| Énergie | `energy-sensor`, `energy-production-sensor`, `teleinformation`, `battery`, `battery-low`, `battery-storage`, `charging-station`, `electrical-vehicle-*`, `grid-sensor` |
| Divers | `signal`, `duration`, `data`, `datarate`, `counter-sensor`, `tamper`, `maintenance`, `unknown` |

**Exemples de types** (`constants.js:1126-1160`) :

- `LIGHT` : `binary`, `brightness`, `hue`, `saturation`, `color`, `temperature`, `power`, `effect-mode`, `effect-speed` ;
- `SENSOR` (types génériques) : `decimal`, `integer`, `binary`, `push`, `unknown` ;
- `SWITCH` : `binary`, `power`, `energy`, `voltage`, `current`, `burglar`, `dimmer`, `target-current`.

**Unités (extrait)** : `celsius`, `fahrenheit`, `percent`, `lux`, `ppm`, `watt`, `kilowatt-hour`, `volt`, `ampere`, `hPa`, `mm`, `degree`, `seconds`…

**Gouvernance.** Tout ajout de catégorie ou de type doit passer la checklist de `docs/specs/device-feature-categories.md` (`AGENTS.md`, `.github/CONTRIBUTING.md`). Les catégories décrivent des capacités, jamais des marques, et s'alignent d'abord sur Matter, puis sur Zigbee.

**Table de correspondance HA déjà présente dans Gladys.** Le service MQTT contient une table **`device_class` HA → catégorie et type Gladys**, pour le protocole de *discovery* MQTT de HA :

- `SENSOR_DEVICE_CLASSES` : `server/services/mqtt/lib/homeAssistant/constants.js:182` ;
- `BINARY_SENSOR_DEVICE_CLASSES` (avec un drapeau `inverted`, car HA « on » sur une porte = ouvert, alors que Gladys 1 = fermé) : `:229` ;
- unités HA : `UNITS`, `:265`.

*Analyse* : c'est un point de départ direct pour notre catalogue de correspondances.

---

## 6. Moteur de scènes

### 6.1 Schéma d'une scène

**Validation.** Le schéma est validé par Joi dans `server/models/scene.js`.

- `actions` : `Joi.array().items(Joi.array().items(actionSchema))` (`:196`).
- `triggers` : `Joi.array().items(triggerSchema)` (`:198-341`).
- Les objets action et trigger n'acceptent **que les clés déclarées**. Il n'y a pas de `.unknown()` global.

**Sémantique de `actions`** (`server/lib/scene/scene.executeActions.js:78-80`) :

- le 1er niveau est exécuté **en série** (`Promise.mapSeries`) ;
- le 2ᵉ niveau est exécuté **en parallèle** (`Promise.map`).

**Sémantique de `triggers`** : **OU**. Chaque déclencheur actif qui correspond lance la scène (`server/lib/scene/scene.checkTrigger.js:17-45`). Seules les scènes `active` sont évaluées (`:21-25`) et planifiées (`scene.addScene.js:157-158`).

**Exécution** (`server/lib/scene/scene.execute.js`) :

- chaque exécution est poussée dans une file `queue({ autostart: true })` (`server/lib/scene/index.js:60-62`) ;
- **aucune concurrence n'est fixée** : la valeur par défaut du paquet `queue` serait illimitée (**non vérifié**, `node_modules` absent) ;
- *analyse* : rien n'empêche donc deux exécutions simultanées d'une même scène ;
- on peut arrêter une exécution (`AbortController` ; route `POST /api/v1/scene/:selector/stop`).

**Erreurs** (`scene.executeActions.js:40-50`) :

- `AbortScene` interrompt la scène (conditions non vérifiées, valeur invalide…) ;
- **toute autre erreur d'action est journalisée puis ignorée, et la scène continue** ;
- les actions `light.*` attrapent même l'erreur appareil par appareil (`scene.actions.js:180-223`).

### 6.2 Déclencheurs

**Types admis par Joi** : toute valeur de `EVENT_LIST` (`constants.js:2686`).

**Types effectivement évalués** : ceux de `server/lib/scene/scene.triggers.js:80-267`, plus ceux planifiés par `scene.addScene.js` et `scene.dailyUpdate.js`. Ce sont aussi ceux exposés par l'éditeur (`front/src/routes/scene/edit-scene/typesCatalog.js:53-77`).

| Type | Paramètres | Notes |
|---|---|---|
| `device.new-state` | `device_feature` (ancien format) ou `device_features[]` (OU), `operator` (`=`, `!=`, `>`, `>=`, `<`, `<=`, `changed`), `value`, `threshold_only` (bool), `for_duration` (ms) | `changed` interdit `value`, `threshold_only` et `for_duration` (`scene.js:297-303`). `threshold_only` : ne déclenche qu'au franchissement. `for_duration` : minuterie, annulée si la condition cesse (`scene.triggers.js:81-172`). |
| `time.changed` + `scheduler_type` | `every-day` (`time` HH:mm), `every-week` (`days_of_the_week[]`, `time`), `every-month` (`day_of_the_month`, `time`), `custom-time` (`date` YYYY-MM-DD, `time`), `interval` (`interval`, `unit` = `second`, `minute` ou `hour`), `time-range` (`time_ranges[{start,end}]`, `days_of_the_week`, `resume_on_startup`) | `scene.addScene.js:162-235`. Fuseau horaire de l'instance. Spec : `docs/specs/scene-time-ranges.md`. |
| `time.sunrise`, `time.sunset` | `house` (sélecteur), `offset` (minutes, de -1440 à 1440) | La maison doit avoir une latitude et une longitude (`scene.dailyUpdate.js:27-50`). |
| `user.back-home`, `user.left-home` | `house`, `user` | présence |
| `house.empty`, `house.no-longer-empty` | `house` | — |
| `area.user-entered`, `area.user-left` | `user`, `area` | zones géographiques |
| `calendar.event-is-coming` | `calendars[]`, `calendar_event_attribute` (`start` ou `end`), `calendar_event_name_comparator`, `calendar_event_name`, `duration`… | évalué par `scene.checkCalendarTriggers.js` |
| `alarm.arm`, `alarm.arming`, `alarm.disarm`, `alarm.partial-arm`, `alarm.panic`, `alarm.too-many-codes-tests` | `house` | — |
| `system.start` | — | au démarrage de Gladys |
| `mqtt.received` | `topic`, `message` (optionnel) | égalité stricte de topic et de message (`scene.triggers.js:250-251`) |
| `weather.alert-raised`, `weather.alert-ended` | `house`, `weather_alert_type`, `weather_alert_severity` | — |
| `energy-contract.price-changed` | `electric_meter` | — |
| `external-integration.scene-event` | `integration`, `trigger_key`, `fields` | déclencheurs déclarés par une intégration externe |

### 6.3 Actions

Liste exhaustive : `ACTIONS` (`constants.js:865-950`). Implémentations : `server/lib/scene/scene.actions.js`.

| Type | Paramètres principaux |
|---|---|
| `light.turn-on`, `light.turn-off`, `light.toggle` | `devices[]` (sélecteurs d'**appareils**). Agit sur la feature `light/binary`. |
| `light.blink` | `devices[]`, `blinking_time`, `blinking_speed` |
| `switch.turn-on`, `switch.turn-off`, `switch.toggle` | `device_features[]` (features `switch/binary` non `read_only`) (`scene.actions.js:80-120`) |
| `device.set-value` | `device_feature`, `value` **ou** `evaluate_value` (formule Handlebars + mathjs). Une feature `text` reçoit une chaîne. |
| `device.get-value` | `device_feature`. Écrit la feature dans le *scope*, à son chemin (`{{i.j.last_value}}`). |
| `delay` | `value` ou `evaluate_value`, `unit` (`milliseconds`, `seconds`, `minutes`, `hours`). Interruptible. |
| `scene.start` | `scene` |
| `message.send` | `user`, `text`, `service` (canal, null = tous) |
| `message.send-camera` | `user`, `camera`, `text` |
| `http.request` | `method`, `url`, `body`, `headers[]`, `request_response_keys` |
| `mqtt.send` | `topic`, `message` |
| `zigbee2mqtt.send` | `topic`, `message` (`scene.actions.js:972`) |
| `music.play-notification` | `device`, `text`, `volume` ou `evaluate_volume` |
| `music.play-media` | `device`, `media{id,type,title}`, `mode` |
| `user.set-seen-at-home`, `user.set-out-of-home` | `house`, `user` |
| `alarm.set-alarm-mode` | `house`, `alarm_mode` |
| `thermostat.set-schedule` | `schedule` |
| `variable.set` | `name`, `text` **ou** `evaluate_value` |
| `time.get-date` | `precision` |
| `ai.ask` | `text` |
| `sms.send` | marquée dépréciée dans le front (`typesCatalog.js:50`) |
| `external-integration.scene-action` | `integration`, `action_key`, `fields` |

### 6.4 Conditions

Il n'y a pas de bloc « conditions » séparé : **les conditions sont des actions** (`CONDITION_ACTIONS`, `constants.js:953-965`) qui lèvent `AbortScene` si elles échouent.

| Type | Sémantique |
|---|---|
| `condition.only-continue-if` | `conditions[]`, chacune avec `variable` **ou** `device_feature` (XOR, `scene.js:129-140`), `operator` (`=`, `!=`, `>`, `>=`, `<`, `<=`) et `value` ou `evaluate_value`. **OU entre les conditions d'une même action** : il suffit d'une seule vraie (`scene.actions.js:516-578`). Pour un **ET**, on enchaîne plusieurs actions en série. Un état inconnu (null) ne vérifie aucune condition. |
| `condition.check-time` | `before` et/ou `after` (HH:mm), `days_of_the_week`. ET, ou OU si la plage passe minuit (`:581-646`). |
| `condition.in-time-range` | `in_range` (bool). Utilise les déclencheurs `time-range` de la scène. |
| `condition.if-then-else` | `if[]` (actions de condition, **toutes** doivent passer = ET, `:1191-1210`), `then[][]`, `else[][]` |
| `condition.while` | `if[]`, `then[][]`, `max_iterations` (10 000 au maximum) |
| `house.is-empty`, `house.is-not-empty`, `user.check-presence`, `alarm.check-alarm-mode`, `calendar.is-event-running`, `calendar.get-events`, `ecowatt.condition`, `edf-tempo.condition`, `energy-contract.current-price` | conditions spécialisées |

### 6.5 Créer, activer et désactiver une scène

| Opération | Lib | API |
|---|---|---|
| Créer | `gladys.scene.create(scene)`. Insertion en base avec les `tags`, puis `addScene` (planification) (`scene.create.js`). | `POST /api/v1/scene` (**admin**) (`routes.js:1093-1097`) |
| Activer ou désactiver | `scene.update` avec le champ `active`. `addScene` annule ou replanifie les déclencheurs (`scene.addScene.js:155-158`). | `PATCH /api/v1/scene/:selector` (admin) |
| Lister, lire | — | `GET /api/v1/scene`, `GET /api/v1/scene/:selector` |
| Dupliquer | — | `POST .../duplicate` |
| Supprimer | — | `DELETE` (admin) |

**Précédent utile : le service MCP crée des scènes.**

- Outil `scene.create` dans `services/mcp/lib/buildSchemas.js:569-610`.
- Il les **étiquette** avec le tag `AI` (`AI_GENERATED_SCENE_TAG`, `constants.js:2669`).
- Il les valide d'abord par un **schéma Zod complet des déclencheurs et actions** (`services/mcp/lib/sceneSchemas.js`), qui est un miroir des schémas Joi.

---

## 7. Services pertinents pour le catalogue V1

**Constat transversal majeur : de nombreuses intégrations natives sont dépréciées.**

- `docs/specs/device-migration.md`, *Context*, liste les intégrations natives dépréciées au profit d'**intégrations externes** (communautaires, en conteneurs Docker) : Netatmo, MELCloud, Tuya, **Philips Hue**, TP-Link, **Zwave JS UI**, Sonos, **Tasmota**, Google Cast, eWeLink, Broadlink, Xiaomi, Nuki.
- Ces intégrations ont `"deprecated": true` dans `front/src/config/integrations/devices.json`.
- Dans le catalogue V1, seuls **zigbee2mqtt, mqtt et matter** restent natifs et non dépréciés.

### 7.1 Tableau de synthèse

| Service | Statut | Découverte et création | Format de l'`external_id` (appareil / feature) | Pour « pré-créer » un appareil importé |
|---|---|---|---|---|
| `zigbee2mqtt` | natif, actif | **Découverte auto** via le topic `zigbee2mqtt/bridge/devices`. L'utilisateur crée depuis l'onglet Découverte (`GET /api/v1/service/zigbee2mqtt/discovered` → `POST /api/v1/device`). Deux modes : Gladys installe Z2M et Mosquitto en Docker, ou **connexion à une installation Z2M existante** via un broker externe (`MQTT_MODE.EXTERNAL`, `lib/constants.js:28-31` ; i18n `setup.remote`). | `zigbee2mqtt:<friendly_name>` (`utils/convertDevice.js:20`) / `zigbee2mqtt:<friendly_name>:<category>:<type>:<property>[:<index>]` (`utils/features/completeFeature.js:16`). Param `IEEE_ADDRESS` pour suivre les renommages (`convertDevice.js:26-31`, `lib/syncRenamedDevices.js`). | Il faut que Z2M soit connecté à Gladys (découvertes en mémoire, `this.discoveredDevices`). La correspondance avec HA est possible par **adresse IEEE**. Les messages MQTT sont routés vers `deviceByExternalId` (`lib/handleMqttMessage.js:~170`). Un appareil créé avec le bon `external_id` reçoit donc ses états. |
| `mqtt` | natif, actif | (a) Appareils **saisis à la main** (« MQTT Virtual Devices », `external_id` commençant par `mqtt:`, `front/.../mqtt/device-page/setup/index.js:65`). (b) **Découverte HA MQTT Discovery** (`homeassistant/#`) : composants `sensor`, `binary_sensor`, `switch`, `light`, `cover`, `lock`, `climate`, `button` (`lib/homeAssistant/constants.js:27`), route `GET /api/v1/service/mqtt/discovery`. | (a) libre, préfixe `mqtt:`. Topics `gladys/master/device/:device_external_id/feature/:feature_external_id/...` et `gladys/device/<ext>/feature/<ext>/state` (`lib/setValue.js:29`). (b) `homeassistant:<identifier>` / `homeassistant:<identifier>:<component>:<object_id>[:<property>]`. Params `ha_discovery_config:<entityKey>` contenant la config JSON (`convertToGladysDevice.js:394-440`). | (a) Toujours possible : création pure dans Gladys, mais le firmware doit publier sur les topics Gladys. (b) Dépend de la publication des messages de *discovery* sur le broker auquel Gladys est connecté. |
| `zwavejs-ui` | natif, **déprécié** (`devices.json`, `DeprecationWarning` dans `ZwaveJSUIPage.jsx:48`) | Connexion **MQTT** à Z-Wave JS UI (`MQTT_URL` et identifiants, `lib/zwaveJSUI.connect.js:11-14`), puis découverte (`POST /api/v1/service/zwavejs-ui/discover`). | `zwavejs-ui:<nodeId>` / `zwavejs-ui:<nodeId>:<endpoint>:<commandClass>:<property>[:<propertyKey>][:<feature>]` (`utils/convertToGladysDevice.js:8-13`, `:139`) | Il faut que Z-Wave JS UI publie en MQTT. Remplaçant externe : **non vérifié** (aucun nom donné dans `device-migration.md`). |
| `philips-hue` | natif, **déprécié**. Remplaçant communautaire « Philips Hue » (`integration-catalog-categories.md` § 7). | Appairage du pont (`POST /api/v1/service/philips-hue/bridge/configure`), puis synchronisation et liste des lampes (`node-hue-api` 4). | `philips-hue-light:<bridgeSerial>:<lightId>` / `...:<lightId>:<type>` (`lib/models/*.js`, `lib/utils/consts.js:10`) | Il faut le numéro de série du pont et l'identifiant de lampe du pont. Correspondance de ces identifiants avec ceux de HA : **non vérifié**. |
| `tasmota` | natif, **déprécié**. Remplaçant : [Gladys-Tasmota](https://github.com/gboulvin/Gladys-Tasmota) (MQTT et HTTP, appareils indexés par adresse MAC) (`device-migration.md`). | Scan MQTT (`stat/+/STATUS…`) ou HTTP (`GET` et `POST /api/v1/service/tasmota/discover/:protocol`) | MQTT `tasmota:<topic>`, HTTP `tasmota:<ip>` (`lib/mqtt/tasmota.mqtt.status.js:17`, `lib/http/tasmota.http.status.js:8`) / `tasmota:<id>:<featureId>[n]` | MQTT : il faut le topic Tasmota. HTTP : il faut l'IP. Alternative : les Tasmota publiant la *discovery* HA sur MQTT sont gérés par le service `mqtt` (b), d'après un commentaire de `server/services/mqtt/lib/homeAssistant/convertToGladysDevice.js:52`. |
| `matter` | natif, actif. 26 clusters gérés sur 132 (`server/services/matter/README.md`). | **Appairage par code** (`POST /api/v1/service/matter/pair-device`, `ManualPairingCodeCodec`, `lib/matter.pairDevice.js:12-35`). Gladys est un contrôleur Matter. | `matter:<nodeId>:<devicePath>` / `...:<clusterIndex>[:click]` (`utils/convertToGladysDevice.js:105-175`) | **Impossible à pré-créer** : le `nodeId` est attribué au *commissioning*. En coexistence, il faut un code d'appairage multi-admin généré côté HA (**non vérifié** côté HA). |
| `matterbridge` | natif, actif | Le service **n'expose aucun appareil**. Il installe et gère un conteneur `luligu/matterbridge:latest` (`services/matterbridge/docker/*.json`) avec une interface web. | — | *Rôle* : Matterbridge sert de pont qui expose d'autres écosystèmes en Matter. Gladys appaire ensuite ces appareils via le service `matter`. Le fonctionnement par plugins de Matterbridge n'est pas visible dans le code Gladys (**non vérifié**). |

### 7.2 Shelly

- **Il n'y a pas de service Shelly natif** (`server/services/index.js`).
- Shelly est une **intégration communautaire externe**, catégories `lighting` et `energy` (`docs/specs/integration-catalog-categories.md:274`).
- Les specs d'intégrations externes citent Shelly comme cas de LAN local (`external-integrations/core/supervisor.md:30`, `verification.md:10`), et le Shelly Pro 3EM comme exemple de charge, avec 24 features (`server/api/middlewares/jsonBodyMiddleware.js:14`).
- Autres voies possibles : MQTT (*discovery* HA si le firmware Shelly la publie, **non vérifié**) et Matter (Shelly Gen3/Gen4, **non vérifié**).

### 7.3 Intégrations externes : ce qui change pour nous

D'après `docs/specs/external-integrations/README.md`, *Scoping decisions* :

- **« No device creation by the integration »** : l'intégration publie ses découvertes (`POST /api/integration/v1/discovered_device`), puis l'utilisateur crée l'appareil depuis l'UI générique (Devices / Discovery / Configuration).
- `external_id` préfixé `ext:<integration>:...` (`contracts/host-api-endpoints.md:20-44`).
- L'unicité de l'`external_id` relève de l'intégration ; celle du sélecteur relève du cœur (`host-api-endpoints.md:44`).
- Une intégration externe est une ligne de `t_service` avec un `type` (`external-integrations/core/data-model.md`).
- *Analyse* : pré-créer un appareil d'intégration externe n'est possible que si l'on connaît exactement l'`external_id` que l'intégration publiera. C'est propre à chaque intégration et c'est **hors de notre contrôle**.

---

## 8. Éléments réutilisables pour un import

| Besoin | Existant dans Gladys | Référence |
|---|---|---|
| Import/export d'appareils ou de scènes | **Aucun.** Il n'y a pas de route d'import/export générique. Seuls existent l'export CSV d'états (`lib/device/device.exportStatesToCsv.js`) et la duplication de scène. | `server/api/routes.js` |
| Migration d'un appareil vers un autre | **`device.migrate`** : déplace l'historique DuckDB, réécrit les scènes et dashboards, hérite de la pièce, supprime la source. `POST /api/v1/device/:device_selector/migrate` (admin, 409 si un verrou est actif, job `device-migrate`). | `server/lib/device/device.migrate.js`, `docs/specs/device-migration.md` |
| Sauvegarde Gladys | Sauvegarde et restauration Gladys Plus : `tar` + `gzip` via `execFile`, vérifications `utils/backupSafety`. Routes `/api/v1/gateway/backup*`. | `server/lib/gateway/gateway.backup.js`, `gateway.downloadBackup.js`, `gateway.restoreBackup.js` |
| Téléversement de fichier | **Pas de multer, busboy ni formidable** (aucune occurrence). Mécanismes de corps existants : JSON de 100 ko (global), JSON de 6 Mo pour les seuls assets de dashboard (base64), brut `audio/*` ou `octet-stream` de 5 Mo (STT). | `server/api/middlewares/jsonBodyMiddleware.js`, `audioRawBodyMiddleware.js` |
| Tâches longues avec progression | **`gladys.job`** : `start(type)`, `updateProgress(id, %, dataPatch)` (diffusé en WebSocket `job.updated`), `finish`, et `wrapper(type, fn)`. Visibles dans *Réglages → Tâches de fond* (`/dashboard/settings/jobs`). Un nouveau type demande une entrée `JOB_TYPES` (`constants.js:2528`), ses clés `data` dans `models/job.js` et ses libellés i18n. | `server/lib/job/*`, `front/src/components/app.jsx:473` |
| Progression spécifique à un service | Exemple : `zigbee2mqtt.install-progress` (WebSocket) + `status.installation`. | `docs/specs/zigbee2mqtt-installation.md` |
| Correspondance HA → Gladys | Tables `device_class` → catégorie et unités du service MQTT (*discovery* HA). | `server/services/mqtt/lib/homeAssistant/constants.js` |
| Validation de scènes | Joi (`models/scene.js`) et Zod (`services/mcp/lib/sceneSchemas.js`) | — |
| Clients HTTP et WebSocket | `axios`, `ws` (serveur), `gladys.http` (`lib/http/http.request.js`) | `server/package.json` |
| Données sur disque | Le chemin se dérive de `path.dirname(gladys.config.storage)`. Ne pas créer de nouveau dossier racine. | `AGENTS.md` (« Development data folder ») |

**Le service `mcp`** (`server/services/mcp/`) :

- c'est un **serveur MCP** (Model Context Protocol, `@modelcontextprotocol/sdk`) qui expose la maison à des agents IA du réseau local ;
- route : `POST /api/v1/service/mcp/proxy` ;
- i18n : « Connect your AI agent to Gladys », `en.json:3843-3848` ;
- outils : `device.get-state`, `device.turn-on-off`, `device.set-light`, `device.set-shutter`, `scene.create`, `scene.start`, `camera.get-image`, `weather.get`… (`buildSchemas.js`) ;
- il ne sert pas à l'import, mais ses schémas Zod de scènes et sa création de scènes avec étiquette constituent un précédent réutilisable.

---

## 9. Points d'attention pour notre service

### 9.1 Contraintes structurantes

1. **Spec et processus.**
   - Le projet touche au modèle de données et c'est un « large project » : il faut une **spec dans `docs/specs/`**, avant le code et dans le même dépôt.
   - Toutes les specs existantes sont **en anglais**, et les PR doivent l'être (`AGENTS.md`). Cela contredit la décision D6 de la vision (specs en français), au moins pour la version à verser dans le dépôt Gladys.
   - CONTRIBUTING recommande une discussion préalable sur le forum.
2. **Couverture de patch à 100 %** sur tout le code serveur ajouté. Cela impose de concevoir le service pour qu'il soit testable : clients HA injectables, et pas d'E/S non *mockables*.
3. **Pas d'i18n serveur.** Le rapport de compatibilité doit être produit sous forme de **codes et faits structurés**, traduits par le front en **4 langues** (en, fr, de, es), avec `compare-translations` en CI.
4. **Téléversement d'une sauvegarde HA.**
   - Ce n'est pas possible avec l'infrastructure actuelle : parseur JSON global de 100 ko, pas de multipart.
   - Il faudra une modification du cœur (nouveau middleware de corps brut ou en flux, avec une exemption du parseur global) ou un autre canal.
   - Il n'y a pas de bibliothèque tar ni YAML côté serveur. L'appel au binaire `tar`, comme pour les sauvegardes, est le précédent existant.
5. **Droits.** Création de pièces, de maisons et de scènes, et migration d'appareils : `admin: true`. Les routes de notre service doivent l'être aussi.

### 9.2 Pièges du modèle de données

6. **Nom de pièce unique sur toute l'instance et limité à 40 caractères** (`models/room.js:20-26`). Deux areas HA homonymes, ou une area au nom long, échoueront. Il faudra une stratégie de dédoublonnage ou de troncature.
7. **`device.create` est un upsert destructif.**
   - Si l'`external_id` existe, les features et params absents du payload sont **supprimés**.
   - Pré-créer un appareil avec un jeu de features partiel ou différent de ce que le service propriétaire publiera fera disparaître des features à la prochaine sauvegarde, ou l'inverse.
   - *Analyse* : il faut **réutiliser la sortie de découverte du service cible** (`getDiscoveredDevices`, *discovery* HA du service MQTT…) plutôt que forger des `external_id`.
8. **Les `external_id` et `selector` des features sont uniques sur toute l'instance.** Une collision (appareil déjà créé à la main) donne un 409 ou, pire, un upsert sur l'appareil existant.
9. **Aucune métadonnée d'origine** sur device, room ou scene.
   - Pour l'idempotence et la réversibilité (principes 5 et 6), il faut tracer ce qu'on a créé : variables du service, ou param d'appareil, ou tag de scène (précédent : tag `AI`).
   - Risque : un param ajouté par nous peut être supprimé par un upsert du service propriétaire qui ne le renvoie pas (comportement de `mergeDevices` côté front et serveur **non vérifié**).
10. **`min` et `max` requis** pour toute feature, `unit` limitée à l'ENUM, et `category`/`type` limités aux ENUM. Une valeur hors liste entraîne un échec de validation Sequelize (422).

### 9.3 Pièges du moteur de scènes (lot 2)

11. **Sémantique booléenne inversée par rapport à HA.**
    - Les conditions d'une même action `only-continue-if` sont en **OU**. HA met ses conditions en ET par défaut : il faut **une action par condition**, en série.
    - Les `if` d'un `if-then-else` sont en ET.
    - Les déclencheurs sont en OU, comme dans HA.
12. **Erreurs d'action non fatales.** Une action en erreur (hors `AbortScene`) est ignorée et la scène continue (`scene.executeActions.js:40-50`). HA arrête l'exécution par défaut (**comportement HA non vérifié ici**). Cela peut violer le critère « aucune scène ne se comporte différemment ».
13. **Pas de modes d'exécution** (`single`, `restart`, `queued`, `parallel` dans HA). La file Gladys n'impose pas de limite de concurrence (**non vérifié**, voir § 6.1).
14. **Paramètres absents de Gladys.**
    - Pas de déclencheur sur **attribut** d'entité ni sur **template**.
    - Pas de `for` sur `time`/`sun`.
    - `mqtt.received` ne fait qu'une égalité stricte, sans *wildcard* vérifiée (**non vérifié**).
    - Pas d'action « régler la luminosité sur plusieurs appareils » : il faut une action `device.set-value` par feature.
15. **Références par sélecteur.** Les scènes référencent des sélecteurs d'appareils ou de features, pas des UUID. L'import des scènes doit donc suivre celui des appareils et réutiliser leurs sélecteurs réels, qui peuvent être suffixés `-2`… par `buildUniqueSelector`.
16. **Clés Joi strictes.** Toute clé non déclarée est refusée. Exemple : `feature_category` est lu par `device.set-value` (`scene.actions.js:127-129`) mais absent du schéma, donc non persistable. Une scène doit donc toujours utiliser `device_feature`.
17. **`icon` obligatoire**, tiré de `server/config/icons.json`. Un précédent existe dans le front : `front/src/routes/scene/guessSceneIcon.js`.

### 9.4 Écosystème

18. **Philips Hue, Tasmota et Z-Wave JS UI sont dépréciés en natif.** Pour le catalogue V1 :
    - le statut ✅ « Migrable » vers ces services natifs pousserait l'utilisateur vers une intégration en fin de vie ;
    - les remplaçants sont des **intégrations externes** sur lesquelles notre service n'a pas de prise : pas de création d'appareil par API d'intégration, et `external_id` propres à chaque intégration.
19. **Shelly** n'existe qu'en intégration externe communautaire (ou via MQTT/Matter, **non vérifié**).
20. **Matter** : pas de pré-création possible (`nodeId` attribué à l'appairage).
21. **Le service MQTT sait déjà ingérer la *discovery* MQTT de HA** (`homeassistant/#`). C'est une voie « 🔀 via alternative » très directe pour tout appareil qui publie cette *discovery* (Tasmota, ESPHome, Z2M avec `homeassistant: true`…). La liste des composants pris en charge est limitée (§ 7.1).
22. **Coexistence Z2M.** Le mode « connexion à une installation existante » de zigbee2mqtt permet à Gladys de se brancher sur le Z2M et le broker de HA, sans réappairage. C'est conforme à la vision, § 9.
23. **`device.migrate`** offre une voie de **bascule ultérieure** : un appareil importé (ex. Tasmota natif) peut être migré plus tard vers une intégration externe en conservant l'historique et les références de scènes. *Analyse* : cela peut servir d'argument pour importer vers des services natifs dépréciés malgré tout, à arbitrer.
