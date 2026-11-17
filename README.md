# HA2Gladys Migration Tool

Outil de migration permettant à un utilisateur de **Home Assistant** de transférer ses **appareils** puis ses **scénarios** (automatisations) vers **Gladys Assistant**.

L'outil prendra la forme d'un **service intégré à Gladys** (« Importer depuis Home Assistant »).

## État du projet

🚧 **Phase de spécification.** Aucun code n'est encore écrit.

- Spécifications : [`docs/`](docs/README.md) (en français)
- Décisions d'architecture : [`docs/decisions/`](docs/decisions/README.md)

## Conventions

| Élément | Langue |
|---|---|
| Spécifications, ADR | Français |
| Code, commentaires, messages de commit, noms de branches | Anglais |

## Processus

1. Chaque section de spec est rédigée sur une branche dédiée (`spec/<sujet>`).
2. Elle est soumise en pull request pour relecture et validation.
3. La validation se fait par merge dans `main`.
4. Le développement ne démarre qu'une fois les sections structurantes validées.
