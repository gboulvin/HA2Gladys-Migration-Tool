# 00 — Vision et périmètre

- **Statut** : 📝 Brouillon
- **Dernière mise à jour** : 2026-11-19

---

## 1. Contexte

Home Assistant (HA) est la plateforme domotique open source la plus répandue. Gladys Assistant mise sur la simplicité et le respect de la vie privée. Un utilisateur de HA qui veut passer à Gladys doit aujourd'hui tout recréer à la main : appareils, pièces et scénarios. Ce coût de sortie freine l'adoption de Gladys.

## 2. Objectif

Fournir dans Gladys un **service d'import depuis Home Assistant**. Il doit permettre à un utilisateur :

1. d'**analyser** son installation HA et de savoir précisément ce qui peut être migré ;
2. de **migrer ses appareils** (avec leur organisation par pièce et leur historique), ses **personnes** et ses **services** vers les intégrations Gladys adéquates ;
3. de **migrer ses scénarios** (scènes, scripts et automatisations HA) vers des scènes Gladys, lorsque le comportement obtenu est similaire en fonctionnement normal (principe 1, S1) ;
4. de savoir **quoi faire pour le reste** : intégration alternative, demande à la communauté, ou action manuelle.

## 3. Utilisateurs cibles

| Profil | Besoin principal |
|---|---|
| Utilisateur HA « standard » (interface graphique, intégrations courantes) | Migration la plus automatique possible, avec un rapport clair |
| Utilisateur HA avancé (YAML, templates, blueprints) | Savoir exactement ce qui ne passera pas, et pourquoi |
| Utilisateur qui veut d'abord essayer Gladys | Faire tourner HA et Gladys en parallèle sans casser son installation HA |

## 4. Périmètre

### 4.1 Dans le périmètre

| Lot | Contenu | Priorité |
|---|---|---|
| **Lot 1 — Appareils** | Appareils et entités HA, **entrées HA en appareils virtuels** (D27), pièces (*areas*), personnes HA vers utilisateurs Gladys, **présence et zones** (D30), intégrations de type « service » (météo, calendriers, notifications…), rattachement aux intégrations Gladys | V1 |
| **Lot 1 bis — Historiques** | Historique des valeurs des appareils migrés (voir § 9 ter) | V1 |
| **Lot 1 bis — Tableaux de bord et modules** | Tableau de bord « Accueil » généré, reprise approchée des tableaux de bord HA (D28). Modules complémentaires HA associés à leurs équivalents Gladys (D29). | V1 |
| **Lot 1 ter — Scènes HA** | Instantanés d'états (« ambiances ») convertis en scènes Gladys sans déclencheur (D25) | V1 |
| **Lot 1 quater — Automatisations simples** | Automatisations HA simples (déclencheurs horaires et solaires, mouvement et ouverture, actions de base) converties en scènes Gladys (D26) | V1 |
| **Lot 2 — Assistant de migration des automatisations** | Outil distinct, pour les automatisations plus élaborées, au fil des évolutions du moteur de scènes de Gladys (D26) | V2 |
| **Transverse** | Rapport de compatibilité, suggestions d'intégrations alternatives, canevas de message pour le forum, rapport final | V1 |

### 4.2 Hors périmètre (pour l'instant)

- Données internes des modules complémentaires (*add-ons*), sauf Zigbee2MQTT (A11).
- Scripts HA à paramètres ou complexes (assistant V2) ; les scripts simples sont en V1 (D31).
- Synchronisation continue : l'import est une opération ponctuelle. L'utilisateur peut le relancer pour récupérer les nouveautés de HA, mais il n'y a pas de miroir permanent (voir D13).

*Les tableaux de bord (D28), les modules complémentaires (D29) et les scripts HA simples (D31), qui figuraient ici, sont passés en V1 (§ 4.1).*

## 5. Principes directeurs

1. **La fiabilité d'abord.** On ne migre que ce qui se comportera dans Gladys **comme dans HA en fonctionnement normal** (appareils joignables, commandes acceptées). Les seuls écarts admis sont **listés** (E1, E2, E4 : 04 § 2.1, S1) et **affichés** sur chaque élément concerné. Le reste est signalé et jamais approximé en silence. **Exception** : les tableaux de bord, qui ne commandent rien, sont repris de façon approchée (08 § 3, D28).
   *Reformulé le 2026-11-19 pour intégrer S1 et D28. Version d'origine : « On ne migre que ce qui sera fidèle à 100 %. »*
