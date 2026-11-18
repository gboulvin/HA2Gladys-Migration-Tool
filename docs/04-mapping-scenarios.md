# 04 — Conversion des scènes et automatisations

- **Statut** : 📝 Brouillon
- **Dernière mise à jour** : 2026-11-17
- **Prérequis** : [00 — Vision](00-vision.md) (D5), [01 — Architecture](01-architecture.md), [02 — Lecture HA](02-extraction-ha.md) (§ 6.3), [03 — Correspondances](03-mapping-devices.md)
- **Note de recherche** : [scene-mapping.md](research/scene-mapping.md) (« S § x »), qui contient la **table de correspondance complète**, élément par élément et sourcée, ainsi que les exemples YAML → JSON.

---

## 1. Ce que l'étude change

Trois constats obligent à préciser D5 (« fidélité à 100 %, sinon à refaire manuellement ») :

1. **Deux écarts touchent presque toutes les automatisations** (S § 4.0) :
   - **E1, indisponibilité** : HA connaît les états `unavailable` et `unknown`. Une condition y devient fausse, et un retour `unavailable → on` redéclenche. Gladys garde la dernière valeur connue.
   - **E2, erreur d'action** : HA **arrête** la séquence à la première action en erreur. Gladys **journalise et continue**.

   Lu à la lettre, D5 interdirait de convertir toute automatisation qui a un déclencheur d'état ou deux actions enchaînées, c'est-à-dire presque toutes.
2. **Gladys n'a pas de mode d'exécution** : toutes les exécutions d'une scène tournent en parallèle, sans limite (S § 1.4). Le mode `single` de HA, qui est le mode par défaut, n'est reproduit fidèlement que pour une séquence **sans attente**.
3. **Plusieurs différences fines** rendent impossibles des éléments qui semblaient simples :
   - une `condition` dans une branche arrête **toute** la scène dans Gladys, et seulement la branche dans HA ;
   - Gladys n'offre ni `wait_for_trigger`, ni condition sur le soleil, ni condition de durée ;
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
- toute référence à la **présence** (`person`, `zone`, `device_tracker`), qui est en V2 (M2) ;
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
| **Actions** | Commander un appareil : marche/arrêt, luminosité, position d'un volet, consigne de température. Lancer une scène HA migrée. Notification à **texte fixe**. |
| **Motif reconnu** | **Lumière sur détection de mouvement avec extinction après un délai**, réécrite en deux scènes (§ 5) |
| **Mode d'exécution** | Séquence sans attente uniquement (§ 3.4) |

**Exclu de la V1**, et donc présenté pour une recréation manuelle : `delay` et attentes hors du motif reconnu, `if`/`choose`/`repeat`/`parallel`, conditions OU/NON, déclencheurs MQTT, d'événement et de bouton, scripts, templates, présence.

Ce socle couvre la majorité des automatisations courantes (§ 8). Il réutilise **les mêmes briques** que la conversion des scènes HA (tables de valeurs, `device.set-value`), et il est testé ligne par ligne comme le reste (§ 9).

### 2.5 Automatisations non converties : aide à la recréation (S6)

Une automatisation non convertie n'est pas une impasse. C'est l'occasion, pour l'utilisateur, de **découvrir l'éditeur de scènes de Gladys**. Pour chacune, le rapport affiche une **fiche lisible** :

> **Volets du salon le soir** — *à recréer*
> - **Quand** : coucher du soleil
> - **Si** : quelqu'un est à la maison *(présence, non migrée)*
> - **Alors** : fermer « Volet salon » (Salon), attendre 10 minutes, éteindre « Lampe salon »
> - **Pourquoi elle n'a pas été convertie** : condition de présence
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
| Lever et coucher du soleil avec décalage : acceptés seulement si l'heure obtenue reste **dans la même journée** | Gladys recalcule ses tâches à minuit : « coucher + 4 h » ne se déclencherait jamais |
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
| Condition `sun` : refus **pour l'instant** | Pas d'équivalent dans le cœur, mais une intégration externe peut le combler (§ 6.1) |

### 3.3 Actions

