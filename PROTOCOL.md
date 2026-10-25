# Protocole Bluetooth JustSalt

Notes de rétro-ingénierie, observées sur un **JustSalt Duo de 4e génération** (`GEN_V4`, logiciel 03.03, sans sonde ORP). Elles complètent le décodage publié sur le [forum HACF](https://forum.hacf.fr/t/electrolyseur-e-pool-technologie-justsalt/24718). Les commandes marquées ✅ ont été vérifiées par capture de l'application officielle (journal HCI Android) et/ou testées depuis l'ESP. Les interprétations marquées « ? » sont des hypothèses.

## Liaison Bluetooth

| Élément | Valeur |
|---|---|
| Service | `09912756-7b32-4629-aeb1-b309d9a338ae` |
| Caractéristique de réception (indications) | `ef785c24-22bb-463d-b651-0b7445ba091c` (handle 0x0013, CCCD 0x0014 = `02 00`) |
| Caractéristique de commande (écriture avec réponse) | `4d32c5e5-2bb5-45c6-8c89-6f59bb3930d2` (handle 0x0010) |

**Sécurité.** LE Secure Connections, « Just Works » (l'électrolyseur se déclare sans écran ni clavier), avec mémorisation des clés (bonding). La réponse d'appairage de l'électrolyseur est `02 03 00 09 10 03 03`.
- L'électrolyseur envoie lui-même une **Security Request** (`0B 09`) à chaque connexion.
- Il n'accepte un **nouvel** appairage qu'en **mode appairage** (déclenché sur le boîtier). Hors de ce mode, il accepte au mieux un appairage temporaire (`mode = 8` côté ESP-IDF), n'envoie rien et coupe après environ 10 s.
- **Un seul appareil appairé** à la fois.
- Côté ESP : ne pas initier d'appairage soi-même en parallèle de la Security Request, sous peine d'obtenir un appairage non persistant.

## Trames reçues

```
00 | longueur totale | type | ID | taille | données… | ID | taille | données… | …
```

- **type 01** : état et configuration. Trame complète (environ 245 octets, découpée en plusieurs notifications) à chaque connexion, puis uniquement les valeurs qui changent.
- **type 02** : historique (enregistrements `E1`…`E5`), envoyé lors d'un nouvel appairage.
- Valeurs sur plusieurs octets : **big-endian**.
- Lire la caractéristique au lieu de recevoir les notifications ne renvoie que `00`.

## Trames de commande

Écriture de **80 octets** sur la caractéristique de commande :

```
00 | 5 + taille | 03 | ID | taille | données… | FF FF | 00 … (jusqu'à 80 octets)
```

Exemples (✅ capture de l'appli) :
- consigne pH 7,2 : `00 06 03 30 01 48 FF FF 00…`
- boost 24 h : `00 06 03 66 01 18 FF FF 00…`
- injection pH- 1 min : `00 07 03 6B 02 01 02 FF FF 00…`

L'électrolyseur confirme en renvoyant la nouvelle valeur ou le nouvel état dans la seconde.

## Registres

R = lecture, W = écriture (même ID), C = commande en écriture seule.

### Mesures et état

| ID | Octets | Accès | Signification |
|---|---|---|---|
| `01` | 1 | R | pH × 10 (72 = 7,2) |
| `02` | 2 | R | pH × 100 (720 = 7,20), plus précis que `01` |
| `03` | 1 | R | Production effective en % : égale à la consigne, **125 en boost** |
| `05` | 2 | R | Compteur d'heures (?), proche de `0E` |
| `08` | 1 | R | ? (100) |
| `09` | 2 | R | Température de l'eau × 10 (°C) |
| `0A` | 1 | R | Sel × 10 (g/L) ; **0 = pas de mesure** (cellule à l'arrêt) |
| `0B` | 1 | R | Lié à la production (?) : 0 à l'arrêt, ~37 en production, augmente avec le % de production |
| `0C` | 1 | R | Lié à la production : 0 à l'arrêt, ~90-100 en production |
| `0D` | 2 | R | ? |
| `0E` | 2 | R | Heures de fonctionnement |
| `0F` | 2 | R | Compteur incrémenté chaque minute |
| `10` | 2 | R | ? |
| `12` | 4 | R | Date d'étalonnage pH : `xx JJ MM AA` |
| `1F` | 2 | R | ? (oscille en permanence) |
| `28` | 2 | R | Sel en mg/L (4600 = 4,6 g/L) ; 0 = pas de mesure |
| `67` | 2 | R | **Temps restant du boost** (minutes), 0 hors boost ✅ |
| `92` | 1 | R | Code d'événement transitoire, remis à 0 après environ 1 s : `07` boost, `09` injection manuelle, `0B` régulation réactivée |
| `93` | 2 | R | **Registre d'état** ✅ : `0x0080` au repos, bit `0x0010` = boost en cours, bit `0x0040` = injection pH- en cours |
| `95` | 4 | R | Code ID / configuration (change avec certains réglages) |
| `96` | 2 | R | Version logicielle (`03 03`) |
| `99` | 22 | R | Nom Bluetooth (ASCII) |
| `9A` | 16 | R | Numéro de série (ASCII) |
| `B0` | 1 | R | Signal reçu par l'électrolyseur (−dBm, envoyé chaque seconde) (?) |
| `B1` | 6 | R | Adresse Bluetooth de l'électrolyseur |
| `D0` | 6 | R | Statut de la dernière commande : `[ID] [état] 00…`. Pour `6B` : `00` en cours, `01` en pause, `02` annulée, `03` terminée |

### Consignes (lecture et écriture)

| ID | Octets | Signification | Vérifié |
|---|---|---|---|
| `11` | 2 | Volume de la piscine (m³) | ✅ |
| `30` | 1 | Consigne pH × 10 | ✅ |
| `32` | 1 | Taux d'acide du produit pH- (%) | ✅ |
| `33` | 1 | Consigne de production (%) | ✅ |
| `39` | 1 | Période d'inversion de polarité (h) | ✅ |
| `51` | 1 | **Régulation pH automatique** : 1 = active, 0 = arrêtée | ✅ |

### Commandes (écriture seule)

| ID | Données | Effet | Vérifié |
|---|---|---|---|
| `66` | 1 octet : durée en heures | **Boost** : `18` (24 h), `0C` (12 h), `00` = arrêt | ✅ |
| `6B` | 2 octets : `01 n` | **Injection manuelle de pH-** pendant n × 30 s (n = 1, 2, 4, 6, 10 → 30 s, 1, 2, 3, 5 min) | ✅ |
| `6B` | 1 octet : `02` | Pause / reprise de l'injection (même commande pour les deux) | ✅ |
| `6B` | 1 octet : `00` | Annulation de l'injection | ✅ |

Écrire dans `67` (temps restant) **ne déclenche pas** le boost : c'est une valeur en lecture seule.

### Modèles avec sonde ORP (d'après le forum, absents sur le Duo)

`06` ORP (mV) · `13` date d'étalonnage ORP · `35` consigne ORP / 10 · `37` seuil d'alarme ORP (h).

## Historique (trames de type 02)

Enregistrements de 15 octets, commençant par une date `AA MM JJ hh mm ss` :
- **`E1`** : événement daté. Il contient un mot d'état au même format que `93` : l'alternance `00 C0` / `00 80` correspond aux impulsions d'injection de pH-.
- **`E2`** : bilan quotidien (?) : durée de filtration du jour (h) et compteur cumulé.
- **`E4`** : relevé horaire (?) : pH × 10, puis des grandeurs liées à la température et à la production.
- **`E3`**, **`E5`** : non identifiés.
