# 06 — Stratégie de test

- **Statut** : 📝 Brouillon
- **Dernière mise à jour** : 2026-11-17
- **Prérequis** : [01 — Architecture](01-architecture.md) (§ 10), [02 — Lecture HA](02-extraction-ha.md), [03 — Correspondances](03-mapping-devices.md), [07 — Historiques](07-historiques.md)
- **Référence** : conventions de contribution de Gladys, résumées dans la [note de recherche](research/gladys-codebase.md) § 1 (`AGENTS.md`, `codecov.yml`, `.github/PULL_REQUEST_TEMPLATE.md`)

---

## 1. Les exigences de Gladys

| Exigence | Source | Conséquence pour nous |
|---|---|---|
| **Couverture de 100 % des lignes modifiées** (*patch*) | `codecov.yml` | Chaque ligne serveur du service doit être exécutée par un test. `AGENTS.md` décrit cette règle comme la cause d'échec la plus fréquente. |
| Couverture globale ≥ 90 % (*project*, serveur) | `codecov.yml` | Pas de baisse due au service |
| Mocha, Chai (chai-as-promised), Sinon, `proxyquire`, `nock` | `server/package.json`, `server/test/hooks.js` | Aucun autre outil de test côté serveur |
| Tests en miroir du code : `server/test/services/<service>/…` | `AGENTS.md` | `server/test/services/migration-home-assistant/{api,lib,utils}` |
| Base SQLite de test pré-remplie (*seeders*) | `server/test/hooks.js`, `server/seeders/` | Pas de nouvelles données de base sans *seeder* |
| Front : pas de tests unitaires, mais **Cypress** pour l'interface | `front/cypress/e2e/`, modèle de PR (« Cypress si UI ») | Un parcours Cypress par écran de l'assistant |
| ESLint (`airbnb-base`, JSDoc obligatoire sur les fonctions exportées), Prettier | `server/.eslintrc.json` | À respecter dès la première ligne |
| Traductions identiques dans les 4 langues | `npm run compare-translations` | Chaque clé ajoutée l'est en en, fr, de et es |

## 2. Pyramide de tests

```
                ┌─────────────────────────┐
                │  Bout en bout réel (§6) │  hors CI Gladys : HA réel, sauvegardes réelles
                ├─────────────────────────┤
                │  Cypress (§5)           │  parcours de l'assistant, API simulée
                ├─────────────────────────┤
                │  Intégration (§4)       │  service complet sur base de test, HA simulé
                ├─────────────────────────┤
                │  Unitaires (§3)         │  le gros du volume : fonctions pures
                └─────────────────────────┘
```

## 3. Tests unitaires

L'architecture isole les fonctions pures (01 § 10). Ce sont elles qui portent l'essentiel de la couverture :

| Composant | Ce qu'on teste | Données |
|---|---|---|
| **Normalisation** (02 § 6) | Formes de MAC, IEEE (`xx:xx` contre `0x…`), UDN. Appareils scindés (2026.8) regroupés par connexion. Appareils enfants rattachés à leur parent. Nom (`name_by_user`) et pièce (entité ou appareil). | Fragments de registres HA |
| **Lecture des fichiers `.storage`** | Versions acceptées, refusées ou signalées (02 § 7). Champs manquants. Format antérieur à 2026.8 refusé (E2). | Un fichier par version de format |
| **Analyseur** (01 § 3.4) | Chaque statut de la vision (§ 8) et de 03 § 8, sur des inventaires construits pour l'occasion | Inventaires et catalogues de test |
| **Rapprocheur** (03 § 3.3) | Chaque opérateur (`equals`, `suffix`, `ends_with`), chaque normalisation, confiance `sure` et `probable`, aucun candidat, plusieurs candidats, exclusions, cardinalités (enfants, Matter par endpoint), convention `GLADYS_HW_*` prioritaire sur les règles déclarées | Paires « appareil HA, appareils découverts » réalistes, tirées de la [note sur les clés](research/matching-keys.md) |
| **Correspondance des fonctions** (03 § 3.4) | Règles par expression régulière, tables de traduction, transformations (`invert`, `scale`), repli « triplet unique », refus quand le triplet est ambigu | |
| **Validation du catalogue** (03 § 3.6) | Schéma JSON, entrée invalide écartée, expression régulière trop longue ou trop lente refusée | Catalogues valides et piégés |
| **Pièces** (03 § 5) | Préfixe d'étage, troncature à 40 caractères, doublons, réutilisation d'une pièce existante | |
| **Historiques** (07) | Raccord statistiques et états bruts, conversion d'unités, binaires et inversion, filtrage `unknown`/`unavailable`, index de compteur, calcul des bornes pour un appareil existant, estimation du volume et de la marge disque, réglage de conservation (H1) | Séries synthétiques |
| **Coffre des secrets** (01 § 7.2) | Effacement en fin de phase, à l'abandon et à l'arrêt. **Aucun secret dans les journaux** : on espionne le *logger* et on vérifie qu'aucune valeur secrète n'y apparaît. | |
| **Export du rapport** | Markdown produit, **expurgé** : aucune valeur secrète ni identifiant matériel | |

