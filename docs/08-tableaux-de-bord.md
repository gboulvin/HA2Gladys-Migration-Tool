# 08 — Tableaux de bord

- **Statut** : 📝 Brouillon
- **Dernière mise à jour** : 2026-11-19
- **Prérequis** : [00 — Vision](00-vision.md) (D28), [03 — Correspondances](03-mapping-devices.md), [04 — Scènes](04-mapping-scenarios.md), [05 — Parcours](05-ux.md)
- **Référence Gladys** : `docs/specs/dashboard-flexible-layout-and-widgets.md`, types de widgets dans `server/utils/constants.js` (`DASHBOARD_BOX_TYPE`)

---

## 1. Pourquoi dès la V1

Le tableau de bord est la **vitrine de Gladys** : c'est la première page après la connexion. Un tableau de bord vide après la migration donnerait l'impression que « tout a disparu ». Gladys ne génère pas de tableau de bord par défaut : sans l'outil, l'utilisateur arriverait donc sur une page vide.

**Principe (D28)** : à la fin de la migration (phase F, 00 § 6 bis), l'utilisateur arrive **toujours** sur un **tableau de bord « Accueil » complet**, construit à partir de ce qui vient d'être migré. Ses **tableaux de bord HA**, repris de façon approchée, viennent **à la suite** (DB2).

## 2. Le tableau de bord « Accueil » (généré)

Il est construit à partir des **données migrées**, pas des cartes HA. Il est donc toujours cohérent, quelle que soit la façon dont l'utilisateur avait organisé HA.

| Zone | Widgets Gladys | Source | Si absent |
|---|---|---|---|
| **En-tête** | `house-view` (illustration de la galerie de Gladys, § 2.1), `weather` si une intégration météo a été migrée, `clock` | Maison, intégrations de service | Pas de météo : widget omis |
| **Ambiances** | `scene` (boutons de scènes) | Scènes HA et scripts HA simples migrés (D25, D31) | Section omise |
| **Une section par pièce**, dans l'ordre des étages puis des pièces HA | `temperature-in-room`, `humidity-in-room` (s'il y a des capteurs), `devices-in-room` | Pièces et appareils migrés | Pièce sans appareil commandable : omise |
| **Caméras** | `camera` | Appareils caméra migrés | Section omise |
| **Énergie** | `energy-consumption` | Compteurs d'énergie migrés (07 § 3.3) | Section omise |
| **Modes** | `devices` | Entrées HA migrées en appareils virtuels (D27) : mode vacances, mode nuit… | Section omise |
| **Migration en cours** (coexistence seulement) | `link` vers Paramètres → Migration (« Terminer ma migration : 3 étapes restantes ») | Journal | Retirée au passage en bascule, ou par l'utilisateur |

- **Pièces très chargées** : au-delà d'environ 12 fonctions, `devices-in-room` est remplacé par un widget `devices` limité aux fonctions **commandables** (lumières, prises, volets, thermostats). Les capteurs restent accessibles dans la page de la pièce. *Seuil à ajuster sur des installations réelles.*
- **Mobile** : sur téléphone, les colonnes d'une section s'empilent. Le réglage `mobile_order` de **chaque section** choisit l'ordre des widgets : colonne par colonne (`columns`, valeur par défaut quand le champ est absent) ou ligne par ligne (`rows`) (`server/models/dashboard.js:193-210`, `server/utils/constants.js:2431-2434`). L'outil retient **`columns`** pour toutes les sections, générées comme reprises : chaque colonne y forme un bloc cohérent (un usage, une pile HA), qu'il vaut mieux garder d'un seul tenant. Le champ est donc omis. *Corrigé le 2026-11-19 (F8) : ce n'est pas une règle d'ordre des sections.*
- **Disposition** : sections et colonnes de Gladys, sans réglage fin. L'objectif est un résultat propre, pas une copie.

### 2.1 Illustration de la maison (DB1)

- L'outil choisit **automatiquement** une illustration de la **galerie de Gladys** (`house-modern`, `house-family`, `house-solar`, `apartment`) :
  - `apartment` si HA ne déclare qu'un étage ou aucun ;
  - une maison sinon ;
  - `house-solar` s'il y a des compteurs de production solaire.
