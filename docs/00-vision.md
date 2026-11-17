# 00 — Vision et périmètre

- **Statut** : 📝 Brouillon
- **Dernière mise à jour** : 2026-11-17

---

## 1. Contexte

Home Assistant (HA) est la plateforme domotique open source la plus répandue. Gladys Assistant mise sur la simplicité et le respect de la vie privée. Un utilisateur de HA qui veut passer à Gladys doit aujourd'hui tout recréer à la main : appareils, pièces et scénarios. Ce coût de sortie freine l'adoption de Gladys.

## 2. Objectif

Fournir dans Gladys un **service d'import depuis Home Assistant**. Il doit permettre à un utilisateur :

1. d'**analyser** son installation HA et de savoir précisément ce qui peut être migré ;
2. de **migrer ses appareils** (avec leur organisation par pièce et leur historique), ses **personnes** et ses **services** vers les intégrations Gladys adéquates ;
3. de **migrer ses scénarios** (automatisations HA) vers des scènes Gladys, lorsque la conversion est fidèle ;
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
| **Lot 1 — Appareils** | Appareils et entités HA, pièces (*areas*), personnes HA vers utilisateurs Gladys, intégrations de type « service » (météo, calendriers, notifications…), rattachement aux intégrations Gladys | V1 |
| **Lot 1 bis — Historiques** | Historique des valeurs des appareils migrés (voir § 9 ter) | V1 |
| **Lot 2 — Scénarios** | Automatisations HA (déclencheurs, conditions, actions) converties en scènes Gladys | V2 |
| **Transverse** | Rapport de compatibilité, suggestions d'intégrations alternatives, canevas de message pour le forum, rapport final | V1 |

### 4.2 Hors périmètre (pour l'instant)

- Tableaux de bord (*dashboards* Lovelace).
- Modules complémentaires (*add-ons*) et leur configuration.
- Scripts HA appelés de façon autonome (à réévaluer au lot 2 : un script appelé par une automatisation pourrait être « déplié » dans la scène).
- Synchronisation continue : l'import est une opération ponctuelle. L'utilisateur peut le relancer pour récupérer les nouveautés de HA, mais il n'y a pas de miroir permanent (voir D13).

## 5. Principes directeurs

1. **La fiabilité d'abord.** On ne migre que ce qui sera fidèle à 100 %. Le reste est signalé et jamais approximé en silence.
2. **Non destructif côté HA.** L'outil ne fait que **lire** HA. Il n'y modifie, ne désactive et ne supprime rien.
3. **Transparence.** Chaque élément HA reçoit un statut explicite et une explication, dès l'analyse et avant tout import.
4. **Contrôle utilisateur.** Rien n'est importé sans une sélection et une confirmation explicites.
5. **Idempotence.** Relancer un import ne crée pas de doublons. Les éléments déjà importés sont reconnus.
6. **Réversibilité.** Un import peut être annulé : on supprime de Gladys ce qu'il a créé.
7. **Confidentialité.** Tout reste local. Aucune donnée de l'installation n'est envoyée à un service tiers. Le jeton HA, le mot de passe de la sauvegarde et la sauvegarde elle-même restent en mémoire le temps de l'analyse et ne sont jamais enregistrés. L'utilisateur est informé de la façon dont chaque secret est traité (voir D10).

## 6. Parcours utilisateur (vue d'ensemble)

```
┌────────────┐   ┌──────────┐   ┌────────────────────┐   ┌────────────┐   ┌───────────┐   ┌─────────┐   ┌─────────────┐
│ 1. Source  │ → │ 2. Mode  │ → │ 3. Analyse et      │ → │ 4. Sélec-  │ → │ 5. Prére- │ → │ 6. Im-  │ → │ 7. Rapport  │
│ API ou     │   │ coexist. │   │ rapport de         │   │ tion       │   │ quis      │   │ port    │   │ final et    │
│ sauvegarde │   │ / bascule│   │ compatibilité      │   │            │   │ (services)│   │         │   │ suites      │
└────────────┘   └──────────┘   └────────────────────┘   └────────────┘   └───────────┘   └─────────┘   └─────────────┘
```

