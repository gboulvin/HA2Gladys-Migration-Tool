# Note de recherche — Intégrations HA les plus utilisées

- **Date** : 2026-11-17
- **Source** : [Suivi des intégrations Gladys](https://william-de71.github.io/gladys-integrations-tracker/) (William-De71/gladys-integrations-tracker), fichier `liste-ha-gladys.csv`, données générées le 2026-11-17 à 18:23 UTC.
- **Objet** : choisir les intégrations prioritaires du catalogue V1 (décision D19, qui remplace D8).

## Limites de la source

- Le tracker fusionne **deux jeux de données** au pourcentage d'usage non homogène :
  - la liste d'octobre-novembre de Pierre-Gilles (lignes 1 à 115), qui recense des intégrations HA **absentes** du cœur de Gladys ;
  - une analyse du forum de juillet (lignes F1 à F82), qui couvre aussi les grandes intégrations déjà prises en charge (MQTT, Matter, Zigbee…).
- Les pourcentages représentent la part des installations HA qui utilisent l'intégration. Le classement fusionné ci-dessous est donc **indicatif**.
- Les statistiques officielles de HA ([analytics.home-assistant.io](https://analytics.home-assistant.io/integrations)) n'ont pas pu être lues : le tableau est chargé dynamiquement. Selon le résumé de la page, 579 869 installations sur 731 005 partagent leurs intégrations. **À recouper** avant de figer le catalogue.

## Classement fusionné (top 20)

| Rang | Intégration HA | Usage HA | Prise en charge Gladys | Intégration Gladys | Source |
|---|---|---|---|---|---|
| 1 | MET Norway (Met.no) | 83,52 % | Store | `guim31/gladys-met-no` | Liste n° 1 |
| 2 | Google Cast | 50,80 % | Store | `gboulvin/Gladys-Google-Cast` | Forum F1 |
| 3 | Bluetooth | 49,00 % | Native | — | Forum F2 |
| 4 | MQTT | 48,60 % | Native | `mqtt` | Forum F3 |
| 5 | Matter | 39,20 % | Native | `matter` | Forum F4 |
| 6 | UPnP / IGD | 32,90 % | Store | `callemand/gladys-upnp-igd` | Forum F5 |
| 7 | Tuya | 29,60 % | Store | `Terdious/gladys-tuya` | Forum F6 |
| 8 | IPP (imprimantes) | 29,30 % | Store | `guim31/gladys-ipp` | Forum F7 |
| 9 | DLNA Digital Media Renderer | 28,20 % | Store | `guim31/gladys-dlna` | Forum F8 |
| 10 | ESPHome | 26,80 % | Store | `William-De71/gladys-esphome` | Forum F9 |
| 11 | Zigbee (ZHA / Zigbee2MQTT) | 26,00 % | Native | `zigbee2mqtt` | Forum F10 |
| 12 | Shelly | 24,90 % | Store | `Terdious/gladys-shelly` | Forum F11 |
| 13 | DLNA Digital Media Server | 20,80 % | Store | `guim31/gladys-dlna` | Forum F12 |
| 14 | Android TV Remote | 20,80 % | Store | `guim31/gladys-integration-android-tv-remote` | Forum F13 |
| 15 | HomeKit Bridge | 17,20 % | Native | `homekit` | Forum F14 |
| 16 | Samsung Smart TV | 17,10 % | Store | `cicoub13/gladys-samsung-tv` | Forum F15 |
| 17 | Sonos | 16,20 % | Store | `vincentBesseau/gladys-sonos` | Forum F16 |
| 18 | Philips Hue | 16,00 % | Store | `cicoub13/gladys-philips-hue` | Forum F17 |
| 19 | Apple TV / AirPlay | 15,80 % | Store | `valentinhttr/gladys-apple-tv` | Forum F18 |
| 20 | HomeKit Controller | 14,90 % | Store | `prohand/gladys-homekitcontroller` | Forum F19 |

Autres intégrations de l'ancienne liste D8 :

| Intégration HA | Usage HA | Prise en charge Gladys | Intégration Gladys |
|---|---|---|---|
| Z-Wave JS | 8,40 % | Store | `sescandell/gladys-zwavejs` |
| Tasmota | 8,10 % | Store | `gboulvin/Gladys-Tasmota` |

## Constats

1. **Les 10 premières intégrations ont toutes un équivalent dans Gladys** : 3 natives et 7 dans le store. Pour elles, l'outil n'a pas à signaler un manque, mais à **installer et rapprocher** (D16).
2. **Le tracker préfigure le catalogue distant (D12).** Il est collaboratif, mis à jour automatiquement et fait déjà le lien entre les intégrations HA et celles de Gladys, avec un type de correspondance (même nom, équivalente, partielle). Il serait la base naturelle du catalogue, à condition de s'entendre avec son auteur et d'enrichir le format (identifiants HA exacts, `store_slug` Gladys, règles de rapprochement, fonctions prises en charge).
3. **Le classement mélange des intégrations « appareil » et des intégrations « service ».** Met.no est une source météo et Bluetooth un transport : ni l'un ni l'autre n'apporte d'appareils à rapprocher au sens strict. Leur traitement sera défini en section 03.
4. Le tracker indique « Libre » pour les intégrations sans aucun équivalent (par exemple BTHome, Oral-B ou Govee Bluetooth). Ce sont les candidates au parcours D15 : créer l'intégration, ou la demander à la communauté.
