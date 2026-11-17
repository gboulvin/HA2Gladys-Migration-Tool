# Brouillon — Fil du forum « Migration depuis Home Assistant »

> **Usage** : texte à copier dans un nouveau sujet du forum Gladys. Tout ce qui suit la ligne `---` est le message. Le forum traduit lui-même pour chaque lecteur. Avant publication : vérifier les liens (les specs pointent vers les pull requests tant qu'elles ne sont pas mergées) et remplacer « William » et « Pierre-Gilles » par leurs @pseudos du forum.

**Titre proposé** : Outil de migration Home Assistant → Gladys : présentation, specs et modifications du cœur

**Catégorie proposée** : Développement (ou celle que Pierre-Gilles préférera)

---

Bonjour à tous,

À la suite de la proposition de Pierre-Gilles, je me lance dans un **outil de migration de Home Assistant vers Gladys**. Ce fil sert à présenter le projet, à **expliquer et discuter les quelques modifications du cœur** qu'il demande, et à recueillir vos avis avant d'écrire la première ligne de code.

## Pourquoi

Quitter Home Assistant pour Gladys, c'est aujourd'hui tout refaire à la main : appareils, pièces, utilisateurs, scènes… et perdre son historique. Ce coût de sortie freine bien des curieux. L'objectif est qu'un utilisateur HA « standard » puisse migrer **en moins d'un quart d'heure**, et sache **précisément** ce qui ne peut pas l'être, et pourquoi.

## Ce que fera l'outil

Un **service intégré à Gladys**, accessible :
- **dès le premier lancement**, à côté de « Restaurer une sauvegarde Gladys Plus » ;
- puis dans **Paramètres → Migration**.

**Version 1**
- **Analyse** de l'installation HA, avec un rapport de compatibilité appareil par appareil : migrable, partiel, via une alternative, non migrable, ignoré.
- **Installation des intégrations** nécessaires depuis le store, en cochant des cases.
- **Appareils** (avec leur nom et leur pièce), **pièces**, **utilisateurs** et **intégrations de service** (météo, calendriers, notifications…).
- **Historiques** complets : statistiques long terme et états récents, y compris les compteurs d'énergie, avec une estimation de l'espace disque nécessaire.

**Version 2**
- **Scènes** : conversion des automatisations HA, **uniquement quand elle est fidèle à 100 %**. Tout le reste est signalé « à refaire », avec la raison.

**Quelques principes**
- **Lecture seule côté HA** : l'outil n'y modifie strictement rien.
- **Deux modes** : *coexistence* (on essaie Gladys sans arrêter HA) ou *bascule*. On passe de l'un à l'autre en relançant l'outil.
- **Annulation garantie** : l'outil ne défait que ce qu'il a fait, et remet Gladys exactement dans son état d'avant.
- **Rien ne sort de l'instance.** Le jeton et les mots de passe HA restent en mémoire, le temps nécessaire.
- **Relançable** : on relance l'outil pour récupérer ce qui a changé dans HA, sans créer de doublons.

**Pour les appareils sans intégration Gladys**, l'outil encourage d'abord l'utilisateur à **la créer lui-même**, grâce au modèle d'intégration officiel. Sinon, il lui propose de **publier une demande** dans « Demandes de fonctionnalités », avec un message déjà rédigé.

## Comment ça marche, en bref

L'outil ne « fabrique » pas d'appareils. Il **installe l'intégration Gladys** qui convient, la laisse **découvrir** les appareils, puis **rapproche** chaque appareil HA de l'appareil découvert par une **clé matérielle** : adresse MAC, adresse Zigbee (IEEE), numéro de série… Il applique ensuite le nom et la pièce venant de HA. Ce mode de fonctionnement marche aussi bien avec les intégrations natives qu'avec celles du store.

L'étude du code des 12 intégrations HA les plus utilisées montre que, pour 8 d'entre elles, ce rapprochement est **sûr** : Zigbee (Zigbee2MQTT et ZHA), Shelly, Tuya, Google Cast, UPnP, DLNA… Aucune n'est entièrement manuelle.

## Ce que je demande au cœur (pour discussion)

J'ai cherché à **toucher le cœur le moins possible**. Les bibliothèques nécessaires (lecture d'archives, YAML, déchiffrement des sauvegardes HA récentes) restent **propres au service**, comme `yaml` pour zigbee2mqtt.