## 4. Tests d'intégration (serveur)

Le service complet tourne sur la base de test de Gladys. Les dépendances externes sont simulées.

| Dépendance | Simulation |
|---|---|
| **API WebSocket de HA** | Un **faux serveur HA** en mémoire (bibliothèque `ws`, déjà présente) qui rejoue des réponses enregistrées : authentification, registres, `automation/config`, historique, statistiques, erreurs (`unauthorized`, version trop ancienne, coupure de connexion). |
| **Téléchargement de sauvegarde** | `nock` sur `/api/backup/download/<id>`, qui renvoie une archive de test (§ 7) |
| **Store des intégrations** | Fonctions `externalIntegration.*` simulées avec Sinon : installation, configuration, appareils découverts |
| **Catalogue distant** | `nock`, avec les cas « injoignable » (repli sur le cache, puis sur l'instantané embarqué) |
| **Espace disque** | `system.getDiskSpace` simulé : vert, orange, rouge |

Scénarios couverts :
1. **Migration complète** en mode A, sur une installation de test : inventaire, installation simulée, rapprochement, création des pièces, appareils et utilisateurs, import d'historique. On vérifie ensuite la base Gladys (objets créés, journal `t_migration_*`).
2. **Relance** : rien n'est dupliqué. Un élément modifié dans HA est signalé, un nouvel élément est proposé (D13).
3. **Reprise** après interruption à chaque phase, y compris au milieu de l'historique (curseur, 07 § 6).
4. **Annulation** (A9) : après une migration sur une instance qui avait déjà des appareils, des pièces et un historique, la base Gladys revient **exactement** à son état initial. On compare un instantané des tables (hors horodatages techniques) avant la migration et après l'annulation. C'est le test qui garantit la promesse faite à l'utilisateur.
5. **Coexistence vers bascule** (A11).
6. **Modes B et C**, et leurs limites (pas de préremplissage en B, automatisations partielles en C).

## 5. Tests Cypress (front)

Comme les autres écrans de Gladys, les réponses du serveur sont simulées avec `cy.intercept` (`front/cypress/e2e/routes/…`) :

| Parcours | Points vérifiés |
|---|---|
| Accueil « Vous venez de Home Assistant ? » | Suggestions de pièces masquées, bouton « Commencer l'import » à la fin |
| Écran Source | Erreurs (jeton non administrateur, version trop ancienne), avertissement HTTP, « Continuer sans sauvegarde » |
| Rapport d'analyse | Onglets, alertes ZHA et Matter, parcours « non migrable » (bouton du forum, canevas sans donnée personnelle) |
| Choix | Préfixe d'étage, conflits de noms, champs obligatoires des personnes |
| Intégrations | Cases à cocher, secrets masqués avec accord individuel, état « À configurer » |
| Appareils | Confirmation groupée des rapprochements probables, choix entre plusieurs candidats, réappairage qui se coche au fil des découvertes (événements WebSocket simulés) |
| Historiques | Jauge verte, orange et rouge, décochage avec recalcul |
| Paramètres → Migration | Reprendre, relancer, annuler avec confirmation renforcée |
| **Mobile** | Les mêmes parcours en fenêtre de téléphone (390 px), comme les tests mobiles existants de Gladys |

## 6. Bout en bout réel (hors CI de Gladys)

La CI de Gladys ne peut pas faire tourner un vrai Home Assistant à chaque pull request. Mais l'outil dépend d'un logiciel tiers qui change tous les mois (D9). D'où une **CI de veille**, développée dans ce dépôt puis **reprise par l'organisation `GladysAssistant`** une fois le projet mergé (T1) :

| Tâche | Fréquence | Contenu |
|---|---|---|
| **Veille des formats HA** | À chaque version de HA (mensuelle) et chaque semaine sur la bêta | Démarre l'**image Docker officielle de HA**, la peuple par script (appareils de démonstration, pièces, étages, personnes, automatisations, historique), crée une sauvegarde **chiffrée**. Puis lance la lecture de l'outil, en mode API et en mode sauvegarde. Compare les versions de formats lues à la table des versions prises en charge (02 § 7). Ouvre un ticket en cas d'écart. |
| **Vecteurs de chiffrement** | À chaque version de `securetar` | Archives chiffrées en v2 et v3 par la **bibliothèque de référence Python**, avec un mot de passe connu, que l'outil doit savoir déchiffrer (§ 7) |
| **Migration réelle** | Avant chaque version de l'outil | HA de test (comme ci-dessus), plus Gladys en Docker avec le service, et quelques intégrations du store. Parcours complet automatisé par l'API, puis annulation et vérification. |

## 7. Jeux de données

| Jeu | Contenu | Origine | Où |
|---|---|---|---|
| **Inventaires synthétiques** | Registres HA construits à la main, au plus petit : un cas par règle | Écrits pour les tests | Dépôt Gladys, `server/test/services/migration-home-assistant/fixtures/` |
| **Réponses WebSocket enregistrées** | Sorties réelles des commandes HA (02 § 3.2), **anonymisées** | Instance HA de test (§ 6) | Idem |
| **Sauvegardes de test** | Petites sauvegardes réelles : sans chiffrement, chiffrées en v2, chiffrées en v3, sans base, sauvegarde du Supervisor avec module Zigbee2MQTT. Mot de passe connu. Base d'historique réduite. | Générées par script (§ 6) avec la bibliothèque de référence | Idem, si la taille le permet (cible : moins de 1 Mo chacune), sinon dans le dépôt d'outillage |
| **Sauvegardes réelles anonymisées** | Installations de membres de la communauté (Q02-3) | Communauté, avec un **script d'anonymisation** fourni (noms, adresses, numéros de série, identifiants remplacés de façon cohérente, secrets effacés) | **Jamais dans un dépôt public.** Utilisées pour les mesures (Q07-1) et les tests de bout en bout. |

**Anonymisation** : le script remplace les données de façon **cohérente**. Une même MAC devient partout la même fausse MAC, sinon le rapprochement ne serait plus testable. Il **supprime** les fichiers sensibles (`auth`, `auth_provider.homeassistant`, `backup`, `secrets.yaml`) et vide `data` et `options` des entrées de configuration, en ne gardant que les **clés**. Il est relu par le porteur du projet avant toute utilisation.

## 8. Dépendances du service et revue

Le service a son **propre `package.json`** (convention de Gladys : `server/services/<service>/package.json`, installé par `server/cli/install_service_dependencies.js`). Le service `zigbee2mqtt` embarque déjà sa propre bibliothèque YAML (`server/services/zigbee2mqtt/package.json`). Les dépendances nouvelles (lecteur tar en flux, YAML, libsodium) peuvent donc rester **propres au service**, sans toucher aux dépendances du cœur. Cela allège la discussion D18, qui se limite alors :
- à la route de téléversement (facultative, 01 § 7.3) ;
- aux params réservés `GLADYS_HW_*` (03 § 2.4) ;
- aux nouvelles tables `t_migration_*` et à la page de paramètres.

**Pour chaque dépendance ajoutée**, la pull request vers Gladys justifie le choix : maintenance active, licence compatible (Apache 2.0 pour Gladys), absence de compilation native (Gladys tourne sur ARM et x86), taille.

## 9. Critères d'acceptation de la V1

La V1 est prête à être proposée à Gladys quand :

1. la CI de Gladys est verte : tests serveur, couverture du *patch* à 100 %, Cypress, lint, traductions, construction ;
2. le **test d'annulation** (§ 4, scénario 4) passe ;
3. la **veille des formats** (§ 6) passe sur la dernière version de HA ;
4. une **image Docker de test** de Gladys, avec le service, a été proposée à la communauté sur le forum (T2), et au moins **trois installations réelles** ont été migrées de bout en bout, en coexistence, puis annulées sans écart ;
5. les critères de réussite de la vision (§ 10) sont mesurés sur ces installations : durée, aucun doublon à la relance, statut et raison pour chaque élément non migré ;
6. la spec anglaise est prête dans `docs/specs/` (D17).

## 10. Décisions

| # | Décision |
|---|---|
| T1 | Tout le travail (service, CI de veille, outillage de test, jeux de données publics) a vocation à être **rendu à l'organisation `GladysAssistant`** une fois mergé. Ce dépôt est un lieu de travail, pas une destination. La CI de veille y est développée, puis transférée (ex-Q06-1). |
| T2 | Une **image Docker de test** de Gladys avec le service est publiée et annoncée sur le forum **avant** la pull request finale, pour les essais de la communauté (ex-Q06-2). |
