# 03 — Correspondances : catalogue, appareils, pièces, personnes, services

- **Statut** : 📝 Brouillon
- **Dernière mise à jour** : 2026-11-17
- **Prérequis** : [00 — Vision](00-vision.md), [01 — Architecture](01-architecture.md) (PR #1), [02 — Lecture HA](02-extraction-ha.md) (PR #2)
- **Notes de recherche** : [clés de rapprochement](research/matching-keys.md) (« K § x »), [usage des intégrations](research/ha-integrations-usage.md), [code Gladys](research/gladys-codebase.md)
- **À relire avec** : William-De71, auteur du [tracker des intégrations](https://william-de71.github.io/gladys-integrations-tracker/) (D21)

---

## 1. Objet

Cette section définit :
1. le **format du catalogue** de correspondances (D12, D21) ;
2. les règles de **rapprochement des appareils** et de **correspondance des fonctions** (D16) ;
3. le traitement des **pièces**, des **personnes** et des **intégrations de type service** (D11, D14, D20).

## 2. Le catalogue en deux niveaux

### 2.1 Pourquoi deux niveaux

Le tracker de William travaille **par intégration** : à chaque intégration HA, il associe l'intégration Gladys qui l'équivaut. Notre outil a besoin d'aller **au niveau de chaque appareil** : comment le retrouver, quelle entité HA correspond à quelle fonction Gladys, quelles clés de configuration reprendre.

Ces règles fines sont connues de **l'auteur de chaque intégration Gladys**, pas d'un mainteneur central. D'où deux niveaux :

| Niveau | Contenu | Où il vit | Qui l'écrit |
|---|---|---|---|
| **1. Table des intégrations** | Domaine HA → intégrations Gladys cibles (natives ou du store), qualité, statut, `store_slug` | Généré par le **tracker** | Automatique, comme aujourd'hui, avec les corrections de `data/mapping.txt` |
| **2. Règles fines** | Rapprochement, fonctions, préremplissage, exclusions | Un fichier **`gladys-migration.json`** à la racine du dépôt de **chaque intégration externe**. Pour les services natifs, un fichier central dans le dépôt du tracker. | L'auteur de l'intégration |

Le tracker publie l'ensemble dans un fichier statique, `migration-catalog.json`, sur sa page GitHub Pages. Le service Gladys le télécharge, le met en cache, et embarque un instantané de secours (01 § 3.3).

### 2.2 Pourquoi pas dans le manifeste des intégrations

Ajouter ces règles au manifeste `gladys-assistant-integration.json` serait plus simple en apparence. Mais **les versions de Gladys déjà installées rejettent les champs de manifeste qu'elles ne connaissent pas** (`docs/specs/external-integrations/contracts/manifest.md`, champ `categories`). Il faudrait modifier le cœur et l'indexeur du store, et les instances existantes casseraient. Un fichier séparé :
- ne touche ni au cœur, ni au store ;
- se publie au rythme de l'intégration, sans nouvelle version ;
- reste **générique** : une clé par système source (`home-assistant` aujourd'hui, d'autres demain, A8).

### 2.3 Ce que fait le tracker en plus

1. Pour chaque intégration Gladys connue, il lit `gladys-migration.json` s'il existe, comme il lit déjà le manifeste.
2. Il **valide** le fichier contre un schéma JSON publié (§ 3.5). Un fichier invalide est écarté et signalé, comme le fait le store avec `rejected.json`.
3. Il fusionne le niveau 1, le niveau 2 et le fichier central des services natifs dans `migration-catalog.json`.
4. Il affiche sur sa page une colonne **« Migration HA »** : règles présentes ou non, confiance. Les auteurs d'intégrations voient ainsi ce qui manque. *Cela donne au tracker la seconde vie évoquée : une fois toutes les intégrations créées, il reste l'outil qui mesure la couverture de la migration.*

## 3. Format du catalogue

### 3.1 Niveau 1 : table des intégrations

```json
{
  "catalog_version": 1,
  "generated_at": "2026-11-17T18:00:00Z",
  "integrations": {
    "shelly": {
      "targets": [
        { "kind": "external", "store_slug": "Terdious/gladys-shelly", "quality": "equivalent", "status": "store" }
      ]
    },
    "zha": {
      "targets": [
        { "kind": "native", "service": "zigbee2mqtt", "quality": "equivalent", "status": "native",
          "notes": { "fr": "Changer de pile Zigbee impose de réappairer les appareils.", "en": "…" } }
      ]
    },
    "met": {
      "targets": [
        { "kind": "external", "store_slug": "guim31/gladys-met-no", "quality": "equivalent", "status": "store" }
      ]
    },
    "bthome": { "targets": [] }
  }
}
```

- `quality` : `same_name`, `equivalent`, `partial` (repris du tracker).
- `status` : `native`, `store`, `in_progress`, `free`. Seuls `native` et `store` sont **installables** par l'outil. Les autres alimentent les messages de la décision D15 (« quelqu'un y travaille », ou « créez-la »).
- Plusieurs `targets` peuvent exister : l'utilisateur choisit, la première étant recommandée.

### 3.2 Niveau 2 : `gladys-migration.json`

Exemple complet pour Shelly (règles tirées de K § 6) :

```json
{
  "$schema": "https://william-de71.github.io/gladys-integrations-tracker/schema/gladys-migration.v1.json",
  "format_version": 1,
  "sources": {
    "home-assistant": {
      "domains": ["shelly"],
      "min_source_version": "2026.8",
      "devices": [
        {
          "match": {
            "ha": { "from": "connections", "type": "mac", "normalize": "mac" },
            "gladys": { "from": "param", "name": "SHELLY_ID", "op": "ends_with", "prefix": "-" }
          },
          "children": "merge_into_parent",
          "confidence": "sure"
        },
        {
          "match": { "ha": { "from": "connections", "type": "bluetooth" } },
          "unsupported": { "fr": "Les appareils Shelly BLU ne sont pas pris en charge.", "en": "…" }
        }
      ],
      "features": [
        {
          "unique_id": "^(?<mac>[0-9A-Fa-f]{12})-(?<key>[a-z0-9]+:[0-9]+)$",
          "feature": "{key}:binary"
        },
        {
          "unique_id": "^(?<mac>[0-9A-Fa-f]{12})-(?<key>switch:[0-9]+)-(?<attr>power|voltage|temperature)$",
          "feature": "{key}:{attr}"
        }
      ],
      "config": [
        { "from": "data.host", "to": "manual_hosts", "transform": "append" },
        { "from": "data.username", "to": "device_username", "secret": false },
        { "from": "data.password", "to": "device_password", "secret": true }
      ]
    }
  }
}
```

### 3.3 Règles de rapprochement (`devices[].match`)

**Côté HA (`ha.from`)** — trois sources suffisent pour les 12 intégrations étudiées (K § 14) :

| `from` | Paramètres | Exemple |
|---|---|---|
| `connections` | `type` (`mac`, `zigbee`, `upnp`, `bluetooth`) | MAC d'un Shelly |
| `identifiers` | `domain`, `pattern` (expression régulière avec un groupe nommé `id`) | `^zigbee2mqtt_(?<id>0x[0-9a-f]{16})$` |
| `config_entry` | `path` (ex. `data.device_name`) | Nom du nœud ESPHome |

**Côté Gladys (`gladys.from`)** :

| `from` | Paramètres | Exemple |
|---|---|---|
| `param` | `name`, `op` | `IEEE_ADDRESS` (Zigbee2MQTT), `SHELLY_ID` |
| `external_id` | `type` (segment après le préfixe), `op` | `…:cast:<id>`, `…:device:<id>` |

- **Opérateurs** (`op`) : `equals` (par défaut), `suffix` (dernier segment de l'`external_id`), `ends_with` (avec un séparateur optionnel `prefix`).
- **Normalisations nommées** (`normalize`), appliquées des deux côtés : `mac`, `ieee`, `upnp_udn`, `strip_prefix:<p>`, `lowercase`, `gladys_mqtt_sanitize` (K § 2.2). Aucune intégration n'apporte son propre code.
- **Préfixe des intégrations externes** : jamais écrit en dur. Le rapprocheur le calcule à partir du sélecteur du service installé (`ext:ext-<owner>-<repo>:` depuis le store, `ext:ext-dev-…:` en développement, K § 2.1).
- **Confiance** (`confidence`) : `sure` ou `probable`. Un rapprochement `probable` est **toujours présenté à l'utilisateur pour confirmation**. Un `sure` unique est présélectionné, mais reste modifiable.
- **Cardinalité** (`children`, `split`) : `merge_into_parent` (enfants HA par canal : Shelly, Tuya, ESPHome), `one_to_many_endpoints` (Matter, un appareil Gladys par endpoint) (K § 2.3).
- **Exclusions** : coordinateur Zigbee, pont et groupes Zigbee2MQTT, groupes ZHA. Elles s'écrivent comme des règles `match` avec `"exclude": true`.
- **`unsupported`** : appareils reconnus mais non pris en charge par la cible. Statut ❌, avec le message fourni.

### 3.4 Correspondance des fonctions (`features`)

Elle est nécessaire pour l'**historique** (07) et les **scènes** (04).

1. **Règle déclarée** : une expression régulière sur le `unique_id` de l'entité HA, avec des groupes nommés, produit le **dernier segment** de l'`external_id` de la feature Gladys. Options possibles :
   - une table de traduction (`map`), pour les propriétés dont le nom diffère ;
   - une transformation de valeur (`transform`) : `invert` pour les ouvertures, `scale` avec un facteur (par exemple KiB/s vers Mbit/s pour UPnP), `unit` pour une conversion déclarée.
2. **Repli générique** si aucune règle ne s'applique : on associe l'entité HA à une feature Gladys de l'appareil rapproché **seulement si une seule** feature libre a le même triplet (catégorie, type, unité). La catégorie et le type sont déduits de l'entité HA par la table du service MQTT de Gladys (`server/services/mqtt/lib/homeAssistant/constants.js`, K § 4.5). C'est la règle qu'applique déjà la migration d'appareils native de Gladys (`docs/specs/device-migration.md`).
3. **Sinon**, la fonction HA est marquée « perdue » (statut ⚠️ « migrable partiellement »). Elle reste visible dans le rapport.

⚠️ La table MQTT de Gladys **ne prédit pas** à elle seule la feature d'une autre intégration. Par exemple, `device_class: power` donne un capteur d'énergie en MQTT, mais une fonction de la catégorie `switch` chez Zigbee2MQTT et Shelly (K § 4.5). Elle ne sert donc qu'à **filtrer les candidats** du repli, jamais à décider seule.

### 3.5 Préremplissage de la configuration (`config`)

- Liste de `{ from, to, transform?, secret }` :
  - `from` désigne `data.*` ou `options.*` de l'entrée de configuration HA (disponible **uniquement** avec la sauvegarde, 02 § 1) ;
  - `to` désigne une clé du `config_schema` de l'intégration externe, ou une **variable de service** pour un service natif.
- Les valeurs `secret: true` sont montrées masquées. L'utilisateur coche une à une celles à reporter (§ 9 bis de la vision).
- Une clé `to` déjà renseignée dans Gladys n'est **jamais** remplacée (A9).

Intérêt selon les intégrations (K § 1) :
- **utile** pour ESPHome (clé de chiffrement), Shelly (hôtes, identifiants), Google Cast (hôtes connus), MQTT et Zigbee2MQTT (broker), UPnP, IPP et DLNA (URL de secours) ;
- **quasi nul** pour Tuya : HA s'y connecte par QR Smart Life, alors que Gladys exige un projet IoT Cloud ;
- **nul** pour Matter.

### 3.6 Schéma et validation

Le schéma JSON `gladys-migration.v1.json` est publié par le tracker. Il est aussi **vendu** (copié) dans le service Gladys, qui revalide chaque entrée du catalogue avant usage : le catalogue est une donnée externe, non modérée, comme l'index du store. Les expressions régulières sont **bornées** (longueur maximale, exécution avec délai) pour éviter les expressions pathologiques.

## 4. Couverture V1 (D19, D20)

Synthèse de K § 1 :

| Intégration HA | Cible Gladys | Rapprochement | Fonctions | Remarque |
|---|---|---|---|---|
| Zigbee2MQTT (via MQTT) | zigbee2mqtt (natif) | ✅ sûr (IEEE) | ✅ sûres | Le `base_topic` doit être celui écouté par Gladys (`zigbee2mqtt`) |
| ZHA | zigbee2mqtt (natif) | ✅ sûr (IEEE) | ⚠️ probable | **Changement de pile Zigbee : réappairage** (01 § 9.3) |
| MQTT générique | mqtt (natif) | ⚠️ probable | ✅ sûres | Préfixe de découverte `homeassistant` obligatoire côté Gladys |
| Matter | matter (natif) | ⚠️ probable (n° de série), sinon manuel | ⚠️ probable | **Nouvel appairage** dans Gladys (code de partage depuis HA). Un appareil HA donne N appareils Gladys. |
| Shelly | Terdious/gladys-shelly | ✅ sûr (MAC) | ⚠️ probable | Shelly BLU non couverts |
| Google Cast | gboulvin/Gladys-Google-Cast | ✅ sûr (UUID) | — | Sans historique |
| ESPHome | William-De71/gladys-esphome | ⚠️ probable (nom du nœud) | ⚠️ probable | Publier la MAC dans un param rendrait la règle sûre (suggestion à William) |
| Tuya | Terdious/gladys-tuya | ✅ sûr (id Tuya) | ✅ sûres | Compte IoT Cloud à créer |
| UPnP/IGD | callemand/gladys-upnp-igd | ✅ sûr (UDN) | ⚠️ probable | Conversions d'unités |
| IPP | guim31/gladys-ipp | ⚠️ probable | ⚠️ probable | |
| DLNA | guim31/gladys-dlna | ✅ sûr (UDN) | — | Sans historique |
| Met.no | guim31/gladys-met-no | sans objet | sans objet | Intégration de service (§ 7) |
| Bluetooth | — | sans objet | — | Transport, pas une intégration d'appareils : les appareils BLE relèvent de leur propre intégration (BTHome, Xiaomi BLE…) |

**Aucune intégration du top 12 n'est entièrement manuelle.** Les cas manuels sont ponctuels : Matter sans numéro de série exploitable, Google Cast et IPP ajoutés par adresse manuelle, entités MQTT sans bloc `device`.

## 5. Pièces (D14)

1. Chaque *area* HA utilisée par au moins un appareil ou une entité retenus donne une **pièce Gladys**, dans la maison choisie par l'utilisateur. Gladys peut avoir plusieurs maisons, HA n'en a qu'une.
2. **Renommage proposé** si HA a plus d'un étage : `« <étage> - <pièce> »`, modifiable.
3. **Contraintes Gladys** : nom de 1 à 40 caractères, unique sur toute l'instance (note Gladys).
   - Un nom trop long est tronqué proprement, avec une proposition modifiable.
   - Un **doublon** est signalé avant l'import : soit on réutilise la pièce Gladys existante du même nom (choix par défaut en coexistence, puisqu'il s'agit probablement de la même pièce), soit on renomme.
4. Pièce d'un appareil : celle de l'**appareil** HA. Si ses entités sont réparties dans plusieurs pièces, c'est celle de l'appareil qui l'emporte, et l'écart est signalé.

## 6. Personnes → utilisateurs (D11)

| Donnée HA | Donnée Gladys | Remarque |
|---|---|---|
| `person.name` | `firstname` (et `lastname` si l'utilisateur sépare le nom) | |
| Utilisateur HA lié : administrateur ou propriétaire | `role: admin` | Sinon `habitant` |
| Personne sans utilisateur HA | `role: habitant` | |
| — | **`email`, `password` (8 caractères minimum), `birthdate`, `language`** | **Obligatoires dans Gladys et absents de HA** : à saisir pour chaque personne retenue |
| Photo | `picture` | Si présente dans la sauvegarde, après accord |
| `device_trackers` | — | Présence : hors V1 (Q03-2) |

- Les mots de passe HA ne sont **jamais** repris : ils sont hachés, et l'outil ne lit pas ces fichiers (02 § 4.2).
- La personne qui fait la migration est déjà un utilisateur Gladys (administrateur). L'outil propose de la **rapprocher** de sa personne HA plutôt que d'en créer une deuxième.

## 7. Intégrations de type service (D20)

Elles ne portent pas d'appareils à rapprocher (`entry_type: service` côté HA), mais une **configuration**.

| Exemple | Ce qui migre |
|---|---|
| Met.no | Installation de l'intégration, coordonnées (celles de la maison Gladys), altitude si elle est connue |
| Calendriers (Google, ICS, CalDAV) | Installation et URL du calendrier quand elle est dans la sauvegarde. Comptes OAuth : nouvelle connexion obligatoire. |
| Notifications (Telegram, ntfy, Pushover…) | Installation, et jeton si présent dans la sauvegarde (secret, avec accord) |

Le niveau 2 du catalogue décrit le préremplissage (`config`) de la même façon que pour les intégrations d'appareils. Une intégration de service sans `config` est simplement **installée**, l'utilisateur la configurant lui-même.

## 8. Statuts produits par l'analyseur

Correspondance avec la vision (§ 8) :

| Situation dans le catalogue | Statut |
|---|---|
| Cible `native` ou `store`, règle `sure` | ✅ Migrable |
| Cible `native` ou `store`, règle `probable` ou rapprochement manuel | ✅ Migrable (confirmation demandée) |
| Cible dont la qualité est `partial`, ou fonctions perdues | ⚠️ Migrable partiellement |
| Le domaine HA n'a pas de cible, mais l'appareil est vu par une autre intégration (MQTT, Matter…) | 🔀 Migrable via alternative |
| Cible `in_progress` | ❌ Non migrable, « en cours de développement par … » |
| Pas de cible (`free`) | ❌ Non migrable : parcours D15 (créer l'intégration, puis le forum) |
| Entité désactivée, masquée, de diagnostic ; règle `exclude` | ⏭️ Ignoré |
| Domaine HA absent du catalogue | ❌ Non migrable, et l'outil propose de signaler le manque au tracker |

## 9. Questions ouvertes

| # | Question |
|---|---|
| Q03-1 | **Nom et hébergement du fichier de niveau 2** : `gladys-migration.json` à la racine du dépôt de l'intégration convient-il à William et aux auteurs d'intégrations ? Faut-il l'ajouter au modèle `integration-template-js` (fichier vide d'exemple), pour que chaque nouvelle intégration y pense dès sa création ? |
| Q03-2 | **Présence** : les `device_trackers` des personnes HA (application mobile, routeur…) n'ont pas d'équivalent direct. On les laisse hors V1 ? |
| Q03-3 | **ZHA → Zigbee2MQTT** : le changement de pile impose un réappairage. Faut-il un parcours guidé dans l'outil (appareil par appareil, le rapprochement se faisant après chaque appairage par l'adresse IEEE, qui ne change pas) ? Cela vaudrait aussi pour Matter. |