2. **Non destructif côté HA.** L'outil ne fait que **lire** HA. Il n'y modifie, ne désactive et ne supprime rien.
3. **Transparence.** Chaque élément HA reçoit un statut explicite et une explication, dès l'analyse et avant tout import.
4. **Contrôle utilisateur.** Rien n'est importé sans une sélection et une confirmation explicites.
5. **Idempotence.** Relancer un import ne crée pas de doublons. Les éléments déjà importés sont reconnus.
6. **Réversibilité.** Un import peut être annulé : on supprime de Gladys ce qu'il a créé et on restaure ce qu'il a modifié (A9). Seule exception, annoncée avant confirmation : l'historique HA importé sur des appareils qui existaient déjà dans Gladys est conservé (01 § 5.4).
7. **Confidentialité.** Tout reste local. Aucune donnée de l'installation n'est envoyée à un service tiers. Le jeton HA et le mot de passe de la sauvegarde restent en mémoire et ne sont jamais enregistrés. La sauvegarde est lue en flux : ses fichiers sensibles restent en mémoire, et seuls la base d'historique et, si besoin, l'archive Zigbee2MQTT sont écrites dans un dossier de travail, purgé à la fin de l'exécution (01 § 7.2). L'utilisateur est informé de la façon dont chaque secret est traité (voir D10).

## 6. Parcours utilisateur (vue d'ensemble)

```
┌──────────┐   ┌────────────┐   ┌────────────────────┐   ┌────────────┐   ┌───────────┐   ┌─────────┐   ┌─────────────┐
│ 1. Mode  │ → │ 2. Source  │ → │ 3. Analyse et      │ → │ 4. Sélec-  │ → │ 5. Prére- │ → │ 6. Im-  │ → │ 7. Rapport  │
│ coexist. │   │ API ou     │   │ rapport de         │   │ tion       │   │ quis      │   │ port    │   │ final et    │
│ / bascule│   │ sauvegarde │   │ compatibilité      │   │            │   │ (services)│   │ (C à F) │   │ suites      │
└──────────┘   └────────────┘   └────────────────────┘   └────────────┘   └───────────┘   └─────────┘   └─────────────┘
```

1. **Mode** : coexistence ou bascule (voir § 9).
2. **Source** : l'utilisateur choisit entre une connexion à son HA (URL et jeton d'accès longue durée) et le téléversement d'une sauvegarde HA.
3. **Analyse** : l'outil inventorie les appareils, entrées, personnes, zones, services, modules, scènes, scripts, automatisations et tableaux de bord, puis classe chaque élément (voir § 8).
4. **Sélection** : l'utilisateur coche ce qu'il veut importer et ajuste les pièces si besoin.
5. **Prérequis** : l'outil indique quels services Gladys activer ou configurer (par exemple le service Zigbee2MQTT) et vérifie qu'ils sont prêts.
6. **Import**, dans l'ordre des phases du § 6 bis : appareils, personnes, entrées, zones et présence (C), scènes (D), historiques (E), tableaux de bord (F).
7. **Rapport final** : bilan, liste des actions manuelles restantes et canevas de messages pour le forum. Le rapport est exportable.

L'ordre Mode puis Source est celui du parcours détaillé (05 § 3, écrans 0 à 9). La correspondance entre écrans et phases est au § 6 bis.

## 6 bis. Un seul outil, en plusieurs phases

Les scènes ne peuvent être converties que si les appareils qu'elles utilisent existent déjà dans Gladys. Or, avec le modèle de rapprochement (D16), un appareil n'existe qu'une fois son intégration installée, configurée, et l'appareil découvert. Cela prend parfois du temps : appairage d'un pont, bouton à presser, découverte réseau…

Plutôt que deux outils, on propose **un seul outil à phases, qui mémorise sa progression** :

Cette liste est la **référence unique des phases** (révisée le 2026-11-19, C5). Elle sert aussi de valeurs au champ `phase` de `t_migration_run` (01 § 5.2) et de structure au parcours (05 § 3).

