# Architecture et plan d'adressage

## Infrastructure physique

L'infrastructure de l'école comprend un switch L2 segmenté en plusieurs VLAN, un pare-feu OPNsense centralisé et commun à plusieurs étudiants, et un nœud Proxmox VE par VLAN. Le serveur de fichiers Windows Server 2019 est hébergé sur le Proxmox du VLAN Serveurs.

Le tunnel VPN est établi depuis un poste client situé dans le VLAN Clients, jusqu'à OPNsense. Une fois le tunnel actif, OPNsense route le trafic vers le serveur de fichiers à travers le VLAN Serveurs.

## Schéma logique

```
                        Internet
                            |
                  +-------------------+
                  |     OPNsense      |
                  | Pare-feu + VPN    |
                  +---------+---------+
                            |
        +-------------------+-------------------+
        |                   |                   |
   VLAN Serveurs        VLAN DMZ         VLAN Clients
        |                                       |
   Proxmox VE                              Poste client
        |                                  (test VPN)
   Windows Server 2019
   (partage SMB)
```

## Plan d'adressage anonymisé

Les valeurs réelles (IPs, noms de domaines) sont remplacées par des placeholders neutres dans toute la documentation publique du repo.

- WAN : `192.168.X.0/24`, passerelle `192.168.X.1` (réseau école, IP privée non routable depuis Internet)
- VLAN Serveurs : `192.168.A.0/24`, passerelle `192.168.A.254`
- VLAN DMZ : `192.168.B.0/24`, passerelle `192.168.B.254`
- VLAN Clients : `192.168.C.0/24`, passerelle `192.168.C.254`
- Réseau du tunnel VPN : `10.10.0.0/24` (IPs distribuées dynamiquement aux clients connectés)

## Architecture des machines virtuelles

Une seule VM joue un rôle de serveur de production : la VM Windows Server 2019 hébergée sur Proxmox. Elle assure les fonctions de contrôleur de domaine Active Directory et de serveur de fichiers SMB.

Une VM Windows 10 sert de poste client de test pour valider la chaîne VPN. En production, ce rôle est tenu par les postes des télétravailleurs.

## Convention de nommage en infrastructure partagée

L'OPNsense étant utilisé par plusieurs projets en parallèle, tout objet créé est préfixé par mon prénom (`ANTONIN_*`) et marqué dans sa description comme désactivable en cas de problème, sans impact sur les autres projets. Voir [firewall-rules.md](firewall-rules.md) pour le détail.

## Justification des choix d'architecture

**Déploiement progressif en trois phases.** La maquette VirtualBox a permis de valider l'ensemble de la chaîne (PKI, tunnel, ACL, partage) en environnement isolé avant de toucher à l'infrastructure de production. Chaque phase apporte une nouvelle contrainte (matériel réel, infrastructure partagée), ce qui rend la dette technique plus facile à isoler.

**VPN hébergé sur OPNsense plutôt que sur Windows Server.** OPNsense intègre nativement OpenVPN avec une gestion graphique de la PKI. Il est positionné en frontal du réseau, ce qui est la position naturelle d'un concentrateur VPN. Cette séparation découple aussi l'authentification (gérée par OPNsense) de l'accès aux ressources (géré par Windows et les ACL NTFS).

**Test du VPN depuis le VLAN Clients.** L'IP WAN d'OPNsense est une IP privée du réseau école, non routable depuis Internet. Le test depuis le VLAN Clients reproduit les conditions fonctionnelles d'un accès distant : le client passe par OPNsense pour atteindre le VLAN Serveurs, ce qui valide la chaîne complète. En production, OPNsense disposerait d'une IP publique routable.