1. **Source** : l'utilisateur choisit entre une connexion à son HA (URL et jeton d'accès longue durée) et le téléversement d'une sauvegarde HA.
2. **Mode** : coexistence ou bascule (voir § 9).
3. **Analyse** : l'outil inventorie les appareils et les automatisations, puis classe chaque élément (voir § 8).
4. **Sélection** : l'utilisateur coche ce qu'il veut importer et ajuste les pièces si besoin.
5. **Prérequis** : l'outil indique quels services Gladys activer ou configurer (par exemple le service Zigbee2MQTT) et vérifie qu'ils sont prêts.
6. **Import** : création des pièces, des appareils puis, au lot 2, des scènes.
7. **Rapport final** : bilan, liste des actions manuelles restantes et canevas de messages pour le forum. Le rapport est exportable.

## 6 bis. Un seul outil, en plusieurs phases

Les scènes ne peuvent être converties que si les appareils qu'elles utilisent existent déjà dans Gladys. Or, avec le modèle de rapprochement (D16), un appareil n'existe qu'une fois son intégration installée, configurée, et l'appareil découvert. Cela prend parfois du temps : appairage d'un pont, bouton à presser, découverte réseau…

Plutôt que deux outils, on propose **un seul outil à phases, qui mémorise sa progression** :

| Phase | Contenu | Peut être reprise plus tard |
|---|---|---|
| A. Analyse | Lecture de HA, classification, rapport de compatibilité | Oui (relance = nouvelle analyse, D13) |
| B. Intégrations | Installation des intégrations externes cochées (store) et configuration des intégrations natives | Oui |
| C. Appareils et personnes | Rapprochement avec les appareils découverts, création, nom, pièce ; création des utilisateurs | Oui, appareil par appareil |
| D. Scènes | Conversion des automatisations dont **tous** les appareils sont migrés | Oui : une automatisation « bloquée par un appareil » devient convertible dès que l'appareil est migré |

L'utilisateur peut quitter l'outil après la phase B, laisser ses intégrations découvrir les appareils, puis revenir. L'outil reprend là où il en était. Ce choix est cohérent avec D13 (relance à la demande).

**Pourquoi un service du cœur, et non une intégration externe ?** L'orientation de Gladys est de sortir un maximum d'intégrations du cœur. Mais l'outil de migration doit installer des intégrations, créer des pièces, des utilisateurs et des scènes. Ce sont des actions d'administration qu'une intégration externe ne peut pas faire, puisqu'elle est isolée dans son conteneur. Il doit donc rester dans le cœur (D1). *À confirmer en section 01 en vérifiant le périmètre exact de l'API hôte.*

## 7. Décisions actées

| # | Décision | Origine |
|---|---|---|
| D1 | L'outil est un **service intégré à Gladys** (backend Node.js et frontend Gladys), pas un outil externe. | Cadrage du 2026-11-17 |
| D2 | **Deux sources** sont proposées : l'API HA en direct et l'import d'un fichier de sauvegarde HA. | Cadrage du 2026-11-17 |
| D3 | Les éléments non migrables sont **présentés avant l'import**. On propose des **intégrations alternatives** quand elles existent et un **canevas de message pour le forum** Gladys quand aucune intégration n'existe. | Cadrage du 2026-11-17 |
| D4 | **Deux modes** sont proposés au choix : **coexistence** (HA continue de tourner) et **bascule** (HA est abandonné). | Cadrage du 2026-11-17 |
| D5 | Les scénarios non convertibles fidèlement sont signalés **« à refaire manuellement »**. Aucune traduction approchée n'est faite. | Cadrage du 2026-11-17 |
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
| D21 | Le catalogue distant (D12) s'appuie sur le **tracker communautaire** de William-De71, développeur Gladys très actif qui avait été pressenti pour cet outil. Le format sera enrichi avec lui. | Cadrage du 2026-11-17 (ex-Q15) |
| D22 | Les **historiques de données** des appareils migrés font partie de la **V1**. Voir § 9 ter. | Cadrage du 2026-11-17 |
| D23 | Historiques : on importe **les deux niveaux**, les états bruts et les statistiques long terme. | Cadrage du 2026-11-17 (ex-Q16) |
| D24 | Historiques : **tout l'historique** est importé, sans limite de profondeur. Avant l'import, l'outil **estime le volume** nécessaire et le compare à **l'espace disque disponible**. Il avertit l'utilisateur, ou bloque l'import, si la place manque. | Cadrage du 2026-11-17 (ex-Q17) |