- Le Bilan (05, écran 9) et la section « Migration en cours » de l'Accueil signalent l'outil communautaire **Isometric House Builder** de guim31, qui permet de dessiner sa propre maison pour ce widget : https://community.gladysassistant.com/t/isometric-house-builder-dessinez-votre-maison-pour-le-widget-vue-de-la-maison
- Les illustrations générées par Gladys Plus restent bien sûr disponibles dans l'éditeur du widget.

## 3. Reprise des tableaux de bord HA (approchée)

En plus de l'Accueil, les tableaux de bord HA sont repris **par défaut**, et placés **après** l'Accueil. Chacun devient un tableau de bord Gladys nommé « <nom HA> (HA) ». Chaque **vue** HA devient une **section** et chaque carte reconnue un widget.

**Ici, une approximation est acceptable** : contrairement aux scènes, un tableau de bord ne commande rien de lui-même. Une différence de rendu n'a pas de conséquence sur la maison.

| Carte HA | Widget Gladys | Remarque |
|---|---|---|
| `entities`, `tile`, `light`, `button` sur un appareil | `devices` | Une ligne par fonction migrée |
| `thermostat` | `thermostat` | |
| `history-graph`, `statistics-graph` | `chart` | Courbes des fonctions migrées |
| `gauge` | `gauge` | Bornes reprises |
| `weather-forecast` | `weather` | |
| `picture-entity`, `picture-glance` (caméra) | `camera` | |
| `button` lançant une scène HA | `scene` | Si la scène a été migrée |
| `area` | `devices-in-room` | |
| `energy-*` | `energy-consumption` | |
| `markdown`, `iframe` | `link` (si un lien) ou non reprise | |
| `horizontal-stack`, `vertical-stack`, `grid` | Contenu aplati dans la section | |
| `conditional`, cartes personnalisées (HACS : mushroom, button-card…), `custom:*` | **Non reprises** | Listées dans le rapport : « 7 cartes non reprises » |

- **Source** (détail : 02 § 3.2 et § 4.2) : WebSocket `lovelace/dashboards/list` et `lovelace/config` (par `url_path`), ou, avec une sauvegarde, `.storage/lovelace`, `.storage/lovelace.<id>` et `.storage/lovelace_dashboards` ([core] `components/lovelace/dashboard.py`, `websocket.py`). *Droits exacts et mode YAML à vérifier.*
- Le **tableau de bord automatique** de HA (aucune configuration enregistrée) n'est pas repris : l'Accueil le remplace avantageusement.
- Les cartes visant des entités **non migrées** sont omises, et comptées dans le rapport.

## 4. Choix de l'utilisateur et sécurité

- **Écran Choix** (05, écran 4) : l'**Accueil est toujours créé**. « Reprendre mes tableaux de bord Home Assistant » est **coché par défaut**, avec la liste de ceux trouvés, chacun décochable.
- **Ordre** : l'Accueil, puis les tableaux de bord HA repris. Si Gladys n'a encore aucun tableau de bord, l'Accueil devient le premier, c'est-à-dire la page d'arrivée. Si l'utilisateur en a déjà, les nouveaux sont **ajoutés à la fin**, sans toucher à l'ordre existant. L'outil propose seulement de placer l'Accueil en premier.
- **Création après les appareils et les scènes** : un tableau de bord référence des fonctions et des scènes qui doivent exister. Étape intégrée à l'écran 9 (Bilan) : aperçu, puis « Créer ».
- **Journal et annulation** (A9) : les tableaux de bord créés sont journalisés (clé : `url_path` HA, ou `home` pour l'Accueil, 01 § 5.1 bis) et supprimés à l'annulation. Une modification de l'ordre (`dashboard.updateOrder`, `server/lib/dashboard/dashboard.updateOrder.js`) est une modification journalisée, avec l'ordre d'avant (01 § 5.4, règle 2), restaurée à l'annulation.
- **Relance** (D13) : un tableau de bord déjà créé n'est **jamais régénéré** par-dessus, puisque l'utilisateur a pu le modifier. L'outil propose seulement d'ajouter les nouveaux appareils dans une section « Nouveautés ».

## 5. Décisions

| # | Décision |
|---|---|
| DB1 | Illustration choisie **automatiquement** dans la galerie de Gladys, avec un renvoi vers l'outil communautaire **Isometric House Builder** de guim31 pour la personnaliser (§ 2.1, ex-Q08-1). |
| DB2 | L'**Accueil est toujours créé**. Les tableaux de bord HA sont repris **par défaut** et placés **à la suite** (§ 3 et 4, ex-Q08-2). |
