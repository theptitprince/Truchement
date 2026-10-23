<div align="center">

# Truchement — Relais NUT Synology → UGOS

**Un onduleur sur le Synology, l'UGREEN prévenu aussi.**

![Version](https://img.shields.io/badge/version-1.0-2ea44f)
![Python](https://img.shields.io/badge/Python-3.12-3776AB?logo=python&logoColor=white)
![Docker Compose](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)
![NUT](https://img.shields.io/badge/protocole-NUT%20%2F%203493-0a7ea4)
![Plateforme](https://img.shields.io/badge/plateforme-Synology%20DSM%20%E2%86%92%20UGREEN%20UGOS-4a4a4a)
![Licence](https://img.shields.io/badge/licence-GPL--3.0--or--later-blue)

</div>

**Français** · [English](README.en.md)

---

> [!NOTE]
> **Truchement a besoin de sa propre adresse IP sur le réseau local** (192.168.1.210 dans
> l'exemple), libre et hors de la plage DHCP de la box : c'est cette adresse, et non celle du
> Synology, qu'on donne à UGOS. Côté DSM, l'adresse à autoriser dans le serveur réseau UPS est
> **172.31.93.2**, celle du relais sur le réseau Docker interne. Les adresses du fichier sont des
> exemples : adaptez celles marquées « à adapter ».

Truchement fait lire à un NAS UGREEN (UGOS) l'onduleur branché en USB sur un
Synology. DSM sait partager son onduleur sur le réseau (serveur NUT, port
3493), et UGOS sait écouter un onduleur réseau ; mais les deux ne
s'entendent pas : UGOS demande l'onduleur `ups0` avec ses propres
identifiants, quand DSM ne connaît que `ups`, avec les identifiants fixes
`monuser` / `secret`.

Truchement se place entre les deux et sert d'interprète : il renomme
l'onduleur dans les deux sens, substitue les bons identifiants, et laisse
passer tout le reste tel quel. Quand le courant tombe, UGOS voit l'onduleur
passer sur batterie (`OB`), puis batterie faible (`LB`) ou arrêt forcé
(`FSD`), exactement comme le Synology.

```
 UGREEN (UGOS)                    TRUCHEMENT                       SYNOLOGY (DSM)
 client NUT        ----------->   192.168.1.210:3493   ---------->  serveur NUT
 « ups0 »                         (macvlan, LAN)                    « ups », USB
 identifiants UGOS                172.31.93.2  ── réseau interne ─> 172.31.93.1:3493
                   <-----------   ups ↔ ups0           <----------  monuser / secret
```

Version actuelle : 1.0 — l'historique des livraisons est dans
[CHANGELOG.md](CHANGELOG.md).

## Sommaire

- [1. Prérequis](#1-prérequis)
- [2. Les fichiers](#2-les-fichiers)
- [3. Installation](#3-installation)
- [4. Réglages](#4-réglages)
- [5. Comment ça marche](#5-comment-ça-marche)
- [6. Le journal](#6-le-journal)
- [7. Sécurité](#7-sécurité)
- [8. En cas de souci](#8-en-cas-de-souci)
- [9. Licence](#9-licence)

---

## 1. Prérequis

- Un **Synology** sous DSM 7 avec **Container Manager**, et l'onduleur
  branché en USB et reconnu (*Panneau de configuration > Matériel et
  alimentation > UPS*).
- Un **NAS UGREEN** sous UGOS, sur le même réseau local.
- **Une adresse IP libre** sur le réseau local pour le relais, en dehors de
  la plage DHCP de la box.

Rien à compiler ni à construire : le conteneur part de l'image officielle
`python:3.12-alpine`, que Container Manager télécharge lui-même, et le script
est inclus dans le fichier compose.

## 2. Les fichiers

| Fichier | Rôle |
|---|---|
| `docker-compose.yml` | Le projet complet : conteneur, script du relais, réseaux. C'est le seul fichier à utiliser. |
| `CHANGELOG.md` | L'historique des versions. |
| `LICENSE`, `NOTICE` | La licence GPL v3 et ses conditions additionnelles. |

## 3. Installation

### Côté Synology : ouvrir le serveur NUT au relais

1. *Panneau de configuration > Matériel et alimentation > UPS*.
2. Cochez **Activer le serveur réseau UPS**.
3. Cliquez sur **Périphériques Synology NAS autorisés** et ajoutez
   **172.31.93.2** (l'adresse du relais sur le réseau interne).
4. Appliquez.

### Côté Synology : créer le projet

1. Ouvrez **Container Manager > Projet > Créer**.
2. Nom du projet : `truchement` ; chemin : un dossier au choix (par exemple
   `/docker/truchement`).
3. Source : **Créer docker-compose.yml**, et collez le contenu de
   [`docker-compose.yml`](docker-compose.yml).
4. Adaptez les lignes marquées « à adapter » (voir [4. Réglages](#4-réglages)) :
   l'adresse du relais sur le LAN, le sous-réseau du LAN, et l'adresse du NAS
   UGREEN dans `ALLOWED_CLIENTS`.
5. Validez : Container Manager télécharge l'image, crée les deux réseaux et
   démarre le conteneur `truchement`.

Dans le journal du conteneur (*Container Manager > Conteneur > truchement >
Journal*), deux lignes doivent apparaître :

```
2026-10-23 14:02:11  Truchement, par ETDEL — https://github.com/theptitprince/Truchement — version 1.0
2026-10-23 14:02:11  Relais NUT 0.0.0.0:3493 -> 172.31.93.1:3493  (UPS ups0 <-> ups, clients 192.168.1.201)
```

### Côté UGREEN : écouter le relais

Dans les réglages de l'onduleur d'UGOS, choisissez le mode **client
d'onduleur réseau (NUT)** et donnez l'adresse du relais, **192.168.1.210**.
Les identifiants demandés par UGOS importent peu : le relais les remplace par
ceux du Synology. UGOS doit alors afficher l'onduleur, son état et sa charge.

## 4. Réglages

### Les adresses

| Où | Valeur d'exemple | À quoi elle sert |
|---|---|---|
| `networks > lan > ipv4_address` | `192.168.1.210` | Adresse du relais sur le LAN, celle qu'on donne à UGOS. **À adapter.** |
| `networks: lan > subnet` (en bas) | `192.168.1.0/24` | Sous-réseau du LAN. **À adapter.** |
| `networks: lan > parent` | `ovs_eth0` | Carte réseau du Synology. `eth0` si Open vSwitch est désactivé, `ovs_bond0` ou `bond0` avec une agrégation de liens. |
| `networks > interne > ipv4_address` | `172.31.93.2` | Adresse du relais sur le réseau interne, à autoriser dans DSM. |
| `networks: interne > gateway` | `172.31.93.1` | Le Synology vu du conteneur. Ne change qu'en cas de conflit avec un autre réseau. |

### Les variables d'environnement

| Variable | Défaut | Rôle |
|---|---|---|
| `UPSTREAM_HOST` | `192.168.1.200` | Serveur NUT à lire. Dans le compose : `172.31.93.1`, le Synology vu du conteneur. |
| `UPSTREAM_PORT` | `3493` | Port du serveur NUT. |
| `LISTEN_HOST` | `0.0.0.0` | Adresse d'écoute du relais. |
| `LISTEN_PORT` | `3493` | Port d'écoute du relais. |
| `ALLOWED_CLIENTS` | *(vide = tous)* | Adresses autorisées à se connecter, séparées par des virgules. **À remplir** : voir [7. Sécurité](#7-sécurité). |
| `CLIENT_UPS` | `ups0` | Nom de l'onduleur côté client (UGOS). |
| `SERVER_UPS` | `ups` | Nom de l'onduleur côté serveur (DSM). |
| `NUT_USER` | `monuser` | Identifiant envoyé au serveur NUT (celui de DSM est fixe). |
| `NUT_PASS` | `secret` | Mot de passe envoyé au serveur NUT (celui de DSM est fixe). |
| `VERBOSE` | `0` | `0` : journal réduit ; `1` : tout le dialogue. |
| `TZ` | `Europe/Paris` | Fuseau horaire des dates du journal. |

Après une modification : *Container Manager > Projet > truchement > Action >
Construire* (ou Arrêter puis Démarrer) pour que le conteneur la prenne en
compte.

## 5. Comment ça marche

**Pourquoi une adresse à lui.** Le serveur NUT de DSM occupe déjà le port
3493 sur l'adresse du Synology : le relais ne peut pas s'y installer aussi.
Il reçoit donc sa propre adresse sur le LAN grâce à un réseau Docker
**macvlan** : pour UGOS, c'est une machine de plus sur le réseau.

**Pourquoi un réseau interne.** Un conteneur en macvlan ne peut pas joindre
la machine qui l'héberge par son adresse LAN : c'est une limite du pilote
macvlan. Le relais passe donc par un second réseau, un petit **bridge**
Docker (`172.31.93.0/24`) dont la passerelle `172.31.93.1` est le Synology
lui-même. C'est par là qu'il lit le serveur NUT, en se présentant avec
l'adresse `172.31.93.2` — d'où l'autorisation à donner dans DSM.

**Ce que fait le relais.** NUT est un protocole texte, une commande par
ligne. Truchement lit chaque ligne et :

- dans le sens UGOS → Synology, remplace `ups0` par `ups`, et les lignes
  `USERNAME` / `PASSWORD` par `monuser` / `secret` ;
- dans le sens Synology → UGOS, remplace `ups` par `ups0`.

Seul le nom d'onduleur pris comme mot isolé est remplacé : les noms de
variables (`ups.status`, `ups.load`…) et les valeurs entre guillemets
restent intacts. Tout le reste passe tel quel, y compris les commandes
`LOGIN` et `FSD` qui déclenchent l'arrêt des machines.

## 6. Le journal

Par défaut (`VERBOSE: "0"`), le journal reste court. Il note :

- la mention d'origine, la version et la configuration au démarrage ;
- chaque authentification (`USERNAME`, `PASSWORD` masqué, `LOGIN`) ;
- les erreurs NUT (sauf `FEATURE-NOT-CONFIGURED`, sans intérêt) ;
- **chaque changement d'état de l'onduleur**, par exemple
  `*** ups.status : OL -> OB DISCHRG` à la coupure de courant ;
- les connexions refusées et un serveur NUT injoignable.

Avec `VERBOSE: "1"`, tout le dialogue s'affiche, sauf le contenu des listes
de variables (seul leur nombre de lignes est noté). Pratique pour mettre au
point, trop bavard au quotidien : UGOS interroge l'onduleur toutes les
quelques secondes.

## 7. Sécurité

Le relais **accepte n'importe quels identifiants** de la part d'un client
autorisé et les remplace par ceux de DSM : la seule barrière est donc
`ALLOWED_CLIENTS`. Remplissez-la avec l'adresse du NAS UGREEN (et,
temporairement, celle d'un PC de test), et donnez au NAS UGREEN une adresse
fixe (réservation DHCP).

Les identifiants `monuser` / `secret` ne sont pas un secret : ce sont ceux,
fixes et publics, du serveur NUT de DSM. Ce qui protège ce serveur, c'est sa
liste de périphériques autorisés — qui ne contient que le relais.

## 8. En cas de souci

| Symptôme | Cause probable |
|---|---|
| `!!! connexion refusée de 192.168.1.x (hors ALLOWED_CLIENTS)` | L'adresse n'est pas dans `ALLOWED_CLIENTS`. Ajoutez-la, puis redémarrez le conteneur. |
| `!!! serveur NUT 172.31.93.1:3493 injoignable` | Serveur réseau UPS non activé dans DSM, ou onduleur non reconnu par le Synology. |
| `ERR ACCESS-DENIED` au `LOGIN` | `172.31.93.2` n'est pas dans les *Périphériques Synology NAS autorisés* de DSM. |
| Le projet refuse de démarrer, erreur sur le réseau `lan` | Mauvaise carte `parent` (`ovs_eth0`, `eth0`, `ovs_bond0`…), ou adresse déjà prise sur le LAN. Le nom exact se lit en SSH avec `ip link`. |
| Erreur sur le réseau `interne` (*pool overlaps*) | `172.31.93.0/24` est déjà utilisé par un autre réseau Docker : choisissez un autre sous-réseau, et mettez à jour l'adresse autorisée dans DSM. |
| Le Synology ne répond pas au ping vers 192.168.1.210 | Normal : c'est la limite du macvlan décrite plus haut. Testez depuis une autre machine. |
| UGOS ne voit pas l'onduleur | Passez `VERBOSE: "1"` et regardez ce qu'UGOS demande et ce que DSM répond. |

Pour tester depuis un PC équipé des outils NUT, ajoutez son adresse à
`ALLOWED_CLIENTS`, puis :

```bash
upsc ups0@192.168.1.210
```

La commande doit lister les variables de l'onduleur (`ups.status: OL`,
`battery.charge`…).

## 9. Licence

Truchement est un **logiciel libre**, distribué sous la licence
**GNU General Public License, version 3 ou ultérieure** (GPL-3.0-or-later).

Copyright © 2026 ETDEL.

Concrètement, vous pouvez utiliser le programme librement, y compris dans un
cadre professionnel, l'étudier, le modifier et le redistribuer — à condition
que toute version redistribuée, modifiée ou non, reste sous la même licence,
avec son code source et la mention de copyright. Le programme est fourni
**sans aucune garantie** : il relaie l'état de l'onduleur, il ne remplace pas
une vérification de la chaîne d'arrêt de vos machines.

### Conditions additionnelles (article 7)

La GPL v3 permet à l'auteur d'ajouter quelques conditions (article 7). Celles de Truchement sont dans
le fichier [`NOTICE`](NOTICE), en français et en anglais :

- **Attribution** : toute copie ou version modifiée doit garder, visible pour l'utilisateur
  et dans sa documentation, la mention :

  > Truchement, par ETDEL — https://github.com/theptitprince/Truchement

  Une version modifiée peut la faire précéder de « D'après » et ajouter ses propres crédits,
  sans la retirer ni la masquer.
- **Origine** : une version modifiée doit dire clairement qu'elle est modifiée, et ne doit se
  présenter ni comme l'original, ni comme approuvée par ETDEL.

Dans Truchement, cette mention s'écrit dans le journal du conteneur à chaque démarrage.

Le texte intégral de la licence est dans le fichier `LICENSE` du dépôt et
sur <https://www.gnu.org/licenses/gpl-3.0.html>. Le fichier
`docker-compose.yml` porte un en-tête qui renvoie à `LICENSE` et `NOTICE`.

---

*Truchement — ETDEL 2026 — GPL-3.0-or-later*