| # | Modification | Pourquoi |
|---|---|---|
| 1 | **Trois nouvelles tables** `t_migration_run`, `t_migration_item` et `t_migration_history_cursor` | Garder la trace de ce qui a été importé : relance sans doublons, reprise après interruption, annulation exacte. Aucun modèle existant n'est modifié. Les tables ont une colonne `source` pour pouvoir servir un jour à d'autres systèmes (Jeedom, Domoticz…). |
| 2 | **Une page Paramètres → Migration** et une ligne sur l'écran d'accueil | Points d'entrée de l'outil |
| 3 | **Des params réservés `GLADYS_HW_*`** (`GLADYS_HW_MAC`, `GLADYS_HW_IEEE`, `GLADYS_HW_SERIAL`, `GLADYS_HW_UDN`…), sur le modèle de `GLADYS_TRANSPORT` | Que chaque intégration publie l'**identité matérielle** de ses appareils sous une forme standard. Le rapprochement devient alors générique, sans règle à écrire par intégration. Bénéfice au-delà de la migration : repérer un même appareil vu par deux intégrations, et proposer `device.migrate`. ⚠️ Un cœur ancien refuse les `GLADYS_*` inconnus : c'est le **SDK** qui ne les enverra qu'aux versions qui les acceptent. |
| 4 | *(Facultatif)* **Une route de téléversement en flux** des sauvegardes HA | Utile seulement si HA est hors d'usage. Dans le cas courant, Gladys **télécharge lui-même** la sauvegarde auprès de HA (comme pour Gladys Plus), déjà déchiffrée par HA. Le corps des requêtes JSON est aujourd'hui limité à 100 ko. |

Un point d'attention aussi : la purge quotidienne des états (90 jours par défaut) effacerait l'historique importé. L'outil proposera donc de passer la conservation à « illimitée », réglage que l'utilisateur pourra modifier ensuite. Cela ne demande aucune modification du cœur, mais je préfère que ce soit dit.

## Pistes d'évolution du cœur (pour la V2 et au-delà)

En étudiant le moteur de scènes pour la conversion des automatisations HA, j'ai relevé des manques. Les combler augmenterait directement ce que l'outil pourra convertir. **Ils ne sont pas nécessaires à la V1** : ce sont des propositions à discuter.

| # | Piste | Intérêt |
|---|---|---|
| P1 | **Modes d'exécution** des scènes (« ignorer si déjà en cours », « redémarrer », « mettre en file ») | Aujourd'hui, toutes les exécutions d'une scène tournent en parallèle. Une scène avec une attente peut donc se lancer plusieurs fois en même temps. C'est le gain le plus important. |
| P2 | Action **« arrêter une scène »** | Motifs à minuterie |
| P3 | Condition **solaire** (il fait jour ou nuit, avant ou après le coucher) | Éclairage, volets |
| P4 | Condition de **durée** (« dans cet état depuis N minutes ») | Alertes |
| P5 | **Déclencheur à l'origine** de la scène utilisable dans les conditions | Scènes à plusieurs déclencheurs et plusieurs comportements |

Au passage, j'ai trouvé un **bogue** : un déclencheur « message MQTT reçu » dont le topic contient un joker (`+` ou `#`) ne se déclenche jamais. Le ticket est ouvert ici : *(lien du ticket)*.

## Ce que je demande aux développeurs d'intégrations

Pas grand-chose, et seulement à la prochaine mise à jour du SDK :
1. publier l'identité matérielle de vos appareils avec l'aide du SDK (point 3 ci-dessus) ;
2. si besoin, ajouter à la racine de votre dépôt un petit fichier **`gladys-migration.json`**. Il décrit les exceptions (fonctions dont le nom diffère de HA, conversions d'unités) et les réglages à reprendre de HA (adresse d'un broker, clé de chiffrement ESPHome…). Le modèle `integration-template-js` en contiendra un exemple.

## Avec William et son tracker

Le **catalogue des correspondances** HA → Gladys s'appuiera sur le [tracker des intégrations](https://william-de71.github.io/gladys-integrations-tracker/) de William. Il fait déjà le lien entre intégrations HA et Gladys, entièrement automatiquement. Il suffirait d'y ajouter la collecte des fichiers `gladys-migration.json` et une colonne « Migration HA ». Une fois toutes les intégrations créées, le tracker resterait l'outil qui **mesure la couverture de la migration**. William, qu'en penses-tu ?

## Où en est-on

Les **spécifications de la V1 sont rédigées**, en français, en attendant leur version anglaise pour la pull request vers Gladys : https://github.com/gboulvin/HA2Gladys-Migration-Tool

| Section | Contenu |
|---|---|
| 00 — Vision | Objectifs, périmètre, décisions |
| 01 — Architecture | Service du cœur, tables, sécurité, annulation |
| 02 — Lecture de HA | API, sauvegardes (y compris chiffrées), historique |
| 03 — Correspondances | Catalogue, rapprochement, pièces, personnes |
| 05 — Parcours | Les écrans de l'assistant |
| 06 — Tests | Couverture à 100 %, CI de veille des versions de HA |
| 07 — Historiques | Import, énergie, purge |

La section 04 (scènes) est en cours. Tout le travail a vocation à être **rendu à l'organisation GladysAssistant** une fois mergé.

## Comment vous pouvez aider

- **Vos avis** sur les modifications du cœur, en particulier les params `GLADYS_HW_*`.
- **Des sauvegardes HA réelles**, que je sais anonymiser avec un script fourni (noms, adresses, numéros de série remplacés, secrets supprimés). Elles servent à tester sur de vraies installations et à mesurer les performances. Elles ne seront **jamais** publiées.
- **Des testeurs** pour l'**image Docker de test**, qui sera annoncée ici avant la pull request finale.

Merci d'avance pour vos retours !
