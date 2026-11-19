# Spécifications — Sommaire

- **Dernière mise à jour** : 2026-11-19

| # | Document | Objet | Statut |
|---|---|---|---|
| 00 | [Vision et périmètre](00-vision.md) | Objectifs, périmètre, principes, décisions actées | ✅ Validé |
| 01 | [Architecture](01-architecture.md) | Forme du service Gladys, composants, flux, sécurité | ✅ Validé |
| 02 | [Lecture des données HA](02-extraction-ha.md) | Lecture des données HA (API et sauvegarde) | ✅ Validé |
| 03 | [Correspondances](03-mapping-devices.md) | Catalogue, rapprochement des appareils, pièces, personnes, services | ✅ Validé |
| 04 | [Scènes](04-mapping-scenarios.md) | Scènes et scripts HA, automatisations simples, présence, substituts | ✅ Validé |
| 05 | [Parcours utilisateur](05-ux.md) | Parcours utilisateur, écrans, rapport de migration | ✅ Validé |
| 06 | [Tests](06-tests.md) | Stratégie de test, jeux de données | ✅ Validé |
| 07 | [Historiques](07-historiques.md) | Import des historiques de données HA | ✅ Validé |
| 08 | [Tableaux de bord](08-tableaux-de-bord.md) | Tableau de bord « Accueil » généré, reprise des tableaux de bord HA | ✅ Validé |
| 09 | [API du service](09-api.md) | Routes REST, messages WebSocket, types de tâches, codes traduits | ✅ Validé |
| — | [`decisions/`](decisions/README.md) | Décisions d'architecture (ADR) | — |
| — | [`research/gladys-codebase.md`](research/gladys-codebase.md) | Note de recherche : étude du code Gladys (commit `71edb17`) | 📚 Référence |
| — | [`research/ha-integrations-usage.md`](research/ha-integrations-usage.md) | Note de recherche : intégrations HA les plus utilisées | 📚 Référence |
| — | [`research/ha-data-sources.md`](research/ha-data-sources.md) | Note de recherche : sources de données HA (API, sauvegardes, recorder) | 📚 Référence |
| — | [`research/matching-keys.md`](research/matching-keys.md) | Note de recherche : clés de rapprochement des 12 intégrations V1 | 📚 Référence |
| — | [`research/scene-mapping.md`](research/scene-mapping.md) | Note de recherche : correspondance automatisations HA → scènes Gladys | 📚 Référence |

## Légende des statuts

- ⏳ À rédiger
- 📝 Brouillon (en cours de discussion)
- 📝 Brouillon (corrections de relecture) : section validée, modifiée par les corrections de la relecture du 2026-11-19, en attente de merge
- ✅ Validé (mergé après relecture)
