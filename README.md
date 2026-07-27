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

## 3. Schéma réseau et configuration Vmare
**Création des différentes interfaces**
- vmnet 2 : 192.168.231.0/24
- vmnet 3 : 192.168.89.0/24
- vmnet 4 : 172.16.44.0/24

**Assignation des interfaces aux machines et IPs statiques**
- Kali : vmnet 2 -> 192.168.231.100
- Debian : vmnet 2 -> 192.168.231.1, vmnet 3 -> 192.168.89.1, vmnet 4 -> 172.16.44.1
- Admin : vmnet 4 -> 172.16.44.2
- Server : vmnet 3 -> 192.168.89.2

Annexe 3 : Création et assignations des interfaces sur Vmare

## 4. Routage
**Routage Pare-feu Debian (Routagee par défaut car chaque réseaux est lié à une interface)**
- 192.168.231.0/24 via vmnet 2 (enp2s0 du point de vue de la machine) 
- 192.168.89.0/24 via vmnet 3 (enp26s0 du point de vue de la machine) 
- 172.168.231.0/24 via vmnet 4 (enp3s0 du point de vue de la machine) 
**Routage admin**
- 192.168.231.0/24 via 172.16.44.1 (vmnet 4)
- 192.168.89.0/24 via 172.16.44.1 (vmnet 4)
**Routage serveur**
- 192.168.231.0/24 via 192.168.89.1 (vmnet 3)
- 172.16.44.0/24 via 192.168.89.1 (vmnet 3)

## Déploiement du pare-feu
**Politique de sécurité**
La politique de sécurité a pour but de garantir la disponibilité des services, l'isolation des différentes zones, et l'accès à l'ensemble de notre réseaux interne à la machine admin.

Notre politique de filtrage consisidère que :
* Le routeur doit pouvoir  ping partout (même à l'extérieur)
* Le routeur doit pouvoir accéder au service Web de l’exterieur mais pas l’inverse
* Le routeur doit pouvoir accéder au service Web de notre serveur interne mais pas l’inverse

* L’exterieur doit pouvoir accéder au service http de notre serveur interne  
* Les zones admin et serveur doivent pouvoir ping l’extérieur mais pas l'inverse
* Les zones admin et serveur doivent pouvoir accéder aux services web (port 80) à l'extérieur 
* La machine admin doit pouvoir accéder en SSH au pare-feu (debian) et au serveur interne, mais pas l'inverse

**NAT**
* DNAT : Les machines à l'exterieur de notre réseaux accèdent aux services Web interne en passant par l’ip du routeur 192.168.231.1 port 80 via l'interface vmnet 2. Le parefeu les redirige vers l'ip 192.168.89.2 port 80.
* SNAT : Lorsqu'une machine souhaite accéder aux services à l'exterieur de notre réseaux, le pare-feu modifie leur adresse source par la sienne (sur l'interface extérieur) : 192.168.231.1

**Connexion Tracking**
On part du principe qu'une connexion n'est autorisé, que si elle correspond à un état prévu par notre politique (stateful firewall). On ajoute ainsi les règles suivantes : 

* Une réponse SSH (sport 22) vers l'admin, n'est autorisé que si elle correspond à une connexion déjà établie
* Une réponse HTTP (sport 80) vers l'admin,le pare-feu ou le serveur interne n'est autorisé que si elle correspond à une connexion déjà établie

* Une réponse HTTP (sport 80) provenant de l'exterieur et en direction du serveur interne ou du firewalle, n'est autorisé que si elle correspond à une connexion déjà établie.

**Journalisation**
Tous les paquets correspondant à une tentative de connexion proscrite (non prévu par la politique de filtrage ou échouant aux règles connexion tracking), on va ainsi préfixer ces cas pour plus tard créer des alertes :

* CT-ERR_FROM-EXTERNAL_ICMP-REP_ : 
    - Réponse ICMP provenant de l'exterieur échouant au connexion tracking
* CT-ERR_FROM-EXTERNAL_HTTP-REP_ :
    - Réponse HTTP provenant de l'exterieur échouant au connexion tracking 
* CT-ERR_FROM-DMZ-TO-ADMIN_HTTP-REP_ : 
    - Réponse HTTP provenant du serveur interne vers admin échouant au connexion tracking 
* CT-ERR_FROM-DMZ-TO-ADMIN_ICMP-REP_ :
    - Réponse ICMP provenant du serveur interne vers admin échouant au connexion tracking
* CT-ERR_FROM-DMZ-TO-ADMIN_SSH-REP_ :
    - Réponse SSH provenant du serveur interne vers admin échouant au connexion tracking

* DENY_FROM-EXTERNAL-TO-ADMIN_HTTP-REQ_ :
    - Requête HTTP de l'exterieur vers admin
* DENY_FROM-DMZ-TO-ADMIN_HTTP-REQ_
    - Requête HTTP du serveur interne vers admin
* DENY_FROM-EXTERNAL-TO-ADMIN_SSH-REQ_ :
    - Requête SSH de l'exterieur vers admin

* DENY_FROM-EXTERNAL-TO-DMZ_SSH-REQ_ :
    - Requête SSH de l'extérieur vers serveur interne
* DENY_FROM-DMZ-TO-ADMIN_SSH-REQ_
    - Requête SSH du serveur interne vers admin
* DENY_FROM-EXTERNAL-TO-ADMIN_ICMP-REQ_ :
    - Requête ICMP de l'extérieur vers admin 
* DENY_FROM-EXTERNAL-TO-DMZ_ICMP-REQ_
    - Requête ICMP de l'exterieur vers serveur interne
* DENY_FROM-DMZ-TO-ADMIN_ICMP-REQ_
    - Requête ICMP du serveur interne vers admin

Note : Les logs du parefeu sont collectés par rsyslog. On les retrouvera dans le fichier kern.log (cela nous sera utile lorsqu'on devra les transmettre à Wazuh)


**Règles pour l'agent Wazuh**
Pour transmettre les logs collectés avec Rsyslog, ainsi que les alertes générées par Suricata, l'agent Wazuh installé sur le parefeu a besoin de contacter la machine admin sur le port 1514. De plus pour s'enregistré auprès de Wazuh Manager, l'agent aura besoin de contacter la machine admin sur le port 1515 (Cette règle n'est utile qu'une seule fois ).
On ajoute les règles suivante sur le pare-feu :
- Les connexions sortantes du parefeu sur les ports 1514 et 1515 sont autorisées
- Les connexions entrantes sur le parefeu sur les ports 1514 et 1515 sont autorisées si elles appartiennes à une connexion déjà établie

**Note technique**
Netfilter, le pare-feu Linux fonctionnent avec 5 points d'accroches vers lesquels transite les paquets (les hooks).
On retiendra que : 
- Le hook prerouting est un hook de pré traitement : il contiendra les règles DNAT
- Le hook postrouting est un hook de post traitement : il contiendra les règles SNAT
- Les hooks input et output filtrent les connexions entrantes et sortantes (resp.) sur la machine contenant le firewall.
- Le hook forward filtrent les connexions qui ne sont pas à destination de la machine 

Aussi, les logs sont positionnés sur une chaine simple (indépendante, qui est appellée à la fin du hook forward), pour rendre la lecture plus lisible.


Annexe 4 : nftables.conf

**Suricata en IDS**

**SIEM Wazuh**

**Test et validation**