| Règle | Pourquoi |
|---|---|
| Commande d'un appareil : **`device.set-value` sur la feature exacte** | `light.turn-on` de Gladys vise la **première** lumière de l'appareil, donc pas forcément le bon canal |
| Valeurs, attributs (luminosité, couleur…) et échelles : traduits par les tables **déclarées dans le catalogue** (§ 4) | Une porte « ouverte » côté HA vaut `0` dans Gladys, et les échelles diffèrent (luminosité 0-255 contre pourcentage) |
| `condition` à l'intérieur d'un `if`, `choose`, `parallel` ou `sequence` : refus | Gladys arrête **toute** la scène, HA seulement la branche |
| `delay` fixe : `delay`. `wait_template`, `wait_for_trigger` : refus | Pas d'attente d'événement dans Gladys |
| `repeat` avec un nombre fixe et raisonnable : actions **dépliées**. `while`, `until` : refus | `condition.while` a un plafond d'itérations et un minimum de 100 ms par tour |
| `notify` à texte fixe : `message.send`, après **confirmation** de la correspondance « service HA → utilisateur et canal Gladys » | Les services de notification n'ont pas d'équivalent automatique |
| Appel d'un **script** HA : contenu **déplié** dans la scène, si le script est lui-même convertible | Gladys n'a pas de script. Le mode propre du script est perdu, cela relève du § 3.4. |
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
| P4 | Condition **solaire** (jour ou nuit, avant ou après le lever ou le coucher) | Motifs d'éclairage très courants. **Peut être comblée sans toucher au cœur**, par une intégration externe (§ 6.1). |
| P5 | Condition de **durée** (« dans cet état depuis N minutes ») | Motifs d'alerte |
| P6 | **Variable du déclencheur** exposée dans l'éditeur | Automatisations à plusieurs rôles, sans réécriture |

### 6.1 Ce qu'une intégration externe peut combler : l'exemple du soleil

Les intégrations externes peuvent ajouter des **déclencheurs** et des **actions** de scène, mais pas de conditions. C'est un choix de la spec de Gladys : une action qui renvoie une valeur, suivie de `condition.only-continue-if`, couvre le besoin (`docs/specs/external-integrations/capabilities/scene-triggers-and-actions.md`). Surtout, elles peuvent publier des **appareils** dont les fonctions servent ensuite de conditions et de déclencheurs ordinaires.

**État du store (vérifié le 2026-11-18, 1 188 intégrations)** : aucune intégration n'expose aujourd'hui l'état « jour / nuit » ni l'élévation du soleil.
- `guim31/gladys-astronomy` calcule crépuscules et aubes, mais pour l'observation du ciel : déclencheurs d'éclipses, de nuits noires, d'étoiles filantes.
- `guim31/gladys-volets-soleil` calcule l'élévation et l'azimut, mais n'expose qu'un conseil de fermeture par façade et un déclencheur « le soleil arrive sur une façade ».

**Or HA traite le soleil comme un appareil** : l'entité `sun.sun` vaut `above_horizon` ou `below_horizon`, avec les attributs `elevation`, `azimuth`, `next_rising`… Une intégration externe **« Soleil »** publiant un appareil équivalent serait courte à écrire, puisque les calculs existent déjà dans les deux intégrations citées. Elle exposerait :
- une fonction binaire **« il fait jour »** ;
- l'**élévation** et l'**azimut**, en degrés.

Avec elle, sans modifier ni le cœur ni l'outil :
- la condition HA `sun` (après le coucher, avant le lever) devient une condition **d'état** sur « il fait jour » ;
- `state: sun.sun` et `numeric_state` sur `elevation`, en condition ou en déclencheur, deviennent des règles **d'appareil** ordinaires, dans le socle V1 ;
- le domaine HA `sun` entre au **catalogue** comme n'importe quelle intégration (03), avec cette cible.

L'écart E4 demeure : les calculs astronomiques diffèrent légèrement d'une bibliothèque à l'autre.

**Généralisation** : c'est le parcours D15 (« créez l'intégration ») appliqué aux scènes. Chaque fois qu'une condition HA porte sur une **entité calculée** (soleil, saison, jour ouvré, fenêtre de tarif…), une intégration qui expose cette valeur comme fonction d'appareil la rend convertible. Le rapport de l'outil doit donc distinguer deux cas de refus :
- **« impossible dans Gladys »** : modes, `wait_for_trigger`… ;
- **« possible avec une intégration qui n'existe pas encore »** : le soleil, par exemple, avec le même encouragement à la créer que pour les appareils.

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
| Chauffage selon la présence | ✋ (V2) |
| Alerte de température (seuil et durée) | ✅ |
| Volets au coucher du soleil avec décalage | ✅ (E4 affiché) |
| Volets selon l'élévation du soleil | ✋ aujourd'hui, ✅ avec une intégration « Soleil » (§ 6.1) |
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
| S5 | **V1** : scènes HA et **automatisations simples** (§ 2.4), dont le motif « lumière sur mouvement » (D26). |
| S6 | Les automatisations non converties sont présentées sous forme de **fiche lisible**, avec un brouillon de scène prérempli (§ 2.5). |
| S7 | Les automatisations élaborées relèvent d'un **assistant distinct** (V2), qui réutilise les fondations de la V1 (§ 2.6). |
| S4 | Les **scènes HA** (instantanés d'états) sont migrées **dès la V1**, en scènes Gladys sans déclencheur (§ 8 bis, ex-Q04-3). |
| S3 | Les pistes d'évolution du cœur (§ 6) sont suivies dans le **fil du forum** du projet. Le bogue P3 (déclencheur MQTT avec joker) fait l'objet d'un **ticket** sur `GladysAssistant/Gladys` (ex-Q04-4). |

## 11. Questions ouvertes

*Aucune pour l'instant.*
