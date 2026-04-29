# Configuration du serveur OpenVPN

Une fois la PKI en place (voir [pki-setup.md](pki-setup.md)), on configure le serveur OpenVPN proprement dit dans OPNsense, depuis VPN, OpenVPN, Instances, Ajouter.

## Paramètres généraux

Le rôle est défini à "Serveur" puisque OPNsense agit ici comme concentrateur VPN. Le protocole retenu est UDP4 et non simplement UDP : ce point est important car j'ai rencontré une erreur de bind sur IPv6 lors des premiers tests, qui se résout en forçant explicitement IPv4. Le détail du diagnostic est dans [troubleshooting.md](troubleshooting.md).

Le port utilisé est 1194, le port standard OpenVPN. Le type d'interface est TUN, qui correspond à un tunnel de niveau 3 (couche IP). Le niveau de verbosité est défini à 3 pour conserver des journaux exploitables sans être trop bavards.

## Gestion des certificats

C'est ici qu'on rattache la PKI configurée précédemment.

Le certificat serveur sélectionné est celui qui a été créé à l'étape PKI. L'autorité de certification est celle qui a signé ce certificat. La vérification du certificat client est définie à "Requis", ce qui constitue un point clé de la double authentification : sans certificat client valide, la connexion est refusée avant même la phase login.

Les algorithmes de chiffrement (Data Ciphers) retenus sont ceux recommandés par OPNsense par défaut.

## Authentification

La source d'authentification est définie à "Local Database", c'est-à-dire la base d'utilisateurs interne d'OPNsense. Le champ "Appliquer le groupe local" est défini sur le groupe `Groupe_Teletravail`, ce qui restreint l'accès VPN aux seuls membres de ce groupe.

Cette restriction est importante : sans elle, n'importe quel utilisateur de la base locale OPNsense pourrait potentiellement se connecter, même sans rôle de télétravailleur.

## Routage

Le réseau du tunnel (champ "Server IPv4") est défini à `10.10.0.0/24`. C'est ce réseau qui sera utilisé pour les IPs distribuées aux clients connectés. OpenVPN prend automatiquement la première IP (`10.10.0.1`) pour le serveur et distribue les suivantes aux clients.

Le réseau local poussé aux clients est celui du VLAN Serveurs. Cela ajoute automatiquement, sur chaque client connecté, une route vers ce réseau, ce qui permet d'atteindre le serveur de fichiers à travers le tunnel.

À noter que le champ "Server IPv4" désigne bien le réseau du tunnel et non l'IP du serveur de fichiers. C'est une confusion fréquente.

## Préparation côté utilisateurs

Avant qu'un client puisse se connecter, plusieurs prérequis doivent être réunis. Un groupe doit être créé dans OPNsense (Système, Accès, Groupes). Chaque utilisateur autorisé doit ensuite être créé dans la base locale (Système, Accès, Utilisateurs) avec un mot de passe complexe, et ajouté au groupe créé. Enfin, un certificat client doit être émis pour chaque utilisateur, comme décrit dans [pki-setup.md](pki-setup.md).

Un utilisateur autorisé dispose donc de trois éléments : un compte dans la base locale OPNsense, une appartenance au bon groupe, et un certificat client signé par la CA interne.

## Génération du profil client

Le profil client est un fichier `.ovpn` qui contient toutes les informations nécessaires à la connexion : adresse du serveur, certificat de la CA, certificat client, paramètres de chiffrement.

Sur les versions récentes d'OPNsense, l'export client n'est plus inclus en standard. Il faut installer le plugin `os-openvpn-legacy` depuis Système, Micrologiciel, Greffons, après avoir coché "Afficher les plugins de la communauté". Une fois le plugin installé, l'option apparaît dans VPN, OpenVPN, Exporter le client.

Pour l'export, on sélectionne l'instance VPN concernée, le type d'exportation (archive ou fichier de configuration unique), le nom d'hôte du pare-feu et le port 1194. Le fichier généré est ensuite distribué à l'utilisateur, qui l'importera dans son client OpenVPN Connect.

## Couche supplémentaire côté Windows Server

La sécurité ne repose pas uniquement sur le tunnel VPN. Côté serveur Windows, le partage de fichiers est protégé par une ACL NTFS qui restreint l'accès aux seuls membres d'un groupe AD `Groupe_Télétravail`. Les autres utilisateurs du domaine sont explicitement bloqués.

Cette double protection illustre le principe de défense en profondeur : même si un attaquant parvenait à se connecter au VPN, il ne pourrait pas lire les fichiers s'il n'est pas dans le groupe AD côté Windows.
