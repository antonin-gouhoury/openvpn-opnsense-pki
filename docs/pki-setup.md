# Mise en place de la PKI sur OPNsense

Une PKI (*Public Key Infrastructure* — infrastructure à clés publiques) permet de gérer des certificats numériques. Dans ce projet, elle authentifie le serveur VPN auprès des clients et chaque utilisateur auprès du serveur. C'est l'un des deux facteurs d'authentification du VPN, en complément du couple identifiant/mot de passe.

## Vue d'ensemble

L'infrastructure PKI repose sur trois objets créés dans OPNsense.

L'**autorité de certification interne** est la racine de confiance. Elle signe et valide tous les certificats émis. Sa clé privée doit être protégée car sa compromission permettrait à un attaquant de signer de faux certificats.

Le **certificat serveur** est présenté par OpenVPN aux clients qui se connectent. Il prouve l'identité du serveur.

Les **certificats clients** sont émis individuellement, un par utilisateur autorisé. Ils prouvent l'identité de l'utilisateur auprès du serveur. Cette individualisation permet de révoquer un seul accès sans toucher aux autres, et de tracer précisément qui s'est connecté.

## Création de l'autorité de certification

Dans OPNsense : Système, Gestion des certificats, Autorités, Ajouter.

J'ai retenu la création d'une autorité de certification interne, avec une clé RSA, un algorithme de hachage SHA-256, et une durée de vie de 3650 jours (10 ans). Le pays est défini à FR et le nom commun est explicite (par exemple `CA_SISR`).

Le choix d'une longue durée pour la CA est volontaire. Si elle expire, tous les certificats qu'elle a signés deviennent invalides en cascade. Une durée de 5 à 20 ans est la norme.

## Création du certificat serveur

Système, Gestion des certificats, Certificats, Ajouter.

Le type est défini à "Certificat de serveur". Le certificat est signé par l'autorité créée à l'étape précédente. Sa durée de vie est fixée à 397 jours, soit environ treize mois. C'est la durée maximale acceptée par les principaux clients TLS modernes pour les certificats serveur. Au-delà, certains clients refuseraient le certificat.

## Création d'un certificat client

Même menu, en changeant simplement le type à "Certificat client". Les autres paramètres restent comparables : signé par la même autorité, durée de vie de 397 jours, nom commun correspondant à l'identifiant de l'utilisateur.

Un certificat est créé par utilisateur. Cette individualisation permet plusieurs choses :

- Révoquer l'accès d'un seul utilisateur sans toucher aux autres, par exemple en cas de départ d'un collaborateur ou de perte d'un poste.
- Identifier dans les journaux quel certificat a été utilisé pour une connexion.
- Adapter la durée ou les attributs au cas par cas si besoin.

## Chaîne de confiance

Quand un client se connecte au VPN, le serveur présente d'abord son certificat. Le client vérifie que ce certificat a bien été signé par la CA qu'il connaît déjà. Ensuite, le client présente son propre certificat. Le serveur vérifie que ce certificat a bien été signé par la même CA.

Si l'une des deux vérifications échoue, la connexion est refusée. Cela se produit avant même la phase d'authentification par identifiant et mot de passe, qui constitue le second facteur.

## Cycle de vie

L'émission d'un nouveau certificat suit la même procédure que celle décrite plus haut. La révocation se fait depuis Système, Gestion des certificats, Listes de révocation. Le renouvellement avant expiration consiste à émettre un nouveau certificat avec le même nom commun. La sauvegarde régulière de la configuration OPNsense complète permet de protéger l'ensemble de la PKI ; en cas de perte, c'est la capacité même de signer ou de révoquer qui disparaît.

## Points d'attention

La protection de la clé privée de la CA est critique. En environnement de production, on génère idéalement la CA hors ligne et on la stocke de manière sécurisée. Pour un projet scolaire, la clé reste sur OPNsense, mais le principe est important à connaître.

L'utilisation effective de la liste de révocation suppose qu'elle soit activée dans la configuration du serveur OpenVPN. Sans cela, un certificat révoqué resterait techniquement utilisable.