| Phase | Valeur de `phase` | Contenu | Dépend de | Écrans (05) | Peut être reprise plus tard |
|---|---|---|---|---|---|
| A. Analyse | `analysis` | Lecture de HA, classification, rapport de compatibilité, choix de l'utilisateur | — | 0 à 4 | Oui (relance = nouvelle analyse, D13) |
| B. Intégrations | `integrations` | Installation des intégrations externes cochées (store) et configuration des intégrations natives et des modules (D29) | A | 5 | Oui |
| C. Appareils, personnes, entrées, zones, présence | `devices` | Rapprochement avec les appareils découverts, création, nom, pièce ; pièces ; utilisateurs ; appareils virtuels (D27) ; zones et coordonnées de la maison ; rattachement des capteurs de présence et scènes de présence générées (04 § 6.3) | B (appareils découverts) ; service MQTT pour les appareils virtuels | 6 | Oui, appareil par appareil |
| D. Scènes | `scenes` | Scènes et scripts HA, automatisations simples dont **tous** les appareils sont migrés | C (appareils, entrées et substituts référencés) | 7 | Oui : un élément « bloqué par un appareil » devient convertible dès que l'appareil est migré |
| E. Historiques | `history` | Import des historiques des fonctions migrées (07) | C (fonctions migrées). Indépendante de D. | 8 | Oui, par curseur (07 § 6) |
| F. Tableaux de bord | `dashboards` | Accueil généré, reprise des tableaux de bord HA (08) | C et D (fonctions et scènes référencées). E facultative : les graphiques se remplissent à mesure que l'historique arrive. | 9 | Oui |

L'utilisateur peut quitter l'outil après la phase B, laisser ses intégrations découvrir les appareils, puis revenir. L'outil reprend là où il en était. Ce choix est cohérent avec D13 (relance à la demande). Une relance, la bascule et l'annulation sont des **exécutions** d'une même migration (01 § 5.1 bis).

**Pourquoi un service du cœur, et non une intégration externe ?** L'orientation de Gladys est de sortir un maximum d'intégrations du cœur. Mais l'outil de migration doit installer des intégrations, créer des pièces, des utilisateurs et des scènes. Ce sont des actions d'administration qu'une intégration externe ne peut pas faire, puisqu'elle est isolée dans son conteneur. Il doit donc rester dans le cœur (D1). *Confirmé en 01 § 2, d'après le périmètre de l'API hôte.*

## 7. Décisions actées

