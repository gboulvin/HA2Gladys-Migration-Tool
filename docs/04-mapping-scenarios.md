# 04 — Conversion des automatisations en scènes (lot 2)

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
| Condition de **durée** (`for`), `sun`, `zone`, `template`, `trigger` : refus | Pas d'équivalent |

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
| P4 | Condition **solaire** (jour ou nuit, avant ou après le lever ou le coucher) | Motifs d'éclairage très courants |
| P5 | Condition de **durée** (« dans cet état depuis N minutes ») | Motifs d'alerte |
| P6 | **Variable du déclencheur** exposée dans l'éditeur | Automatisations à plusieurs rôles, sans réécriture |

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
| Volets selon l'élévation du soleil | ✋ |
| Bouton sans fil (Zigbee) | ✅ si le catalogue déclare les événements (§ 4) |
| Automatisations à plusieurs rôles (`choose` sur `trigger.id`) | 🔁 réécriture |
| Tout ce qui contient un template | ✋ |

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
| S3 | Les pistes d'évolution du cœur (§ 6) sont suivies dans le **fil du forum** du projet. Le bogue P3 (déclencheur MQTT avec joker) fait l'objet d'un **ticket** sur `GladysAssistant/Gladys` (ex-Q04-4). |

## 11. Questions ouvertes

| # | Question |
|---|---|
| Q04-3 | **Scènes HA** : les migrer aussi **pour elles-mêmes** (scène Gladys sans déclencheur, lancée à la main), en plus de leur traduction dans les automatisations qui les appellent ? |
