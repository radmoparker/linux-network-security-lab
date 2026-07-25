# linux-network-security-lab
## Lab SOC - Firewall Debian (nftable), Suricata et Wzuh

## 1. Présentation
**Objectif du laboratoire**
La mise en place d'un laboratoire virtuel est essentiel pour prendre en main la plupart des outils de sécurité informatique.

Ce laboratoire a été conçu afin de mettre en place une architecture réseau sécurisée intégrant un pare-feu Linux, un IDS réseau et une solution SIEM. Il permet de simuler différents scénarios d'attaque et d'observer leur détection dans un environnement virtualisé.


Note : Ce Lab est également un moyen d'apréhender un parefeu plus technique que pfsense (pratiqué lors de projets universitaire)

**Machine et outil de virtualisation**
La machine dans laquelle est implémenté ce lab est un Macbookpro 2023 (apple silicon M2 Pro 16GB RAM). On est donc sur un processeur en AMRM64.

L'outil de virtualisation choisi est VMware Fusion pour plusieur raisons :
- Disponible sur MacOS
- Permet d'atttacher plusieurs interfaces réseau à une V, ce qui est essentiel pour ce Lab.

**Technologies utilisées**
- Nftable : pour manipuler le parefeu Netfilter (parefeu linux)
- rsyslog : permet de ollecter les journeaux générés par le parefeu
- Suricata : Utilisé en IDS sur l'interface Wan du parefeu afin d'identifier les tentatives d'intrusions
- Wazuh : Générer des alertes à partir de Suricata ainsi que des logs du parefeu collectés par rsyslog
- Scapy : Pour générer pour générer des paquets identifié par le parefeu comme risqués (notament sur le connexion tracking)
- Nmap : Pour Tester l'IDS suricata

- Kali Linux : Machine utilisé pour tester les tentatives d'intrusions ou de cartographie.
- Linux Desktop / Server : L'un pour l'administration et la supervision, l'autre pour le service Web (minimale)

**Compétences mises en œuvre**
- Mise en place d'une politique de filtrage (politique de sécurité réseau)
- Mise en place du NAT (SNAT et DNAT)
- Routage et Ip statiques
- Mise en place de règles et d'alertes autour du "connexion tracking"


## 2. Architecture
**Schéma réseau**
Annexe 1
**Description des machines**
- Kali : Machine exterieur dédiée aux attaques, c'est avec elle qu'on va tester notre architectures (accès ou non accès aux services, tentatives d'intrusions, scan de ports)
- Debian : C'est cette machine qui va intégrer les fonctions de routage et de parefeu avec nftable. Elle va également générer des logs, intégrer Suricata en IDS et faire tourner un Agent Wazuh.
- Ubunut Desktop : C'est la machine admin qui va intégrer Wazuh (en single node) et établir des connexions SSH avec l'interieur de notre réseaux
- Ubuntu Serveur : Service Web simple et accessible depuis l'exterieur.

**Flux Réseaux**
Annexe 2 : Flux de communication Réseaux

## 3. Configuration Vmare
**Création des différentes interfaces**
- vmnet 2 : 192.168.231.0/24
- vmnet 3 : 192.168.89.0/24
- vmnet 4 : 172.16.44.0/24

**Assignation des interfaces aux machines**
- Kali : vmnet 2 
- Debian : vmnet 2, vmnet 3, vmnet 4
- Admin : vmnet 4
- Server : vmnet 3

Annexe 3 : Création et assignations des interfaces sur Vmare




