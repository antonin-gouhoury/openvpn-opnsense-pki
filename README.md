# VPN d'accès distant sécurisé avec PKI sur OPNsense

Projet BTS SIO option SISR — Épreuve E5/E6 — Année 2025/2026
Réalisé en alternance chez APPIMAC (Saint-Germain-Laval).

## Présentation

Mise en place d'une solution complète d'accès distant chiffré au serveur de fichiers d'une PME. Le projet a été déployé en trois phases successives : maquette VirtualBox pour valider la chaîne complète, migration du serveur de fichiers vers Proxmox VE, puis intégration de la partie VPN sur l'OPNsense physique partagé entre plusieurs étudiants.

L'authentification combine un certificat client signé par une autorité de certification interne et un identifiant personnel. L'accès au partage de fichiers est doublé d'une ACL NTFS côté Windows Server, pour une défense en profondeur.

## Stack technique

- OPNsense (pare-feu, serveur OpenVPN, PKI)
- OpenVPN
- Proxmox VE (hyperviseur du serveur de fichiers)
- Windows Server 2019 (Active Directory, partage SMB)
- VirtualBox (maquette initiale)

## Architecture

L'OPNsense physique sert de pare-feu et de concentrateur VPN pour l'ensemble de l'infrastructure. Le réseau est segmenté en VLAN distincts pour les serveurs, les postes clients et la DMZ. Le serveur de fichiers Windows est hébergé sur le Proxmox du VLAN Serveurs.

Le tunnel VPN est établi depuis un poste client (VLAN Clients) jusqu'à OPNsense, puis routé vers le Windows Server à travers le VLAN Serveurs. Sans VPN actif, le serveur reste injoignable depuis le VLAN Clients.

Détail dans [docs/architecture.md](docs/architecture.md).

## Sécurité

Plusieurs couches ont été mises en œuvre :

- Tunnel chiffré OpenVPN entre le client et OPNsense.
- Authentification à deux facteurs : certificat client signé par la CA interne, plus identifiant et mot de passe utilisateur.
- Restriction d'accès au VPN limitée aux membres d'un groupe OPNsense dédié.
- ACL NTFS sur le partage Windows, restreinte au groupe AD des télétravailleurs.
- Isolation réseau : le serveur n'est joignable depuis le VLAN Clients qu'à travers le tunnel.

Détail dans [docs/pki-setup.md](docs/pki-setup.md) et [docs/openvpn-server.md](docs/openvpn-server.md).

## Cohabitation sur infrastructure partagée

L'OPNsense est utilisé simultanément par plusieurs étudiants. Pour éviter tout conflit, j'ai adopté une convention stricte :

- Tous les objets créés sont préfixés par mon prénom (`ANTONIN_*`).
- Chaque règle porte une description explicite mentionnant qu'elle peut être désactivée en cas de problème, sans impact sur les autres projets.
- Les règles utilisent des aliases plutôt que des IPs en dur, ce qui permet de modifier une cible à un seul endroit.

Détail dans [docs/firewall-rules.md](docs/firewall-rules.md).

## Phases de déploiement

**Phase 1 — Maquette VirtualBox.** Trois VMs isolées (OPNsense, Windows Server 2019, client Windows 10) pour valider la chaîne complète sans toucher à l'infrastructure de production.

**Phase 2 — Migration vers Proxmox.** Export de la VM Windows Server au format OVA depuis VirtualBox, pré-installation des pilotes VirtIO NetKVM, import sur l'hyperviseur Proxmox VE.

**Phase 3 — Intégration sur OPNsense physique.** Reproduction de la configuration VPN sur le pare-feu partagé en respectant la convention de nommage.

## Validation

Tous les tests fonctionnels prévus ont été validés :

- Tunnel établi depuis le VLAN Clients.
- Accès au partage `\\<serveur>\Entreprise_DATA` à travers le tunnel.
- Serveur injoignable sans VPN actif.
- Cohabitation sans impact sur les projets des autres étudiants.

## Documentation

- [Architecture et plan d'adressage](docs/architecture.md)
- [Mise en place de la PKI](docs/pki-setup.md)
- [Configuration du serveur OpenVPN](docs/openvpn-server.md)
- [Aliases et règles de pare-feu](docs/firewall-rules.md)
- [Diagnostic et résolution d'incidents](docs/troubleshooting.md)

## Auteur

Antonin Gouhoury — Étudiant BTS SIO SISR, UTEC Melun.
[LinkedIn](https://linkedin.com/in/antonin-gouhoury) · gouhouryantonin@gmail.com

---

Projet scolaire mené en autonomie. Les noms de domaines, IP et identifiants présents dans la documentation sont fictifs ou propres à un environnement de maquette.
