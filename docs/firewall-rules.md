# Aliases et règles de pare-feu

## Le défi de la cohabitation

L'OPNsense de l'école est utilisé simultanément par plusieurs étudiants pour leurs projets respectifs. Sans précaution, on risque de modifier des règles créées par un autre étudiant, de créer des conflits d'IP ou de noms d'objets, de casser le projet d'un camarade en désactivant la mauvaise règle, ou de ne pas pouvoir tracer qui a fait quoi.

J'ai adopté une convention de nommage stricte permettant à chacun d'identifier ses propres objets et de les désactiver ou supprimer sans risque pour les autres.

## Convention adoptée

Tous les objets que je crée sur l'OPNsense partagé sont préfixés par mon prénom (`ANTONIN_*`), décrits explicitement avec une mention `[ANTONIN]` en début de description, et marqués `DESACTIVER SI PROBLEME` dans la description des règles. Cette dernière mention permet à n'importe qui (professeur, autre étudiant) de comprendre qu'on peut désactiver la règle sans tout casser.

## Aliases créés

Un alias dans OPNsense est un nom symbolique qui regroupe une ou plusieurs valeurs (IP, réseau, port). Au lieu d'écrire en dur une IP dans plusieurs règles différentes, on crée un alias et on le référence partout. Si l'IP change, on modifie l'alias une seule fois.

J'ai créé trois aliases :

- `ANTONIN_SRV` (type Hôte) qui pointe vers l'IP du Windows Server.
- `ANTONIN_VPN_TUNNEL` (type Réseau) qui correspond au réseau du tunnel `10.10.0.0/24`.
- `ANTONIN_PORTS` (type Ports) qui regroupe les ports SMB (445) et OpenVPN (1194).

## Règles de pare-feu créées

Deux règles ont été créées pour le projet.

La première est une règle WAN qui autorise le trafic UDP entrant sur le port 1194, ce qui permet aux clients VPN d'atteindre OPNsense. Source à "any", destination à l'adresse WAN, port défini via l'alias `ANTONIN_PORTS`. Description : "[ANTONIN] VPN entrant — DESACTIVER SI PROBLEME".

La seconde est une règle OpenVPN qui autorise le trafic du tunnel à atteindre le serveur de fichiers. Source définie via `ANTONIN_VPN_TUNNEL`, destination via `ANTONIN_SRV`, protocole "any". Description : "[ANTONIN] VPN vers serveur — DESACTIVER SI PROBLEME".

Les deux règles sont précises : pas de "any vers any" par paresse. La désactivation se fait en un clic sur l'icône d'état de chaque règle, sans suppression.

## Bénéfices de cette approche

Cette approche apporte plusieurs avantages concrets. Le filtrage de la liste des règles sur la chaîne `ANTONIN` donne une visibilité immédiate sur tout ce qui m'appartient. La désactivation est réversible et ne supprime rien. La mention `DESACTIVER SI PROBLEME` rend la communication facile en cas d'incident, sans qu'un tiers ait besoin de comprendre tout le projet pour agir. La description elle-même fait office de documentation, lisible directement dans l'interface.

Cette logique n'est pas spécifique à un environnement scolaire. En entreprise, les pare-feux sont souvent mutualisés entre équipes ou projets, et adopter une nomenclature dès la conception facilite les audits, réduit les erreurs lors des changements et permet d'identifier les règles obsolètes à nettoyer.