Ces décisions seront formalisées en ADR dans `decisions/` au fil de la rédaction des sections concernées.

## 8. Classification des éléments

Chaque appareil HA reçoit un des statuts ci-dessous.

| Statut | Signification | Ce que propose l'outil |
|---|---|---|
| ✅ **Migrable** | Un service Gladys gère l'appareil et on sait le recréer fidèlement. | Import direct |
| 🔀 **Migrable via alternative** | L'intégration HA d'origine n'a pas d'équivalent direct, mais l'appareil est pris en charge autrement : intégration au nom différent, protocole générique (MQTT, Matter…), autre passerelle. | Explication de l'alternative et des étapes à suivre |
| ⚠️ **Migrable partiellement** | L'appareil est pris en charge, mais certaines de ses fonctions n'existent pas dans Gladys. | Import des fonctions prises en charge, liste de celles perdues |
| ❌ **Non migrable** | Aucune prise en charge dans Gladys. | Canevas de message pour le forum (demande d'intégration), pré-rempli avec les informations utiles (intégration HA, fabricant, modèle, fonctions) |
| ⏭️ **Ignoré** | Élément sans intérêt pour Gladys : entités système, diagnostics, mises à jour, entités désactivées dans HA… | Rien, mais l'élément reste visible dans le rapport détaillé |

Pour les automatisations (lot 2), les statuts sont :

| Statut | Signification |
|---|---|
| ✅ **Convertible** | Tous les déclencheurs, conditions et actions ont un équivalent exact dans Gladys, et tous les appareils référencés sont migrés. |
| 🔗 **Bloquée par un appareil** | Elle serait convertible, mais un appareil référencé n'est pas migré. Elle le devient une fois l'appareil migré. |
| ✋ **À refaire manuellement** | Elle contient au moins un élément sans équivalent fidèle (template Jinja, blueprint non résolu, service sans équivalent…). L'outil indique précisément lequel. |

Le **catalogue de correspondances** (intégration HA → service Gladys, alternatives, fonctions prises en charge) est une donnée centrale du projet. Il doit pouvoir être enrichi par la communauté sans toucher au code. Il sera détaillé dans `03-mapping-devices.md`.

## 9. Modes de migration

| | **Coexistence** | **Bascule** |
|---|---|---|
| Intention | Essayer Gladys sans renoncer à HA | Remplacer HA définitivement |
| HA après migration | Continue de tourner | Arrêté par l'utilisateur |
| Scènes importées | Créées **désactivées** par défaut, pour éviter qu'elles s'exécutent en double avec les automatisations HA | Créées actives (au choix de l'utilisateur) |
| Contraintes | Les deux systèmes doivent pouvoir accéder aux mêmes appareils | Il faut parfois réappairer les appareils |

**Point d'attention majeur : les contrôleurs radio.** Une clé Zigbee ou Z-Wave ne peut être pilotée que par un seul logiciel à la fois.

- Si HA utilise **Zigbee2MQTT** (ou Z-Wave JS UI) avec MQTT, Gladys peut se brancher sur la même instance. La coexistence est alors possible sans réappairage.
- Si HA utilise **ZHA** (Zigbee intégré à HA), la coexistence suppose une seconde clé, et la bascule suppose de réappairer ou de passer par Zigbee2MQTT.

L'outil doit détecter ces situations et les expliquer. Le détail sera traité dans `03-mapping-devices.md` et `05-ux.md`.

## 9 bis. Traitement des secrets

L'utilisateur doit savoir où se trouvent ses secrets et ce qu'ils deviennent. Ce qui suit a été vérifié dans le code de Gladys ; le comportement de HA est à confirmer dans `02-extraction-ha.md`.

| Secret | Où il se trouve | Ce qu'en fait l'outil |
|---|---|---|
| Jeton d'accès longue durée HA | Saisi par l'utilisateur | Gardé en mémoire le temps de l'analyse, jamais enregistré. L'utilisateur est invité à le révoquer dans HA après la migration. |
| Mot de passe ou clé de la sauvegarde HA | Saisi par l'utilisateur | Gardé en mémoire le temps du déchiffrement, jamais enregistré. |
| Identifiants des intégrations HA (broker MQTT, comptes cloud…) | **En clair** dans la configuration de HA (`.storage/`), donc en clair dans la sauvegarde une fois déchiffrée *(à confirmer)* | Proposés pour préremplir la configuration du service Gladys cible, uniquement avec l'accord explicite de l'utilisateur. |
| Configuration des services Gladys | **En clair** dans la base SQLite de Gladys (table `t_variable`, sans chiffrement : `server/models/variable.js`). Les sauvegardes Gladys Plus, elles, sont chiffrées. | L'outil ne change rien à ce fonctionnement, mais l'écran de saisie le signale. |

> ⚠️ Contrairement à ce qu'on pourrait supposer, ni HA ni Gladys ne chiffrent les identifiants qu'ils stockent localement. La protection repose sur l'accès à la machine. L'outil ne doit **pas créer de nouvelle copie** de ces secrets : pas de journaux, pas de fichiers temporaires laissés sur disque, rapport exporté expurgé.

## 9 ter. Historiques de données

Ce qui suit sur Gladys a été vérifié dans le code. Ce qui concerne HA vient de sa documentation et sera vérifié dans `02-extraction-ha.md`.

**Côté Gladys (vérifié)**
- L'historique est stocké dans DuckDB (`t_device_feature_state`).
- Le cœur sait déjà **insérer en masse des valeurs datées** (`device.saveMultipleHistoricalStates`) et **remplacer atomiquement l'historique à partir d'une date** (`device.replaceHistoricalStatesFrom`). Ce second mécanisme permet de relancer un import sans doublons (D13).
- Seules les **valeurs numériques** sont historisées. Les états textuels (par exemple « en lecture ») ne le sont pas.

**Côté HA (à vérifier)**
- HA garde deux niveaux d'historique dans son *recorder* :
  - les **états bruts**, conservés **10 jours** par défaut ;
  - les **statistiques long terme**, des agrégats horaires (moyenne, minimum et maximum, ou cumul pour les compteurs d'énergie) **conservés indéfiniment**, mais uniquement pour les capteurs qui déclarent une `state_class`.
- La base est en SQLite par défaut, mais peut être en MariaDB ou PostgreSQL. Dans ce cas, elle **n'est pas dans la sauvegarde**, et seule l'API permet d'y accéder.
- L'utilisateur peut aussi exclure la base de données de ses sauvegardes.

**Principes retenus**
1. On n'importe l'historique que pour les fonctions d'appareils **effectivement migrées**.
2. L'import se fait en **tâche de fond avec progression** (`gladys.job`) et par lots, car le volume peut atteindre plusieurs millions de points.
3. Les **compteurs d'énergie** demandent une conversion à part : index cumulé ou consommation par période. Elle sera traitée avec les correspondances de fonctions (section 03).
4. Une relance n'importe que ce qui manque (D13).

Ce sujet a sa propre section de spec : `07-historiques.md`.

## 10. Critères de réussite

- Un utilisateur dont l'installation ne contient que des intégrations prises en charge peut migrer **tous ses appareils et leurs pièces en moins de 15 minutes**, sans saisie manuelle hormis la configuration des services Gladys requis.
- **Aucune scène importée ne se comporte différemment** de l'automatisation HA d'origine (critère de fiabilité, vérifié par les tests).
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
