# JustSalt → Home Assistant (ESPHome)

Pilotage d'un électrolyseur de piscine **JustSalt** (E-Pool / Pool Technologie) depuis **Home Assistant**, via un **ESP32** sous **ESPHome** qui dialogue en Bluetooth Low Energy avec le boîtier, comme le fait l'application officielle.

> *English summary: ESPHome Bluetooth gateway for JustSalt salt chlorinators (tested on a JustSalt Duo, 4th generation). Reads all measurements and controls setpoints, boost mode, manual pH- injection and automatic pH regulation from Home Assistant. Protocol details in [PROTOCOL.md](PROTOCOL.md).*

⚠️ **Projet non officiel**, issu de rétro-ingénierie, sans lien avec le fabricant. Il pilote un appareil qui **injecte de l'acide** dans votre piscine : utilisez-le en connaissance de cause, à vos risques.

## Matériel testé

| Élément | Détail |
|---|---|
| Électrolyseur | JustSalt **Duo**, 4e génération (nom Bluetooth `GEN_V4` / `GEN_xxxx-xxxxxx-xxx`, logiciel 03.03), sans sonde ORP |
| Passerelle | ESP32 classique (ESP32-WROOM-32 / DevKit). Pas d'ESP32-S2 : il n'a pas de Bluetooth. |
| Logiciel | ESPHome 2026.10, framework `esp-idf` |

D'autres générations ou modèles (avec sonde ORP notamment) utilisent le même protocole d'après le forum, mais n'ont pas été testés avec cette configuration. Retours bienvenus.

## Fonctionnalités

**Lecture**
- pH (au centième), température de l'eau, sel (g/L)
- Production de chlore en cours (déduite), production effective (%)
- Injection pH- en cours (manuelle ou automatique)
- Boost : état et temps restant
- Consignes, heures de fonctionnement, date d'étalonnage pH, version, numéro de série
- Diagnostic : signal Bluetooth (des deux côtés), signal Wi-Fi, réception des données

**Commandes**
- Consigne pH, consigne de production, taux d'acide, volume de la piscine, période d'inversion
- **Boost** : interrupteur + sélecteur de durée (12 h / 24 h)
- **Injection manuelle de pH-** : interrupteur + sélecteur de durée (30 s, 1, 2, 3, 5 min), pause / reprise
- **Régulation automatique du pH** : marche / arrêt
- **Verrou des commandes** : bloque toute écriture (hivernage, intervention…)

## Installation

1. Copier `justsalt.yaml` dans le dossier de configuration ESPHome, et créer `secrets.yaml` à partir de `secrets.example.yaml`.
2. Renseigner l'**adresse Bluetooth** de l'électrolyseur dans `mac_justsalt`, en tête du fichier. Pour la trouver, depuis un PC Linux à proximité : `bluetoothctl`, puis `scan on`, et repérer l'appareil nommé `GEN_…`. L'application nRF Connect sur téléphone fonctionne aussi.
3. Compiler et flasher l'ESP32 (USB la première fois, puis OTA).
4. Appairer l'ESP avec l'électrolyseur (voir ci-dessous).
5. Ajouter l'appareil dans Home Assistant : **Settings → Devices & services → ESPHome**, avec la clé `api_key`.

## Appairage : à lire absolument

L'électrolyseur exige un appairage Bluetooth chiffré, et **n'accepte qu'un seul appareil appairé**. Une fois l'ESP appairé, l'application téléphone ne peut plus s'y connecter, et inversement.

1. Couper le Bluetooth du téléphone, ou au moins fermer l'application.
2. Mettre l'électrolyseur en **mode appairage** depuis le boîtier.
3. L'ESP se connecte et s'appaire tout seul. Dans les logs : `auth success … mode = 9`, puis les trames et l'historique.

Une fois appairé, la liaison survit aux redémarrages de l'ESP et aux coupures de l'électrolyseur (par exemple quand il est alimenté avec la filtration). Le log affiche à chaque connexion et déconnexion le nombre d'appairages mémorisés.

**Si vous écrivez votre propre configuration**, trois points ont été déterminants :
- **ne pas déclencher d'appairage depuis l'ESP** (pas de `pair()` dans `on_connect`). L'électrolyseur réclame lui-même la sécurisation à chaque connexion, et un appairage lancé en parallèle donne des appairages non persistants : `Deleting unpaired device` à la déconnexion, puis impossibilité de se reconnecter sans refaire l'appairage ;
- `esp32_ble: auth_req_mode: sc_bond`, pour demander un appairage mémorisé ;
- dans les logs, `mode = 8` signifie appairage **temporaire** : l'électrolyseur n'envoie rien et coupe au bout d'environ 10 s. Il faut `mode = 9`.

Pour réappairer proprement : bouton **« Oublier l'appairage Bluetooth »**, puis mode appairage sur le boîtier.

## Conseils Home Assistant

- **Ajoutez une confirmation** sur l'interrupteur « Injection pH- » dans vos cartes de dashboard (`confirmation: true` dans le `tap_action`), pour éviter une injection d'acide par erreur.
- Le capteur **« Données électrolyseur reçues »** passe à OFF si l'électrolyseur n'envoie plus rien depuis 30 s. Une automatisation (« OFF depuis 10 min alors que l'électrolyseur est visible ») permet d'être prévenu si un réappairage devient nécessaire.
- Les interrupteurs Boost, Injection et Régulation affichent l'**état réel** lu sur l'électrolyseur. Ils n'envoient rien au démarrage de l'ESP (`restore_mode: DISABLED`).

## Dépannage

| Symptôme | Cause probable |
|---|---|
| `Error reading char … status=15` | Liaison non chiffrée : appairage absent ou refusé |
| `auth success … mode = 8`, puis déconnexion après ~10 s | Appairage temporaire : l'électrolyseur n'était pas en mode appairage |
| `LE encryption failed (status 0x06)` | L'électrolyseur ne connaît plus la clé : réappairer |
| Sel « inconnu » | Normal quand la cellule ne produit pas (0 dans la trame) |
| L'application téléphone ne se connecte plus | Normal, un seul appareil appairé |

Pour étudier de nouveaux registres : passer `logger: level` à `DEBUG` pour voir les trames brutes. Le détecteur de changements journalise en `INFO` toute valeur qui bouge. L'action `esphome.justsalt_ecrire_registre` permet d'écrire un registre arbitraire depuis HA.

## Protocole

Format des trames, liste des registres, commandes et notes sur l'appairage : **[PROTOCOL.md](PROTOCOL.md)**.

## Remerciements

- **Ricky_D** ([Ricky197503/haJustsalt](https://github.com/Ricky197503/haJustsalt)) et les contributeurs du fil HACF [Électrolyseur E-Pool Technologie Justsalt](https://forum.hacf.fr/t/electrolyseur-e-pool-technologie-justsalt/24718), pour le décodage initial du protocole et les premières versions ESPHome.
- Ce dépôt ajoute notamment : la commande du boost (`0x66`), l'injection manuelle de pH- (`0x6B`), la régulation pH (`0x51`), le registre d'état `0x93`, et la correction des problèmes d'appairage.
- **L'IA**

## Licence

MIT, voir [LICENSE](LICENSE).
