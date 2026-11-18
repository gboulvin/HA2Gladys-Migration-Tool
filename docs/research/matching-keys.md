# Note de recherche : clés de rapprochement HA → Gladys

- **Date** : 2026-11-17
- **Objet** : établir, pour chaque intégration du périmètre, la règle factuelle qui relie un appareil du registre HA à l'appareil **découvert** par l'intégration Gladys cible, puis les entités HA aux *features* Gladys (import d'historique). Sert l'architecture § 6 (D16) et la future section 03.
- **Convention** : chaque affirmation est sourcée `[dépôt] fichier:ligne`. Sans source : **non vérifié**.

## 0. Versions étudiées

| Dépôt (alias) | Commit / version | Date du commit |
|---|---|---|
| [gladys] GladysAssistant/Gladys | `71edb178` (package 5.3.3) | 2026-11-17 |
| [core] home-assistant/core | `9b5a3964` (2026.12.0.dev0) | 2026-11-17 |
| [z2m] Koenkk/zigbee2mqtt | `3c841fdf` | 2026-11-01 |
| [zha] zigpy/zha (bibliothèque) | `2da1a5b5` (`main` ; HA épingle `zha==2.4.3`, [core] `components/zha/manifest.json:28`) | 2026-11-07 |
| [zigpy] zigpy/zigpy | `64b51efe` | 2026-11-07 |
| [aioesphomeapi] esphome/aioesphomeapi | `e3447e7b` (tag v46.30.1 ; HA épingle 46.30.0, [core] `components/esphome/manifest.json:22`) | 2026-11-16 |
| [shelly] Terdious/gladys-shelly | `d810bacc` (manifeste 1.4.1) | 2026-10-19 |
| [cast] gboulvin/Gladys-Google-Cast | `e22a310e` (manifeste 1.0.0) | 2026-11-16 |
| [esphome] William-De71/gladys-esphome | `3515d872` (manifeste 1.4.0) | 2026-11-17 |
| [tuya] Terdious/gladys-tuya | `84035ccd` (manifeste 1.32.0) | 2026-11-03 |
| [upnp] callemand/gladys-upnp-igd | `0d241987` (manifeste 1.1.0) | 2026-10-20 |
| [ipp] guim31/gladys-ipp | `342b4d3e` (manifeste 1.3.2) | 2026-10-29 |
| [dlna] guim31/gladys-dlna | `ee9e36e6` (manifeste 1.2.0) | 2026-11-17 |
| [met] guim31/gladys-met-no | `1ae50bd2` (manifeste 1.0.2) | 2026-11-17 |

Les écarts entre `main` et la version épinglée par HA (zha, aioesphomeapi) n'ont pas été vérifiés.

---

## 1. Synthèse

| Intégration HA | Cible Gladys | Clé HA | Clé Gladys | Normalisation | Confiance appareil | Confiance entités |
|---|---|---|---|---|---|---|
| `zha` | zigbee2mqtt (natif) | connexion `zigbee` (ou identifiant `zha`) | param `IEEE_ADDRESS` | IEEE : retirer `:` et `0x`, minuscules | **sûre** | probable (table cluster → propriété Z2M à écrire) |
| `mqtt` (Zigbee2MQTT) | zigbee2mqtt (natif) | identifiant `mqtt` = `zigbee2mqtt_0x…` | param `IEEE_ADDRESS` | retirer le préfixe `zigbee2mqtt_` | **sûre** | **sûre** (propriété Z2M dans l'`unique_id`) |
| `mqtt` (discovery générique) | mqtt (natif, `lib/homeAssistant`) | identifiants `mqtt` (ou 1ʳᵉ connexion) | `external_id` `homeassistant:<id>` | assainissement Gladys `[^a-zA-Z0-9_-]` → `-` | probable (ordre des identifiants perdu) | **sûre** (`unique_id` conservé dans les params) |
| `matter` | matter (natif) | identifiant `matter` `serial_<n° série>` | param `SERIAL_NUMBER` | retirer `serial_` | probable (série absente ou « test » → manuel) | probable (endpoint + cluster) |
| `shelly` | Terdious/gladys-shelly | connexion `mac` | param `SHELLY_ID` (`<modèle>-<mac>`) | MAC sans séparateur, minuscules, suffixe de `SHELLY_ID` | **sûre** | probable (clé RPC identique) |
| `cast` | gboulvin/Gladys-Google-Cast | identifiant `cast` (UUID sans tirets) | suffixe de l'`external_id` `…:cast:<id>` | minuscules, alphanumérique | **sûre** (hors adresses manuelles) | sans objet (pas d'historique musical) |
| `esphome` | William-De71/gladys-esphome | `data.device_name` de l'entrée | suffixe de l'`external_id` `…:esphome:<nom>` | casse | probable (MAC non publiée côté Gladys) | probable (nom d'entité) |
| `tuya` | Terdious/gladys-tuya | identifiant `tuya` = id Tuya | param `DEVICE_ID` / suffixe `…:device:<id>` | aucune | **sûre** | **sûre** pour les DP codes |
| `upnp` | callemand/gladys-upnp-igd | connexion `upnp` (UDN) | param `udn` | retirer `uuid:`, minuscules | **sûre** | probable (table de 7 clés, conversion d'unité) |
| `ipp` | guim31/gladys-ipp | identifiant `ipp` (UUID ou n° de série) | suffixe `…:printer:<uuid>` | minuscules, retrait `urn:uuid:` | probable (repli sur l'hôte → manuel) | probable (nom du consommable) |
| `dlna_dmr` | guim31/gladys-dlna | connexion `upnp` (UDN) | param `UDN` | retirer `uuid:`, minuscules | **sûre** | sans objet (lecteur multimédia) |
| `met` | guim31/gladys-met-no | appareil `service` | aucun appareil (intégration `weather`) | — | **sans objet** : rien à rapprocher | sans objet ; configuration seulement |

Aucune intégration n'est entièrement « impossible → manuel », mais trois cas le deviennent localement : Matter sans numéro de série exploitable, Cast et IPP ajoutés par adresse manuelle côté Gladys (identifiant dérivé de l'hôte), et MQTT générique quand l'appareil n'a ni identifiant ni connexion.

---

## 2. Fondations communes

### 2.1 Format des `external_id` Gladys

- **Services natifs** : préfixe libre propre au service (`zigbee2mqtt:`, `homeassistant:`, `matter:`), voir chaque section.
- **Intégrations externes** : le serveur refuse tout `external_id` d'appareil ou de feature qui ne commence pas par `ext:<selector>:` ([gladys] `server/lib/external-integration/externalIntegration.setDiscoveredDevices.js:44,52-53,68-69` ; [gladys] `docs/specs/external-integrations/contracts/conventions.md:10`).
- Le `selector` vaut `ext-` + slug de `owner/repo` pour une installation depuis le store, `ext-dev-<nom>` en installation de développement ([gladys] `server/lib/external-integration/externalIntegration.buildSelector.js:17-28`, `constants.js:87-88`). Exemple : `ext:ext-terdious-gladys-shelly:…`.
- Le SDK construit `device : ext:<selector>:<type>:<id>` et `feature : ext:<selector>:<type>:<id>:<code>` (fabrique `gladys.externalIds(type, id)`), documenté dans [tuya] `src/tuya/utils/tuya.externalId.js:3-5` et [upnp] `src/devices/convertDevice.js:5-8`. Le SDK lui-même n'a pas été lu (**non vérifié** au-delà de ces commentaires concordants).

**Conséquence pour le catalogue** : il ne doit pas stocker le préfixe complet (il dépend du mode d'installation), mais le **segment `<type>`** et la position de l'identifiant. Le rapprocheur calcule le préfixe à partir du `selector` du service installé.

### 2.2 Normalisations nécessaires

| Nom | Règle | Utilisée par |
|---|---|---|
| `mac` | supprimer `:`, `-`, `.` ; minuscules → 12 hex | Shelly, (ESPHome, UPnP, DLNA en secours) |
| `ieee` | supprimer `:` et le préfixe `0x` ; minuscules → 16 hex | ZHA, Zigbee2MQTT |
| `upnp_udn` | supprimer le préfixe `uuid:` (insensible à la casse) ; minuscules | UPnP, DLNA |
| `strip_prefix(p)` | supprimer un préfixe connu (`zigbee2mqtt_`, `serial_`, `deviceid_`) | Z2M, Matter |
| `gladys_mqtt_sanitize` | remplacer `[^a-zA-Z0-9_-]` par `-` ([gladys] `server/services/mqtt/lib/homeAssistant/handleHomeAssistantDiscoveryMessage.js:35`) | MQTT générique |

Le registre HA normalise déjà les MAC en `aa:bb:cc:dd:ee:ff` dans `connections` (voir `ha-data-sources.md` § 2.1), mais **pas** les valeurs placées dans `identifiers` (Shelly y met la MAC brute de l'entrée, § 5.1).

### 2.3 Cardinalités à prévoir

- **N appareils HA → 1 appareil Gladys** : Shelly (appareils enfants par canal, [core] `components/shelly/utils.py:852-875`), Tuya (enfants `<id>_channel_<n>`, [core] `components/tuya/entity.py:52-57`), ESPHome (sous-appareils `<mac>_<device_id>`, [core] `components/esphome/manager.py:1424`). Les enfants n'ont pas de clé matérielle propre : ils héritent du rapprochement de leur parent (`parent_device_id` / `via_device_id`).
- **1 appareil HA → N appareils Gladys** : Matter (Gladys crée un appareil **par endpoint**, [gladys] `server/services/matter/lib/matter.handleNode.js:67-77`, `utils/convertToGladysDevice.js:105`).
- **Doublons HA** : depuis la scission des composites (2026.8), une même MAC/IEEE peut apparaître sur plusieurs appareils HA (`ha-data-sources.md` § 2.1). Regrouper avant de chercher le candidat.

---

## 3. Zigbee

### 3.1 ZHA → service natif zigbee2mqtt

**Côté HA**

- Registre : `connections={(CONNECTION_ZIGBEE, str(ieee))}`, `identifiers={("zha", str(ieee))}` ([core] `components/zha/helpers.py:1106-1115` ; mêmes valeurs dans `DeviceInfo`, [core] `components/zha/entity.py:139-148`).
- Format : `str(EUI64)` = octets en hexadécimal **minuscule**, séparés par `:`, du plus significatif au moins significatif (`__repr__` inverse l'ordre de stockage) ([zigpy] `zigpy/types/named.py:65-68` ; `FixedList` hérite de `list`, dont `__str__` appelle `__repr__`). Exemple : `00:15:8d:00:01:ab:cd:ef`.
- Pas `entry_type: service`, sauf les groupes ZHA (modèle `Group`, sans IEEE, `ha-data-sources.md` § 2.4).
- `unique_id` des entités : `"{ieee}-{endpoint}-{cluster_id}"` par défaut, suivi de `-{suffixe}` si l'entité en déclare un ([zha] `zha/application/platforms/__init__.py:710-719,562-563`). Groupes : `"{platform}_zha_group_0x{id:04x}"` (`:813`). Exemples de suffixes : `rms_voltage`, `apparent_power`, `total_active_power` ([zha] `zha/application/platforms/sensor/__init__.py:1297,1508,1229`).

**Côté Gladys**

- Appareil : `external_id = zigbee2mqtt:<friendly_name>` ; si l'IEEE est connu, `ieee_address` et `params = [{ name: 'IEEE_ADDRESS', value: ieee }]` ([gladys] `server/services/zigbee2mqtt/utils/convertDevice.js:17-31`, `lib/constants.js:34-38`). L'IEEE est la forme Z2M `0x00158d0001abcdef` (champ `ieee_address` de Z2M ; [z2m] `lib/model/device.ts:23-28`).
- L'`external_id` dépend du **nom** et change au renommage ; Gladys resynchronise via le param IEEE ([gladys] `server/services/zigbee2mqtt/lib/syncRenamedDevices.js:9,192-194`).
- Feature : `zigbee2mqtt:<friendly_name>:<category>:<type>:<propriété Z2M>[:<index>]` ([gladys] `server/services/zigbee2mqtt/utils/features/completeFeature.js:15-16`).

**Règle** : `connections[zigbee]` (ou `identifiers[zha]`) → `ieee` normalisé = param `IEEE_ADDRESS` normalisé. Confiance : **sûre**. L'IEEE est gravé dans la radio et survit à un nouvel appairage.

Points d'attention :
- Le réseau Zigbee appartient au coordinateur ZHA. Passer à Zigbee2MQTT suppose soit un réappairage de chaque appareil (l'IEEE ne change pas, donc le rapprochement reste sûr), soit une reprise du réseau (sauvegarde du coordinateur). La faisabilité d'une reprise automatique n'a pas été étudiée (**non vérifié**).
- Le coordinateur ZHA est un appareil HA avec IEEE ([core] `components/zha/__init__.py:264-265`) ; côté Gladys il est filtré (`type !== 'Coordinator'`, [gladys] `server/services/zigbee2mqtt/lib/syncRenamedDevices.js:192`). Il faut l'exclure du rapprochement.

**Entités → features** : il faut une table `cluster_id (+ suffixe, + plateforme)` → propriété Z2M. Exemples (identifiants de clusters ZCL standard : **non vérifié** dans le code, à confirmer par une fixture) :

| Entité HA (domaine, device_class) | `unique_id` HA (motif) | Propriété Z2M | Feature Gladys (category / type) | Source Gladys |
|---|---|---|---|---|
| `sensor`, `temperature` | `<ieee>-<ep>-1026` | `temperature` | TEMPERATURE_SENSOR / decimal, °C | [gladys] `zigbee2mqtt/exposes/numericType.js:447-455` |
| `sensor`, `humidity` | `<ieee>-<ep>-1029` | `humidity` | HUMIDITY_SENSOR / decimal | `numericType.js:313` |
| `sensor`, `battery` | `<ieee>-<ep>-1` | `battery` | BATTERY / integer | `numericType.js:133-137` |
| `binary_sensor`, `occupancy` | `<ieee>-<ep>-1030` | `occupancy` | MOTION_SENSOR / binary | `binaryType.js:29-34` |
| `light` / `switch` | `<ieee>-<ep>-6` | `state` (ou `state_<ep>`) | LIGHT / binary ou SWITCH / binary selon le type parent | `binaryType.js:77-93` |

Les capteurs IAS (cluster 1280) donnent `contact`, `water_leak` ou `occupancy` selon le type de zone : il faut lire le `device_class` HA (`door`/`window` → `contact`, `moisture` → `water_leak`). **Inversion** : Gladys stocke `contact` en valeur inversée (`reversedValue: true`, [gladys] `binaryType.js:10-15`) ; un historique HA « on = ouvert » doit être inversé à l'import.

**Préremplissage** : l'entrée ZHA contient `data.device.path`, `data.device.baudrate`, `data.device.flow_control` et `data.radio_type` ([core] `components/zha/config_flow.py:184-187`, `const.py:39-43`). Gladys attend `ZIGBEE2MQTT_DRIVER_PATH`, `ZIGBEE_DONGLE_NAME`, `Z2M_ADAPTER_MODE`, `Z2M_NETWORK_ADAPTER_URL`, `Z2M_NETWORK_ADAPTER_TYPE` ([gladys] `server/services/zigbee2mqtt/lib/constants.js:4-12`). `path` → `ZIGBEE2MQTT_DRIVER_PATH` est direct si le chemin USB est le même sur l'hôte Gladys (**non vérifié** : Gladys tourne dans Docker). `radio_type` (`ezsp`, `znp`…) → `ZIGBEE_DONGLE_NAME` exige une table de correspondance (**non vérifié**). Un chemin `socket://…` correspondrait au mode réseau. Le service zigbee2mqtt n'est pas une intégration externe : il n'a pas de `config_schema` de manifeste.

### 3.2 Zigbee2MQTT exposé à HA via MQTT → service natif zigbee2mqtt

**Côté HA** : l'appareil appartient à l'entrée `mqtt`. HA recopie `identifiers` et `connections` du payload de découverte : `identifiers={("mqtt", id) for id in …}`, `connections={(c[0], c[1]) …}` ([core] `components/mqtt/entity.py:1494-1506`). Ce n'est pas un appareil `service`.

**Ce que publie Zigbee2MQTT** ([z2m] `lib/extension/homeassistant.ts`) :

| Élément | Valeur | Source |
|---|---|---|
| `device.identifiers` (appareil) | `["zigbee2mqtt_<ieee>"]`, `ieee` au format `0x00158d0001abcdef` | `:2176-2186` ; `ID` = `ieeeAddr` ([z2m] `lib/model/device.ts:27-28`) |
| `device.identifiers` (groupe) | `["zigbee2mqtt_<base_topic encodé>_<id groupe>"]` | `:2177,2186` |
| `device.identifiers` (pont) | `["zigbee2mqtt_bridge_<ieee coordinateur>"]` | `:485,2186` |
| `device.connections` | **aucune** | `:2185-2228` |
| `via_device` | identifiant du pont | `:2226-2228` |
| `model` / `model_id` / `manufacturer` | `definition.description` / `definition.model` / `definition.vendor` | `:2198-2200` |
| `unique_id` des entités | `<ieee>_<object_id>_<base_topic>` | `:1853-1855` |
| `object_id` capteurs | nom de l'expose, suffixé `_<endpoint>` | `:1203,1277` |
| `object_id` lumière, interrupteur, volet, serrure, thermostat | `light`, `switch`, `cover`, `lock`, `climate` (suffixés `_<endpoint>`) | `:650,711,985,951,750` |

**Côté Gladys** : identique au § 3.1.

**Règle** : `identifiers[mqtt]` correspondant à `^zigbee2mqtt_(0x[0-9a-f]{16})$` → groupe 1 = param `IEEE_ADDRESS`. Confiance : **sûre**. Ignorer `zigbee2mqtt_bridge_*` et les groupes. Le `base_topic` lu dans le suffixe des `unique_id` doit valoir celui écouté par Gladys (`zigbee2mqtt/#` par défaut, [gladys] `server/services/zigbee2mqtt/lib/constants.js:168,177`).

**Entités → features** : l'`unique_id` contient directement la propriété Z2M, qui est le dernier segment de l'`external_id` de la feature Gladys. On extrait `object_id`, on retire le suffixe `_<base_topic>`, puis :

| Entité HA | `unique_id` | Feature Gladys |
|---|---|---|
| `sensor.salon_temperature` | `0x00158d0001abcdef_temperature_zigbee2mqtt` | `zigbee2mqtt:Salon:temperature-sensor:decimal:temperature` |
| `sensor.…_humidity` | `…_humidity_zigbee2mqtt` | `…:humidity-sensor:decimal:humidity` |
| `binary_sensor.…_contact` | `…_contact_zigbee2mqtt` | `…:opening-sensor:binary:contact` (inversion, § 3.1) |
| `sensor.…_power` | `…_power_zigbee2mqtt` | `…:switch:power:power` ([gladys] `numericType.js:383`) |
| `light.…` | `…_light_zigbee2mqtt` | `…:light:binary:state` (l'`object_id` `light` correspond à la propriété `state`) |

Confiance : **sûre** pour les capteurs (même nom d'expose des deux côtés). Les entités composites (`light`, `climate`, `cover`) demandent une table `object_id` → propriétés.

**Préremplissage** : l'entrée `mqtt` de HA (`broker`, `port`, `username`, `password` ; [core] `components/mqtt/const.py:34` et `homeassistant/const.py:191,199,253`) peut alimenter le mode MQTT externe de Gladys zigbee2mqtt (`Z2M_MQTT_MODE`, `Z2M_MQTT_URL`, `Z2M_MQTT_USERNAME`, `Z2M_MQTT_PASSWORD`, [gladys] `server/services/zigbee2mqtt/lib/constants.js:8,13-17`). Le comportement exact du mode externe n'a pas été étudié (**non vérifié**).

---

## 4. MQTT discovery générique → service natif mqtt

### 4.1 Côté HA

- Registre : identifiants `("mqtt", <chaque valeur de device.identifiers>)` et connexions recopiées telles quelles ([core] `components/mqtt/entity.py:1494-1506`). Le schéma impose `identifiers` ou `connections` ([core] `components/mqtt/schemas.py:135,165-177`).
- `unique_id` d'entité = `unique_id` du payload. Les payloads eux-mêmes ne sont pas dans `.storage` : ils sont sur le broker, en principe en message *retained* (**non vérifié** pour chaque firmware).
- Préfixe de découverte configurable (`discovery_prefix`, défaut `homeassistant`) ([core] `components/mqtt/const.py:44,283`).

### 4.2 Côté Gladys (`server/services/mqtt/lib/homeAssistant/`)

- Abonnement fixe à `homeassistant/#` ([gladys] `server/services/mqtt/lib/init.js:15` ; `constants.js:9-10`). **Un préfixe HA personnalisé n'est pas vu par Gladys.**
- Identifiant d'appareil : **premier** élément de `device.identifiers`, sinon `device.connections[0].join('-')`, sinon `node_id` ou `object_id` du topic, puis assainissement `[^a-zA-Z0-9_-]` → `-` ([gladys] `handleHomeAssistantDiscoveryMessage.js:20-36`).
- `external_id` appareil : `homeassistant:<identifiant>` ([gladys] `handleHomeAssistantDiscoveryMessage.js:92` ; `convertToGladysDevice.js:418,437`).
- Clé d'entité : `<component>:[<node_id>:]<object_id>` (topic classique) ou `<platform>:<object_id>:<clé du composant>` (découverte par appareil) ([gladys] `handleHomeAssistantDiscoveryMessage.js:203,211`).
- Feature : `homeassistant:<id>:<clé d'entité>` plus `:state`, `:brightness`, `:color_temp`, `:color`, `:position`, `:target_temperature`, `:current_temperature` pour les composants multi-features ([gladys] `convertToGladysDevice.js:94-95` ; `constants.js:30-38`).
- Params : un param `ha_discovery_config:<clé d'entité>` par entité convertie, dont la valeur est le JSON de la config **sans** `device`, `origin` et `availability` ([gladys] `convertToGladysDevice.js:420-431` ; `constants.js:12`). **Ce JSON garde `unique_id`** (abréviation `uniq_id` développée, `constants.js:154`).

### 4.3 Règle

- Appareil : pour **chaque** identifiant `("mqtt", v)` de l'appareil HA, calculer `homeassistant:` + `gladys_mqtt_sanitize(v)` et chercher l'`external_id`. À défaut d'identifiants, essayer `<type>-<valeur>` pour chaque connexion. Le registre HA stocke un ensemble : l'ordre du payload (dont dépend Gladys) est perdu, d'où l'essai de chaque valeur. Confiance : **probable**. Collision possible car deux identifiants distincts peuvent donner la même chaîne assainie.
- Connexions : HA normalise la MAC (`aa:bb:…`) mais Gladys part du payload brut (`AA:BB:…` → `mac-AA-BB-…`) : comparer sans tenir compte de la casse.
- Appareil sans identifiant ni connexion : HA le refuse ([core] `components/mqtt/schemas.py:135`). Côté Gladys, une entité sans bloc `device` devient un appareil nommé d'après le topic. **Impossible → manuel** pour ces entités orphelines.

### 4.4 Entités → features

Règle **sûre** : `unique_id` de l'entité HA (plateforme `mqtt`) = champ `unique_id` du JSON d'un param `ha_discovery_config:<clé>` → clé d'entité → features `homeassistant:<id>:<clé>[:<propriété>]`. Le type de feature se déduit ensuite de la table du § 4.5.

| Entité HA | Feature(s) Gladys |
|---|---|
| `sensor` `device_class: temperature` | `…:sensor:<obj>` → TEMPERATURE_SENSOR / decimal |
| `binary_sensor` `device_class: door` | `…:binary_sensor:<obj>` → OPENING_SENSOR / binary, **inversée** |
| `light` avec luminosité | `…:light:<obj>:state` (LIGHT / binary) + `…:brightness` |
| `climate` | `…:climate:<obj>:target_temperature` (THERMOSTAT) + `:current_temperature` (TEMPERATURE_SENSOR) |
| `cover` | `…:cover:<obj>:state` + `:position` (SHUTTER ou CURTAIN) |

Composants non pris en charge par Gladys (`fan`, `select`, `number`, `event`…) : features perdues ([gladys] `constants.js:27`).

### 4.5 La table HA → Gladys existante : structure et réutilisabilité

Fichier [gladys] `server/services/mqtt/lib/homeAssistant/constants.js` (338 lignes).

| Constante | Lignes | Contenu |
|---|---|---|
| `SUPPORTED_COMPONENTS` | 27 | `sensor`, `binary_sensor`, `switch`, `light`, `cover`, `lock`, `climate`, `button` |
| `FEATURE_PROPERTIES` | 30-38 | suffixes des composants multi-features |
| `ABBREVIATIONS`, `DEVICE_ABBREVIATIONS`, `ORIGIN_ABBREVIATIONS` | 42-179 | abréviations du protocole de découverte |
| `SENSOR_DEVICE_CLASSES` | 182-221 | 30 `device_class` de `sensor` → `{category, type}` |
| `BINARY_SENSOR_DEVICE_CLASSES` | 229-249 | 17 `device_class` de `binary_sensor` → `{category, inverted?}` |
| `CURTAIN_COVER_DEVICE_CLASSES` | 253 | `curtain` → CURTAIN, sinon SHUTTER |
| `COLOR_TEMP_BOUNDS` | 257-262 | bornes mireds/Kelvin |
| `UNITS` | 265-293 | 27 unités HA → unités Gladys |
| `DEFAULT_PAYLOADS`, `COVER_STATE_BY_PAYLOAD_KEY` | 296-321 | charges utiles par défaut, états de volet |

Couverture `sensor` : `temperature`, `humidity`, `moisture`, `pressure`, `atmospheric_pressure`, `illuminance`, `battery`, `power`, `apparent_power`, `energy`, `voltage`, `current`, `carbon_dioxide`, `carbon_monoxide`, `pm25`, `pm10`, `nitrogen_dioxide`, `ozone`, `sulphur_dioxide`, `volatile_organic_compounds`, `signal_strength`, `speed`, `wind_speed`, `precipitation`, `duration`, `aqi` (`:183-220`). Couverture `binary_sensor` : `motion`, `moving`, `door`, `window`, `garage_door`, `opening`, `lock`, `presence`, `occupancy`, `smoke`, `carbon_monoxide`, `moisture`, `vibration`, `tamper`, `battery`, `rain` (`:230-248`). Les classes absentes (`gas`, `problem`, `safety`…) sont **volontairement** exclues (`:227-228`) et donnent la catégorie `unknown` (`convertToGladysDevice.js:62,99`).

Logique hors table (dans `convertToGladysDevice.js`) : capteur sans `device_class` gardé en `unknown` seulement s'il a une unité ou une `state_class` (`:53-63`) ; `switch` → SWITCH / binary (`:121-123`) ; `button` → BUTTON (`:148-149`) ; `lock` → LOCK (`:314-315`) ; `climate` → THERMOSTAT + TEMPERATURE_SENSOR (`:341-357`).

**Réutilisabilité comme table générique « entité HA → feature Gladys »** :
- **Oui pour l'axe `(domaine, device_class)` → `(category, type)` + unité + inversion.** C'est exactement ce que le rapprocheur doit savoir d'une entité HA, quelle que soit l'intégration. Le drapeau `inverted` est indispensable à l'import d'historique des ouvertures et serrures.
- **Limites** : (1) elle ne couvre que 8 domaines (pas `fan`, `media_player`, `select`, `number`, `water_heater`, `vacuum`…) ; (2) un même `device_class` correspond à des **catégories différentes selon l'intégration cible** : Zigbee2MQTT range `power` en SWITCH / power ([gladys] `zigbee2mqtt/exposes/numericType.js:383`), Shelly aussi ([shelly] `src/shelly/features.js:83-86`), alors que la table MQTT donne ENERGY_SENSOR / power (`constants.js:193`). La table ne peut donc pas **prédire** la feature exacte d'une autre intégration ; (3) elle ne connaît pas `state_class`, utile pour distinguer index et puissance.
- **Usage recommandé** : s'en servir comme **filtre de compatibilité** (la catégorie attendue est-elle plausible ?) et comme **repli** générique, appliqué comme l'appariement automatique de la migration native de Gladys : une feature est appariée seulement si **une seule** feature libre a le même triplet `(category, type, unit)` ([gladys] `docs/specs/device-migration.md:41`). La correspondance principale reste propre à chaque intégration (`unique_id` → `external_id`). Il faut copier la table dans le catalogue de l'outil (aucune API Gladys ne l'expose : **non vérifié**) et l'étendre par intégration.

### 4.6 Préremplissage

Entrée HA `mqtt` : `broker`, `port`, `username`, `password` (`data`), `discovery_prefix` (options) ([core] `components/mqtt/const.py:34,44` ; `config_flow.py:4469-4470`). Variables Gladys : `MQTT_URL` (`mqtt://<broker>:<port>`), `MQTT_USERNAME`, `MQTT_PASSWORD`, `MQTT_EMBEDDED_BROKER` ([gladys] `server/services/mqtt/lib/constants.js:1-6`, `saveConfiguration.js:53-56`). Si `discovery_prefix ≠ homeassistant`, l'outil doit **signaler** que Gladys ne verra pas les appareils. Pas de manifeste (service natif).

---

## 5. Matter → service natif matter

**Côté HA**

- Identifiants ([core] `components/matter/adapter.py:248-269`) :
  - `("matter", "deviceid_<OperationalInstanceId>-MatterNodeDevice")`, ou `…-<endpoint>` pour un appareil ponté ([core] `components/matter/helpers.py:81-94`). `OperationalInstanceId` = `<compressed_fabric_id 16 hex MAJ>-<node_id 16 hex MAJ>` ([core] `helpers.py:59-68`) ;
  - `("matter", "serial_<numéro de série>")` si le numéro existe, ne contient pas « test » et, pour un appareil ponté, diffère de celui du pont (`adapter.py:255-269`). Constantes `deviceid` et `serial` ([core] `components/matter/const.py:18-19`).
- `serial_number`, `model_id` (= productID, sauf appareil ponté), `manufacturer` = vendorName ([core] `adapter.py:286-311`). Pas de connexion, pas `entry_type: service`.
- `unique_id` d'entité : `<device_id>-<endpoint_id>-<clé de description>-<cluster_id>-<attribute_id>` ([core] `helpers.py:123-133`).

**Côté Gladys**

- Un appareil **par endpoint** : `external_id = matter:<nodeId>:<devicePath>` ; `devicePath` concatène les numéros d'endpoint ([gladys] `server/services/matter/utils/convertToGladysDevice.js:102-109` ; `lib/matter.handleNode.js:67-77`).
- Params : `UNIQUE_ID` (BasicInformation.uniqueId) et `SERIAL_NUMBER` (BasicInformation.serialNumber) s'ils existent (`convertToGladysDevice.js:118-129`).
- Feature : `matter:<nodeId>:<devicePath>:<clusterId>[:suffixe]` (suffixes `click`, `position`, `state`, `color`, `local-temperature`, `heating`… ; `:149,175,222,233,267,293`).

**Règle** : `identifiers[matter]` de forme `serial_<s>` → `<s>` = param `SERIAL_NUMBER` (égalité stricte). Comme un nœud Gladys produit plusieurs appareils, chacun portant le même numéro, rattacher l'appareil HA au **groupe** d'appareils Gladys du nœud. Confiance : **probable** (numéro fourni par le fabricant, souvent unique mais pas garanti). `deviceid_*` est **inutilisable** : la fabric et le `node_id` de Gladys diffèrent de ceux de HA. Pas de numéro de série, ou numéro rejeté par HA → **impossible → manuel** (aide : `manufacturer`, `model`, `model_id` ; HA ne stocke pas `uniqueId`). Préalable : l'appareil doit être appairé à Gladys (partage multi-admin depuis HA ou nouvel appairage). L'API HA d'ouverture de fenêtre d'appairage n'a pas été lue (**non vérifié**).

**Entités → features** : `endpoint_id` HA = dernier segment de `devicePath` Gladys, et `cluster_id` HA = `clusterId` Gladys. Confiance : **probable**.

| Entité HA | Motif `unique_id` HA | Feature Gladys |
|---|---|---|
| `switch`/`light` on/off | `…-<ep>-…-6-0` | `matter:<n>:<path>:6` SWITCH / binary ([gladys] `convertToGladysDevice.js:142-152`) |
| `sensor` température | `…-<ep>-…-1026-0` | `…:1026` TEMPERATURE_SENSOR / decimal (`:202-213`) |
| `sensor` humidité | `…-<ep>-…-1029-0` | `…:1029` (`:272`) |
| `binary_sensor` occupation | `…-<ep>-…-1030-0` | `…:1030` (`:179`) |
| `cover` | `…-<ep>-…-258-…` | `…:258:position` / `:258:state` (`:214-233`) |

Les identifiants de clusters (6, 1026, 1029, 1030, 258) sont ceux de la spécification Matter : **non vérifié** dans le code lu, qui utilise des constantes `OnOff.Complete.id`, etc.

**Préremplissage** : l'entrée HA ne contient que `url` (serveur Matter) et `use_addon` ([core] `components/matter/config_flow.py:44-56,344-345`). Gladys embarque son propre contrôleur. **Rien** à préremplir.

---

## 6. Shelly → Terdious/gladys-shelly

**Côté HA**

- Appareil principal : `connections={("mac", mac)}`, `identifiers={("shelly", mac)}` ([core] `components/shelly/coordinator.py:206-218`), avec `mac = config_entry.unique_id` (`coordinator.py:191`), fixé à partir du champ `mac` de l'appareil ([core] `components/shelly/config_flow.py:444,750`). La valeur de `connections` est normalisée par le registre (`aa:bb:…`) ; celle d'`identifiers` reste brute (MAC sans séparateur, casse de l'appareil : **non vérifié**, voir `config_flow.py:290`).
- Enfants : `identifiers={("shelly", f"{mac}-{key}")}` (canal) ou `f"{mac}-{key}-{phase}"` (phase EM) ([core] `components/shelly/utils.py:852-875`) ; appareils BLE : `connections={("bluetooth", addr)}` (`utils.py:891-892`).
- `unique_id` : `"{mac}-{key}"` (entité de composant, ex. `switch:0`) et `"{mac}-{key}-{attribut}"` (capteur RPC) ([core] `components/shelly/entity.py:443,576` ; `switch.py:185`) ; Gen1 : `"{mac}-{block.description}"` (`entity.py:400`), REST : `"{mac}-{attribut}"` (`entity.py:536`). Exemple : `sensor` « power » = composant `switch`, sous-clé `apower` ([core] `sensor.py:544-550`).

**Côté Gladys**

- `external_id = ext:<selector>:device:<shellyId>` ([shelly] `src/shelly/constants.js:10-11`, `src/shelly/discovery.js:383,435`).
- `shellyId` : `id` renvoyé par `/shelly` en Gen2+ (`shellypro4pm-ece334ea4d10`, [shelly] `src/shelly/selector.js:48`) ; en Gen1, reconstruit en `<modèle minuscule alphanumérique>-<mac minuscule>` ([shelly] `src/shelly/gen1/normalize.js:343-352`).
- Params : `IP_ADDRESS`, `SHELLY_ID`, `SHELLY_MODEL`, `SHELLY_GEN`, `SHELLY_SLEEPY` ([shelly] `src/shelly/deviceMapping.js:128-139` ; `constants.js:88-93`). La MAC seule n'est pas publiée, mais elle est **contenue** dans `SHELLY_ID` (commentaire `deviceMapping.js:110`).
- Feature : `…:device:<shellyId>:<composant>:<id>:<mesure>`, ex. `switch:0:binary`, `switch:0:power`, `em:0:a_active_power` ([shelly] `src/shelly/features.js:64-130,153`, `toGladysFeature` `:1216-1219`).

**Règle** : `connections[mac]` → `mac` normalisé ; candidat si `SHELLY_ID` (ou le suffixe de l'`external_id`) se termine par `-<mac>`. Confiance : **sûre**, à condition que l'`id` Gen2 se termine bien par la MAC (convention Shelly vérifiée seulement pour Gen1, `normalize.js:335-338` ; l'exemple Gen2 du code la contient). Appareils BLE (connexion `bluetooth`) : non couverts par la cible (**non vérifié**) → manuel. Enfants HA : rattachés au parent.

**Entités → features** : on retire `<mac>-` de l'`unique_id` HA, on garde `<key>` (`switch:0`) et `<attribut>`. Probable :

| Entité HA | `unique_id` | Feature Gladys |
|---|---|---|
| `switch` | `<mac>-switch:0` | `switch:0:binary` (SWITCH / binary) |
| `sensor` power | `<mac>-switch:0-power` | `switch:0:power` (SWITCH / power, W) |
| `sensor` energy | `<mac>-switch:0-energy` | `switch:0:energy` (unité à comparer : Wh/kWh, **non vérifié**) |
| `sensor` voltage | `<mac>-switch:0-voltage` | `switch:0:voltage` |
| `sensor` temperature | `<mac>-switch:0-temperature` | `switch:0:temperature` |

Les noms d'attributs HA (`power`, `voltage`, `energy`) ne sont pas toujours ceux des mesures Gladys (`total_active_power`, `a_active_power`…) : la table de correspondance doit être écrite par composant (`switch`, `em`, `em1`, `pm1`, `cover`, `light`).

**Préremplissage** (`config_schema` [shelly] `gladys-assistant-integration.json`) :

| HA (`data` de chaque entrée, [core] `components/shelly/config_flow.py:465-475,649-656`) | Gladys |
|---|---|
| `host` (+ `port` si ≠ 80), pour les appareils absents du mDNS | `manual_hosts` (liste) |
| `username` / `password` (protection HTTP de l'appareil) | `device_username` / `device_password`, ou `device_passwords` si les mots de passe diffèrent |
| — | `cloud_*`, `mqtt_*`, `feature_names`… : rien |

---

## 7. Google Cast → gboulvin/Gladys-Google-Cast

**Côté HA** : `identifiers={("cast", str(uuid).replace("-", ""))}` ; `unique_id` de l'entité `media_player` = `str(uuid)` avec tirets ([core] `components/cast/media_player.py:333-339`). Pas de connexion, pas `service`. Une seule entrée de configuration (`unique_id = "cast"`, [core] `components/cast/config_flow.py:79`).

**Côté Gladys**

- `external_id = ext:<selector>:cast:<castId>` ([cast] `src/devices/castDevice.js:16,74-77,115-119`).
- `castId` = champ TXT mDNS `id`, en minuscules, sans caractère non alphanumérique ([cast] `src/cast/discovery.js:51-52`). Adresse manuelle sans mDNS : `manual-<slug>-<port>` (`discovery.js:118`).
- Params : `CAST_HOST`, `CAST_PORT`, `CAST_MODEL` (`castDevice.js:20-24,122-126`).
- Features : catégorie MUSIC uniquement, `keep_history: false` (`castDevice.js:127-140`).

**Règle** : identifiant `cast` (déjà sans tirets, minuscules via `str(UUID)`) = suffixe `castId`. Le TXT `id` est l'UUID sans tirets : **non vérifié** dans pychromecast, mais concordant avec le traitement HA. Confiance : **sûre** pour les appareils découverts ; **impossible → manuel** pour ceux saisis à la main (`manual-…`). Groupes d'enceintes : uuid propre des deux côtés, option `include_groups`.

**Entités** : une entité `media_player` par appareil. Pas d'historique à importer (Gladys ne garde pas l'historique musical). Ce qui compte, ce sont les références de scènes : l'action « notification sonore » (`MUSIC.PLAY_NOTIFICATION`) existe côté Gladys (`castDevice.js:39,71` ; [gladys] `docs/specs/device-migration.md:16`).

**Préremplissage** : `data.known_hosts` (liste, [core] `components/cast/config_flow.py:88-110` ; `const.py:30`) → `manual_hosts` (chaîne séparée par des virgules, [cast] manifeste). `ignore_cec` : rien. `include_groups` : laisser la valeur par défaut (`true`).

---

## 8. ESPHome → William-De71/gladys-esphome

**Côté HA**

- Appareil principal : `connections={("mac", device_info.mac_address)}`, **sans** identifiants ([core] `components/esphome/manager.py:1402-1411`). Le nom vient de `friendly_name` ou `name` du nœud.
- Sous-appareils : `identifiers={("esphome", f"{mac}_{device_id}")}` (`manager.py:1424`).
- `unique_id` d'entité : version 3 `"{mac}/{device_id}/{type}/{nom}"`, avec migration depuis la version 1 `"{mac}-{type}-{object_id}"` ([core] `components/esphome/entity.py:149,606-615` ; `entry_data.py:414-415` ; [aioesphomeapi] `aioesphomeapi/model.py:2426-2513`). Types : `sensor`, `binary_sensor`, `switch`, `light`, `text_sensor`… (`model.py:2396-2421`). Format exact de `mac_address` (casse, séparateurs) : **non vérifié**.
- Entrée : `data = {host, port, password, noise_psk, device_name}` ([core] `components/esphome/config_flow.py:650-659` ; `const.py:24-25`).

**Côté Gladys**

- `external_id = ext:<selector>:esphome:<nom du nœud>` ; params `ADDRESS` (`host:port`) et `ESPHOME_VERSION` ([esphome] `src/devices.js:344-383` ; `src/esphome/constants.js:24-31`). **La MAC n'est publiée nulle part** (aucune occurrence de `mac` dans `src/`).
- Feature : `…:esphome:<nœud>:<type>-<objectId>:<clé>` ; `clé` = `state` (capteurs), `brightness`, `color`, `position`, `target_temperature`… ([esphome] `src/devices.js:355-357,681` ; `src/esphome/mapping.js:418-425`).

**Règle** : `config_entry.data.device_name` (nom du nœud ESPHome) = segment `<nom>` de l'`external_id`, comparaison insensible à la casse (Gladys indexe ses connexions en minuscules, [esphome] `src/esphome/EsphomeManager.js:127`). Repli : `data.host:data.port` = param `ADDRESS`. Confiance : **probable** (le nom est unique sur le réseau par construction mDNS, mais peut changer). La règle par MAC est **impossible** tant que gladys-esphome ne publie pas la MAC (évolution simple à proposer en amont).

**Entités → features** : HA v3 → `type` + `nom` ; Gladys utilise `objectId` (v1, nom en *snake_case* assaini). Il faut recalculer `objectId` depuis le nom, ou se servir de la clé v1 si le registre HA en garde une. Probable :

| Entité HA (v3) | Feature Gladys | Catégorie |
|---|---|---|
| `aabb…/0/sensor/Temperature` | `…:sensor-temperature:state` | TEMPERATURE_SENSOR ([esphome] `mapping.js:107`) |
| `…/0/sensor/Humidity` | `…:sensor-humidity:state` | HUMIDITY_SENSOR (`:108`) |
| `…/0/binary_sensor/Motion` | `…:binary_sensor-motion:state` | selon `device_class` |
| `…/0/switch/Relay` | `…:switch-relay:state` | SWITCH / binary |
| `…/0/light/Lamp` | `…:light-lamp:state` + `:brightness` | LIGHT |

Les sous-appareils (`device_id ≠ 0`) n'ont pas d'équivalent identifié côté Gladys (**non vérifié**).

**Préremplissage** :

| HA | Gladys |
|---|---|
| `data.noise_psk` (si identique pour tous les nœuds) | `encryption_key` |
| `data.noise_psk` par nœud | `encryption_keys` au format `nœud\|clé` ([esphome] manifeste) |
| `data.host[:port]` (nœuds non trouvés par mDNS) | `nodes` |
| `data.password` (API legacy) | rien (aucune clé de mot de passe dans le manifeste) |

---

## 9. Tuya → Terdious/gladys-tuya

**Côté HA** : `identifiers={("tuya", device.id)}` ; enfants de canal `("tuya", f"{device.id}_channel_{n}")` ; `unique_id = f"tuya.{device.id}{description.key}"`, où `key` est le DP code ([core] `components/tuya/entity.py:52-80`). Pas de connexion, pas `service`.

**Côté Gladys**

- `external_id = ext:<selector>:device:<tuyaDeviceId>` ; feature `…:<code>` ([tuya] `src/tuya/utils/tuya.externalId.js:3-5` ; `src/tuya/device/tuya.convertDevice.js:142-143` ; `tuya.convertFeature.js:231` ; `constants.js:74`).
- Params : `DEVICE_ID`, `LOCAL_KEY`, `IP_ADDRESS`, `PROTOCOL_VERSION`, `PRODUCT_ID`, `PRODUCT_KEY`, `DEVICE_TYPE`, etc. ([tuya] `src/tuya/constants.js:34-60`). L'ancien service natif utilisait `tuya:<id>` (`tuya.externalId.js:6`).

**Règle** : identifiant `tuya` (sans `_channel_`) = param `DEVICE_ID` (ou suffixe de l'`external_id`). Égalité stricte. Confiance : **sûre** : l'id Tuya est le même pour l'API Smart Life utilisée par HA et pour l'API IoT Cloud de Gladys (même plateforme ; **non vérifié** pour les sous-appareils de passerelle).

**Entités → features** : `unique_id` sans `tuya.<id>` = DP code = dernier segment Gladys. **Sûre** quand la clé HA est un DP code.

| Entité HA | `unique_id` | Feature Gladys |
|---|---|---|
| `switch` | `tuya.<id>switch_1` | `…:switch_1` SWITCH / binary ([tuya] `src/devices/global.js:52`) |
| `sensor` power | `tuya.<id>cur_power` | `…:cur_power` SWITCH / power (`global.js:94`) |
| `sensor` voltage | `tuya.<id>cur_voltage` | `…:cur_voltage` (`global.js:100`) |
| `sensor` energy | `tuya.<id>add_ele` | `…:add_ele` SWITCH / energy (`global.js:82`) |
| `sensor` température | `tuya.<id>va_temperature` | `…:va_temperature` ([tuya] `src/devices/temperatureHumiditySensor.js:7`) |

Les échelles diffèrent (ex. `va_temperature = 216` pour 21,6 °C, `temperatureHumiditySensor.js:7`) : HA enregistre la valeur mise à l'échelle, Gladys aussi après conversion (**non vérifié**). Comparer les unités avant import.

**Préremplissage** : HA utilise la connexion Smart Life par QR code (`user_code`, `token_info{uid, access_token…}`, `terminal_id`, `endpoint`, [core] `components/tuya/config_flow.py:123-134`). Gladys exige un projet IoT Cloud (`access_key`, `secret_key`, `app_account_id`, `endpoint` parmi `centralEurope`, `westernEurope`…). Seuls `token_info.uid` → `app_account_id` (même UID de compte app : **non vérifié**) et l'URL `endpoint` → région (table à écrire) sont exploitables. Les identifiants du projet doivent être saisis par l'utilisateur.

---

## 10. UPnP/IGD → callemand/gladys-upnp-igd (intégration « box »)

**Côté HA** : ce n'est **pas** un appareil `service`, c'est un vrai appareil. `identifiers` : `("upnp", usn)` avec `usn = "<udn>::<deviceType>"`, `("upnp_host", host)`, `("upnp_serial_number", série)` ; `connections` : `("upnp", ssdp_udn)`, `("upnp", udn)` et `("mac", mac)` si connue ([core] `components/upnp/__init__.py:124-134` ; `device.py:143-191`). `unique_id` des entités : `"{original_udn}_{unique_id ou key}"` ([core] `components/upnp/entity.py:40-43`). Entrée : `udn`, `st`, `original_udn`, `mac_address`, `host`, `location` ([core] `components/upnp/config_flow.py:268-275` ; `const.py:31-37`).

**Côté Gladys** : `external_id = ext:<selector>:box:<udn sans "uuid:", assaini>` (repli `host-<hôte>`) ; params `descriptionUrl`, `udn` (UDN brut de la racine, `uuid:…`) ([upnp] `src/devices/convertDevice.js:24-36,63-77` ; `src/constants.js:74-78` ; `src/upnp/soap.js:80-82`). Features : `connection-status`, `external-ip`, `uptime`, `download-rate`, `upload-rate`, `packets-sent`, `packets-received` (`src/constants.js:63-71`).

**Règle** : connexion `upnp` → `upnp_udn` = `upnp_udn(param udn)`. Confiance : **sûre**.

**Entités → features** (probable) :

| Clé HA ([core] `sensor.py`, `binary_sensor.py`) | Feature Gladys ([upnp] `src/devices/box.js`) | Remarque |
|---|---|---|
| `wan_status` | `connection-status` SWITCH / binary | texte HA → binaire |
| `ip` | `external-ip` TEXT | pas d'historique numérique |
| `uptime` | `uptime` DURATION (s) | |
| `KiB/sec_received` | `download-rate` DATARATE (Mbit/s) | **conversion** × 0,008192 |
| `KiB/sec_sent` | `upload-rate` DATARATE (Mbit/s) | idem |
| `packets_sent` / `packets_received` | `packets-sent` / `packets-received` | |

**Préremplissage** : `data.location` → `description_url`, utile seulement si la découverte SSDP échoue.

---

## 11. IPP → guim31/gladys-ipp

**Côté HA** : `identifiers={("ipp", device_id)}` avec `device_id = config_entry.unique_id or entry_id` ([core] `components/ipp/entity.py:26-34` ; `coordinator.py:31`). `unique_id` = UUID de l'imprimante (TXT `UUID` en zeroconf ou `printer.info.uuid`), sinon numéro de série ([core] `components/ipp/config_flow.py:101-112,132,170-180`). `serial_number` renseigné. Pas `service`. Entités : `f"{device_id}_{key}"` ; consommables `marker_<index>` nommés `marker.name` ([core] `sensor.py:154-160`).

**Côté Gladys** : `external_id = ext:<selector>:printer:<printer-uuid sans "urn:uuid:">`, sinon `host-<hôte>` ; param `PRINTER_URL` ; features `state` (texte, sans historique) et `marker:<slug(nom)>` (LEVEL_SENSOR, %, avec historique) ([ipp] `src/device.js:25-27,107-112,133-181` ; `src/printer.js:174-178`).

**Règle** : identifiant `ipp`, normalisé (minuscules, sans `urn:uuid:`) = suffixe de l'`external_id`. Confiance : **probable**. Le format de `printer.info.uuid` de pyipp (avec ou sans `urn:uuid:`) n'a pas été vérifié, ni l'égalité entre TXT `UUID` et `printer-uuid`. Si HA a retenu le numéro de série ou l'`entry_id`, ou si Gladys est en repli `host-…` → **impossible → manuel** (aide : hôte de `data.host` contre `PRINTER_URL`).

**Entités** : `marker_<i>` → `marker:<slug(original_name)>` (même nom de consommable des deux côtés ; suffixe `-2` en cas de doublon côté Gladys, `src/printer.js`/`naming.js` : **non vérifié** en détail). Probable. Le capteur d'état `printer` → `state` (texte, pas d'historique). Les compteurs de pages HA (`pages_completed`…) n'existent pas côté Gladys : perdus.

**Préremplissage** : `data.host`, `port`, `base_path`, `ssl` ([core] `config_flow.py:36-43`) → `printer_hosts` sous forme `ipp://host:port/base_path`, seulement pour les imprimantes non découvertes.

---

## 12. DLNA DMR → guim31/gladys-dlna

**Côté HA** : `connections` = UDN de la racine, UDN du DMR et UDN de l'entrée (`("upnp", …)`), plus `("mac", …)` si connue ; `unique_id` de l'entité `media_player` = UDN ([core] `components/dlna_dmr/media_player.py:438-460,568-571`). Entrée : `url`, `device_id` (UDN), `type`, `mac` ([core] `components/dlna_dmr/config_flow.py:202-207`). Pas `service`.

**Côté Gladys** : `external_id = ext:<selector>:renderer:<platformId>` avec `platformId` = UDN sans `uuid:`, minuscules, caractères hors `[a-z0-9._-]` → `_` ; params `LOCATION`, `UDN`, `MANUFACTURER`, `MODEL` ([dlna] `src/devices/index.js:10,32-49` ; `src/upnp/description.js:120-129` ; `src/devices/renderer.js:207-226`). Les serveurs de médias sont des appareils `server` sans équivalent `dlna_dmr`.

**Règle** : connexion `upnp` → `upnp_udn` = `upnp_udn(param UDN)`. Confiance : **sûre**.

**Entités** : `media_player` → features MUSIC. Pas d'historique utile ; seules les références de scènes comptent.

**Préremplissage** : `data.url` → `description_urls`, seulement si la découverte SSDP échoue. Favoris : rien.

---

## 13. Met.no → guim31/gladys-met-no

**Côté HA** : appareil `entry_type: service`, `identifiers={("met", config_entry.entry_id)}` ([core] `components/met/weather.py:126-133`). `unique_id` de l'entité météo : `"{lat}-{lon}"` (+ suffixe horaire) ou dérivé du domicile si `track_home` (`weather.py:81-89`). Entrée : `latitude`, `longitude`, `elevation`, `name`, ou `track_home: true` ([core] `components/met/config_flow.py:40-50,105-115,132` ; `const.py:45`).

**Côté Gladys** : le manifeste est de type `weather`, avec `location: true` et **aucun appareil** : les prévisions sont servies par maison, à partir des coordonnées de la maison Gladys ([met] `gladys-assistant-integration.json` ; `src/houses.js:90` ; `src/app.js:210-222`).

**Règle** : rien à rapprocher : **sans objet**. L'appareil HA « service » est exclu du rapprochement et signalé « remplacé par l'intégration météo ».

**Ce qui migre** : la configuration seulement.

| HA | Gladys |
|---|---|
| `track_home: true` | rien : Gladys utilise la maison ; vérifier que ses coordonnées sont reprises de `core.config` HA (autre étape de la migration) |
| `elevation` | `altitude` (option globale, à laisser vide sauf en montagne d'après le manifeste) |
| entrée avec `latitude`/`longitude` ≠ domicile | aucun équivalent (Gladys ne fait la météo que par maison) → signaler |

Historique de l'entité `weather.*` : non migrable (pas de feature cible).

---

## 14. Remarques sur le format du catalogue

1. **Trois sources de clé côté HA** suffisent : `identifiers[<domaine>]` filtré par motif (regex à groupe nommé), `connections[<type>]`, et un champ de l'**entrée de configuration** (`data.device_name` pour ESPHome). Le format § 6.2 de l'architecture doit prévoir ce troisième cas.
2. **Trois cibles côté Gladys** : un param nommé (`IEEE_ADDRESS`, `SERIAL_NUMBER`, `DEVICE_ID`, `udn`, `UDN`, `SHELLY_ID`), un segment de l'`external_id` après `<préfixe>:<type>:`, ou une valeur **contenue** dans un param (`SHELLY_ID` se termine par `-<mac>`). Il faut donc des opérateurs `equals`, `suffix` et `ends_with`.
3. **Normalisations nommées** (§ 2.2) plutôt que du code par intégration.
4. **Préfixe externe dynamique** : stocker `type` (`device`, `cast`, `esphome`, `box`, `printer`, `renderer`), jamais `ext:ext-…:` en dur.
5. **Cardinalité** déclarée par règle (`1-1`, `enfants → parent`, `1 → n par endpoint` pour Matter).
6. **Règle d'entité** = regex sur `unique_id` avec groupes nommés → gabarit d'`external_id` de feature, plus une table de traduction (`object_id` → propriété, `cluster` → propriété, clé UPnP → code), plus des **transformations de valeur** : `invert` (ouvertures), `scale` (KiB/s → Mbit/s), et contrôle d'unité. Le repli générique est l'appariement unique `(category, type, unit)` de Gladys (§ 4.5).
7. **Préremplissage** = liste `{source: "data.noise_psk" | "options.x", cible: "<clé config_schema>", transformation}`, avec des cibles qui sont des **variables de service** pour les services natifs (pas de `config_schema`).
8. Prévoir des **exclusions** : coordinateur Zigbee, pont et groupes Z2M, groupes ZHA, appareils `service` sans cible (Met).
