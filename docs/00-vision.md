# 00 — Vision et périmètre

- **Statut** : 📝 Brouillon
- **Dernière mise à jour** : 2026-11-17

---

## 1. Contexte

Home Assistant (HA) est la plateforme domotique open source la plus répandue. Gladys Assistant mise sur la simplicité et le respect de la vie privée. Un utilisateur de HA qui veut passer à Gladys doit aujourd'hui tout recréer à la main : appareils, pièces et scénarios. Ce coût de sortie freine l'adoption de Gladys.

## 2. Objectif

Fournir dans Gladys un **service d'import depuis Home Assistant**. Il doit permettre à un utilisateur :

1. d'**analyser** son installation HA et de savoir précisément ce qui peut être migré ;
2. de **migrer ses appareils** (et leur organisation par pièce) vers les intégrations Gladys adéquates ;
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
| **Lot 1 — Appareils** | Appareils et entités HA, pièces (*areas*), personnes HA vers utilisateurs Gladys, rattachement aux services Gladys | V1 |
| **Lot 2 — Scénarios** | Automatisations HA (déclencheurs, conditions, actions) converties en scènes Gladys | V2 |
| **Transverse** | Rapport de compatibilité, suggestions d'intégrations alternatives, canevas de message pour le forum, rapport final | V1 |

### 4.2 Hors périmètre (pour l'instant)

- Historique des états et statistiques (énergie, graphiques).
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
| D8 | Intégrations HA prioritaires dans le catalogue V1 : **Zigbee2MQTT, MQTT, Z-Wave, Philips Hue, Tasmota, Shelly, Matter**. | Cadrage du 2026-11-17 (ex-Q6) |
| D9 | Versions supportées : **Gladys 5.3 minimum**, **HA dans sa version courante** au moment de la sortie. L'outil évoluera avec les versions des deux logiciels, ce qui suppose de suivre leurs changements de format. | Cadrage du 2026-11-17 (ex-Q2) |
| D10 | Les **mots de passe** sont demandés à l'utilisateur quand c'est nécessaire : sauvegarde HA chiffrée, intégrations et appareils qui en exigent un. Ils ne quittent pas l'instance. L'utilisateur est **prévenu** de l'usage et du stockage de chaque secret (voir § 9 bis). | Cadrage du 2026-11-17 (ex-Q3) |
| D11 | Les **personnes HA** sont migrées vers des **utilisateurs Gladys**, avec le même processus d'analyse, de sélection et de rapport que les appareils. | Cadrage du 2026-11-17 (ex-Q5) |
| D12 | Le **catalogue de correspondances** est **distant**, collaboratif et alimenté de la façon la plus automatisée possible. Il est mis à jour indépendamment des versions de Gladys. | Cadrage du 2026-11-17 (ex-Q7) |
| D13 | **Resynchronisation à la demande** : relancer l'outil détecte les nouveautés de HA (appareils, personnes, automatisations) et les propose à l'import. Il n'y a pas de synchronisation en direct. | Cadrage du 2026-11-17 (ex-Q8) |

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

## 10. Critères de réussite

- Un utilisateur dont l'installation ne contient que des intégrations prises en charge peut migrer **tous ses appareils et leurs pièces en moins de 15 minutes**, sans saisie manuelle hormis la configuration des services Gladys requis.
- **Aucune scène importée ne se comporte différemment** de l'automatisation HA d'origine (critère de fiabilité, vérifié par les tests).
- **100 % des éléments non migrés** ont un statut, une raison et une suggestion d'action.
- Relancer un import sur une même source ne crée **aucun doublon**.

## 11. Questions ouvertes

| # | Question | Section concernée |
|---|---|---|
| — | *Q1, Q2, Q3, Q5, Q6, Q7 et Q8 sont tranchées : voir D7 à D13.* | — |
| Q4 | Faut-il migrer les **étages** (*floors*) et les **étiquettes** (*labels*) de HA, alors que Gladys organise maisons et pièces différemment ? | 03 |
| Q9 | Forum : copier-coller du canevas uniquement, ou lien pré-rempli vers le forum (si celui-ci le permet) ? | 05 |
| Q10 | Adopter un **modèle de rapprochement** plutôt que de création : l'intégration Gladys cible découvre l'appareil, l'outil le rapproche de l'appareil HA par un identifiant matériel (IEEE, MAC, n° de série), puis applique nom et pièce. *Motif : les intégrations externes et Matter fixent elles-mêmes l'identifiant des appareils (voir note de recherche, § 7).* | 01, 03 |
| Q11 | Réviser D8 : plusieurs intégrations natives de la liste (Philips Hue, Tasmota, Z-Wave JS UI) sont **dépréciées** dans Gladys au profit d'intégrations externes ; Shelly n'existe qu'en externe. Règle proposée : « natif s'il est maintenu, sinon l'intégration externe recommandée ». | 03 |
| Q12 | Livrer une **version anglaise** de la spec dans `docs/specs/` avec la pull request vers Gladys, toutes les specs du dépôt Gladys étant en anglais. La spec de travail reste en français ici. | 01 |
| Q13 | L'import d'une sauvegarde HA demande de **modifier le cœur** de Gladys (pas de téléversement de fichier, corps des requêtes limité à 100 ko). À faire valider par les mainteneurs. | 01, 02 |

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