| # | Décision | Origine |
|---|---|---|
| D1 | L'outil est un **service intégré à Gladys** (backend Node.js et frontend Gladys), pas un outil externe. | Cadrage du 2026-11-17 |
| D2 | **Deux sources** sont proposées : l'API HA en direct et l'import d'un fichier de sauvegarde HA. | Cadrage du 2026-11-17 |
| D3 | Les éléments non migrables sont **présentés avant l'import**. On propose des **intégrations alternatives** quand elles existent et un **canevas de message pour le forum** Gladys quand aucune intégration n'existe. | Cadrage du 2026-11-17 |
| D4 | **Deux modes** sont proposés au choix : **coexistence** (HA continue de tourner) et **bascule** (HA est abandonné). | Cadrage du 2026-11-17 |
| D5 | Les scénarios non convertibles fidèlement sont signalés **« à refaire manuellement »**. Aucune traduction approchée n'est faite. **Amendée le 2026-11-19 (S1, D28)** : « fidèlement » s'entend comme un comportement **similaire en fonctionnement normal**. Les seuls écarts admis sont listés (E1, E2, E4, 04 § 2.1) et affichés sur chaque scène concernée. Tout autre écart conduit à « à refaire manuellement », sans traduction approchée. Les **tableaux de bord** font exception : une reprise approchée est acceptable (08 § 3). | Cadrage du 2026-11-17, amendée le 2026-11-19 |
| D6 | Spécifications en **français**. Code, commentaires et commits en **anglais**. | Cadrage du 2026-11-17 |
| D7 | **Contribution officielle** au dépôt `GladysAssistant/Gladys`, à l'initiative de son fondateur (Pierre-Gilles). On suit donc les conventions, l'outillage et le processus de revue du projet Gladys. | Cadrage du 2026-11-17 (ex-Q1) |
| D8 | ~~Intégrations HA prioritaires dans le catalogue V1 : Zigbee2MQTT, MQTT, Z-Wave, Philips Hue, Tasmota, Shelly, Matter.~~ **Remplacée par D19.** | Cadrage du 2026-11-17 (ex-Q6) |
| D9 | Versions supportées : **Gladys 5.3 minimum**, **HA dans sa version courante** au moment de la sortie. L'outil évoluera avec les versions des deux logiciels, ce qui suppose de suivre leurs changements de format. | Cadrage du 2026-11-17 (ex-Q2) |
| D10 | Les **mots de passe** sont demandés à l'utilisateur quand c'est nécessaire : sauvegarde HA chiffrée, intégrations et appareils qui en exigent un. Ils ne quittent pas l'instance. L'utilisateur est **prévenu** de l'usage et du stockage de chaque secret (voir § 9 bis). | Cadrage du 2026-11-17 (ex-Q3) |
| D11 | Les **personnes HA** sont migrées vers des **utilisateurs Gladys**, avec le même processus d'analyse, de sélection et de rapport que les appareils. | Cadrage du 2026-11-17 (ex-Q5) |
| D12 | Le **catalogue de correspondances** est **distant**, collaboratif et alimenté de la façon la plus automatisée possible. Il est mis à jour indépendamment des versions de Gladys. | Cadrage du 2026-11-17 (ex-Q7) |
| D13 | **Resynchronisation à la demande** : relancer l'outil détecte les nouveautés de HA (appareils, personnes, automatisations) et les propose à l'import. Il n'y a pas de synchronisation en direct. | Cadrage du 2026-11-17 (ex-Q8) |
| D14 | Les **étages** HA ne sont pas migrés : Gladys n'a pas d'équivalent. Si l'installation HA compte plus d'un étage, l'outil **propose** de renommer les pièces en « Étage - Pièce ». Ce renommage est facultatif et modifiable, dans les limites de Gladys : nom de pièce unique et 40 caractères au maximum. Les **étiquettes** HA ne sont pas migrées non plus. | Cadrage du 2026-11-17 (ex-Q4) |
| D15 | Pour un appareil non migrable, l'outil **encourage d'abord l'utilisateur à créer l'intégration manquante**, en mettant en avant la simplicité du modèle officiel (`GladysAssistant/integration-template-js`) et la publication sans validation dans le store. Il termine par : « Impossible pour vous ? Demandez à la communauté ! », avec un **lien vers le forum** et le canevas de message prérempli. | Cadrage du 2026-11-17 (ex-Q9) |
| D16 | **Modèle de rapprochement** : l'outil ne fabrique pas d'appareils. L'intégration Gladys cible les découvre, et l'outil les rapproche des appareils HA par un identifiant matériel, puis leur applique nom et pièce. L'outil **installe lui-même les intégrations externes** nécessaires : liste de cases à cocher, puis validation, via l'API d'installation du store déjà présente dans le cœur. Voir § 6 bis. | Cadrage du 2026-11-17 (ex-Q10) |
| D17 | La spec de travail reste **en français** dans ce dépôt. Une **version anglaise** est livrée dans `docs/specs/` avec la pull request vers Gladys. | Cadrage du 2026-11-17 (ex-Q12) |
| D18 | Les **modifications du cœur** de Gladys (téléversement de sauvegarde, etc.) sont expliquées et argumentées dans un **fil dédié du forum Gladys**, ouvert par le porteur du projet. Pierre-Gilles valide l'ensemble des pull requests. | Cadrage du 2026-11-17 (ex-Q13) |
| D19 | Le catalogue V1 couvre en priorité les **10 intégrations HA les plus utilisées** d'après le [tracker communautaire](https://william-de71.github.io/gladys-integrations-tracker/) : Met.no, Google Cast, Bluetooth, MQTT, Matter, UPnP/IGD, Tuya, IPP, DLNA et ESPHome. Toutes ont un équivalent Gladys, natif ou dans le store. Détail et limites de la source : [`research/ha-integrations-usage.md`](research/ha-integrations-usage.md). | Cadrage du 2026-11-17 (ex-Q11) |
| D20 | Les 10 premières intégrations sont un **point de départ, pas une limite**. Le catalogue V1 y ajoute **Zigbee (ZHA et Zigbee2MQTT) et Shelly**. L'objectif à terme est de couvrir le maximum d'appareils, idéalement **tous**. Les intégrations de type **service**, sans appareil physique (météo, calendrier, notifications…), sont migrées elles aussi. | Cadrage du 2026-11-17 (ex-Q14) |
| D21 | ~~Le catalogue distant (D12) s'appuie sur le **tracker communautaire** de William-De71, développeur Gladys très actif qui avait été pressenti pour cet outil. Le format sera enrichi avec lui.~~ **Remplacée par D32.** | Cadrage du 2026-11-17 (ex-Q15) |
| D22 | Les **historiques de données** des appareils migrés font partie de la **V1**. Voir § 9 ter. | Cadrage du 2026-11-17 |
| D23 | Historiques : on importe **les deux niveaux**, les états bruts et les statistiques long terme. | Cadrage du 2026-11-17 (ex-Q16) |
| D24 | Historiques : **tout l'historique** est importé, sans limite de profondeur. Avant l'import, l'outil **estime le volume** nécessaire et le compare à **l'espace disque disponible**. Il avertit l'utilisateur, ou bloque l'import, si la place manque. | Cadrage du 2026-11-17 (ex-Q17) |
| D25 | Les **scènes HA** (instantanés d'états, sans déclencheur) sont migrées **dès la V1**, en scènes Gladys sans déclencheur lancées à la main. ~~La conversion des **automatisations** reste au lot 2 (V2).~~ **Complétée par D26** : les automatisations simples sont converties dès la V1. Voir 04 § 8 bis. | Cadrage du 2026-11-17, amendée le 2026-11-19 |
| D26 | **Migrer tout ce qui est migrable, sans que cela devienne un casse-tête pour l'utilisateur.** La V1 convertit les scènes HA (D25) et les **automatisations simples** : déclencheurs horaires et solaires, mouvement ou ouverture détectés, conditions simples, actions de base (04 § 2.4). Les automatisations plus élaborées relèvent d'un **assistant de migration des automatisations distinct** (V2), car le fonctionnement des deux moteurs diffère profondément. Les automatisations non converties sont présentées sous une forme lisible, pour que l'utilisateur les recrée lui-même : c'est aussi sa porte d'entrée dans les scènes de Gladys. | Cadrage du 2026-11-18 |
| D27 | Les **entrées HA** (*helpers* : `input_boolean`, `input_number`, `counter`) sont migrées **dès la V1** en **appareils virtuels** Gladys, avec les appareils. Plus généralement, le convertisseur cherche un **substitut** (appareil existant désigné par l'utilisateur, appareil virtuel, intégration) à toute entité HA sans équivalent dans le cœur (04 § 6). | Cadrage du 2026-11-18 |
| D28 | **Tableaux de bord dès la V1** : un tableau de bord « Accueil » est généré à partir des données migrées, pour que l'utilisateur ne découvre jamais une page vide. Les tableaux de bord HA peuvent en plus être repris de façon approchée (08). | Cadrage du 2026-11-18 |
| D29 | **Modules complémentaires HA (*add-ons*) dès la V1** : chacun est associé, via le catalogue, à un service natif de Gladys, à une intégration externe, ou signalé « sans objet » ou « sans équivalent ». Leurs données ne sont pas reprises, sauf celles de Zigbee2MQTT (A11). Voir 03 § 7 ter. | Cadrage du 2026-11-18 |
| D30 | La **présence** passe en **V1** (revient sur M2) : traqueurs réseau et Bluetooth rapprochés par MAC et rattachés aux utilisateurs, zones HA en zones Gladys, déclencheurs d'arrivée et de départ. Les traqueurs GPS demandent une configuration guidée. Voir 04 § 6.3. | Cadrage du 2026-11-18 |
| D31 | Les **scripts HA simples** (sans paramètres, actions du socle V1, attentes fixes comprises) sont migrés **dès la V1** en scènes Gladys sans déclencheur. Les automatisations qui les appellent deviennent ainsi convertibles. Voir 04 § 8 ter. | Cadrage du 2026-11-18 |
| D32 | Le **catalogue** est **produit par l'organisation GladysAssistant**, via l'**indexeur du store** (`GladysAssistant/integration-store`), qui lit `gladys-migration.json` au tag publié et publie `migration-catalog.json` à côté de `index.json`. Les règles des services natifs sont versionnées dans le dépôt Gladys. Le **tracker** de William fournit la table des domaines HA et mesure la couverture par niveaux. Sous réserve de l'accord de Pierre-Gilles. Voir 03 § 2. | Forum, 2026-11-19 (relecture de William-De71) |

Ces décisions seront formalisées en ADR dans `decisions/` au fil de la rédaction des sections concernées.

## 8. Classification des éléments : table de référence des statuts

Cette section est la **référence unique des statuts** (révisée le 2026-11-19, C4). Les autres sections (03 § 8, 04 § 2.3, 05, 09 § 4) y renvoient et n'en définissent pas d'autres. Chaque icône n'a qu'un sens.

### 8.1 Les statuts

| Statut | Signification | Ce que propose l'outil |
|---|---|---|
| ✅ **Migrable** / **convertible** | Un service Gladys gère l'élément et on sait le recréer, avec un comportement similaire en fonctionnement normal (principe 1). | Import direct |
| ✅ **Migrable (confirmation demandée)** | Comme ✅, mais le rapprochement est `probable` ou manuel (03 § 3.3). | Confirmation, groupée si possible (05, écran 6) |
| 🔀 **Migrable via alternative** | L'intégration HA d'origine n'a pas d'équivalent direct, mais l'appareil est pris en charge autrement : intégration au nom différent, protocole générique (MQTT, Matter…), autre passerelle (règle de détection : 03 § 8). | Explication de l'alternative et des étapes à suivre |
| ⚠️ **Migrable partiellement** / **partielle** | L'élément est pris en charge, mais une partie ne l'est pas : fonctions absentes de Gladys, attributs d'une scène non convertibles, cartes d'un tableau de bord non reprises, appel `scene.start` différé jusqu'à la bascule (04 § 7). | Import de ce qui est pris en charge, liste de ce qui manque |
| 🔗 **Bloquée par un appareil** | Scène, script ou automatisation qui serait convertible, mais un appareil référencé n'est pas migré. Statut **temporaire**, réévalué à chaque relance. | Liste des appareils manquants |
| 🔁 **Convertible par réécriture** | Automatisation reproductible seulement en la réécrivant en plusieurs scènes (04 § 5, S2). | Aperçu des scènes produites, accord élément par élément, décoché par défaut |
| ✋ **À refaire manuellement** | Au moins un élément sans équivalent (template Jinja, blueprint non résolu, service sans équivalent…). L'outil indique précisément lequel, avec une **sous-raison** : « impossible dans Gladys » ou « possible avec un appareil » (04 § 6.1). | Fiche lisible et brouillon de scène (04 § 2.5) |
| ❌ **Non migrable** | Aucune prise en charge dans Gladys. | Parcours D15 : créer l'intégration, puis canevas de message pour le forum, prérempli (intégration HA, fabricant, modèle, fonctions) |
| ⏭️ **Ignoré** / **sans objet** | Élément sans intérêt pour Gladys : entités système, diagnostics, mises à jour, entités désactivées dans HA, module « non nécessaire avec Gladys »… | Rien, mais l'élément reste visible dans le rapport détaillé |
| ⏳ **En attente de découverte** | Intégration installée, mais l'appareil n'a pas encore été découvert (01 § 6.1). | Conseil propre à l'intégration, bouton « Actualiser » |
| ☑️ **Déjà migré** | À la relance : élément déjà migré, objet Gladys toujours présent (01 § 5.3). | Rien |
| ✏️ **Modifié dans HA** | À la relance : élément déjà migré, mais modifié dans HA depuis (01 § 5.3). | Mettre à jour ou ignorer, au choix |
| 🗑️ **Supprimé dans HA** | À la relance : élément migré qui n'existe plus dans HA (01 § 5.3). | Signalé. L'objet Gladys n'est pas supprimé. |

Les **niveaux de couverture** 0 à 3 (03 § 2.3) mesurent un domaine HA. Ce ne sont pas des statuts d'élément.

### 8.2 Statuts possibles par type d'élément

| Type d'élément | Statuts possibles | Détail |
|---|---|---|
| **Appareils** (et leurs entités) | ✅, ✅ (confirmation), 🔀, ⚠️ (fonctions perdues), ❌, ⏭️, ⏳ | 03 § 8 |
| **Entrées HA** (*helpers*) | ✅ (`input_boolean`, `input_number`, `counter`), ❌ (autres types, avec la raison), ⏭️ | 03 § 7 bis, 04 § 6.2 |
| **Personnes** | ✅ (utilisateur créé), ☑️ (rapprochée du compte de l'administrateur, « C'est vous ? »), ⏭️ (non retenue) | 03 § 6 |
| **Présence** (traqueurs) | ✅ (réseau ou Bluetooth, rapproché par MAC), ⚠️ (GPS : configuration guidée), ⏳, ❌ | 04 § 6.3 |
| **Zones** | ✅ (`zone.home` vers les coordonnées de la maison, autres zones vers des zones Gladys), ⚠️ (zone créée, mais sans effet tant que la localisation de l'utilisateur n'est pas configurée) | 04 § 6.3 |
| **Services** (intégrations de type service) | ✅, ⚠️ (reconnexion obligatoire, par exemple OAuth), ❌ | 03 § 7 |
| **Modules** complémentaires | ✅ (équivalent natif ou externe), ⏭️ (sans objet), ❌ (sans équivalent) | 03 § 7 ter |
| **Scènes HA** et **scripts HA** | ✅, ⚠️ (partielle), 🔗, ✋ (scripts non simples), ⏭️ (scènes fournies par une intégration, non reprises) | 04 § 8 bis, § 8 ter |
| **Automatisations** | ✅, ⚠️ (appel `scene.start` différé, 04 § 7), 🔁, 🔗, ✋ | 04 § 2.3 |
| **Tableaux de bord** | ✅ (Accueil généré, tableau de bord HA repris), ⚠️ (cartes non reprises, listées), ⏭️ (tableau de bord automatique de HA) | 08 |
| **Tout type, à la relance** | ☑️, ✏️, 🗑️, ou le statut normal pour un nouvel élément | 01 § 5.3 |

L'ordre d'évaluation des scènes, scripts et automatisations (structure, puis appareils) est donné en 04 § 2.3.

Le **catalogue de correspondances** (intégration HA → service Gladys, alternatives, fonctions prises en charge) est une donnée centrale du projet. Il doit pouvoir être enrichi par la communauté sans toucher au code. Il sera détaillé dans `03-mapping-devices.md`.

## 9. Modes de migration

| | **Coexistence** | **Bascule** |
|---|---|---|
| Intention | Essayer Gladys sans renoncer à HA | Remplacer HA définitivement |
| HA après migration | Continue de tourner | Arrêté par l'utilisateur |
| Scènes issues d'**automatisations** | Créées **désactivées** par défaut, pour éviter qu'elles s'exécutent en double avec les automatisations HA | Créées actives (au choix de l'utilisateur) |
| Scènes issues de **scènes et scripts HA**, scènes de présence générées | Créées **actives** : sans déclencheur (ou sans effet sur les appareils), elles ne font pas double emploi (04 § 6.3, § 8 bis, § 8 ter) | Créées actives |
| Contraintes | Les deux systèmes doivent pouvoir accéder aux mêmes appareils | Il faut parfois réappairer les appareils |

**Point d'attention majeur : les contrôleurs radio.** Une clé Zigbee ou Z-Wave ne peut être pilotée que par un seul logiciel à la fois.

- Si HA utilise **Zigbee2MQTT** (ou Z-Wave JS UI) avec MQTT, Gladys peut se brancher sur la même instance. La coexistence est alors possible sans réappairage.
- Si HA utilise **ZHA** (Zigbee intégré à HA), la coexistence suppose une seconde clé, et la bascule suppose de réappairer ou de passer par Zigbee2MQTT.

L'outil doit détecter ces situations et les expliquer. Le détail sera traité dans `03-mapping-devices.md` et `05-ux.md`.

## 9 bis. Traitement des secrets

L'utilisateur doit savoir où se trouvent ses secrets et ce qu'ils deviennent. Ce qui suit a été vérifié dans le code de Gladys, et pour HA en 02 § 1 et § 8.

| Secret | Où il se trouve | Ce qu'en fait l'outil |
|---|---|---|
| Jeton d'accès longue durée HA | Saisi par l'utilisateur | Gardé en mémoire (coffre éphémère) le temps des phases qui l'utilisent, jamais enregistré. Redemandé si Gladys redémarre pendant une exécution qui en a encore besoin (01 § 7.2). L'utilisateur est invité à le révoquer dans HA après la migration. |
| Mot de passe ou clé de la sauvegarde HA | Saisi par l'utilisateur | Gardé en mémoire le temps de la lecture de la sauvegarde, jamais enregistré. Redemandé si la sauvegarde doit être relue après un redémarrage (01 § 7.2). |
| Identifiants des intégrations HA (broker MQTT, comptes cloud…) | **En clair** dans la configuration de HA (`.storage/`), donc en clair dans la sauvegarde une fois déchiffrée (confirmé : 02 § 1, constat 1) | Proposés pour préremplir la configuration du service Gladys cible, uniquement avec l'accord explicite de l'utilisateur. |
| Configuration des services Gladys | **En clair** dans la base SQLite de Gladys (table `t_variable`, sans chiffrement : `server/models/variable.js`). Les sauvegardes Gladys Plus, elles, sont chiffrées. | L'outil ne change rien à ce fonctionnement, mais l'écran de saisie le signale. |

> ⚠️ Contrairement à ce qu'on pourrait supposer, ni HA ni Gladys ne chiffrent les identifiants qu'ils stockent localement. La protection repose sur l'accès à la machine. L'outil ne doit **pas créer de nouvelle copie** de ces secrets : pas de journaux, pas de fichiers temporaires laissés sur disque, rapport exporté expurgé.

## 9 ter. Historiques de données

Ce qui suit sur Gladys a été vérifié dans le code. Ce qui concerne HA a été vérifié en 02 § 1 et § 5.

**Côté Gladys (vérifié)**
- L'historique est stocké dans DuckDB (`t_device_feature_state`).
- Le cœur sait déjà **insérer en masse des valeurs datées** (`device.saveMultipleHistoricalStates`), par lots internes de 10 000 états (`server/models/index.js:400-407`). C'est la seule fonction d'écriture utilisée par l'outil (07 § 4).
- ~~Il sait aussi **remplacer atomiquement l'historique à partir d'une date** (`device.replaceHistoricalStatesFrom`), ce qui permet de relancer un import sans doublons (D13).~~ **Corrigé le 2026-11-19 (F1)** : cette fonction supprime **tous** les états à partir de la date donnée, sans borne de fin (`server/models/index.js:424`). Elle effacerait l'historique enregistré par Gladys depuis la migration. L'outil ne l'utilise donc pas : l'absence de doublons repose sur des bornes d'import strictes (07 § 6).
- Seules les **valeurs numériques** sont historisées. Les états textuels (par exemple « en lecture ») ne le sont pas.

**Côté HA (vérifié en 02 § 5)**
- HA garde deux niveaux d'historique dans son *recorder* :
  - les **états bruts**, conservés **10 jours** par défaut ;
  - les **statistiques long terme**, des agrégats horaires (moyenne, minimum et maximum, ou cumul pour les compteurs d'énergie) **conservés indéfiniment**, mais uniquement pour les capteurs qui déclarent une `state_class`.
- La base est en SQLite par défaut, mais peut être en MariaDB ou PostgreSQL. Dans ce cas, elle **n'est pas dans la sauvegarde**, et seule l'API permet d'y accéder.
- L'utilisateur peut aussi exclure la base de données de ses sauvegardes.

**Principes retenus**
1. On n'importe l'historique que pour les fonctions d'appareils **effectivement migrées**.
2. L'import se fait en **tâche de fond avec progression** (`gladys.job`) et par lots, car le volume peut atteindre plusieurs millions de points.
3. Les **compteurs d'énergie** demandent une conversion à part : index cumulé ou consommation par période. Tranché en 07 § 3.3 (H2) : import de l'index réel, puis recalcul par Gladys.
4. Une relance n'importe que ce qui manque (D13) : un historique n'est importé que pour les fonctions **nouvellement migrées**, et jamais pour une période postérieure à la date de migration d'une fonction, puisque Gladys l'enregistre lui-même depuis (07 § 6).

Ce sujet a sa propre section de spec : `07-historiques.md`.

## 10. Critères de réussite

- Un utilisateur dont l'installation ne contient que des intégrations prises en charge peut migrer **tous ses appareils et leurs pièces en moins de 15 minutes**, sans saisie manuelle hormis la configuration des services Gladys requis.
- **Aucune scène importée ne se comporte différemment** de l'élément HA d'origine **en fonctionnement normal**. Les seuls écarts sont ceux listés en 04 § 2.1 (S1), affichés sur la scène concernée (critère de fiabilité, vérifié par les tests). *Amendé le 2026-11-19 (S1).*
- **100 % des éléments non migrés** ont un statut, une raison et une suggestion d'action.
- Relancer un import sur une même source ne crée **aucun doublon**.

## 11. Questions ouvertes

| # | Question | Section concernée |
|---|---|---|
| — | *Q1 à Q17 sont tranchées : voir D7 à D24. Aucune question ouverte à ce stade.* | — |

## 12. Glossaire

| Terme | Définition |
|---|---|
| **Intégration (HA)** | Module HA qui gère une marque, un protocole ou un service (ex. : Philips Hue, ZHA, MQTT). |
| **Appareil (HA)** | Objet physique ou logique du registre des appareils de HA. Il regroupe des entités. |
| **Entité (HA)** | Une fonction d'un appareil (ex. : `light.salon`, `sensor.salon_temperature`). |
| **Area (HA)** | Pièce dans HA. |
| **Automatisation (HA)** | Règle déclencheurs → conditions → actions. |
| **Service (Gladys)** | Intégration dans Gladys (ex. : Zigbee2MQTT, Philips Hue). |
| **Device / Feature (Gladys)** | Appareil Gladys et ses fonctions (*features*), chacune typée par une catégorie et un type. |
| **Scène (Gladys)** | Équivalent Gladys d'une automatisation : déclencheurs et actions (les conditions sont exprimées sous forme d'actions de contrôle). |
| **Catalogue de correspondances** | Référentiel qui associe intégrations et entités HA aux services et fonctions Gladys. |
| **Migration** | Ensemble du travail mené depuis une source HA, de la première analyse à la bascule ou à l'annulation. Une seule est ouverte à la fois (01 § 5.1 bis). |
| **Exécution** | Un passage de l'outil au sein d'une migration : analyse initiale, relance, bascule ou annulation (01 § 5.1 bis). |
| **Phase** | Étape A à F d'une exécution (§ 6 bis). |
