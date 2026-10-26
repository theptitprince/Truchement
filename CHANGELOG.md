# Journal des versions

Toutes les livraisons de Truchement, de la plus récente à la plus ancienne.

## Comment se lit un numéro de version

`X.Y.Z` : `1.1.0`, `1.1.1`, `1.2.0`… (la 1.0 a été publiée avant ce
découpage en trois chiffres).

- Le **premier chiffre** change quand le fichier compose ne se reprend plus
  tel quel — une variable renommée, un réseau organisé autrement.
- Le **deuxième** monte à chaque fonction ou comportement nouveau ; un
  fichier compose de la version précédente se reprend tel quel.
- Le **troisième** monte à chaque correction.

Le numéro vit à un seul endroit dans le code, `APP_VERSION` dans le script
inclus dans `docker-compose.yml`, recopié dans l'en-tête du fichier, les
badges et la « Version actuelle » des deux README. Il est écrit au journal
du conteneur à chaque démarrage.

---

## 1.1.0

Truchement réveille le NAS UGREEN quand le secteur revient alors qu'il s'est
déjà arrêté. Sur batterie, le UGREEN s'arrête avant le Synology ; si le
courant revient avant que l'onduleur coupe ses prises, le UGREEN n'a jamais
perdu son alimentation, ne redémarre pas seul et restait éteint. Truchement
lui envoie désormais un paquet magique Wake-on-LAN. La fonction est inactive
tant que `WOL_MAC` est vide : un fichier compose 1.0 se reprend tel quel, et
le relais se comporte exactement comme avant.

- Sondage autonome de l'état de l'onduleur sur le serveur NUT de DSM, toutes
  les `WOL_POLL` secondes, sur une connexion gardée ouverte et sans `LOGIN` :
  il ne dépend plus des questions d'UGOS, muettes quand le UGREEN est éteint.
  Les changements d'état restent journalisés une seule fois, qu'ils viennent
  du sondage ou du relais.
- Armement au passage sur batterie (`OB`, `LB` ou `FSD`) ; un serveur NUT
  injoignable (DSM en mode sécurisé) ne désarme pas.
- Au retour du secteur stable depuis `WOL_STABLE` secondes, si le UGREEN ne
  répond pas sur `WOL_IP:WOL_PORT` : paquet magique (ports UDP 9 et 7, trois
  fois) vers le broadcast du LAN, renouvelé toutes les `WOL_RETRY` secondes,
  jusqu'à `WOL_MAX` essais. Désarmement dès que le UGREEN répond.
- Armement au lancement si le Synology a démarré depuis moins de `WOL_BOOT`
  secondes, pour couvrir la coupure complète.
- Un UGREEN éteint à la main n'est jamais rallumé en dehors de ces deux cas.
- Au démarrage, une ligne de configuration du réveil, et une alerte si le
  broadcast ne sortirait pas par l'interface qui joint le UGREEN.
- Toute erreur du réveil est journalisée (`!!! WoL : …`) sans jamais toucher
  au relais ; un réglage invalide désactive seulement le réveil.
- Nouvelles variables : `WOL_MAC`, `WOL_IP`, `WOL_PORT`, `WOL_BROADCAST`,
  `WOL_POLL`, `WOL_STABLE`, `WOL_RETRY`, `WOL_MAX`, `WOL_BOOT`.
- README : nouvelle section « Réveil du NAS UGREEN (Wake-on-LAN) ».

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
