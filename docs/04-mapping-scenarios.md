# 04 — Conversion des scènes et automatisations

- **Statut** : 📝 Brouillon
- **Dernière mise à jour** : 2026-11-17
- **Prérequis** : [00 — Vision](00-vision.md) (D5), [01 — Architecture](01-architecture.md), [02 — Lecture HA](02-extraction-ha.md) (§ 6.3), [03 — Correspondances](03-mapping-devices.md)
- **Note de recherche** : [scene-mapping.md](research/scene-mapping.md) (« S § x »), qui contient la **table de correspondance complète**, élément par élément et sourcée, ainsi que les exemples YAML → JSON.

---

## 1. Ce que l'étude change

Trois constats obligent à préciser D5 (« fidélité à 100 %, sinon à refaire manuellement ») :

1. **Deux écarts touchent presque toutes les automatisations** (S § 4.0) :
   - **E1, indisponibilité** : HA passe une entité à `unavailable` **dès** la perte de liaison. Une condition devient alors fausse, et un retour `unavailable → on` redéclenche. Gladys a bien une notion voisine, mais **différente et absente des scènes** :
     - « **Pas de valeur récente** » : calculée par le serveur (`last_value_is_too_old`, `server/lib/device/device.get.js`), elle signale une valeur plus ancienne qu'un **nombre d'heures** réglable (`DEVICE_STATE_NUMBER_OF_HOURS_BEFORE_STATE_IS_OUTDATED`). C'est une **ancienneté**, pas une perte de liaison immédiate. Elle n'est utilisée que par l'affichage, et aucun déclencheur ni aucune condition de scène ne la lit ;
     - `GLADYS_TRANSPORT = unreachable` : publié par certaines intégrations externes (pastille sur l'appareil), lui aussi absent des scènes.

     Il n'y a donc pas de correspondance pour les scènes aujourd'hui. Une condition ou un déclencheur « valeur récente / appareil joignable » rapprocherait les deux comportements : voir la piste P7 (§ 6).
   - **E2, erreur d'action** : HA **arrête** la séquence à la première action en erreur. Gladys **journalise et continue**.

   Lu à la lettre, D5 interdirait de convertir toute automatisation qui a un déclencheur d'état ou deux actions enchaînées, c'est-à-dire presque toutes.
2. **Gladys n'a pas de mode d'exécution** : toutes les exécutions d'une scène tournent en parallèle, sans limite (S § 1.4). Le mode `single` de HA, qui est le mode par défaut, n'est reproduit fidèlement que pour une séquence **sans attente**.
3. **Plusieurs différences fines** rendent impossibles des éléments qui semblaient simples :
   - une `condition` dans une branche arrête **toute** la scène dans Gladys, et seulement la branche dans HA ;
   - **une scène Gladys ne peut pas se mettre en pause jusqu'à un événement**. C'est ce que fait `wait_for_trigger` dans HA : *au milieu* d'une exécution, attendre par exemple « fin du mouvement », puis **continuer la même exécution**, avec un délai maximal facultatif. Un déclencheur Gladys, lui, **démarre** une scène, il ne la reprend pas. L'équivalent passe donc par **deux scènes** (réécriture de motif, § 5). Les attentes **fixes** (`delay`), elles, existent dans Gladys et sont converties ;
   - il n'y a pas, **dans le cœur**, de condition sur la position du soleil ni de condition « dans cet état depuis N minutes ». La première se comble par un **appareil** (§ 6.1). Les **déclencheurs** lever et coucher du soleil avec décalage (§ 3.1), les conditions d'heure, de plages horaires, de calendrier, de présence et d'alarme existent bien ;
   - un `time_pattern` `/n` ne peut pas être aligné sur l'horloge ;
   - un déclencheur MQTT avec joker ne se déclenche jamais dans Gladys.

## 2. Doctrine de conversion (S1)

### 2.1 Un fonctionnement similaire, écarts affichés

**Objectif : un fonctionnement similaire, pas identique.** Gladys n'est pas HA, et des différences sont admises. Le seuil reste exigeant. **On convertit une automatisation si son comportement est le même dans HA et dans Gladys en fonctionnement normal**, c'est-à-dire appareils joignables et commandes acceptées. Les écarts qui ne jouent qu'en cas de panne sont **acceptés et affichés** :

| Écart | Traitement |
|---|---|
| E1 — indisponibilité d'un appareil | Accepté. Mention sur chaque scène concernée : « Si un appareil devient indisponible, Gladys utilise sa dernière valeur connue. » |
| E2 — erreur d'une action | Accepté. Mention : « Si une action échoue, les suivantes sont exécutées quand même. » |
| E4 — heures de lever et de coucher du soleil légèrement différentes (bibliothèques astronomiques différentes, altitude ignorée par Gladys) | Accepté. Mention sur les scènes solaires. |
| **Tout autre écart** | **Refus**, statut « à refaire manuellement » avec la raison |

Ces mentions apparaissent dans le rapport (05, écran 3) et, si Gladys le permet, dans la description de la scène créée.

### 2.2 Liste blanche, refus par défaut

Le convertisseur ne reconnaît que les éléments notés « exact » ou « exact sous condition » dans la table de correspondance (S § 4.2 à 4.4), **conditions vérifiées**. Tout le reste conduit à un refus, avec :
- le **chemin** de l'élément fautif (par exemple `actions[2].choose[0].sequence[1]`) ;
- un **code de raison** traduit par le front.

L'utilisateur sait ainsi exactement quoi refaire.

**Refus immédiats**, avant toute analyse détaillée :
- toute chaîne contenant un **template** Jinja (`{{` ou `{%`), où qu'elle soit ;
- tout déclencheur `event`, `template`, `webhook`, `tag`, `conversation`, `sun` autre que lever et coucher, `time_pattern` en `/n`, MQTT avec joker.

### 2.3 Ordre d'évaluation et statuts

1. **Structure** : chaque élément est-il convertible ? Si non, ✋ **à refaire manuellement**. C'est **définitif** tant que Gladys n'évolue pas.
2. **Appareils** : chaque entité référencée est-elle migrée ? Si non, 🔗 **bloquée par un appareil**, avec la liste. C'est **temporaire** : le statut est réévalué à chaque relance (D13).
3. Sinon : ✅ **convertible**, avec la liste des écarts acceptés (§ 2.1).

Cet ordre évite de laisser croire qu'une automatisation à template deviendra convertible une fois ses appareils migrés.

### 2.4 Périmètre V1 : les automatisations simples (S5)

**Principe (D26)** : migrer tout ce qui est migrable **sans que cela devienne un casse-tête pour l'utilisateur**. La V1 convertit donc un **socle d'automatisations simples**, dont le résultat se comprend au premier coup d'œil dans l'éditeur de scènes de Gladys. Tout le reste est présenté de façon lisible, pour que l'utilisateur le recrée lui-même (§ 2.5). Les automatisations élaborées relèvent d'un **assistant distinct** (§ 2.6).

| | Accepté en V1 |
|---|---|
| **Déclencheurs** | Heure fixe, avec ou sans jours de la semaine. Lever et coucher du soleil, avec un décalage qui reste dans la journée. **Changement d'état d'un capteur binaire** (mouvement, ouverture, présence d'eau…), avec ou sans durée (« sans mouvement depuis 5 minutes »). Seuil simple sur une mesure (température, humidité…), avec ou sans durée. |
| **Conditions** | Plage horaire et jours de la semaine. État simple d'un appareil ou seuil simple. Plusieurs conditions **combinées en ET** uniquement. |
| **Actions** | Commander un appareil : marche/arrêt, luminosité, position d'un volet, consigne de température. Lancer une scène HA ou un **script** HA migrés (§ 8 ter). Notification à **texte fixe**. |
| **Motif reconnu** | **Lumière sur détection de mouvement avec extinction après un délai**, réécrite en deux scènes (§ 5) |
| **Mode d'exécution** | Séquence sans attente uniquement (§ 3.4) |

**Exclu de la V1**, et donc présenté pour une recréation manuelle : `delay` et attentes hors du motif reconnu, `if`/`choose`/`repeat`/`parallel`, conditions OU/NON, déclencheurs MQTT, d'événement et de bouton, scripts **non simples** (§ 8 ter), templates.

Ce socle couvre la majorité des automatisations courantes (§ 8). Il réutilise **les mêmes briques** que la conversion des scènes HA (tables de valeurs, `device.set-value`), et il est testé ligne par ligne comme le reste (§ 9).

### 2.5 Automatisations non converties : aide à la recréation (S6)

Une automatisation non convertie n'est pas une impasse. C'est l'occasion, pour l'utilisateur, de **découvrir l'éditeur de scènes de Gladys**. Pour chacune, le rapport affiche une **fiche lisible** :

> **Volets du salon le soir** — *à recréer*
> - **Quand** : coucher du soleil
> - **Si** : la température extérieure est sous 5 °C *(capteur non migré)*
> - **Alors** : fermer « Volet salon » (Salon), attendre 10 minutes, éteindre « Lampe salon »
> - **Pourquoi elle n'a pas été convertie** : appareil non migré
> - Bouton **« Créer un brouillon dans Gladys »** : crée une scène **désactivée**, préremplie de tout ce qui est convertible (déclencheur, actions), et ouvre l'éditeur. L'utilisateur complète le reste, puis active la scène.

- Les éléments sont décrits avec les **noms Gladys** des appareils migrés, pas avec les identifiants HA.
- La fiche est produite à partir des mêmes tables que la conversion : les éléments connus sont formulés en clair, les autres sont cités tels quels (« template », « script `xyz` »).
- Le brouillon n'est créé qu'à la demande de l'utilisateur, et **désactivé** : il ne se déclenche pas tant que l'utilisateur ne l'a pas activé. C'est le fonctionnement normal de Gladys, où l'on crée d'abord une scène, puis on la complète dans l'éditeur (`front/src/actions/createScene.js`). Le brouillon porte l'étiquette « Home Assistant », figure au journal, et est donc supprimé en cas d'annulation (A9).

### 2.6 Assistant de migration des automatisations (V2, S7)

Les automatisations élaborées feront l'objet d'un **assistant distinct**, lancé depuis **Paramètres → Migration** une fois la migration de base faite. Ce choix se justifie ainsi :
- le fonctionnement des automatisations HA et des scènes Gladys diffère profondément (§ 1). L'assistant évoluera au rythme du **moteur de scènes de Gladys** (pistes du § 6, notamment les modes d'exécution) ;
- il **réutilise** les fondations de la V1 : inventaire, journal, catalogue, liste blanche, fiches lisibles. Il ne fait qu'élargir la liste blanche et ajouter des réécritures de motifs ;
- la migration de base reste **simple et rapide**. L'utilisateur prend la main sur Gladys avec ses fonctionnements essentiels, puis peaufine à son rythme.

Le reste de cette section (§ 3 à § 7) décrit l'**ensemble** des règles de conversion. La V1 en applique le sous-ensemble du § 2.4, et l'assistant V2 l'étendra.

## 3. Règles de traduction essentielles

La table complète est dans la note de recherche (S § 4). Voici les règles qui demandent une attention particulière.

### 3.1 Déclencheurs

| Règle | Pourquoi |
|---|---|
| Tout déclencheur d'état ou de seuil devient `device.new-state` **avec `threshold_only: true`** | Sans cette option, Gladys déclenche à **chaque rapport** de l'appareil, même si la valeur n'a pas changé. Avec une durée (`for_duration`), il déclencherait en boucle. |
| `above` / `below` deviennent `>` / `<`, **jamais** `>=` / `<=` | HA compare strictement |
| `state` **sans** `from` ni `to` : refus | HA se déclenche aussi sur un changement d'**attribut**, pas Gladys |
| Lever et coucher du soleil, avec ou sans décalage (« 30 minutes avant le lever ») : déclencheurs **natifs** de Gladys (`time.sunrise`, `time.sunset`), convertis directement. Une seule limite : l'heure obtenue doit rester **dans la même journée**. | Gladys recalcule ses tâches à minuit : « coucher + 4 h » (après minuit) ou « lever − 8 h » (la veille) ne se déclencheraient jamais. Les décalages courants (quelques minutes à quelques heures) ne sont pas concernés. |
| `time` entre 02:00 et 03:00 : refus | Changement d'heure : saut ou double exécution, selon le système |
| `homeassistant: start` devient `system.start` | Accepté, avec mention : c'est le démarrage de **Gladys** |

### 3.2 Conditions

| Règle | Pourquoi |
|---|---|
| Liste de conditions (ET implicite de HA) : **une action de condition par condition**, en série | `condition.only-continue-if` combine ses conditions en **OU** |
| `or` de comparaisons simples : **une** action à plusieurs conditions | OU natif |
| `or` contenant un `and` : refus | Pas de ET imbriqué dans un OU |
| `not` : opérateur inversé (`=` ↔ `!=`, `>` ↔ `<=`) | Exact, hors E1 |
| Condition de **durée** (`for`), `zone`, `template`, `trigger` : refus | Pas d'équivalent |
| Condition `sun` : convertie **via un substitut** (appareil « il fait jour »), sinon refus | Pas d'équivalent dans le cœur, mais un appareil peut le fournir (§ 6.1) |

### 3.3 Actions

| Règle | Pourquoi |
|---|---|
| Commande d'un appareil : **`device.set-value` sur la feature exacte** | `light.turn-on` de Gladys vise la **première** lumière de l'appareil, donc pas forcément le bon canal |
| Valeurs, attributs (luminosité, couleur…) et échelles : traduits par les tables **déclarées dans le catalogue** (§ 4) | Une porte « ouverte » côté HA vaut `0` dans Gladys, et les échelles diffèrent (luminosité 0-255 contre pourcentage) |
| `condition` à l'intérieur d'un `if`, `choose`, `parallel` ou `sequence` : refus | Gladys arrête **toute** la scène, HA seulement la branche |
| `delay` fixe : `delay`. `wait_template`, `wait_for_trigger` : refus | Pas d'attente d'événement dans Gladys |
| `repeat` avec un nombre fixe et raisonnable : actions **dépliées**. `while`, `until` : refus | `condition.while` a un plafond d'itérations et un minimum de 100 ms par tour |
| `notify` à texte fixe : `message.send`, après **confirmation** de la correspondance « service HA → utilisateur et canal Gladys » | Les services de notification n'ont pas d'équivalent automatique |
| Appel d'un **script** HA : `scene.start` vers la scène issue du script (§ 8 ter), en **dernière** position. Si l'appel n'est pas en dernière position et que le script est court et sans attente, son contenu est **déplié** dans la scène appelante. | Gladys n'attend pas la fin d'une scène lancée par `scene.start`, alors que HA attend la fin d'un script appelé directement. |
| Activation d'une **scène HA** (`scene.turn_on`) : actions `device.set-value` en parallèle, reproduisant les états de la scène | Si chaque état est convertible (Q04-3) |
| `automation.trigger` / `script.turn_on` : `scene.start`, **en dernière position** | Gladys n'attend pas la scène lancée. Pour `automation.trigger`, la cible ne doit pas avoir de condition, puisque HA les ignore par défaut. |

### 3.4 Modes d'exécution

| Mode HA | Converti si… |
|---|---|
| `single` (défaut), `restart`, `queued`, `parallel` | La séquence est **sans attente** : ni `delay`, ni `wait`, ni `repeat` long, ni `scene.start` attendu. Deux exécutions ne peuvent alors pas se chevaucher utilement. |
| `queued` | Idem, **et** sans action dépendant de l'état (`toggle`, condition sur une valeur modifiée par la scène elle-même) |
| Tout mode, **avec** attente | Refus, sauf `parallel` dont le plafond ne peut pas être atteint |

**Conséquence** : la plupart des automatisations « avec délai » sont refusées tant que Gladys n'a pas de mode d'exécution. C'est la première piste d'évolution du cœur (§ 6).

## 4. Compléments au catalogue (03) et à l'inventaire (01)

La conversion des scènes demande au catalogue (03 § 3.4) plus que la correspondance entité → feature :

| Ajout au niveau 2 | Exemple |
|---|---|
| **Tables de valeurs** | `on`/`off` → `1`/`0`, porte `on` (ouverte) → `0`, serrure, modes de chauffage |
| **Attributs** | `brightness`, `current_temperature`, `color_temp_kelvin` → feature et conversion |
| **Événements d'appareil** | Bouton `remote_button_short_press` → valeur de la feature bouton |
| **Échelles** | Luminosité HA 0-255 → échelle de la feature Gladys |

L'**inventaire** (01 § 4) doit aussi porter, pour chaque automatisation :
- `mode`, `max`, et l'état **activé ou désactivé** dans HA ;
- l'**origine blueprint** ;
- les scripts et scènes HA appelés ;
- la **configuration HA** (fuseau horaire, coordonnées, unités). Si elle diffère de celle de Gladys, les déclencheurs horaires et solaires et les seuils concernés sont refusés.

## 5. Réécritures de motifs (S2)

Certains motifs très courants sont impossibles **élément par élément**, mais leur **comportement observable** est reproductible autrement. Le cas typique est la lumière sur détection de mouvement :

```yaml
# HA : une automatisation
mode: restart
triggers: [{ trigger: state, entity_id: binary_sensor.couloir, to: "on" }]
actions:
  - action: light.turn_on
    target: { entity_id: light.couloir }
  - wait_for_trigger: [{ trigger: state, entity_id: binary_sensor.couloir, to: "off" }]
  - delay: 120
  - action: light.turn_off
    target: { entity_id: light.couloir }
```

Il devient **deux scènes Gladys** :
1. mouvement = 1 → allumer ;
2. mouvement = 0 → éteindre, avec l'option de déclencheur **« Exécuter la scène après que la condition a été valide pendant : 2 minutes »** (`for_duration`, avec `threshold_only`).

La minuterie de Gladys est annulée par un retour du mouvement, exactement comme le `restart` de HA.

**Ce n'est plus une traduction, c'est une réécriture.** Elle n'est fidèle que si la lumière n'est pilotée que par ce motif. Proposition :
- une **bibliothèque fermée de motifs reconnus**, chacun couvert par des tests de comportement. Pour commencer : lumière sur mouvement, et automatisation à plusieurs rôles éclatée par déclencheur (`choose` sur `trigger.id`) ;
- une réécriture **jamais appliquée d'office**. Elle est proposée sous un statut distinct, 🔁 « convertible par réécriture », avec l'explication et l'aperçu des scènes produites. L'utilisateur l'accepte ou la refuse, scène par scène.

## 6. Pistes d'évolution du cœur de Gladys

L'étude fait apparaître des manques dont la correction augmenterait directement la couverture. Ils sont **hors du périmètre** de l'outil, mais méritent d'être remontés à Pierre-Gilles :

| # | Piste | Effet sur la migration |
|---|---|---|
| P1 | **Modes d'exécution** des scènes (`single`, `restart`, `queued`) | Débloquerait la plupart des automatisations avec délai. C'est le gain le plus important. |
| P2 | Action **« arrêter une scène »** | Débloquerait des motifs à minuterie |
| P3 | **Déclencheur MQTT avec jokers** (`+`, `#`) | Aujourd'hui, un tel déclencheur ne se déclenche jamais. Gladys s'abonne bien au filtre avec joker, mais compare ensuite le topic **reçu** au filtre par égalité stricte (`server/lib/scene/scene.addScene.js:240-253`, `scene.triggers.js:250-251`). **Vérifié dans le code : c'est un bogue, à signaler séparément.** |
| P4 | Condition **solaire** (jour ou nuit, avant ou après le lever ou le coucher) | Motifs d'éclairage très courants. **Déjà comblée en pratique par un appareil** (virtuel ou intégration), § 6.1 : piste de confort seulement. |
| P5 | Condition de **durée** (« dans cet état depuis N minutes ») | Motifs d'alerte |
| P6 | **Variable du déclencheur** exposée dans l'éditeur | Automatisations à plusieurs rôles, sans réécriture |
| P7 | Condition ou déclencheur **« appareil joignable / valeur récente »**, s'appuyant sur `last_value_is_too_old` ou `GLADYS_TRANSPORT` | Rapprocherait le comportement de HA en cas d'indisponibilité (E1), et utile en soi : « alerter si le capteur ne répond plus » |

### 6.1 Substituts : ce que le cœur ne fournit pas, un appareil peut le fournir (S8)

**Principe.** Dans Gladys, ce que le cœur ne fournit pas est couramment comblé par un **appareil** : un appareil virtuel ou une intégration externe. La plupart des utilisateurs ont, par exemple, un appareil qui indique s'il fait jour ou nuit. Une fois cet appareil en place, ses fonctions servent de **déclencheurs et de conditions ordinaires**. Or HA fonctionne de la même façon : le soleil (`sun.sun`), les jours ouvrés (`binary_sensor.workday`) ou les « entrées » créées par l'utilisateur (`input_boolean`, `input_number`…) sont des **entités**.

Le convertisseur ne s'arrête donc pas à « pas d'équivalent dans le cœur ». Pour toute entité HA référencée par une automatisation, il cherche un **substitut**, dans cet ordre :

| # | Substitut | Exemple | Comment |
|---|---|---|---|
| 1 | **Appareil Gladys existant**, désigné par l'utilisateur | « Il fait jour » d'un appareil virtuel déjà en place | L'outil demande **une seule fois** : « Quel appareil indique s'il fait jour ? » Le choix vaut pour toutes les automatisations et est mémorisé dans le journal. |
| 2 | **Appareil virtuel créé par l'outil** à partir d'une entrée HA (*helper*) | `input_boolean.mode_vacances` → interrupteur virtuel « Mode vacances » | Migration de l'entrée comme un appareil (§ 6.2) |
| 3 | **Intégration externe** du store | `binary_sensor.workday` → intégration « Holidays & Workdays » (`guim31/gladys-holidays`) | Comme pour tout appareil : catalogue, installation, rapprochement (03) |
| 4 | **Aucun substitut** | Capteur `template` de HA, calculé par une formule Jinja | Refus : « à recréer » |

Dès qu'un substitut existe, les éléments qui le visent sont traités comme des **règles d'appareil ordinaires**, et entrent dans le socle V1 (§ 2.4) :

| HA | Avec substitut |
|---|---|
| Condition `sun` : `after: sunset`, `before: sunrise` (sans décalage) | Condition d'état sur « il fait jour » = 0 |
| Condition `state: sun.sun = above_horizon` | Condition d'état sur « il fait jour » = 1 |
| Déclencheur `numeric_state` sur l'élévation du soleil | Déclencheur de seuil, si le substitut expose l'élévation |
| Condition sur `binary_sensor.workday` | Condition d'état sur la fonction « jour ouvré » |
| Déclencheur ou condition sur `input_boolean` | Déclencheur ou condition d'état sur l'interrupteur virtuel |
| Action `input_boolean.turn_on` / `input_number.set_value` | `device.set-value` sur l'appareil virtuel |

Les **décalages** d'une condition `sun` (`after_offset`, `before_offset`) n'ont pas d'équivalent sur un état binaire, et restent refusés. L'écart E4 (calculs astronomiques) demeure, et il est affiché.

**État du store** (vérifié le 2026-11-18, 1 188 intégrations) : aucune intégration n'expose encore « il fait jour » ni l'élévation du soleil. `guim31/gladys-astronomy` et `guim31/gladys-volets-soleil` font les calculs, sans publier ces valeurs. Le substitut n° 1 (appareil existant de l'utilisateur) couvre le cas en attendant. Une intégration « Soleil » dédiée serait courte à écrire, et proposée selon le parcours D15.

**Dans le rapport**, un refus faute de substitut est distingué d'un refus définitif :
- **« impossible dans Gladys »** : modes d'exécution, `wait_for_trigger`… ;
- **« possible avec un appareil »** : indiquer l'appareil existant qui convient, ou installer ou créer l'intégration.

### 6.2 Entrées HA (*helpers*) → appareils virtuels — V1 (S9)

Les entrées HA sont très présentes dans les automatisations : mode vacances, mode nuit, consigne réglable, compteur… Gladys les remplace par des **appareils virtuels** (« Appareils virtuels MQTT » : appareils créés à la main, pilotables dans les scènes et sur le tableau de bord).

| Entrée HA | Appareil virtuel Gladys |
|---|---|
| `input_boolean` | Fonction interrupteur (binaire) |
| `input_number` | Fonction numérique, avec les mêmes min, max, pas et unité |
| `counter` | Fonction numérique entière |
| `input_select`, `input_text`, `input_datetime`, `timer`, `schedule` | Pas d'équivalent simple : refus en V1 |

- Valeur initiale : l'état actuel dans HA.
- Prérequis : le service MQTT de Gladys configuré, avec un broker. Gladys sait installer le sien. Une commande sur un appareil virtuel publie sur MQTT (`server/services/mqtt/lib/setValue.js`) et échoue sans broker.
- Ces appareils figurent au journal comme les autres (annulation A9) et portent l'étiquette « Home Assistant ».
- *À vérifier dans le code de Gladys : création programmatique d'un appareil virtuel MQTT, et enregistrement de l'état sans retour (`has_feedback: false`).*

**Décision (S9, D27)** : les entrées HA sont migrées **dès la V1**, avec les appareils (05, écran 6). Cela rend convertibles une grande partie des automatisations « à modes » (vacances, nuit, absence) du socle V1.

### 6.3 Présence (D30)

La présence passe en **V1** : sans elle, une grande partie des automatisations courantes (chauffage, alarme, lumières à l'arrivée) resterait à recréer. Correspondances :

| HA | Gladys | Statut |
|---|---|---|
| `person` | Utilisateur (déjà en V1, 03 § 6) | ✅ |
| `device_tracker` **réseau** (routeur : FRITZ!Box, UniFi, ping, nmap) ou **Bluetooth** | Capteur de présence (`presence-sensor`) du service `lan-manager` ou `bluetooth`, ou d'une intégration du store (FRITZ!Box, UniFi), **rattaché à l'utilisateur**. Rapprochement par **adresse MAC**, comme pour les appareils (03). | ✅ |
| `device_tracker` **GPS** (application mobile HA) | Pas de reprise automatique : l'utilisateur configure la localisation côté Gladys. *Sources de localisation disponibles dans Gladys à vérifier (application, Gladys Plus, OwnTracks).* | ⚠️ guidé |
| `zone.home` | La maison Gladys (coordonnées) | ✅ |
| Autres `zone` (travail, école…) | **Zones** Gladys (`server/lib/area`) : centre et rayon | ✅ si la localisation de l'utilisateur est configurée |

| Automatisation HA | Scène Gladys |
|---|---|
| `person` passe à `home` / quitte `home` | Déclencheurs `user.back-home` / `user.left-home` |
| Zone : entrée ou sortie d'une personne | Déclencheurs d'entrée ou de sortie de zone |
| `zone.home` à 0 (dernier parti) / ≥ 1 | Conditions `house.is-empty` / `house.is-not-empty` |
| Condition « la personne X est à la maison » | *Condition par utilisateur : à vérifier dans le moteur de scènes. À défaut, substitut (§ 6.1).* |

**Écart affiché** : HA et Gladys ne déterminent pas la présence de la même façon (délais de détection, règles de consolidation entre plusieurs traqueurs). Le comportement est **similaire**, pas identique, ce qui est conforme à S1.

## 7. Création des scènes

- **Coexistence** : scènes créées **désactivées** (D4), avec une étiquette commune « Home Assistant », comme le fait le service MCP de Gladys avec l'étiquette `AI`. Elle permet de les retrouver, de les activer en bascule et de les supprimer en cas d'annulation (A9).
- ⚠️ Une scène désactivée **reste exécutable** par `scene.start` depuis une autre scène, ou à la main (S § 7.1, X7). En coexistence, l'outil ne crée donc **pas** d'appel `scene.start` vers une scène importée **tant que la cible n'est pas activée**. L'appel est créé au moment de la bascule.
- **Bascule** : activation proposée scène par scène (05 § 5). Une automatisation **désactivée dans HA** reste désactivée dans Gladys.
- **Identité** : l'`id` HA de l'automatisation, ou son empreinte si elle n'en a pas (02 § 6.3).
- **Validation** : chaque scène produite est validée par le schéma Joi de Gladys (`server/models/scene.js`) avant création. Un échec est une **erreur du convertisseur**, signalée comme telle et jamais présentée comme un refus normal.

## 8. Couverture attendue

Estimation qualitative, faute de statistiques d'usage des automatisations HA (S § 6) :

| Motif courant | Statut attendu |
|---|---|
| Allumer sur mouvement (sans extinction) | ✅ |
| Éteindre après N minutes sans mouvement (`to: off`, `for`) | ✅ |
| Lumière sur mouvement avec extinction, en **une** automatisation | 🔁 réécriture (§ 5), ✅ si P1 |
| Notification d'ouverture de porte, texte fixe | ✅ |
| Notification avec le nom de l'entité (template) | ✋ |
| Chauffage selon l'heure et les jours | ✅ |
| Chauffage selon la présence (« dernier parti », « premier arrivé ») | ✅ si la présence est migrée (§ 6.3) |
| Alerte de température (seuil et durée) | ✅ |
| Volets au coucher du soleil avec décalage | ✅ (E4 affiché) |
| Volets selon l'élévation du soleil | ✅ si un appareil expose l'élévation (§ 6.1), sinon ✋ |
| Lumière seulement s'il fait nuit (condition `sun`) | ✅ avec l'appareil « il fait jour » de l'utilisateur (§ 6.1) |
| Automatisations à modes (vacances, nuit) via `input_boolean` | ✅ si les entrées sont migrées en appareils virtuels (§ 6.2) |
| Bouton sans fil (Zigbee) | ✅ si le catalogue déclare les événements (§ 4) |
| Automatisations à plusieurs rôles (`choose` sur `trigger.id`) | 🔁 réécriture |
| Tout ce qui contient un template | ✋ |

## 8 bis. Scènes HA (instantanés d'états) — V1 (S4)

**À ne pas confondre** : une scène HA est un **instantané d'états** (« Soirée cinéma » = lampe du salon à 20 %, volets fermés), sans déclencheur ni condition. L'équivalent Gladys d'une automatisation HA est une scène Gladys, mais l'équivalent d'une scène HA est une **scène Gladys sans déclencheur**, que l'utilisateur lance à la main (bouton du tableau de bord, application, assistant vocal).

**Pourquoi dès la V1** : ces « ambiances » sont très utilisées. Leur conversion est **simple**, car elle ne demande ni déclencheur, ni condition, ni mode d'exécution. Il suffit d'une liste d'actions `device.set-value` en parallèle. Le schéma de Gladys accepte une scène sans déclencheur (`triggers` est un tableau, éventuellement vide, `server/models/scene.js`).

| Point | Règle |
|---|---|
| **Source** | Mode A et B : entités `scene.*` (`get_states`) et configuration par la route REST de HA `/api/config/scene/config/<id>` (administrateur, lit `scenes.yaml`). Mode C : `scenes.yaml` de la sauvegarde. *Détail à ajouter en 02.* |
| **Périmètre** | Scènes **créées dans HA** (éditeur ou YAML), avec un `id`. Les scènes **fournies par une intégration** (scènes Hue, Tuya…) sont exclues : elles vivent dans l'appareil ou le cloud du fabricant. Elles reviennent à l'intégration Gladys correspondante, si elle les expose. |
| **Traduction** | Une action `device.set-value` par fonction, toutes **en parallèle** (même groupe), comme HA applique les états d'une scène simultanément |
| **États et attributs pris en charge en V1** | Marche / arrêt (lumières, prises, interrupteurs, ventilateurs), **luminosité**, **position** et ouverture / fermeture des volets, **consigne** de température. Couleur et température de couleur **si le catalogue déclare la conversion** pour l'intégration (§ 4). |
| **Attribut non convertible** (effet, transition, mode de chauffage sans table…) | La scène est ⚠️ **partielle**. Le rapport liste ce qui manque, et l'utilisateur choisit de l'importer ainsi ou non. |
| **Appareil non migré** | 🔗 bloquée par un appareil, comme pour les automatisations (§ 2.3) |
| **Activation** | Une scène sans déclencheur ne se lance jamais toute seule. Elle est donc créée **active dans les deux modes** : pas de risque de double exécution en coexistence. |
| **Nom et étiquette** | Nom de la scène HA, étiquette « Home Assistant » (§ 7) |
| **Appel depuis une automatisation (lot 2)** | Les actions de la scène sont recopiées dans la scène Gladys appelante (§ 3.3), ou bien la scène importée est lancée par `scene.start` en dernière position. *Choix à faire au lot 2.* |

**Conséquence sur la V1** : les **tables de valeurs et d'attributs** du catalogue (§ 4) passent en V1, pour le sous-ensemble ci-dessus. Elles servent aussi aux automatisations simples (§ 2.4).

## 8 ter. Scripts HA simples — V1 (S11)

**Ce qu'est un script HA** : une **suite d'actions nommée, sans déclencheur**, avec la même syntaxe que la partie « actions » d'une automatisation. On le lance à la main (bouton, assistant vocal), depuis une automatisation ou depuis un autre script. Il peut avoir des **paramètres** (`fields`) et un **mode d'exécution**.

**Équivalent Gladys** : une **scène sans déclencheur**, comme pour les scènes HA (§ 8 bis), lancée à la main ou par `scene.start`.

| Point | Règle |
|---|---|
| **Source** | Mode A et B : WebSocket `script/config` (administrateur). Mode C : `scripts.yaml` (02, R § 1.6). |
| **Périmètre V1** | Scripts **sans paramètres** (`fields`), dont toutes les actions sont dans la liste blanche du socle (§ 2.4), **attentes fixes (`delay`) comprises** |
| **Attentes et mode** | Un script n'est lancé qu'à la main ou par un appel : deux exécutions simultanées ne surviennent que sur un double appel. C'est un écart **affiché**, pas un refus (S1). |
| **Activation** | Scène créée **active** dans les deux modes : sans déclencheur, elle ne se lance jamais seule. |
| **Appels** | Une automatisation ou un script qui appelle un script migré utilise `scene.start` (§ 3.3). C'est ce qui rend convertibles les nombreuses automatisations qui se contentent d'appeler un script. |
| **Hors V1** | Scripts à paramètres, avec templates, `choose`, `repeat`, `wait_for_trigger` : **fiche lisible** et brouillon (§ 2.5), puis assistant V2 (§ 2.6) |
| **Tableau de bord** | Les scripts migrés figurent, comme les scènes HA, dans la section « Ambiances » de l'Accueil (08) |

**Scripts à paramètres, pistes pour l'assistant V2.** Gladys n'a pas de scène paramétrée, mais deux techniques s'en approchent :
1. **Appareil virtuel servant de variable**, une pratique courante chez les utilisateurs de Gladys. L'appelant écrit la valeur dans une fonction de l'appareil virtuel, puis lance la scène, qui la lit. C'est simple et visible dans l'interface. En revanche, deux appels simultanés s'écrasent mutuellement la valeur.
2. **Portée transmise par `scene.start`** : la scène lancée reçoit une **copie** de la portée de la scène appelante (`server/lib/scene/scene.actions.js`, action `scene.start`), y compris les valeurs posées par `variable.set`. Il n'y a pas de conflit entre appels simultanés. Mais les valeurs sont repérées par leur **position** dans la scène appelante (`{{colonne.ligne.value}}`), ce qui demande une convention stricte entre l'appelant et la scène appelée. *À expérimenter.*

## 9. Tests

En complément de 06 :
- **chaque ligne** de la table de correspondance a un test de conversion : YAML HA → JSON validé par le schéma Joi de Gladys ;
- **chaque refus** a un test qui vérifie la raison et le chemin de l'élément ;
- les **réécritures** ont des tests de **comportement** : séquences d'événements simulées et vérification des commandes émises, comparées au comportement HA attendu.

## 10. Décisions

| # | Décision |
|---|---|
| S1 | Objectif : un **fonctionnement similaire**. Les écarts propres à Gladys sont admis et **affichés** : indisponibilité, erreur d'action, heures solaires. Le reste suit la liste blanche (§ 2, ex-Q04-1). |
| S2 | **Réécritures de motifs** retenues : bibliothèque fermée, statut distinct, accord scène par scène (§ 5, ex-Q04-2). |
| S11 | Les **scripts HA simples** (sans paramètres, actions du socle, attentes fixes comprises) sont migrés **dès la V1** en scènes sans déclencheur (§ 8 ter, D31). |
| S10 | La **présence** passe en **V1** (§ 6.3, D30) : traqueurs réseau et Bluetooth rapprochés par MAC, zones, déclencheurs d'arrivée et de départ, conditions de maison vide ou occupée. |
| S9 | Les **entrées HA** (`input_boolean`, `input_number`, `counter`) sont migrées **dès la V1** en appareils virtuels (§ 6.2, D27). |
| S8 | Le convertisseur cherche un **substitut** (appareil existant, appareil virtuel, intégration) à toute entité HA qui n'a pas d'équivalent dans le cœur, avant de refuser (§ 6.1). |
| S5 | **V1** : scènes HA et **automatisations simples** (§ 2.4), dont le motif « lumière sur mouvement » (D26). |
| S6 | Les automatisations non converties sont présentées sous forme de **fiche lisible**, avec un brouillon de scène prérempli (§ 2.5). |
| S7 | Les automatisations élaborées relèvent d'un **assistant distinct** (V2), qui réutilise les fondations de la V1 (§ 2.6). |
| S4 | Les **scènes HA** (instantanés d'états) sont migrées **dès la V1**, en scènes Gladys sans déclencheur (§ 8 bis, ex-Q04-3). |
| S3 | Les pistes d'évolution du cœur (§ 6) sont suivies dans le **fil du forum** du projet. Le bogue P3 (déclencheur MQTT avec joker) fait l'objet d'un **ticket** sur `GladysAssistant/Gladys` (ex-Q04-4). |

## 11. Questions ouvertes

*Aucune pour l'instant.*
