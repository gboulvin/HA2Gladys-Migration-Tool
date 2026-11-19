# 02 — Lecture des données Home Assistant

- **Statut** : 📝 Brouillon
- **Dernière mise à jour** : 2026-11-19
- **Prérequis** : [00 — Vision](00-vision.md), [01 — Architecture](01-architecture.md) (PR #1), [note de recherche HA](research/ha-data-sources.md)
- **Référence HA étudiée** : `home-assistant/core` `9b5a396` (2026.12 dev), comparé au tag publié **2026.11.2**. Bibliothèque `securetar` `08db90e`. Supervisor `e610ca8`.

Les renvois « R § x » désignent la [note de recherche](research/ha-data-sources.md), qui cite le code source de HA ligne par ligne.

---

## 1. Ce que l'étude change

L'étude du code de HA confirme la vision sur l'essentiel, mais impose cinq ajustements :

| # | Constat | Conséquence |
|---|---|---|
| 1 | L'API ne donne **jamais** les identifiants des intégrations (`config_entries/get` omet `data` et `options`). Ils ne sont que dans `.storage/core.config_entries`, en clair (R § 1.3, § 3.2). | Le préremplissage des configurations Gladys (§ 9 bis de la vision) exige la **sauvegarde**. |
| 2 | La configuration des automatisations, les blueprints et la liste des utilisateurs exigent un jeton **administrateur**. Un jeton non administrateur reçoit un inventaire **filtré sans erreur** (R § 1.2, § 8). | L'outil vérifie le statut administrateur dès la connexion et refuse un jeton qui ne l'est pas. |
| 3 | Par l'API, les automatisations arrivent **déjà résolues** : `!include`, packages, `!secret` et blueprints substitués. Par la sauvegarde, il faut réimplémenter tout le chargement YAML de HA (R § 1.5, § 3.3). | Les deux sources sont équivalentes pour les **registres**, mais pas pour les **automatisations**. |
| 4 | Les sauvegardes récentes sont chiffrées en **SecureTar v3** (par défaut depuis HA 2026.3) : Argon2id, BLAKE2b et XChaCha20-Poly1305, ce qui exige **libsodium**. En revanche, **HA sait servir lui-même une sauvegarde déchiffrée**, à un administrateur qui fournit le mot de passe (R § 4.4). | Voie privilégiée : Gladys télécharge la sauvegarde **déjà déchiffrée** auprès de HA (§ 3). |
| 5 | En 2026.11, aucune commande d'API ne **compte** les points d'historique. `recorder/database_usage` arrive probablement en 2026.12. Avec la base de la sauvegarde, un simple `COUNT` suffit (R § 1.9). | La sauvegarde est aussi la meilleure source pour l'estimation du volume (D24). |

**Conclusion** : l'API et la sauvegarde ne s'opposent pas, elles se **complètent**. On en tire trois modes de lecture (§ 2).

## 2. Les trois modes de lecture

| Mode | Entrée de l'utilisateur | Ce que l'outil utilise | Quand |
|---|---|---|---|
| **A. Connexion complète** (recommandé) | URL de HA, jeton administrateur, mot de passe de la sauvegarde | API pour l'inventaire et les automatisations **+ sauvegarde téléchargée et déchiffrée par HA** pour les identifiants, la base d'historique et les données des modules complémentaires | HA est en marche. C'est le cas général. |
| **B. Connexion seule** | URL de HA et jeton administrateur | API uniquement | L'utilisateur ne veut pas ou ne peut pas fournir de sauvegarde. Pas de préremplissage des identifiants. Historique par l'API, plus lent. |
| **C. Fichier de sauvegarde seul** | Fichier `.tar` et son mot de passe | Lecture locale complète, avec déchiffrement par Gladys | HA est arrêté ou hors d'usage. Automatisations en fidélité réduite (§ 6.3). |

Le mode A combine le meilleur des deux sources : automatisations résolues par HA, identifiants et historique complets par la sauvegarde, **sans que Gladys ait à déchiffrer** quoi que ce soit.

## 3. Mode A et B : lecture par l'API

### 3.1 Connexion et contrôles

1. Connexion WebSocket à `<url>/api/websocket`, authentification par jeton longue durée (R § 1.1). La version de HA est lue dans le message `auth_required`.
2. **Contrôle de version** (§ 7).
3. **Contrôle du statut administrateur** : appel de `config/auth/list`. Une erreur `unauthorized` arrête l'outil avec un message expliquant comment créer un jeton depuis un compte administrateur.
4. Le client annonce `coalesce_messages` et gère les réponses groupées (R § 1.1). Les `id` de commande sont strictement croissants.

### 3.2 Commandes utilisées

| Donnée | Commande | Remarque |
|---|---|---|
| Appareils (et appareils enfants) | `config/device_registry/list` | Un seul message, parfois plusieurs Mo (R § 1.10) |
| Anciens appareils scindés | `config/device_registry/list_composite_splits` | Pour résoudre les `device_id` d'automatisations antérieures à 2026.8 (R § 2.1) |
| Entités | `config/entity_registry/list`, puis `get_entries` par lots | `get_entries` donne `device_class`, `original_device_class` et `capabilities` (`state_class`) |
| Unités, entités sans registre | `get_states` | L'unité n'est exposée que par l'état (R § 1.3) |
| Pièces, étages | `config/area_registry/list`, `config/floor_registry/list` | Étiquettes ignorées (D14) |
| Intégrations | `config_entries/get` | Sans identifiants |
| Personnes | `person/list` (`storage` et `config`), `config/auth/list` | D11 |
| Automatisations | `get_states` sur `automation.*`, puis **WS `automation/config`** par entité | Couvre toutes les automatisations, y compris sans `id`, dans des packages ou issues d'un blueprint (R § 1.5). La route REST ne lit que `automations.yaml` et n'est pas utilisée. |
| Scripts | `get_states` sur `script.*`, puis `script/config` par entité | V1 : scripts simples et appels depuis les automatisations (04 § 8 ter) |
| Scènes HA | `get_states` sur `scene.*` (attribut `id`, présent seulement pour une scène qui en a un : [core] `components/homeassistant/scene.py:375-381`), puis REST `GET /api/config/scene/config/<id>` (administrateur : [core] `components/config/view.py:44`, `:85-86`) | V1 (04 § 8 bis) |
| Entrées (*helpers*) | `input_boolean/list`, `input_number/list`, `counter/list` (commandes `<domaine>/list` des collections : [core] `helpers/collection.py:600-606`), états par `get_states` | V1 (D27). Les entrées définies en YAML n'apparaissent que dans `get_states`. |
| Zones | `zone/list` (zones de l'éditeur), `get_states` sur `zone.*` (zones YAML et `zone.home`) | V1 (D30) |
| Configuration HA | `get_config` : fuseau horaire, coordonnées, altitude, système d'unités ([core] `components/websocket_api/commands.py:734-745`) | Contrôles de 04 § 4, coordonnées de la maison (04 § 6.3) |
| Tableaux de bord | `lovelace/dashboards/list` ([core] `components/lovelace/__init__.py:252`), puis `lovelace/config` par `url_path` ([core] `components/lovelace/websocket.py:130-135`) | V1 (08 § 3). *Droits exacts et mode YAML à vérifier.* |
| Modules complémentaires | `supervisor/api` vers `/addons` ([core] `components/hassio/websocket_api.py:111-148`, administrateur) | V1 (D29, 03 § 7 ter). Absent sans Supervisor. |
| Photos des personnes | `person/list` donne le champ `picture` (URL d'image) ; image par REST *à vérifier* | 03 § 6 |
| Sauvegardes disponibles | `backup/info` | Mode A (§ 3.3) |
| Historique des états | `history/history_during_period`, par fenêtres de temps | Pas de pagination côté HA (R § 1.7) |
| Statistiques long terme | `recorder/list_statistic_ids`, `recorder/statistics_during_period` (`period: "hour"`) | Toujours passer `units` explicitement, sinon HA convertit (R § 1.8) |
| Comptage d'historique | `recorder/database_usage` si disponible (2026.12+), sinon estimation (§ 5.3) | |

### 3.3 Téléchargement de la sauvegarde (mode A)

1. `backup/info` liste les sauvegardes existantes et leurs emplacements (*agents*). L'outil propose **la plus récente** qui contient HA (`homeassistant_included`) et, si possible, la base d'historique (`database_included`) ([core] `components/backup/models.py:35-39`). Il indique sa date. *Corrigé le 2026-11-19 (F6) : `exclude_database` n'existe que dans le `backup.json` de l'archive (§ 4.1), pas dans la réponse de l'API.*
2. L'utilisateur saisit le mot de passe de cette sauvegarde. Pour les sauvegardes automatiques, c'est la « clé de chiffrement » affichée dans les réglages de HA, à transmettre **telle quelle** (R § 4.4).
3. Gladys appelle `GET /api/backup/download/<id>?agent_id=<agent>&password=<mot de passe>`. HA **déchiffre lui-même** et renvoie un tar en clair (R § 4.4). Gladys le lit **en flux pendant le téléchargement**, avec l'extraction sélective du § 4.3 (règle 1) : l'archive en clair n'est **jamais écrite telle quelle sur disque** (01 § 7.2, 01 § 7.3 voie 1).
4. Lecture locale identique au mode C (§ 4), **sans l'étape de déchiffrement**.

> ⚠️ Le mot de passe passe dans l'URL. La connexion doit donc être en HTTPS, ou rester sur le réseau local. Le mot de passe peut aussi apparaître dans les journaux d'accès de HA ou d'un proxy inverse. L'utilisateur en est averti, et pourra changer ce mot de passe dans HA après la migration.

**Et s'il n'y a pas de sauvegarde récente ? (E1)** L'outil **ne crée jamais de sauvegarde** : il reste en lecture seule (principe 2 de la vision). On part du principe qu'un utilisateur de HA sait en faire une. L'outil affiche la date de la plus récente, invite l'utilisateur à en lancer une nouvelle dans HA s'il le souhaite, puis à relancer la détection. En dernier recours seulement, un lien vers la page d'aide de HA sur les sauvegardes. Il est utile, mais devra être tenu à jour.

## 4. Mode A et C : lecture de la sauvegarde

### 4.1 Structure

Un tar externe contient `backup.json` en clair et une archive interne par composant (R § 4.1) :

- `homeassistant.tar.gz` : le dossier de configuration de HA, sous `data/` ;
- pour les sauvegardes du Supervisor (HA OS, Supervised), une archive par module complémentaire et par dossier (`share`, `ssl`, `media`…).

`backup.json` est lu en premier. Il donne la version de HA, l'indicateur `protected`, `exclude_database`, et peut indiquer `homeassistant: null`, c'est-à-dire une sauvegarde partielle **sans** HA. Dans ce dernier cas, l'outil refuse le fichier et explique pourquoi.

### 4.2 Fichiers lus

| Fichier dans `data/` | Usage | Sensible |
|---|---|---|
| `.storage/core.device_registry`, `core.entity_registry`, `core.area_registry`, `core.floor_registry` | Inventaire | Non |
| `.storage/core.config_entries` | Intégrations, **identifiants pour le préremplissage** | **Oui** |
| `.storage/person` | Personnes | Non |
| `.storage/auth` | Utilisateurs (noms, statut administrateur) | **Oui** (contient des jetons) |
| `.storage/input_boolean`, `.storage/input_number`, `.storage/counter` (version 1 : [core] `components/input_boolean/__init__.py:53-54`, de même pour `input_number` et `counter`) | Entrées (D27) | Non |
| `.storage/zone` (version 1 : [core] `components/zone/__init__.py:128-129`) | Zones (D30) | Non |
| `.storage/core.config` ([core] `core_config.py:87`) | Configuration HA : fuseau horaire, coordonnées, unités (04 § 4, § 6.3) | Non |
| `.storage/image` et dossier `image/<id>/original` ([core] `components/image_upload/__init__.py:28`, `:116-119`) | Photos des personnes (03 § 6), seulement si l'utilisateur l'accepte | Non |
| `.storage/lovelace`, `.storage/lovelace.<id>`, `.storage/lovelace_dashboards` ([core] `components/lovelace/dashboard.py:35-38`) | Tableaux de bord (08 § 3) | Non |
| `automations.yaml`, `scripts.yaml`, `scenes.yaml`, `blueprints/`, `configuration.yaml` et inclusions, `secrets.yaml` | Automatisations, scripts et scènes HA (V1, § 6.3) | `secrets.yaml` : **oui** |
| `home-assistant_v2.db` **et** `home-assistant_v2.db-wal` | Historique et comptage | Non |
| `.HA_VERSION` | Version | Non |
| `backup.json` (liste `addons`) et archive du module Zigbee2MQTT | Modules (03 § 7 ter) ; données de Zigbee2MQTT pour la bascule (01 § 9.3) | Archive Zigbee2MQTT : **oui** (clé réseau). Écrite dans le dossier de travail seulement si la bascule la reprend. |

**Ne sont jamais lus ni extraits** : `auth_provider.homeassistant` (mots de passe hachés), `.storage/backup` (contient le mot de passe des sauvegardes), ni le reste du dossier.

### 4.3 Règles de lecture

1. **Extraction sélective et en flux, pendant le transfert** (mode A : téléchargement ; mode C : téléversement ou lecture du fichier déposé) : on ne garde que les fichiers du § 4.2. Les fichiers sensibles sont lus **en mémoire** et jamais écrits sur disque. Seuls la base d'historique et son WAL, trop volumineux, et, si besoin, l'archive du module Zigbee2MQTT sont écrits dans le dossier de travail, conservé tant que l'exécution est ouverte (01 § 7.2). Le reste de l'archive est ignoré.
2. **Sécurité des archives** : les contrôles de Gladys sont réutilisés (`server/utils/backupSafety.js` : noms de fichiers sûrs, aucune traversée de chemin).
3. **Lecteur tar en JavaScript** : les archives internes chiffrées se lisent en flux (déchiffrement, puis gunzip, puis tar). Le binaire `tar`, prévu en 01 § 3.1, ne suffit pas (R § 4.4). Une bibliothèque de lecture tar en flux est donc ajoutée (dépendance à valider, D18).
4. **Base d'historique** : la base et son fichier WAL sont extraits **côte à côte**, sinon les écritures récentes manquent. La base est ouverte en **lecture seule** avec `sqlite3`, déjà présent dans Gladys (R § 8, point 14).
5. **Versions des fichiers** : chaque fichier `.storage` porte `version` et `minor_version`, et le schéma de la base est dans `schema_changes`. L'outil vérifie ces versions par rapport à une table des versions prises en charge (§ 7), et **ne migre jamais** un format lui-même.

### 4.4 Déchiffrement par Gladys (mode C uniquement)

| Version SecureTar | Produite par | Algorithmes | Faisable en Node.js |
|---|---|---|---|
| v1, v2 | HA < 2026.3 | AES-128-CBC, clé par SHA-256 itéré 100 fois | Oui, avec `crypto` natif |
| **v3** | HA ≥ 2026.3 (par défaut) | Argon2id, BLAKE2b avec sel et personnalisation, XChaCha20-Poly1305 *secretstream* | Oui, avec **libsodium** en WebAssembly (sans compilation native) |

- La v3 permet de **vérifier le mot de passe** avant tout déchiffrement (clé de validation). Un mauvais mot de passe est signalé immédiatement.
- Avec la v1 et la v2, un mauvais mot de passe ne se détecte qu'au premier bloc.
- **Ajout d'une dépendance libsodium, propre au service** (A7, § 9 ; présentée sur le forum, D18). Un **test de bout en bout sur de vraies sauvegardes v2 et v3** est obligatoire. Plusieurs détails d'implémentation restent *à vérifier*, notamment le remplissage de la personnalisation BLAKE2b (R § 4.4).

## 5. Historiques (D22 à D24)

### 5.1 Choix de la source

| Source | États bruts | Statistiques | Comptage | Performance |
|---|---|---|---|---|
| Base de la sauvegarde (modes A et C) | ✅ SQL | ✅ SQL, unité de stockage | ✅ `COUNT` exact | Locale, rapide |
| API (mode B, ou base absente de la sauvegarde) | ✅ `history_during_period` | ✅ `statistics_during_period` | ⚠️ estimation (2026.11), exact en 2026.12+ | Réseau, découpage par fenêtres |

**Règle** : si une base SQLite est disponible dans la sauvegarde, elle est la source de l'historique. Sinon, on passe par l'API. Une base MariaDB ou PostgreSQL n'est jamais dans la sauvegarde : l'API est alors la seule voie (R § 4.3).

### 5.2 Ce qui est importé

| Donnée HA | Rétention dans HA | Import vers Gladys |
|---|---|---|
| États bruts (`states`) | 10 jours par défaut | Valeurs **numériques** uniquement. `unknown`, `unavailable` et les textes sont écartés (R § 5.3). |
| Statistiques horaires (`statistics`) | Indéfinie | `measurement` : valeur **moyenne** de l'heure. Compteurs (`total`, `total_increasing`) : traitement à part (§ 5.4). |
| Statistiques 5 minutes | Comme les états | Non importées : elles doublonnent les états bruts sur la même période. |
| Statistiques externes (fournisseurs d'énergie…) | Indéfinie | Hors V1 : elles ne sont rattachées à aucune entité (R § 5.5). |

**Raccord entre les deux niveaux** : pour une même fonction Gladys, les statistiques horaires couvrent le passé lointain et les états bruts les derniers jours. La frontière est la date du plus ancien état brut conservé : les statistiques sont importées jusqu'à cette date, les états bruts à partir d'elle. On évite ainsi de compter deux fois la même période.

**Mesures angulaires** (`mean_type = 2`, par exemple la direction du vent) : la moyenne circulaire calculée par HA est importée telle quelle. La colonne `statistics_meta.mean_type` n'existe qu'à partir du schéma 49 ([core] `components/recorder/const.py:47`, `CIRCULAR_MEAN_SCHEMA_VERSION`). Si elle est absente, l'historique des mesures angulaires n'est pas importé, et c'est signalé (§ 7).

### 5.3 Estimation du volume (D24)

| Situation | Méthode |
|---|---|
| Base SQLite disponible | `COUNT` par entité sur `states` et `statistics` (R § 5.4) |
| HA ≥ 2026.12, mode B | `recorder/database_usage` (comptes exacts, administrateur) |
| HA 2026.11, mode B | Statistiques : bornes de la période × 1 point par heure (surestime légèrement si HA a été arrêté). États : comptage réel, acceptable puisque la rétention est courte. |

Le nombre de points obtenu alimente le calcul de 01 § 8.

### 5.4 Compteurs d'énergie

Pour `total` et `total_increasing`, HA stocke deux séries (R § 5.5) :
- `state` : l'index brut du compteur, qui peut revenir à zéro ;
- `sum` : un cumul continu, qui « traverse » les remises à zéro.

La consommation d'une heure est `sum(h) − sum(h−1)`. Le choix de la série à importer est **tranché en 07 § 3.3 (H2)** : l'index réel (`state`), puis recalcul des consommations, coûts et productions par Gladys. L'interaction avec la consommation à 30 minutes reste à mesurer (Q07-1).

### 5.5 Unités et horodatages

- Les statistiques sont importées dans leur **unité de stockage** (`statistics_unit_of_measurement`), puis converties vers l'unité de la fonction Gladys. Une conversion inconnue rend l'historique de cette entité non importable, et c'est signalé.
- Horodatages : secondes epoch (float) en SQL et pour l'historique WS, **millisecondes** pour les statistiques WS (R § 8, point 9). Tout est converti en ISO 8601 UTC pour `device.saveMultipleHistoricalStates`.

## 6. Normalisation vers l'inventaire

### 6.1 Appareils et rapprochement

- **Adresse IEEE** : ZHA publie `("zha", ieee)` et `("zigbee", ieee)` au format `xx:xx:…`, alors que Zigbee2MQTT et Gladys utilisent `0x…`. La normalisation retire les `:` et le préfixe `0x`, et passe en minuscules (R § 2.4).
- **Adresse MAC** : déjà normalisée par HA (`aa:bb:cc:dd:ee:ff`, R § 2.1).
- **Appareils scindés (HA 2026.8)** : une même connexion peut figurer sur plusieurs appareils HA. Les appareils sont **regroupés par connexion** avant le rapprochement, sinon chaque appareil Gladys trouverait plusieurs candidats (R § 2.1).
- **Appareils enfants (HA 2026.9)** : rattachés à leur parent par `parent_device_id`.
- **Appareils de type service** (`entry_type = service`) : ils servent à reconnaître les intégrations de type service (D20). Ce ne sont pas des appareils à rapprocher.
- **Nom** : `name_by_user` en priorité, sinon `name`. **Pièce** : celle de l'entité si elle est définie, sinon celle de l'appareil.

### 6.2 Entités exclues par défaut

Entités désactivées (`disabled_by`), masquées (`hidden_by`), de catégorie `diagnostic` ou `config`. Elles sont présentes dans l'inventaire avec le statut « ignoré » (vision § 8) et restent visibles dans le rapport détaillé.

### 6.3 Automatisations, scripts et scènes HA (V1)

| Mode | Automatisations disponibles |
|---|---|
| A, B | **Toutes**, résolues par HA (`automation/config`, `script/config`, configuration REST des scènes, § 3.2) |
| C | `automations.yaml`, `scripts.yaml` et `scenes.yaml` (éditeur de HA), avec substitution des blueprints référencés. Les automatisations définies ailleurs (packages, fichiers inclus, clés `automation xxx:`) sont **listées** si on les trouve, mais classées « à refaire manuellement ». La fiabilité prime (D5). |

Pour le mode C, on ne réimplémente **pas** l'intégralité du chargeur YAML de HA. On se limite à `automations.yaml`, `scripts.yaml`, `scenes.yaml`, `!secret` et la substitution `!input` des blueprints, ce qui couvre l'usage de l'éditeur graphique. *Ce périmètre pourra être élargi plus tard si le besoin est avéré.*

**Identité d'une automatisation** pour l'idempotence (D13) : son `id` HA. Sans `id` (YAML), une **empreinte du contenu** est utilisée, l'`entity_id` dérivé de l'alias étant instable (R § 8, point 18). La clé stable de **chaque type d'élément** (scènes, scripts, entrées, zones, tableaux de bord, modules…) est définie en 01 § 5.1 bis (T9).

## 7. Versions prises en charge (D9)

| Élément | Version de référence (HA 2026.11) | Politique proposée |
|---|---|---|
| `core.device_registry` | 3.4 | Majeure 3 uniquement, soit **HA ≥ 2026.8** (E2). Les formats antérieurs sont refusés avec un message explicite. |
| `core.entity_registry` | 1.24 | Mineures connues acceptées |
| `core.area_registry` / `floor_registry` | 1.9 / 1.3 | Idem |
| `core.config_entries` | 1.5 | Idem |
| Schéma du recorder | 53 | Schémas ≥ 49 acceptés (colonne `mean_type`, § 5.2). En pratique, E2 (HA ≥ 2026.8) impose déjà un schéma plus récent. *Corrigé le 2026-11-19 (F7) : le minimum était 48 (R § 5.2).* |
| SecureTar | v1, v2, v3 | Toutes |
| HA par l'API | 2026.11 | Avertissement au-delà de la dernière version testée, refus en dessous du minimum |

Une **version plus récente qu'attendu** déclenche un avertissement, pas un refus. HA migre ses formats au chargement : une sauvegarde porte le format de la version qui l'a produite (R § 3.1). La table des versions vit dans le code du service et suit chaque version mensuelle de HA, conformément à D9. *Un suivi automatisé (CI qui relit les constantes de HA à chaque version) serait souhaitable.*

## 8. Sécurité (compléments à 01 § 7)

- Le **jeton administrateur** HA donne un accès complet à HA. Il reste en mémoire (coffre éphémère), et l'utilisateur est invité à le **révoquer** après la migration.
- La sauvegarde déchiffrée contient en clair `secrets.yaml`, `core.config_entries` et `auth`, qui inclut **les jetons de rafraîchissement, dont celui fourni à l'outil**. D'où l'extraction sélective en mémoire **pendant le transfert** (§ 4.3), en mode A comme en mode C : ces fichiers ne sont jamais écrits sur disque, et l'archive complète non plus.
- Le **mot de passe de la sauvegarde** transite dans l'URL en mode A (§ 3.3).
- Les identifiants extraits ne servent qu'au **préremplissage consenti** : l'utilisateur voit, intégration par intégration, ce qui sera reporté dans Gladys.

## 9. Dépendances et modifications du cœur (D18)

| Élément | Nécessaire pour | Statut |
|---|---|---|
| Client WebSocket | Modes A, B | `ws` déjà présent |
| `sqlite3` | Lecture de la base d'historique | Déjà présent |
| Lecteur tar en flux (JavaScript) | Modes A, C | **À ajouter**, propre au service (A7) |
| Analyseur YAML avec balises personnalisées | Mode C : `automations.yaml`, `scripts.yaml`, `scenes.yaml` (V1, D26, D31) | **À ajouter**, propre au service (A7) |
| libsodium (WebAssembly) | Mode C, sauvegardes v3 | **À ajouter**, propre au service (A7) |
| Route de téléversement en flux | Mode C depuis le navigateur | Souhaitable (01 § 7.3, voie 2) : modification du cœur facultative |

En V1, le **mode A** demande une nouvelle dépendance (le lecteur tar) et le **mode C** trois (lecteur tar, YAML, libsodium). Toutes sont **propres au service** (A7, 01 § 3.1) : aucune dépendance n'est ajoutée au cœur, et aucune route du cœur n'est modifiée, hormis la route facultative de téléversement. *Corrigé le 2026-11-19 (C12) : le YAML sert dès la V1, en mode C.*

## 10. Décisions

| # | Décision |
|---|---|
| E1 | **Lecture seule stricte** : l'outil ne demande jamais à HA de créer une sauvegarde. L'utilisateur la fait lui-même, éventuellement aidé d'un lien vers l'aide de HA (ex-Q02-1). |
| E2 | Sauvegardes et instances acceptées à partir de **HA 2026.8** (registre des appareils 3.x) (ex-Q02-2). |

## 11. Questions ouvertes

| # | Question |
|---|---|
| Q02-3 | **Sauvegardes réelles anonymisées** pour vérifier l'emplacement des données de Zigbee2MQTT (module complémentaire, 01 § 9.3), le déchiffrement v2 et v3, et pour constituer les jeux de test (section 06). *En cours : le porteur du projet se renseigne auprès de la communauté.* |
