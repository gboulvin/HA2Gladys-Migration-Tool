# Recherche — Conversion des automatisations HA en scènes Gladys (lot 2)

- **Date** : 2026-11-17
- **Statut** : note de recherche factuelle (entrée de la section 04 « scènes » et du convertisseur `lib/scenes/` prévu en 01 § 11)
- **Commits étudiés** :
  - `GladysAssistant/Gladys` : `71edb17` (« feat(nuki): deprecate the native Nuki integration (#3391) »), clone dans `/home/user/ref/gladys`.
  - `home-assistant/core` : `9b5a396` (branche `dev`, 2026.12.0.dev0), clone sparse dans `/home/user/ref/ha-core`. Dossiers ajoutés pour cette note : `sun`, `zone`, `light`, `cover`, `climate`, `notify`, `scene`, `input_boolean`, `timer`, `calendar`, `webhook`, `template`, `device_automation`, `persistent_notification`.
  - Paquet npm `queue@6.0.2` (version verrouillée par Gladys), téléchargé avec `npm pack` pour lire sa valeur de concurrence par défaut.
- **Documents d'entrée** : `00-vision.md` (main), `01-architecture.md` (spec/architecture), `02-extraction-ha.md` et `research/ha-data-sources.md` § 1.5 et § 6 (spec/extraction-ha), `03-mapping-devices.md` (spec/mapping), `research/gladys-codebase.md` § 6 et § 9.3.

Conventions :

- **[G]** = dépôt Gladys, **[HA]** = dépôt `home-assistant/core`. `fichier:NN` renvoie à la ligne NN au commit étudié.
- « **non vérifié** » : affirmation qui ne vient pas de la lecture du code étudié.
- Les paragraphes **« Analyse »** sont des déductions pour notre convertisseur, pas des faits sur l'un ou l'autre logiciel.
- Verdicts de la table de correspondance (§ 4) : **exact** (fidèle à 100 % dans tous les cas), **exact sous condition** (fidèle seulement si la condition indiquée est vérifiée par l'analyseur), **impossible** (raison donnée). Les **écarts transversaux E1 à E5** (§ 4.0) s'appliquent en plus à tous les verdicts.

---

## 1. Modèle de scène Gladys

### 1.1 Table `t_scene`

| Champ | Type, contrainte | Source |
|---|---|---|
| `id` | UUID, clé primaire, généré | [G] `server/models/scene.js:357-361` |
| `name` | chaîne, **obligatoire** | [G] `server/models/scene.js:362-365` |
| `description` | chaîne, facultative | [G] `server/models/scene.js:366-369` |
| `icon` | **obligatoire**, énumération tirée de `server/config/icons.json` (noms Feather/Lucide : `activity`, `alarm-clock`…) | [G] `server/models/scene.js:23,370-373` |
| `active` | booléen, obligatoire, **défaut `true`** | [G] `server/models/scene.js:374-378` |
| `selector` | unique, `^[a-z0-9]+(?:-[a-z0-9]+)*$`, ajouté s'il manque (`slugify(name, true)`, suffixe aléatoire) | [G] `server/models/scene.js:379-386,418`, `server/lib/scene/scene.create.js:15-21` |
| `actions` | JSON, obligatoire, validé par Joi (§ 1.3) | [G] `server/models/scene.js:387-398` |
| `triggers` | JSON, validé par Joi | [G] `server/models/scene.js:399-409` |
| `last_executed` | date, écrite à chaque exécution | [G] `server/models/scene.js:410-412`, `scene.execute.js:59-64` |
| `tags` | association `t_tag_scene` (`scene_id`, `name`, clé composite) | [G] `server/models/scene.js:420-426`, `server/models/tag_scene.js:1-27` |

Il n'existe **ni `triggers_logic`, ni bloc `conditions`, ni champ `mode`** : les seuls champs sont ceux ci-dessus ([G] `server/models/scene.js:354-415`). Les conditions sont des **actions** (§ 2.3).

### 1.2 Création, mise à jour, activation

| Opération | Comportement | Source |
|---|---|---|
| `gladys.scene.create(scene)` | `db.Scene.create` avec `include` des `tags` (la validation Joi du modèle s'exécute), puis `addScene` qui planifie les déclencheurs | [G] `server/lib/scene/scene.create.js:13-38` |
| `gladys.scene.update(selector, scene)` | met à jour, remplace les tags si fournis, ignore `last_executed`, puis `addScene` | [G] `server/lib/scene/scene.update.js:15-58` |
| Activation | `addScene` annule tous les déclencheurs puis ne replanifie que si `scene.active` | [G] `server/lib/scene/scene.addScene.js:149-158` |
| Scène inactive | ignorée par `checkTrigger`, `checkCalendarTriggers` et `dailyUpdate` | [G] `scene.checkTrigger.js:21-25`, `scene.checkCalendarTriggers.js:32-36`, `scene.dailyUpdate.js:44-47` |
| Exécution d'une scène inactive | **possible** : `execute()` ne teste pas `active` (lancement manuel ou action `scene.start`) | [G] `server/lib/scene/scene.execute.js:19-35` |
| Étiquette | précédent : le service MCP ajoute l'étiquette `AI` aux scènes qu'il crée | [G] `server/utils/constants.js:2669`, `server/services/mcp/lib/buildSchemas.js:31-42` |

### 1.3 Validation Joi

- `actions` = `Joi.array().items(Joi.array().items(actionSchema))` ([G] `server/models/scene.js:196`).
- `triggers` = `Joi.array().items(triggerSchema)` ([G] `server/models/scene.js:329`).
- Seules les clés déclarées sont acceptées (pas de `.unknown()` sur les objets action et déclencheur) ([G] `server/models/scene.js:49-194,198-327`).
- Règles particulières :
  - opérateur `changed` : `value`, `threshold_only` et `for_duration` interdits ([G] `server/models/scene.js:297-303`) ;
  - `time-range` : au moins une plage et un jour ; `start ≠ end` ; heures réelles `HH:mm` ([G] `server/models/scene.js:47,227-243,316-327`) ;
  - condition d'`only-continue-if` : **exactement un** de `variable` ou `device_feature` (`xor`) ; opérateurs `=`, `!=`, `>`, `>=`, `<`, `<=` ([G] `server/models/scene.js:129-141`) ;
  - `variable.set` : `text` et `evaluate_value` mutuellement exclusifs ([G] `server/models/scene.js:186-188`) ;
  - `offset` solaire : entier de −1440 à 1440 ([G] `server/models/scene.js:272-275`) ;
  - `before`/`after` de `check-time` : `HH:mm` sans secondes ([G] `server/models/scene.js:79-80`) ; `time` d'un déclencheur : idem ([G] `:259`).
- **Aucune vérification d'existence** des sélecteurs référencés (appareil, feature, utilisateur, scène) : seule la forme est validée ([G] `server/models/scene.js:54-60`).

### 1.4 Sémantique d'exécution

| Point | Comportement exact | Source |
|---|---|---|
| Déclencheurs | **OU** : chaque déclencheur d'une scène active dont le type correspond à l'événement est évalué ; s'il est vérifié, la scène est exécutée (un `execute` par déclencheur vérifié) | [G] `scene.checkTrigger.js:19-47` |
| Niveau 1 de `actions` | **en série** (`Promise.mapSeries`) | [G] `scene.executeActions.js:78` |
| Niveau 2 de `actions` | **en parallèle** (`Promise.map` de bluebird ; concurrence illimitée par défaut, **non vérifié** dans bluebird) | [G] `scene.executeActions.js:80-83` |
| Erreur d'une action | si ce n'est pas un `AbortScene` : **journalisée (`warn`) puis ignorée, la scène continue** | [G] `scene.executeActions.js:39-51` |
| Arrêt par condition | `AbortScene` remonte jusqu'à `execute`, qui l'avale : **toute la scène s'arrête**, y compris depuis une branche `then`/`else` ou un groupe parallèle | [G] `scene.executeActions.js:43-44`, `scene.execute.js:65-72`, `coreErrors.js:65-75` |
| Arrêt manuel | `AbortController` par exécution ; testé **avant** chaque action et pendant `delay` ; une action déjà en cours n'est pas interrompue | [G] `scene.execute.js:44-51`, `scene.executeActions.js:22-24`, `scene.actions.js:284-358`, `scene.stop.js:10-37` |
| Ré-entrance | chaque exécution est poussée dans une file `queue({ autostart: true })` **sans `concurrency`** ; `queue@6.0.2` prend `options.concurrency \|\| Infinity` : **toutes les exécutions tournent en parallèle, sans limite** | [G] `server/lib/scene/index.js:60-62`, `scene.execute.js:35`, `server/package-lock.json:18590-18593` ; `queue@6.0.2` `index.js:14` |
| Arrêter une autre scène depuis une scène | **aucune action** ne le permet (pas de `scene.stop` dans `ACTIONS`) | [G] `server/utils/constants.js:865-958` |
| Contexte (`scope`) | objet partagé par toutes les actions d'une exécution. `triggerEvent` = événement déclencheur. Une action écrit son résultat au chemin `colonne.ligne` (ex. `{{0.1.last_value}}`) : `device.get-value`, `http.request`, `variable.set`, `time.get-date`, `calendar.*` | [G] `scene.checkTrigger.js:40-42`, `scene.actions.js:427-460,461-515,665-690` |
| Gabarits | Handlebars dans `message.send`, `http.request`, `mqtt.send`, `evaluate_value` (Handlebars puis mathjs) | [G] `scene.actions.js:299-306,665-690,961-971,121-179` |
| Boucle de scènes | `scene.start` refuse une scène déjà exécutée dans la même chaîne (`alreadyExecutedScenes`) et lance la cible **sans l'attendre** | [G] `scene.actions.js:360-373`, `scene.execute.js:26-27` |

**Analyse — modes HA.** Le comportement de Gladys équivaut au mode HA `parallel` **sans plafond** (`max` infini). Il n'existe ni `single`, ni `restart`, ni `queued`. Voir § 5.

---

## 2. Catalogue exhaustif Gladys

### 2.1 Déclencheurs

Joi accepte toute valeur de `EVENT_LIST` ([G] `server/models/scene.js:200-202`, `constants.js:2686`), mais **seuls les types ayant un vérificateur dans `triggersFunc` fonctionnent** : pour les autres (ex. `light.turned-on`, `temperature.changed`, [G] `constants.js:703,718`), aucun événement n'est émis vers `checkTrigger`, qui lèverait de toute façon « has no checker function » ([G] `scene.checkTrigger.js:13-15`). Le convertisseur ne doit produire que les types ci-dessous.

| Type | Paramètres | Sémantique exacte | Source |
|---|---|---|---|
| `device.new-state` | `device_features[]` (OU entre features ; ancien `device_feature`), `operator`, `value`, `threshold_only`, `for_duration` (ms) | Événement émis **à chaque valeur reçue**, même identique à la précédente (pas de déduplication) avec `previous_value` = dernière valeur en RAM. Voir § 2.2 | [G] `scene.triggers.js:81-174`, `device.saveState.js:23-24,58-64`, `device.saveStringState.js:20-21,53-59`, `device.newStateEvent.js:15-73` |
| `time.changed` / `every-day` | `time` (`HH:mm`) | `node-schedule` avec `tz` de l'instance, seconde 0 | [G] `scene.addScene.js:173-218` |
| `time.changed` / `every-week` | `days_of_the_week[]`, `time` | idem ; liste vide = tous les jours | [G] `scene.addScene.js:183-199` |
| `time.changed` / `every-month` | `day_of_the_month`, `time` | idem | [G] `scene.addScene.js:177-182` |
| `time.changed` / `custom-time` | `date` (`YYYY-MM-DD`), `time` | une seule fois | [G] `scene.addScene.js:205-212` |
| `time.changed` / `interval` | `interval`, `unit` (`second`, `minute`, `hour`) | **`setInterval` démarré au chargement de la scène** (démarrage de Gladys, modification) : non aligné sur l'horloge | [G] `scene.addScene.js:219-238` |
| `time.changed` / `time-range` | `time_ranges[{start,end}]`, `days_of_the_week`, `resume_on_startup` | exécute la scène au début et à la fin de chaque plage, `triggerEvent.range_event` et `in_range` | [G] `scene.addScene.js:56-136`, `scene.triggers.js:185-228`, `docs/specs/scene-time-ranges.md` |
| `time.sunrise`, `time.sunset` | `house`, `offset` (minutes entières) | horaires calculés **chaque jour à 00:00** par `suncalc` (latitude/longitude de la maison, sans altitude) ; les tâches de la veille sont annulées à minuit ; une heure déjà passée n'est pas planifiée | [G] `scene.dailyUpdate.js:18-85`, `scene.init.js:38-41`, `scene.triggers.js:14-15` |
| `user.back-home`, `user.left-home` | `house`, `user` | présence Gladys | [G] `scene.triggers.js:233-236` |
| `house.empty`, `house.no-longer-empty` | `house` | — | [G] `scene.triggers.js:237-238` |
| `area.user-entered`, `area.user-left` | `user`, `area` | zones géographiques Gladys | [G] `scene.triggers.js:239-242` |
| `calendar.event-is-coming` | `calendars[]`, `calendar_event_attribute` (`start`/`end`), `calendar_event_name_comparator`, `calendar_event_name`, `duration`, `unit` | sondage **chaque minute**, fenêtre ±25 s autour de `maintenant + duration`, **calendriers partagés uniquement** (`shared = true`) | [G] `scene.init.js:44`, `scene.checkCalendarTriggers.js:23-110` |
| `alarm.arm`, `alarm.arming`, `alarm.disarm`, `alarm.partial-arm`, `alarm.panic`, `alarm.too-many-codes-tests` | `house` | alarme de la maison Gladys | [G] `scene.triggers.js:243-248` |
| `system.start` | — | émis une fois les services démarrés | [G] `scene.triggers.js:249`, `server/lib/index.js:283-288` |
| `mqtt.received` | `topic`, `message` | égalité **stricte** du topic et, si `message` non vide, du message. Un filtre à jokers (`+`, `#`) est bien souscrit au broker, mais le topic concret reçu n'est jamais égal au filtre : **un déclencheur à jokers ne se déclenche jamais** | [G] `scene.triggers.js:250-251`, `scene.addScene.js:240-254`, `services/mqtt/lib/subscribe.js:10-17`, `services/mqtt/lib/handleNewMessage.js:30-35` |
| `weather.alert-raised`, `weather.alert-ended` | `house`, `weather_alert_type`, `weather_alert_severity` | — | [G] `scene.triggers.js:27-33,252-253` |
| `energy-contract.price-changed` | `electric_meter` | — | [G] `scene.triggers.js:257-265` |
| `external-integration.scene-event` | `integration`, `trigger_key`, `fields` | égalité ou appartenance, filtres vides = jokers | [G] `scene.triggers.js:36-78` |

### 2.2 Sémantique précise de `device.new-state`

Comparaison par `compare(operator, a, b)` : `=` est `===`, `!=` est `!==`, `>`, `<`, `>=`, `<=` sont les opérateurs JavaScript ([G] `server/utils/compare.js:10-32`). Le type compte : `1 === '1'` est faux.

| Configuration | Se déclenche quand… | Source |
|---|---|---|
| `operator: changed` | `last_value !== previous_value` | [G] `scene.triggers.js:102-104` |
| comparaison, `threshold_only` absent ou `false`, sans `for_duration` | **chaque** valeur reçue vérifie la règle, même répétée (un capteur qui renvoie 25 toutes les minutes avec `> 20` déclenche toutes les minutes) | [G] `scene.triggers.js:122-130` |
| comparaison, `threshold_only: true`, sans `for_duration` | la nouvelle valeur vérifie la règle **et** la précédente non (franchissement) | [G] `scene.triggers.js:125-127` |
| avec `for_duration` | au premier événement qui vérifie la règle (et franchit, si `threshold_only`), une minuterie est créée, une seule à la fois par (scène, feature, opérateur, valeur). Elle est annulée si une valeur ne vérifie plus la règle. À échéance, un événement synthétique relit la valeur courante et déclenche si elle vérifie toujours la règle | [G] `scene.triggers.js:110-120,132-171` |
| avec `for_duration`, **sans** `threshold_only` | après le déclenchement, la minuterie est supprimée ; la valeur suivante qui vérifie la règle en crée une nouvelle : **déclenchements répétés** tant que la condition dure et que l'appareil émet | [G] `scene.triggers.js:134-139,141-170` |
| `previous_value` nul (feature jamais renseignée) | `null` est comparé tel quel : `null < 20` est vrai, `null > 20` est faux | [G] `scene.triggers.js:107-108,125`, `device.saveState.js:24` |

**Analyse — piège de l'événement synthétique.** L'événement émis à la fin d'un `for_duration` porte le type `device.new-state` et passe par `checkTrigger` pour **toutes** les scènes ([G] `scene.triggers.js:157-166`, `scene.checkTrigger.js:19-33`). Une autre scène qui surveille la même feature peut donc revoir la transition `ancienne → courante` une seconde fois et se déclencher deux fois. Le cas est rare (la valeur doit avoir changé pendant la minuterie tout en vérifiant toujours la règle), mais il existe.

### 2.3 Actions

Liste exhaustive : `ACTIONS` ([G] `server/utils/constants.js:865-958`) ; implémentations dans [G] `server/lib/scene/scene.actions.js`.

| Type | Paramètres | Sémantique | Source |
|---|---|---|---|
| `device.set-value` | `device_feature`, `value` **ou** `evaluate_value` | feature `text`/`select` : chaîne ; sinon nombre fini, sinon `AbortScene` | [G] `scene.actions.js:121-179` |
| `device.get-value` | `device_feature` | copie la feature dans le scope | [G] `scene.actions.js:427-430` |
| `light.turn-on`, `light.turn-off`, `light.toggle` | `devices[]` (**sélecteurs d'appareils**) | agit sur la **première** feature `light/binary` de l'appareil ; erreur avalée par appareil ; `toggle` : `last_value === 0 ? 1 : 0` (un état `null` donne 0) | [G] `scene.actions.js:180-224`, `server/utils/device.js:70-79` |
| `light.blink` | `devices[]`, `blinking_time`, `blinking_speed` | — | [G] `scene.actions.js:225-273` |
| `switch.turn-on`, `switch.turn-off`, `switch.toggle` | `device_features[]` (features `switch/binary` modifiables), repli `devices[]` | `toggle` : `last_value ? 0 : 1` | [G] `scene.actions.js:80-118,274-283` |
| `delay` | `value` ou `evaluate_value`, `unit` (`milliseconds`, `seconds`, `minutes`, `hours`) | arrondi à la ms, interruptible | [G] `scene.actions.js:284-358` |
| `scene.start` | `scene` | non attendu, anti-boucle | [G] `scene.actions.js:360-373` |
| `message.send` | `user`, `text`, `service` (`null` = tous les canaux, `__gladys_only__` = conversation Gladys seule) | texte Handlebars | [G] `scene.actions.js:374-381`, `constants.js:517`, `message.sendToUser.js:47-49` |
| `message.send-camera` | `user`, `camera`, `text` | — | [G] `scene.actions.js:382-388` |
| `http.request` | `method`, `url`, `body`, `headers[]`, `request_response_keys` | réponse dans le scope | [G] `scene.actions.js:665-690` |
| `mqtt.send`, `zigbee2mqtt.send` | `topic`, `message` | sans effet si le service n'existe pas | [G] `scene.actions.js:961-982` |
| `alarm.set-alarm-mode` | `house`, `alarm_mode` (`armed`, `disarmed`, `partially-armed`, `panic`) | — | [G] `scene.actions.js:947-960`, `constants.js:2580-2585` |
| `user.set-seen-at-home`, `user.set-out-of-home` | `house`, `user` | — | [G] `scene.actions.js:659-664` |
| `thermostat.set-schedule` | `schedule`, `devices[]` | — | [G] `scene.actions.js:983-1002` |
| `variable.set` | `text` ou `evaluate_value` | — | [G] `scene.actions.js:461-515` |
| `time.get-date` | `precision` | — | [G] `scene.actions.js:431-460` |
| `music.play-notification`, `music.play-media`, `ai.ask`, `sms.send` (déprécié), `external-integration.scene-action` | — | hors besoin du convertisseur | [G] `scene.actions.js:389-426,1003-1084` |

### 2.4 Conditions (actions qui lèvent `AbortScene`)

| Type | Sémantique exacte | Source |
|---|---|---|
| `condition.only-continue-if` | `conditions[]` en **OU** (une seule vraie suffit). Chaque condition compare la **valeur courante** d'une feature (`last_value`, ou `last_value_string` pour une feature texte) ou une variable du scope. Une feature à `null` ne vérifie **aucune** condition, quel que soit l'opérateur. `>`, `<`… exigent une valeur numérique finie. Feature supprimée : `AbortScene` | [G] `scene.actions.js:516-580` |
| `condition.check-time` | `after` : **strictement** après `HH:mm:00` ; `before` : strictement avant ; si `after ≥ before`, OU (plage traversant minuit), sinon ET ; puis `days_of_the_week` sur le jour courant | [G] `scene.actions.js:581-646` |
| `condition.in-time-range` | dans/hors des plages `time-range` de la scène | [G] `scene.actions.js:1145-1190` |
| `condition.if-then-else` | `if[]` exécutés **en parallèle**, tous doivent passer (ET). Une erreur non-`AbortScene` dans `if` remonte (et est avalée au niveau supérieur). `then[][]`/`else[][]` exécutés comme des `actions` : un `AbortScene` dans une branche **arrête toute la scène** | [G] `scene.actions.js:1191-1218` |
| `condition.while` | `if[]` en série avant chaque tour, `then[][]`, `max_iterations` (défaut 1000, plafond 10 000), **au moins 100 ms par tour** | [G] `scene.actions.js:45-47,1085-1144` |
| `house.is-empty`, `house.is-not-empty`, `alarm.check-alarm-mode`, `calendar.is-event-running`, `calendar.get-events`, `ecowatt.condition`, `edf-tempo.condition`, `energy-contract.current-price` | conditions spécialisées | [G] `scene.actions.js:647-658,713-940,941-946`, `constants.js:961-973` |

**Analyse.** Il n'existe **ni NON, ni ET explicite** : un ET s'obtient en enchaînant des actions en série (ou dans `if[]`), un NON en inversant l'opérateur. Il n'existe **aucune condition** sur le soleil, la durée depuis le dernier changement d'état ou l'identité du déclencheur.

---

## 3. Automatisations Home Assistant

### 3.1 Structure et modes

| Élément | Fait | Source |
|---|---|---|
| Clés | `id`, `alias`, `description`, `initial_state`, `triggers`, `conditions`, `actions`, `variables`, `trigger_variables`, `mode`, `max`, `max_exceeded` (anciennes clés `trigger`, `condition`, `action`, `platform`, `service` encore acceptées) | `research/ha-data-sources.md` § 6 ; [HA] `components/automation/config.py:56-84` |
| Modes | `single` (défaut), `restart`, `queued`, `parallel` ; `max` défaut 10 ; `max_exceeded` défaut `WARNING` | [HA] `helpers/script.py:117-134,309-316` |
| `single` déjà en cours | nouvelle exécution **refusée** (`failed_single`) | [HA] `helpers/script.py:1964-1968` |
| `queued`/`parallel` au plafond | nouvelle exécution refusée (`failed_max_runs`) ; `queued` exécute les suivantes à la file | [HA] `helpers/script.py:1970-1977,2023-2027` |
| `restart` | la nouvelle exécution arrête les précédentes | [HA] `helpers/script.py:2030-2036` |
| Conditions de l'automatisation | **ET** de toutes les conditions ; évaluées **avant** de lancer la séquence ; erreur d'évaluation = pas d'exécution | [HA] `helpers/condition/__init__.py:659-670`, `components/automation/entity.py:524-547` |
| `enabled: false` | possible sur un déclencheur, une condition, une action (booléen ou **template**) | [HA] `helpers/config_validation.py:1546,1616,1936`, `helpers/script.py:591-611` |
| `id` de déclencheur | exposé à la condition `trigger` | [HA] `helpers/config_validation.py:1934`, `helpers/condition/__init__.py:555-571` |

### 3.2 Déclencheurs

| Déclencheur | Sémantique exacte | Source |
|---|---|---|
| `state` sans `from`/`to`/`not_*` | se déclenche sur **tout changement d'état ou d'attribut** | [HA] `components/homeassistant/triggers/state.py:157-167,204-209` |
| `state` avec `to` et/ou `from` (y compris `to: null`) | exige `old != new` sur l'état ; les changements d'attribut seuls sont ignorés | [HA] `state.py:142-154,204-209` |
| `state` avec `attribute` | compare l'attribut, ignore les autres changements | [HA] `state.py:183-202` |
| `state` avec `for` | minuterie `async_track_same_state` ; annulée si l'état ne vaut plus la cible (`to`) ou revient à `from` (si seulement `from`) ; `for` peut être un template | [HA] `state.py:231-279`, `:68` |
| `numeric_state` | `above` **strict** (`> above`), `below` **strict** (`< below`), les deux = plage ; « armé » quand la valeur est hors critère, déclenche à l'entrée dans le critère (franchissement). `unavailable`/`unknown`/`None` = hors critère (réarme). Seuils possibles depuis une entité ; `value_template`, `attribute`, `for` | [HA] `triggers/numeric_state.py:81-105,167-177,223-255`, `helpers/condition/conditions.py:358-364,401,432` |
| `time` | `at` : `HH:MM[:SS]`, entité `input_datetime`/`sensor` (timestamp) avec `offset`, template limité ; `weekday` facultatif | [HA] `triggers/time.py:67-91,119-128,259-297` |
| `time_pattern` | `hours`/`minutes`/`seconds`, valeurs ou `/n` ; **aligné sur l'horloge** ; unités inférieures à 0 si une supérieure est donnée | [HA] `triggers/time_pattern.py:36,74-83,100-101` |
| `sun` (ancien `platform: sun`) | `event` `sunrise`/`sunset`, `offset` (durée, **secondes possibles**) ; bibliothèque `astral`, observateur avec **altitude** de HA. Nouveaux déclencheurs `sun.*` : `dawn`, `dusk`, `solar_noon`, `golden_hour_*`, `elevation_*`… | [HA] `components/sun/trigger.py:86-97,181-265,576-628`, `helpers/sun.py:41-45` |
| `zone` | entrée/sortie d'une personne ou d'un `device_tracker` dans une zone | [HA] `components/zone/trigger.py:165-172` |
| `homeassistant` | `start` (une fois au démarrage, si armé avant `STARTED`) ou `shutdown` | [HA] `triggers/homeassistant.py:13-19,51-66` |
| `event` | événement du bus HA | [HA] `helpers/trigger/__init__.py:164-171` |
| `mqtt` | `topic` (jokers, template limité), `payload` (template), `value_template`, `qos`, `encoding` ; une charge vide ne correspond jamais à `payload` | [HA] `components/mqtt/trigger.py:43-54,81-104` |
| `calendar` | début/fin d'événement d'une entité calendrier, `offset` ; rafraîchissement toutes les 15 min | [HA] `components/calendar/trigger.py:44-58` |
| `template`, `webhook`, `tag`, `conversation`… | — | `research/ha-data-sources.md` § 6 (non lus en détail ici) |
| `device` | délégué à l'intégration ; pour une entité « toggle » (`turned_on`/`turned_off`), réécrit en déclencheur `state` `to: on/off`, `for` éventuel ; `entity_id` peut être l'**UUID du registre** | [HA] `components/device_automation/toggle_entity.py:82-98,158-183` |
| Déclencheurs « par domaine » (2025+) : `light.turned_on`, `light.brightness_crossed_threshold`… | `target` (entité, appareil, pièce, étiquette), `options.behavior` `each`/`first`/`all` ; **excluent `unavailable`/`unknown`** des états d'origine et d'arrivée | [HA] `components/light/trigger.py:51-56`, `helpers/trigger/entity_trigger.py:55-58,103-110,129-136,172-182` |

### 3.3 Conditions

| Condition | Sémantique exacte | Source |
|---|---|---|
| `state` | égalité de chaîne ; liste d'états = OU ; plusieurs entités : `match: all` (défaut, ET) ou `any` ; `attribute` ; `for` (depuis `last_changed`) ; état cible pouvant être une entité | [HA] `helpers/condition/conditions.py:219-290`, `helpers/condition/__init__.py:486-520` |
| `numeric_state` | `> above`, `< below` stricts ; `unavailable`/`unknown` = **faux** ; plusieurs entités = ET | [HA] `conditions.py:314-436`, `helpers/condition/__init__.py:443-472` |
| `time` | `after ≤ maintenant < before` (après **inclusif**, avant exclusif), inversé si la plage passe minuit ; secondes possibles ; bornes depuis `input_datetime`/`time`/capteur ; `weekday` | [HA] `conditions.py:470-564` |
| `and`, `or`, `not`, `template`, `trigger`, `device` | cœur | [HA] `helpers/condition/__init__.py:192-200` |
| `sun`, `zone` | intégrations | `research/ha-data-sources.md` § 6 |
| Conditions « par domaine » (`light.is_on`…) | idem déclencheurs par domaine | [HA] `components/light/condition.py:39-43` |

### 3.4 Actions

| Action | Sémantique exacte | Source |
|---|---|---|
| Appel de service (`action:`) | **bloquant** (`blocking=True`) ; `target` = `entity_id`/`device_id`/`area_id`/`floor_id`/`label_id` ; l'expansion indirecte exclut les entités masquées et celles à `entity_category` | [HA] `helpers/script.py:1088-1140` (`:1125`), `helpers/target.py:209-217,350-357` |
| Erreur d'une action | **arrête la séquence** sauf `continue_on_error: true`, qui ne couvre que les `HomeAssistantError` hors erreurs de configuration | [HA] `helpers/script.py:139-145,646-668` |
| `condition` (action) | faux → `_ConditionFail` : arrête **la séquence courante**. Dans une sous-séquence (`if`, `choose`, `parallel`, `sequence`, `repeat`), seule cette sous-séquence s'arrête : `_ConditionFail` n'est pas propagé | [HA] `helpers/script.py:548-557,833-860` |
| `stop` | arrête tout le script (propagé depuis les sous-séquences) ; `error: true` = abandon en erreur | [HA] `helpers/script.py:558-565,1055-1075` |
| `delay` | durée ou template | [HA] `helpers/script.py:1245-1249` |
| `wait_template`, `wait_for_trigger` | attente d'un état/événement avec `timeout`, `continue_on_timeout` | [HA] `helpers/script.py:1312-1414` |
| `choose` | première option dont **toutes** les conditions passent, sinon `default` | [HA] `helpers/script.py:808-831` |
| `if`/`then`/`else` | conditions en ET | [HA] `helpers/script.py:862-882` |
| `parallel` | chaque branche est un sous-script ; la première exception est relancée après la fin de toutes les branches | [HA] `helpers/script.py:746-762` |
| `sequence`, `repeat` (`count`, `while`, `until`, `for_each`) | sous-scripts | [HA] `helpers/script.py:764-768,1045-1053` |
| `scene` | `scene.turn_on` bloquant | [HA] `helpers/script.py:1170-1180` |
| `variables`, `event`, `device`, `set_conversation_response` | — | [HA] `helpers/script.py:1079-1084,1141-1169,1415` |
| `automation.trigger` | `skip_condition` **vrai par défaut** | [HA] `components/automation/services.py:75` |

Services de domaine utiles :

| Service | Paramètres et conversions | Source |
|---|---|---|
| `light.turn_on` | `brightness` (0-255), `brightness_pct` → `round(255 × pct / 100)`, `brightness_step(_pct)`, couleurs, `color_temp_kelvin`, `transition`, `effect`, `flash` | [HA] `components/light/helper.py:132-134,235-239` |
| `cover.set_cover_position` | `position` entier 0-100 | [HA] `components/cover/services.py:36-44` |
| `climate.set_temperature` | `temperature` ou `target_temp_low/high`, `hvac_mode` facultatif ; conversion de l'unité du système HA vers l'unité native de l'appareil | [HA] `components/climate/services.py:100-162` |

---

## 4. Table de correspondance HA → Gladys

### 4.0 Écarts transversaux (s'appliquent à toute scène convertie)

| # | Écart | Effet | Sources |
|---|---|---|---|
| **E1** | **Indisponibilité.** HA a des états `unavailable`/`unknown` ; Gladys garde la dernière valeur connue d'une feature. | HA : `on → unavailable → on` redéclenche un `state to: on` ; une condition `state: on` ou `numeric_state` est **fausse** pendant l'indisponibilité. Gladys : aucun redéclenchement, condition évaluée sur une valeur périmée. | [HA] `state.py:204-209`, `conditions.py:358-364` ; [G] `device.saveState.js:23-29`, `scene.actions.js:551-564` |
| **E2** | **Erreur d'action.** HA arrête l'exécution ; Gladys journalise et continue. | Si une commande échoue (service arrêté, appareil injoignable qui lève une erreur), les actions suivantes s'exécutent dans Gladys, pas dans HA. | [HA] `script.py:646-668` ; [G] `scene.executeActions.js:39-51` |
| **E3** | **Concurrence.** Gladys = `parallel` sans plafond. | Voir § 5 : seul un mode HA dont les exécutions ne peuvent pas se chevaucher utilement est fidèle. | [G] `index.js:60-62`, `queue@6.0.2 index.js:14` |
| **E4** | **Calcul astronomique.** HA : `astral` avec altitude ; Gladys : `suncalc` sans altitude, offset en minutes entières, recalcul quotidien à 00:00. | Heure de lever/coucher légèrement différente (écart non mesuré, **non vérifié**, de l'ordre de la minute selon l'altitude). | [HA] `helpers/sun.py:41-45` ; [G] `scene.dailyUpdate.js:28-37` |
| **E5** | **Configuration de référence.** Fuseau horaire, latitude/longitude et unités doivent être identiques entre HA et Gladys. | Sinon, heures et seuils décalés. | [G] `scene.init.js:14-17`, `scene.dailyUpdate.js:27-34` ; [HA] `climate/services.py:113-162` |

**Analyse.** Lu à la lettre, D5 (« fidèle à 100 % ») rend E1 et E2 bloquants pour **toute** automatisation dotée d'un déclencheur d'état ou d'au moins deux actions en série. Il faut une décision explicite (proposition : accepter E1, E2 et E4 comme écarts documentés dans le rapport, en refusant tout le reste ; voir § 7).

### 4.1 Valeurs et références

| Élément HA | Gladys | Verdict | Condition / raison |
|---|---|---|---|
| `entity_id` | sélecteur de feature (`device_feature`) issu du journal de migration (03 § 3.4) | exact sous condition | entité migrée ; sinon statut 🔗 « bloquée par un appareil » |
| Entité désignée par **UUID du registre** (déclencheurs/conditions `device`, `entity_ids_or_uuids`) | idem après résolution par le registre des entités | exact sous condition | UUID résolu ([HA] `state.py:123-126`, `toggle_entity.py:82`) |
| `device_id` (cible, déclencheur `device`) | features de l'appareil Gladys rapproché | exact sous condition | résolution des anciens composites (`research/ha-data-sources.md` § 2.1) |
| État `on`/`off` d'un `light`, `switch`, `binary_sensor` | `1`/`0` | exact sous condition | table de valeurs par catégorie : **porte/fenêtre/ouverture et serrure inversées** (HA `on` = ouvert / déverrouillé ; Gladys `1` = fermé / verrouillé) ([G] `services/mqtt/lib/homeAssistant/constants.js:224-235`, `services/zigbee2mqtt/exposes/binaryType.js:10-16`) |
| État textuel (`cover` `open`/`closed`/`opening`, `climate` `heat`/`auto`, `alarm_control_panel`, `media_player`…) | — | impossible sauf table déclarée | Gladys `shutter/state` porte des **commandes** (`OPEN 1`, `CLOSE -1`, `STOP 0`), pas des états ([G] `constants.js:117-121`) ; pas de table de valeurs dans 03 § 3.4 |
| Valeur numérique | même nombre | exact sous condition | même unité, ou conversion exacte déclarée (`transform` de 03 § 3.4 appliqué aussi aux seuils) ; précision identique (**non vérifié**) |
| Attribut (`attribute: brightness`, `current_temperature`…) | feature distincte (ex. `light/brightness`) | exact sous condition | correspondance attribut → feature **à ajouter au catalogue** (03 § 3.4 ne couvre que les entités) et échelle identique |
| Cible `area_id`, `floor_id`, `label_id` | liste figée de features | exact sous condition | expansion calculée comme HA (domaine du service, entités masquées et `entity_category` exclues, pièce de l'entité prioritaire sur celle de l'appareil) **au moment de la conversion** ; l'utilisateur est prévenu que la liste ne suivra plus la pièce ([HA] `helpers/target.py:209-217,350-357`) |
| `entity_id: all` | — | impossible | ensemble dynamique |

### 4.2 Déclencheurs

| HA | Gladys | Verdict | Condition / raison |
|---|---|---|---|
| `state` `to: X` | `device.new-state`, `=` X, **`threshold_only: true`** | exact sous condition | E1. Sans `threshold_only`, Gladys déclencherait à chaque rapport (§ 2.2) |
| `state` `to: [X, Y]` | un déclencheur par valeur (OU) | exact sous condition | idem |
| `state` `from: X` seul | `!=` X, `threshold_only: true` | exact sous condition | E1 (HA voit aussi `X → unavailable`) |
| `state` `from: X, to: Y` | `=` Y avec `threshold_only` + `only-continue-if` sur la variable `triggerEvent.previous_value` `=` X | exact sous condition | binaire : `from: off, to: on` ≡ `= 1` seul. Sinon, passage par une variable de scope que l'éditeur ne propose pas (**non vérifié** dans le front) ; incompatible avec `for` (l'événement final réécrit `previous_value`, [G] `scene.triggers.js:159-165`) ; une seule entité déclencheuse |
| `state` `not_to`/`not_from` | `!=`, `threshold_only` | exact sous condition | une seule valeur exclue |
| `state` `to: null` | `operator: changed` | exact sous condition | E1 |
| `state` **sans** `from`/`to` | — | **impossible** | HA déclenche aussi sur un changement d'**attribut** ; Gladys `changed` ne voit que la valeur |
| `state` + `attribute` | `device.new-state` sur la feature de l'attribut | exact sous condition | voir § 4.1 « Attribut » |
| `state` + `for` (durée fixe) | `for_duration` (ms) + `threshold_only: true` | exact sous condition | E1. Sans `threshold_only` : déclenchements répétés (§ 2.2) |
| `for` en template | — | impossible | template |
| plusieurs `entity_id` | `device_features[]` | exact sous condition | minuteries indépendantes par feature des deux côtés ([HA] `state.py:273-279` ; [G] `scene.triggers.js:110`) |
| `numeric_state` `above: N` | `>` N, `threshold_only: true` | exact sous condition | **`>` et non `>=`** (HA strict). E1 : HA réarme après `unavailable` |
| `numeric_state` `below: N` | `<` N, `threshold_only: true` | exact sous condition | idem ; attention : `previous_value` nul vaut « vérifie la règle » pour `<` ([G] `compare.js:14-15`), donc pas de déclenchement au tout premier rapport. HA, lui, est armé au démarrage si la valeur est hors critère ([HA] `numeric_state.py:167-177`) |
| `numeric_state` `above` **et** `below` | deux déclencheurs (`> above`, `< below`, `threshold_only`) + deux `only-continue-if` en série (`> above`, `< below`) | exact sous condition | automatisation à **déclencheur unique** et sans `for` (les conditions ajoutées s'appliqueraient aux autres déclencheurs) |
| `numeric_state` + `for` | `for_duration` + `threshold_only` | exact sous condition | seuil simple uniquement |
| seuil depuis une entité, `value_template` | — | impossible | seuil dynamique / template |
| `time` `at: "HH:MM"` (ou `HH:MM:00`) | `time.changed` `every-day` | exact sous condition | E5. Heure dans la plage du changement d'heure (02:00-03:00 en Europe) : comportement de `node-schedule` documenté comme différent (saut ou double exécution, [G] `scene.addScene.js:79-82`) ; comportement HA **non vérifié** → refuser |
| `time` + `weekday` | `every-week` + `days_of_the_week` | exact sous condition | idem |
| `time` avec secondes ≠ 0, entité `input_datetime`/`sensor`, `offset`, template | — | impossible | Gladys : `HH:mm` fixe seulement |
| `time_pattern` `hours: H` / `minutes: M` fixes (ex. `minutes: 0` = chaque heure) | N déclencheurs `every-day` (un par occurrence quotidienne) | exact sous condition | `seconds` = 0, N raisonnable (proposition : ≤ 24) |
| `time_pattern` `/n` | — | impossible | `interval` Gladys part du chargement de la scène, non aligné sur l'horloge ([G] `scene.addScene.js:237`) ; décomposition en N déclencheurs seulement si N ≤ limite |
| `sun` `sunrise`/`sunset` + `offset` | `time.sunrise`/`time.sunset`, `house`, `offset` minutes | exact sous condition | E4, E5 ; offset multiple de 60 s ; **heure résultante dans la même journée civile** : une heure après minuit (coucher + 4 h) est planifiée la veille puis annulée par le recalcul de 00:00, une heure avant minuit (lever − 8 h) est « passée » au recalcul : **elle ne se déclenche jamais** ([G] `scene.dailyUpdate.js:19-22,62-77`, `scene.init.js:38-41`) |
| `sun.dawn`, `dusk`, `solar_noon`, élévation, heure dorée… | — | impossible | aucun équivalent |
| `homeassistant` `start` | `system.start` | exact sous condition | transposé au démarrage de Gladys (moment différent par nature, à indiquer) |
| `homeassistant` `shutdown` | — | impossible | pas d'événement d'arrêt |
| `zone`, `device_tracker`, `person` | `area.user-*`, `user.back-home`… | impossible (V2) | présence non migrée (03 § 6, M2) et zones HA non migrées |
| `event` | — | impossible | pas de bus d'événements HA |
| `template`, `webhook`, `tag`, `conversation` | — | impossible | pas d'équivalent (le webhook Gladys est réservé aux intégrations externes, [G] `constants.js:654`) |
| `mqtt` `topic` sans joker, `payload` constant non vide | `mqtt.received` `topic`, `message` | exact sous condition | même broker utilisé par le service MQTT de Gladys ; pas de `value_template` ; QoS ignorée |
| `mqtt` avec joker `+`/`#` | — | impossible | ne se déclenche jamais dans Gladys (§ 2.1) |
| `calendar` | `calendar.event-is-coming` | impossible (V2 à réévaluer) | source de synchronisation, fréquence (15 min HA / sondage minute Gladys), calendrier partagé obligatoire, `contains` en `LIKE` : équivalence non démontrée |
| `device` `turned_on`/`turned_off` (entité toggle) | comme `state to: on/off` | exact sous condition | idem `state` |
| `device` d'intégration (bouton ZHA/Z2M/Hue : `remote_button_short_press`…) | `device.new-state` sur la feature bouton | exact sous condition | table « type d'événement → valeur de feature » **déclarée dans le catalogue** (absente de 03) |
| Déclencheurs par domaine (`light.turned_on`…) `behavior: each` | comme `state to:` | exact sous condition | cible figée (§ 4.1). Ces déclencheurs **excluent** `unavailable` côté HA : E1 disparaît pour eux |
| idem `behavior: first`/`all` | — | impossible | condition sur un groupe |
| `enabled: false` | déclencheur omis | exact | — |
| `enabled` en template, `variables` de déclencheur | — | impossible | template |

### 4.3 Conditions

| HA | Gladys | Verdict | Condition / raison |
|---|---|---|---|
| Liste de conditions (ET implicite) | une action de condition **par** condition, en série | exact sous condition | E1 |
| `state` `state: X` | `only-continue-if` `{device_feature, '=', X}` | exact sous condition | E1 : HA faux si indisponible, Gladys teste la valeur périmée |
| `state` `state: [X, Y]` | une `only-continue-if` à deux conditions (OU) | exact sous condition | idem |
| `state` plusieurs entités `match: all` / `any` | actions en série / une action à plusieurs conditions | exact sous condition | idem |
| `state` + `for` | — | impossible | aucune condition de durée ; `last_value_changed` de Gladys est mis à jour même quand la valeur ne change pas ([G] `device.saveState.js:26-29`) |
| `state` + `attribute` | sur la feature de l'attribut | exact sous condition | § 4.1 |
| `numeric_state` `above`/`below` | `>` / `<` (deux actions en série si les deux) | exact sous condition | E1 ; `>` et non `>=` |
| `numeric_state` seuil entité / template | — | impossible | — |
| `time` `after`/`before` `HH:MM` | `condition.check-time` | exact sous condition | E5 ; bornes à la minute (secondes refusées). Écart résiduel : `after` inclusif chez HA, strict chez Gladys, soit la seule milliseconde `HH:MM:00.000` |
| `time` `weekday` | `days_of_the_week` | exact | correspondance `mon` → `monday`… |
| `time` borne `input_datetime`/capteur | — | impossible | dynamique |
| `or` de conditions simples | une `only-continue-if` à plusieurs conditions | exact sous condition | chaque membre exprimable comme une comparaison unique de feature |
| `or` contenant un `and` (ou une plage `above`+`below`) | — | impossible | pas de ET imbriqué dans un OU (`if-then-else` dupliquerait la suite de la scène) |
| `and` | actions en série | exact sous condition | membres exprimables |
| `not` d'une comparaison | opérateur inversé (`=`↔`!=`, `>`↔`<=`, `<`↔`>=`) | exact sous condition | E1 renforcé : HA `not state: on` est **vrai** si indisponible |
| `not` d'un `or` / `and` | De Morgan → ET / OU | exact sous condition | idem |
| `sun`, `zone`, `template` | — | impossible | aucun équivalent |
| `trigger` (`id`) | — | impossible en l'état | Gladys ne sait pas quel déclencheur a lancé la scène (sauf variables de `triggerEvent`, non exposées par l'éditeur). Alternative : **éclater** l'automatisation en une scène par déclencheur quand chaque branche ne dépend que de son déclencheur (exact sous condition, § 7) |
| `device` (`is_on`…), conditions par domaine | comme `state` | exact sous condition | — |

### 4.4 Actions

| HA | Gladys | Verdict | Condition / raison |
|---|---|---|---|
| `light.turn_on` / `turn_off` sans données | `device.set-value` (feature `light/binary`, 1/0) | exact sous condition | E2. **Préférer `device.set-value` à `light.turn-on`**, qui vise la première feature `light/binary` de l'appareil et donc le mauvais canal d'un appareil multi-canal fusionné (03 `merge_into_parent`) ([G] `scene.actions.js:180-194`, `utils/device.js:74`) |
| `light.toggle` | `light.toggle` ou `switch.toggle` | exact sous condition | appareil mono-canal et état connu (`null` → éteint pour `light.toggle`, [G] `scene.actions.js:210-224`) |
| `light.turn_on` + `brightness`/`brightness_pct` | `device.set-value` binaire **et** luminosité, en parallèle | exact sous condition | échelle et conversion déclarées par intégration (`round(255 × pct / 100)` côté HA) ; le comportement « régler la luminosité allume » doit être identique dans l'intégration Gladys (**non vérifié**, par intégration) |
| `light.turn_on` + couleur, `color_temp_kelvin`, `effect` | `device.set-value` `color`/`temperature`/`effect-mode` | exact sous condition | conversion d'espace colorimétrique et d'unités déclarée ; sinon impossible |
| `transition`, `flash`, `brightness_step*`, `profile` | — | impossible | pas de paramètre équivalent (pas d'incrément relatif atomique) |
| `switch.turn_on`/`turn_off`/`toggle` | `switch.turn-on`… avec `device_features[]`, ou `device.set-value` | exact sous condition | E2 |
| `cover.set_cover_position` | `device.set-value` `shutter/position` | exact sous condition | même convention 0 = fermé, ou `transform: invert` |
| `cover.open_cover`/`close_cover`/`stop_cover` | `device.set-value` `shutter/state` 1 / −1 / 0 | exact sous condition | feature d'état présente |
| `cover` tilt | — | impossible | pas de feature d'inclinaison générique (**non vérifié** pour chaque intégration) |
| `climate.set_temperature` `temperature` | `device.set-value` `target-temperature` | exact sous condition | même unité (E5) |
| `climate.set_temperature` + `hvac_mode`, `target_temp_low/high`, `set_hvac_mode`, `set_preset_mode` | `thermostat/mode`… | impossible sauf table de valeurs déclarée | — |
| `notify.<service>` (texte fixe, sans `title`) | `message.send` (`user`, `text`, `service`) | exact sous condition | correspondance « service HA → utilisateur Gladys + canal » **confirmée par l'utilisateur** (Telegram, ntfy…). `notify.mobile_app_*` : pas d'équivalent garanti. Texte sans `{{` (Gladys l'interpréterait en Handlebars) |
| `notify` avec `title`, `data`, cible multiple, texte en template | — | impossible | — |
| `persistent_notification.create` | — | impossible | affichage différent (notification d'interface partagée, `notification_id`) ; `__gladys_only__` n'est qu'un message de conversation d'un utilisateur |
| `delay` fixe | `delay` en `milliseconds` (ou unité exacte) | exact | E3 pour l'interaction avec le mode |
| `delay` en template | — | impossible | — |
| `wait_template`, `wait_for_trigger` | — | impossible | pas d'attente d'événement ; `condition.while` sonde (≥ 100 ms, plafond d'itérations) |
| `condition` (action) au premier niveau | action de condition | exact sous condition | comme § 4.3 |
| `condition` dans une branche (`if`, `choose`, `parallel`, `sequence`, `repeat`) | — | **impossible** | HA n'arrête que la branche, Gladys arrête **toute** la scène ([HA] `script.py:548-557` ; [G] `scene.actions.js:1191-1218`) |
| `if`/`then`/`else` | `condition.if-then-else` (`else: []` si absent) | exact sous condition | conditions exprimables ; branches sans `condition` ni `stop` |
| `choose` | `if-then-else` imbriqués (`else` = option suivante, puis `default`) | exact sous condition | idem |
| `parallel` d'actions simples | un groupe de niveau 2 | exact sous condition | E2 (HA : erreur relancée après la fin des branches) |
| `parallel` dont une branche est une séquence | — | impossible en V1 | le niveau 2 n'accepte que des actions isolées ; le contournement par `if-then-else` à `if: []` (toujours vrai, [G] `scene.actions.js:1197-1204`) n'est pas représentable proprement dans l'éditeur (**non vérifié**) |
| `sequence` | aplatie dans la séquence parente | exact sous condition | sans `condition` interne |
| `repeat` `count: n` fixe | actions dépliées n fois | exact sous condition | n raisonnable, sans `condition` interne |
| `repeat` `while`/`until`/`for_each` | — | impossible | `condition.while` : plafond d'itérations et 100 ms minimum par tour |
| `stop` en fin de séquence principale | omis | exact | — |
| `stop` ailleurs, `stop` `error: true` | — | impossible | pas d'action « arrêter » inconditionnelle |
| `variables`, `event`, `set_conversation_response` | — | impossible | — |
| `scene: scene.x` / `scene.turn_on` | actions `device.set-value` en parallèle reproduisant les états de la scène HA | exact sous condition | toutes les entités et attributs de la scène HA convertibles (comme `light.turn_on`) ; **les scènes HA ne sont pas dans le périmètre de 00** |
| `script.<nom>` (bloquant) | contenu du script déplié | exact sous condition | script lui-même convertible ; son `mode` propre est perdu (E3) |
| `script.turn_on`, `automation.trigger` | `scene.start` | exact sous condition | cible convertie ; appel en **dernière** position (Gladys n'attend pas) ; pour `automation.trigger`, `skip_condition` vaut vrai par défaut : la cible ne doit **pas** avoir de conditions, sinon impossible ([HA] `automation/services.py:75`) |
| `homeassistant.turn_on`/`turn_off`/`toggle` | selon le domaine de l'entité | exact sous condition | idem domaine |
| `continue_on_error: true` | aucun champ | exact sous condition | Gladys continue toujours ; écart seulement pour les erreurs de configuration HA, qui n'existent pas dans une automatisation chargée |
| Toute action avec template dans `data`/`target` | — | impossible | template |
| `enabled: false` | action omise | exact | — |

---

## 5. Mode d'exécution

Rappel : Gladys exécute **toutes** les exécutions d'une scène en parallèle, sans plafond (§ 1.4). Aucune action ne permet d'arrêter une exécution en cours.

**Définition (analyse).** Une séquence est « **sans attente** » si elle ne contient ni `delay`, ni `wait_*`, ni `repeat` non déplié, ni `scene.start`/script en attente. Sa durée est alors celle d'une suite d'appels de services ; deux exécutions ne peuvent se chevaucher que si deux déclenchements arrivent dans cet intervalle.

| Mode HA | Comportement HA | Traduction | Verdict |
|---|---|---|---|
| `parallel`, `max` ≥ 2 | jusqu'à `max` exécutions simultanées | aucune | **exact sous condition** : sans attente, ou plafond non atteignable (déclencheurs exclusifs) |
| `single` (défaut) | déclenchement ignoré si déjà en cours | aucune | **exact sous condition** : séquence sans attente (chevauchement négligeable). **Impossible** avec `delay`/`wait` : Gladys lancerait une seconde exécution (ex. deux notifications) |
| `restart` | la nouvelle exécution arrête l'ancienne | aucune | **impossible**, sauf séquence sans attente (redémarrer une exécution presque finie revient à en lancer une seconde) |
| `queued` | exécutions à la file, dans l'ordre | aucune | **exact sous condition** : sans attente **et** sans action dépendant de l'état (`toggle`, conditions sur une feature modifiée par la scène), car des exécutions parallèles peuvent s'entrelacer |
| `max_exceeded` | niveau de journalisation | — | sans effet sur le comportement |

**Motif reconnaissable : la lumière sur détection de mouvement.** Le motif HA le plus courant (« allumer, attendre la fin du mouvement, attendre N s, éteindre », en mode `restart`) est **impossible** élément par élément (`wait_for_trigger`, `restart`). Son comportement observable est pourtant reproductible par **deux scènes** : « mouvement = 1 → allumer » et « mouvement = 0 pendant N s → éteindre » (`for_duration`, `threshold_only`). La minuterie Gladys est annulée par un retour du mouvement, comme le `restart` de HA. **Analyse** : c'est une réécriture de motif, pas une traduction ; elle n'est fidèle que si l'éclairage n'est piloté que par ce mouvement. Elle ne doit être proposée qu'en option explicite (§ 7).

---

## 6. Couverture estimée

Aucune statistique d'usage des automatisations HA n'a été trouvée : les estimations ci-dessous sont **qualitatives (analyse, non vérifié)**.

| Motif courant | Convertible fidèlement ? | Élément bloquant éventuel |
|---|---|---|
| Allumer une lumière sur mouvement (sans extinction) | oui | — |
| Éteindre après N min sans mouvement (`to: off`, `for`) | oui | — |
| Lumière sur mouvement avec extinction après délai en **une** automatisation (`wait_for_trigger`/`delay` + `restart`) | non (sauf motif optionnel § 5) | `wait_for_trigger`, `restart` |
| Notification sur ouverture de porte, texte fixe | oui | correspondance du canal |
| Notification avec le nom de l'entité (`{{ trigger… }}`) | non | template |
| Chauffage selon l'heure et les jours | oui | — |
| Chauffage selon la présence (personne, zone) | non (V2) | présence |
| Alerte de température (seuil, durée) | oui | — |
| Volets au coucher du soleil avec décalage | oui, si l'heure reste dans la journée | E4 |
| Volets selon l'élévation du soleil, `sun` condition | non | pas d'équivalent |
| Bouton sans fil (événement d'appareil ZHA/Z2M) | oui, si le catalogue déclare les valeurs | table d'événements |
| Automatisations de blueprints | selon le contenu substitué | souvent des templates |
| `choose` sur `trigger.id` (une automatisation à plusieurs rôles) | non (oui si éclatée) | condition `trigger` |
| Tout ce qui contient un template Jinja | non | template |

### 6.1 Cas convertibles

**C1. Allumer sur mouvement.**

```yaml
alias: Couloir allumé sur mouvement
mode: single
triggers:
  - trigger: state
    entity_id: binary_sensor.couloir_mouvement
    from: "off"
    to: "on"
actions:
  - action: light.turn_on
    target: { entity_id: light.couloir }
```

```json
{
  "name": "Couloir allumé sur mouvement",
  "icon": "lightbulb",
  "active": false,
  "tags": [{ "name": "Home Assistant" }],
  "triggers": [
    { "type": "device.new-state", "device_features": ["capteur-couloir-motion"],
      "operator": "=", "value": 1, "threshold_only": true }
  ],
  "actions": [
    [ { "type": "device.set-value", "device_feature": "lampe-couloir-binary", "value": 1 } ]
  ]
}
```

`from: off, to: on` sur un binaire ≡ `= 1` avec franchissement (la précédente valeur Gladys ne peut être que 0 ou `null`). `mode: single` sans attente : exact sous condition (§ 5). Écart E1. `active: false` en coexistence (00 § 9). Les noms d'icône et de sélecteurs sont des exemples.

**C2. Éteindre après 2 min sans mouvement.**

```yaml
triggers:
  - trigger: state
    entity_id: binary_sensor.couloir_mouvement
    to: "off"
    for: "00:02:00"
actions:
  - action: light.turn_off
    target: { entity_id: light.couloir }
```

```json
{
  "triggers": [
    { "type": "device.new-state", "device_features": ["capteur-couloir-motion"],
      "operator": "=", "value": 0, "threshold_only": true, "for_duration": 120000 }
  ],
  "actions": [
    [ { "type": "device.set-value", "device_feature": "lampe-couloir-binary", "value": 0 } ]
  ]
}
```

**C3. Notification sur ouverture de porte.**

```yaml
triggers:
  - trigger: state
    entity_id: binary_sensor.porte_entree_contact
    to: "on"
actions:
  - action: notify.telegram_famille
    data: { message: "La porte d'entrée est ouverte" }
```

```json
{
  "triggers": [
    { "type": "device.new-state", "device_features": ["porte-entree-opening"],
      "operator": "=", "value": 0, "threshold_only": true }
  ],
  "actions": [
    [ { "type": "message.send", "user": "alice", "service": "telegram",
        "text": "La porte d'entrée est ouverte" } ]
  ]
}
```

HA `on` (ouvert) devient Gladys **`0`** (capteur d'ouverture inversé, § 4.1). Le destinataire Telegram de `notify.telegram_famille` doit être confirmé par l'utilisateur ; une notification à plusieurs destinataires donne une action par utilisateur, dans le même groupe parallèle.

**C4. Chauffage du matin en semaine.**

```yaml
triggers:
  - trigger: time
    at: "06:30:00"
    weekday: [mon, tue, wed, thu, fri]
actions:
  - action: climate.set_temperature
    target: { entity_id: climate.salon }
    data: { temperature: 20 }
```

```json
{
  "triggers": [
    { "type": "time.changed", "scheduler_type": "every-week", "time": "06:30",
      "days_of_the_week": ["monday", "tuesday", "wednesday", "thursday", "friday"] }
  ],
  "actions": [
    [ { "type": "device.set-value", "device_feature": "thermostat-salon-target-temperature", "value": 20 } ]
  ]
}
```

Conditions : même fuseau horaire, même unité de température (E5).

**C5. Alerte de température, seulement le jour.**

```yaml
triggers:
  - trigger: numeric_state
    entity_id: sensor.chambre_temperature
    above: 28
    for: { minutes: 10 }
conditions:
  - condition: time
    after: "08:00:00"
    before: "22:00:00"
actions:
  - action: notify.ntfy
    data: { message: "Chambre au-dessus de 28 °C depuis 10 minutes" }
```

```json
{
  "triggers": [
    { "type": "device.new-state", "device_features": ["capteur-chambre-temperature"],
      "operator": ">", "value": 28, "threshold_only": true, "for_duration": 600000 }
  ],
  "actions": [
    [ { "type": "condition.check-time", "after": "08:00", "before": "22:00" } ],
    [ { "type": "message.send", "user": "alice", "service": "ntfy",
        "text": "Chambre au-dessus de 28 °C depuis 10 minutes" } ]
  ]
}
```

`above` → `>` (pas `>=`). La condition est la première action, en série.

**C6. Volets au coucher du soleil, sauf si la fenêtre est ouverte.**

```yaml
triggers:
  - trigger: sun
    event: sunset
    offset: "-00:15:00"
conditions:
  - condition: state
    entity_id: binary_sensor.fenetre_salon
    state: "off"
actions:
  - action: cover.close_cover
    target: { entity_id: cover.volet_salon }
```

```json
{
  "triggers": [ { "type": "time.sunset", "house": "maison", "offset": -15 } ],
  "actions": [
    [ { "type": "condition.only-continue-if",
        "conditions": [ { "device_feature": "fenetre-salon-opening", "operator": "=", "value": 1 } ] } ],
    [ { "type": "device.set-value", "device_feature": "volet-salon-state", "value": -1 } ]
  ]
}
```

Fenêtre fermée = HA `off` = Gladys `1`. Écarts E1 (fenêtre indisponible) et E4 (heure du coucher).

### 6.2 Cas non convertibles

| Cas | YAML (extrait) | Raison |
|---|---|---|
| N1. Lumière sur mouvement en une automatisation | `mode: restart` ; `wait_for_trigger: [{trigger: state, entity_id: binary_sensor.couloir_mouvement, to: "off"}]` ; `delay: 120` ; `light.turn_off` | `wait_for_trigger` sans équivalent ; `restart` impossible avec une attente (§ 5). Réécriture possible seulement via le motif optionnel § 5 |
| N2. Notification avec template | `message: "{{ trigger.to_state.name }} ouverte"` | template Jinja |
| N3. Chauffage si quelqu'un est à la maison | `condition: zone` / `state: person.alice = home` | présence non migrée (V2, 03 § 6) |
| N4. Arrosage toutes les 15 min | `trigger: time_pattern, minutes: "/15"` | `interval` non aligné sur l'horloge ; 96 déclencheurs fixes au-delà de la limite proposée |
| N5. Volets à l'élévation du soleil | `trigger: sun.elevation_crossed_threshold` | aucun équivalent |
| N6. Automatisation `choose` sur `trigger.id` | `condition: trigger, id: arrivee` | Gladys ignore quel déclencheur a lancé la scène (éclatement possible, § 7) |

---

## 7. Pièges et recommandations pour le convertisseur

### 7.1 Écarts et contradictions avec 00, 01, 02 et 03

| # | Constat | Document concerné |
|---|---|---|
| X1 | « Fidèle à 100 % » (D5, critère de réussite 00 § 10) est **inatteignable à la lettre** pour presque toute automatisation à cause de E1 (indisponibilité) et E2 (erreur d'action). Une décision doit fixer quels écarts sont acceptés et affichés | 00 § 5, § 8, § 10 |
| X2 | L'absence de **modes** est structurante : `single` (défaut HA) avec `delay` est impossible, alors que c'est la forme la plus fréquente d'une automatisation à délai (§ 5) | 00 § 8 |
| X3 | La **présence** est en V2 (03 § 6, M2) : toute automatisation à `person`, `zone`, `device_tracker` reste « à refaire » tant que M2 n'est pas livré | 03 § 6 |
| X4 | 03 § 3.4 associe des **entités** à des features, mais le convertisseur a aussi besoin : des **valeurs** (`on`/`off` → 1/0, inversions ouverture/serrure, états textuels), des **attributs** (`brightness`, `current_temperature`), des **événements d'appareil** (boutons) et des **échelles** pour les seuils et consignes | 03 § 3.4 |
| X5 | L'inventaire `Automation` de 01 § 4 doit aussi porter : `max`, `max_exceeded`, `initial_state`, l'état activé/désactivé dans HA, `variables`/`trigger_variables`, l'origine blueprint, les scripts et scènes HA appelés, les zones, et la configuration HA (fuseau, latitude, longitude, altitude, unités) | 01 § 4 |
| X6 | Les **scènes HA** (`scene.turn_on`) et les **scripts** sont appelés par de nombreuses automatisations, mais sont hors périmètre (00 § 4.2 ne cite que les scripts, « à réévaluer ») | 00 § 4.2 |
| X7 | 00 § 9 crée les scènes **désactivées** en coexistence. Une scène désactivée reste exécutable par `scene.start` depuis une autre scène ou à la main ([G] `scene.execute.js:19-35`) : en bascule progressive, activer A qui démarre B (désactivée) exécute B quand même | 00 § 9, 01 § 9.3 |
| X8 | `research/gladys-codebase.md` § 6.1 et § 9.3 laissaient deux points « non vérifiés », désormais tranchés : la concurrence de la file est **infinie** (`queue@6.0.2`), et HA **arrête** la séquence sur erreur ([HA] `script.py:646-668`). Le point MQTT « sans joker vérifié » est aussi tranché : un déclencheur à joker **ne se déclenche jamais** | research/gladys-codebase.md |

### 7.2 Recommandations

1. **Refus par défaut, liste blanche par élément.** Le convertisseur ne reconnaît que les lignes « exact » et « exact sous condition » du § 4, avec leurs conditions vérifiées. Tout le reste produit ✋ avec le **chemin** de l'élément fautif (`actions[2].choose[0].sequence[1]`) et un code de raison traduit par le front (01 § 9.2).
2. **Templates partout = refus.** Toute chaîne contenant `{{` ou `{%` (dans `data`, `target`, `for`, `delay`, `enabled`, `message`) refuse l'automatisation. Rappel : `raw_config` contient les templates non rendus (`research/ha-data-sources.md` § 1.5).
3. **`threshold_only: true` systématique** pour tout déclencheur `state`/`numeric_state` traduit en comparaison ; `>`/`<` stricts pour `above`/`below`.
4. **`device.set-value` plutôt que `light.*`/`switch.*` par appareil**, pour viser la feature exacte (canal).
5. **Appareils non migrés** : si toutes les autres règles passent, statut 🔗 « bloquée par un appareil » avec la liste des entités manquantes. L'ordre d'évaluation doit être : d'abord la convertibilité structurelle (✋ est définitif), puis les appareils (🔗 est temporaire), pour ne pas laisser croire qu'une automatisation à template deviendra convertible.
6. **`device_id` composites** : résoudre via `list_composite_splits` ou `composite_device_id` avant la recherche dans le journal (`research/ha-data-sources.md` § 2.1). Entités en UUID de registre : résoudre via le registre des entités.
7. **Blueprints** : l'automatisation substituée est convertie comme les autres (mode A/B). Indiquer l'origine blueprint dans le rapport : ces automatisations contiennent souvent des templates.
8. **Vérifications de configuration** avant toute conversion temporelle (E5) : fuseau HA = fuseau Gladys, coordonnées de la maison Gladys = coordonnées HA, unités identiques. Sinon, refuser les déclencheurs `time`/`sun` et les seuils concernés.
9. **Heures limites** : refuser `time` entre 02:00 et 03:00 (changement d'heure) et tout `sun` + offset qui sort de la journée civile (§ 4.2).
10. **Coexistence** : `active: false` à la création ; étiquette commune (ex. `Home Assistant`, sur le modèle de `AI`) pour la retrouver, l'activer en bascule et l'annuler (01 § 5.4). Signaler X7.
11. **Idempotence** : l'identité d'une scène est l'`id` HA de l'automatisation, ou son empreinte (02 § 6.3). Une automatisation HA **désactivée** est proposée, mais créée inactive dans les deux modes (proposition).
12. **Éclatement en plusieurs scènes** (option, à valider) : une automatisation `choose` sur `trigger.id` dont chaque branche ne dépend que de son déclencheur peut devenir une scène par déclencheur. Comme le motif « mouvement » (§ 5), c'est une **réécriture**, à présenter comme telle et à n'appliquer qu'avec accord explicite.
13. **Tests** : chaque ligne du § 4 doit avoir un test de conversion (YAML → JSON validé par le schéma Joi de [G] `server/models/scene.js`), et chaque verdict « impossible » un test de refus avec sa raison (01 § 10).
