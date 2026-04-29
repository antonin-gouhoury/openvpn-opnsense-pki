# Diagnostic et résolution d'incidents

Cette page documente les incidents rencontrés lors du déploiement, leur diagnostic et les solutions appliquées. Les écueils techniques apprennent souvent plus que les configurations qui marchent du premier coup, autant en garder trace.

## Connection Timeout à la connexion VPN

Le client OpenVPN Connect affichait systématiquement le message "Connection failed to establish within given time" lors des premières tentatives.

J'ai d'abord vérifié que le service OpenVPN était bien actif côté serveur, ce qui était le cas. J'ai ensuite consulté les journaux OpenVPN dans VPN, OpenVPN, Fichier journal. Une erreur revenait à chaque tentative : `Connection Attempt write UDPv6: Can't assign requested address`.

L'erreur indique que le serveur OpenVPN tente de se mettre à l'écoute en IPv6 alors qu'IPv6 n'est pas configuré sur le réseau. La cause vient du fait que la valeur "UDP" sans précision de famille est ambiguë et que l'implémentation préfère IPv6 par défaut quand elle est disponible côté système.

La solution a consisté à remplacer "UDP" par "UDP4" dans le paramètre Protocole de l'instance VPN. Cette valeur force explicitement l'écoute en IPv4. Après sauvegarde et application, la connexion s'établit immédiatement.

À retenir : toujours lire les logs avant de chercher en ligne. Le message d'erreur exact était la clé. Et quand un protocole propose plusieurs variantes (UDP, UDP4, UDP6), être explicite plutôt que se reposer sur la valeur par défaut, surtout si l'environnement n'est pas dual-stack.

## IP refusée par Windows lors de l'attribution statique

Lors de la migration de la VM Windows Server depuis VirtualBox vers Proxmox, j'ai voulu attribuer une IP statique. Windows a refusé l'attribution avec un statut `AddressState: Invalid`.

L'IP était déjà utilisée par un autre équipement sur le réseau, vraisemblablement la VM d'un autre étudiant dans le VLAN partagé. Windows détecte le conflit via une requête ARP gratuite et refuse l'attribution.

J'ai testé plusieurs IPs voisines avec un simple ping, choisi une IP qui ne répondait pas, et confirmé que son attribution donnait bien le statut `Preferred`. La mise à jour s'est faite ensuite uniquement au niveau de l'alias OPNsense correspondant, sans avoir à modifier les règles puisque celles-ci référencent l'alias et non l'IP en dur. C'est précisément l'intérêt des aliases.

À retenir : en infrastructure partagée, vérifier la disponibilité d'une IP avant de l'attribuer. L'usage d'aliases plutôt que d'IPs en dur dans les règles paye à ce moment précis : un seul changement à un seul endroit.

## Plugin d'export client absent dans OPNsense récent

Pour distribuer le profil `.ovpn` aux utilisateurs, je m'attendais à trouver une option d'export client dans le menu OpenVPN. Sur la version récente d'OPNsense, l'option n'apparaissait nulle part.

Le plugin officiel `os-openvpn-client-export` a été retiré des dépôts standards à partir d'OPNsense 24.x. La fonctionnalité est désormais gérée différemment, mais l'export en un clic n'est plus disponible nativement.

J'ai installé le plugin de la communauté `os-openvpn-legacy`, qui restaure l'ancienne interface d'export client. Son installation passe par Système, Micrologiciel, Greffons, après avoir coché "Afficher les plugins de la communauté". Après installation, l'onglet Exporter le client réapparaît dans VPN, OpenVPN.

À retenir : une fonctionnalité absente ne signifie pas qu'elle n'existe pas. Elle peut avoir été déplacée, renommée, ou déléguée à un plugin. Penser à activer les plugins de la communauté, dont la case n'est pas cochée par défaut.

## Carte réseau invisible après migration vers Proxmox

Après import de la VM Windows Server dans Proxmox, le système bootait correctement mais aucune carte réseau n'apparaissait dans le panneau de configuration Windows.

Proxmox utilise par défaut des cartes VirtIO paravirtualisées pour de meilleures performances. Windows ne possède pas nativement les pilotes VirtIO, contrairement à Linux. Sans pilote, la carte n'est pas reconnue par le système.

La solution est préventive et doit s'appliquer avant la migration, pendant que la VM est encore dans VirtualBox. On télécharge l'ISO `virtio-win.iso` depuis le dépôt officiel Red Hat, on le monte dans la VM, on navigue dans `NetKVM\w2k19\amd64`, et on installe le fichier `netkvm.inf` par clic droit puis "Installer". Le pilote est alors présent dans Windows. À la migration, dès que Proxmox attribue une carte VirtIO, Windows la reconnaît automatiquement.

À retenir : toute migration entre hyperviseurs implique de vérifier la compatibilité des pilotes côté OS invité. Pour Windows en particulier, VirtIO doit être installé avant le changement d'hyperviseur, sinon la VM démarre sans réseau et donc sans accès Internet pour télécharger les pilotes manquants.

## Impossibilité de tester depuis Internet

Le VPN fonctionnait en interne (depuis le VLAN Clients), mais il était impossible de tester depuis l'extérieur de l'école.

L'IP WAN d'OPNsense est une IP privée non routable sur Internet. C'est l'environnement réseau de l'école : OPNsense est en NAT derrière un autre routeur, sans IP publique allouée. Ce n'est pas un défaut de configuration, c'est une contrainte de l'infrastructure scolaire.

Le test final a été réalisé depuis le VLAN Clients de l'école, en simulant un poste distant au sens fonctionnel : dans un VLAN séparé du VLAN Serveurs, plutôt qu'à distance géographique. La chaîne complète a été validée : authentification PKI et login, établissement du tunnel, accès SMB au partage à travers le tunnel.

À retenir : bien identifier la différence entre test fonctionnel et test en condition de production. En production réelle, OPNsense aurait une IP publique et le VPN serait accessible depuis n'importe quel réseau Internet. La configuration logique reste identique, seule la couche d'adressage change. À l'oral, savoir expliquer cette limite plutôt que la masquer démontre une compréhension correcte de l'environnement.
