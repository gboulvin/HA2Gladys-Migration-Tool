# Brouillon — Réponse à Will sur le forum (sujet 11619)

> **Usage** : texte à copier sous la ligne `---`, en réponse au message n° 2 de Will_71. À vérifier avant publication : le pseudo de Pierre-Gilles sur le forum (supposé `@pierre-gilles`), et le lien vers la pull request de la spec une fois mergée.

---

Merci @Will_71 pour cette relecture, elle est précieuse. Je reprends presque tout. La spec est mise à jour (section 03, PR en cours sur le dépôt) :

**1. Table des domaines HA.** Tu as raison, j'avais supposé un catalogue indexé par domaine sans vérifier que le tracker les connaissait. On part sur une vraie table **domaine HA → intégration(s) Gladys**, distincte des lignes du tracker. Si tu peux en générer l'essentiel, je m'occupe de la vérification manuelle du reste. Les domaines absents seront fréquents : ils sont traités comme « non migrables », avec le parcours « créez l'intégration / demandez à la communauté ».

**2. Installation automatique.** D'accord, 97 %, c'est trop juste pour installer quelque chose chez quelqu'un. Une cible n'est **installable automatiquement** que si elle a des règles de migration, publie `GLADYS_HW_*`, ou a été **confirmée à la main** (champ `confirmed`). Les autres sont seulement suggérées, avec un avertissement.

**3. Domaines autorisés.** Très bon point de sécurité, que j'avais manqué. Les règles d'un dépôt ne sont retenues **que pour les domaines qui lui sont rattachés** dans la table. Tout le reste est écarté et signalé dans un rapport public, comme `rejected.json`.

**4. Version des règles.** `gladys-migration.json` est lu **au tag de la version publiée** dans le store, avec un `min_integration_version` facultatif.

**5. Plusieurs cibles.** Je reprends ton ordre tel quel : règles de migration présentes, puis publiée dans le store, puis activité récente.

**6. Production du catalogue.** C'est le changement principal, et je suis d'accord avec toi : des instances Gladys ne doivent pas dépendre d'un dépôt personnel ni du planificateur de GitHub Actions. Nouvelle répartition :
- **l'indexeur du store** (`GladysAssistant/integration-store`) lit `gladys-migration.json` au tag publié, en même temps que le manifeste, et publie `migration-catalog.json` à côté de `index.json` ;
- les règles des **services natifs** (zigbee2mqtt, mqtt, matter) vont dans le **dépôt Gladys**, versionnées avec le cœur ;
- **le tracker** fournit la table des domaines (par PR vers le dépôt de l'indexeur) et **mesure la couverture**.

**7. Niveaux de couverture.** Adoptés : 0 (aucun équivalent), 1 (installable, configuration manuelle), 2 (appareils rapprochés automatiquement), 3 (fonctions et historique repris), avec un taux pondéré par l'usage dans HA, présenté comme un ordre de grandeur. L'outil réutilisera les mêmes niveaux dans son rapport d'analyse : l'utilisateur et le tracker parleront la même langue.

**8. `GLADYS_HW_VENDOR_ID`.** Juste. La valeur sera **préfixée par un espace de noms** déclaré dans la convention (`tuya:…`, `cast:…`), associé à un domaine HA dans la table.

**9. ESPHome.** Merci pour la MAC. Le rapprochement ESPHome passera ainsi de « probable » à « sûr ».

---

@pierre-gilles, plusieurs points dépendent de toi. Voici ce que nous te demandons, pour en discuter :

1. **Indexeur du store** (`GladysAssistant/integration-store`) :
   - lire `gladys-migration.json` à la racine de chaque dépôt, **au tag de la version publiée** ;
   - le valider contre un schéma JSON publié, et écarter les règles portant sur des domaines non rattachés au dépôt, avec un rapport public de rejets ;
   - héberger la **table des domaines HA** (fichier de données alimenté par PR depuis le tracker) ;
   - publier **`migration-catalog.json`** à côté de `index.json`.
2. **Point signalé par Will** : l'indexeur bute sur la **limite de 1 000 résultats** de la recherche GitHub, et une vingtaine d'intégrations portant le topic manquent aujourd'hui dans l'index. C'est indépendant de la migration, mais cela la concerne directement, puisqu'une intégration absente de l'index ne peut pas être proposée.
3. **Params réservés `GLADYS_HW_*`** dans le cœur (`GLADYS_HW_MAC`, `GLADYS_HW_IEEE`, `GLADYS_HW_SERIAL`, `GLADYS_HW_UDN`, `GLADYS_HW_VENDOR_ID` avec espace de noms), et l'aide correspondante dans le **SDK**, qui ne les envoie qu'aux versions du cœur qui les acceptent.
4. **Règles des services natifs** : un fichier de règles de migration dans le dépôt Gladys, versionné avec le cœur.

Le reste des demandes au cœur est inchangé par rapport au premier message (tables `t_migration_*`, page Paramètres → Migration, route de téléversement facultative).
