# Journal des versions

Toutes les livraisons de Truchement, de la plus récente à la plus ancienne.

## Comment se lit un numéro de version

`X.Y` : `1.0`, `1.1`…

- Le **premier chiffre** change quand le fichier compose ne se reprend plus
  tel quel — une variable renommée, un réseau organisé autrement.
- Le **second** monte à chaque livraison — une correction, un réglage ou une
  fonction de plus.

Le numéro vit à un seul endroit dans le code, `APP_VERSION` dans le script
inclus dans `docker-compose.yml`. Il est écrit au journal du conteneur à
chaque démarrage.

---

## 1.0

Première publication. Un seul fichier, `docker-compose.yml`, à coller dans
Container Manager : il fait partager l'onduleur branché sur un Synology à un
NAS UGREEN (UGOS), que le client NUT d'UGOS ne savait pas lire directement.

- Relais NUT (port 3493) avec sa propre adresse sur le réseau local
  (macvlan), qui lit le serveur NUT de DSM par un petit réseau Docker interne.
- Traduction du nom de l'onduleur dans les deux sens : `ups0` côté UGOS,
  `ups` côté Synology.
- Remplacement des identifiants envoyés par UGOS par ceux, fixes, du serveur
  NUT de DSM (`monuser` / `secret`).
- Liste des clients autorisés (`ALLOWED_CLIENTS`) ; toute autre adresse est
  refusée et notée au journal.
- Journal réduit par défaut : connexions, erreurs, et chaque changement d'état
  de l'onduleur (`OL`, `OB`, `LB`, `FSD`…) ; `VERBOSE=1` affiche tout le
  dialogue.
- Mention d'origine et version écrites au journal à chaque démarrage.
