# Cahier des Charges — Proxmox Secure Test Lab v3.0

> **Version** : 3.0
> **Date** : 10 février 2026
> **Auteur** : Tom Daluzeau
> **Statut** : En cours de rédaction
> **Classification** : Portfolio — Usage personnel

---

## Table des matières

1. [Introduction](#1-introduction)
2. [Description du matériel](#2-description-du-matériel)
3. [Contraintes](#3-contraintes)
4. [Exigences fonctionnelles](#4-exigences-fonctionnelles)
5. [Exigences techniques](#5-exigences-techniques)
6. [Architecture cible](#6-architecture-cible)
7. [Plan de réalisation](#7-plan-de-réalisation)
8. [Livrables attendus](#8-livrables-attendus)
9. [Critères de validation](#9-critères-de-validation)
10. [Gestion des risques](#10-gestion-des-risques)
11. [Glossaire](#11-glossaire)

---

## 1. Introduction

### 1.1. Objet du document

Le présent document constitue le cahier des charges du projet « Proxmox Secure Test Lab ». Il a pour vocation de formaliser l'ensemble des besoins fonctionnels et techniques, les contraintes identifiées, les choix d'architecture retenus et les livrables attendus pour la réalisation d'un environnement de test sécurisé basé sur la virtualisation Proxmox VE.

Ce document servira de référence tout au long du cycle de vie du projet, de la phase de conception à la recette finale. Il pourra être amendé en fonction des retours d'expérience rencontrés lors de la mise en œuvre.

### 1.2. Contexte du projet

Dans le cadre de la préparation d'un Bachelor en Administration Système DevOps (ESGI, rentrée septembre 2026) et d'une recherche active d'alternance en Île-de-France, il est nécessaire de disposer d'un environnement de test professionnel permettant de pratiquer l'administration système, la gestion réseau, l'automatisation d'infrastructure et les pratiques DevOps.

Ce projet repose sur le recyclage de deux anciens ordinateurs portables transformés en infrastructure serveur, démontrant une capacité à optimiser des ressources limitées — une compétence valorisée en entreprise.

### 1.3. Objectifs du projet

#### 1.3.1. Objectif principal

Concevoir, déployer et documenter un environnement de test sécurisé et segmenté, hébergé sur Proxmox VE, accompagné d'un serveur NAS de sauvegarde, le tout étant reproductible, automatisé et publié en tant que projet portfolio sur GitHub.

#### 1.3.2. Objectifs secondaires

- Pratiquer la segmentation réseau via VLANs et firewall avec pfSense
- Mettre en place une chaîne CI/CD complète (Forgejo, Woodpecker CI, Docker Registry)
- Implémenter un monitoring centralisé (VictoriaMetrics, Grafana)
- Appliquer les bonnes pratiques de sécurité (hardening SSH, Fail2ban, MFA, chiffrement)
- Automatiser l'ensemble du déploiement via Ansible et scripts Bash
- Mettre en place une stratégie de sauvegarde et de reprise d'activité (Disaster Recovery)
- Produire une documentation professionnelle exploitable en contexte portfolio

### 1.4. Public cible

Ce projet s'adresse en premier lieu à l'auteur dans un cadre de montée en compétences. La documentation produite et le dépôt GitHub sont destinés à être présentés à des recruteurs, responsables techniques et équipes DevOps dans le cadre d'une recherche d'alternance.

### 1.5. Périmètre du projet

**Inclus dans le périmètre :**

- Installation et configuration de l'hyperviseur Proxmox VE sur le PC 1
- Installation et configuration du serveur NAS Debian sur le PC 2
- Mise en place de la segmentation réseau (4 VLANs) via pfSense
- Déploiement des services : reverse proxy, base de données, monitoring, CI/CD
- Sécurisation de l'ensemble de l'infrastructure
- Automatisation complète via Ansible, Bash et optionnellement Terraform
- Stratégie de backup et plan de reprise d'activité documenté
- Publication sur GitHub avec documentation complète

**Exclu du périmètre :**

- Mise en production de services accessibles publiquement sur Internet
- Haute disponibilité (cluster Proxmox multi-nœuds)
- Achat de matériel supplémentaire
- Gestion de noms de domaine publics et certificats Let's Encrypt

---

## 2. Description du matériel

### 2.1. PC 1 — Nœud Proxmox principal

Le PC 1 constitue le cœur de l'infrastructure. Il héberge l'hyperviseur Proxmox VE ainsi que l'ensemble des machines virtuelles et containers de l'environnement de test.

| Composant | Spécification | Notes |
|-----------|---------------|-------|
| **Processeur** | Intel Core i5-8265U (4C/8T, 8e gen, Whiskey Lake) | Support VT-x et VT-d confirmé |
| **Mémoire vive** | 8 Go DDR4 | Contrainte principale du projet |
| **Stockage** | 250 Go SSD | Ext4, pas de ZFS (un seul disque) |
| **Virtualisation** | VT-x / VT-d | Activation requise dans le BIOS |
| **Rôle** | Hyperviseur Proxmox VE | Héberge toutes les VMs et containers LXC |

### 2.2. PC 2 — Serveur NAS / Backup

Le PC 2 assure la fonction de serveur de sauvegarde. Il reçoit les backups des machines virtuelles, des bases de données et des fichiers de configuration depuis le PC 1.

| Composant | Spécification | Notes |
|-----------|---------------|-------|
| **Processeur** | Intel Core i5-5200U (2C/4T, 5e gen, Broadwell) | Suffisant pour NFS et rsync |
| **Mémoire vive** | 8 Go DDR3L | Confortable pour un serveur de fichiers |
| **Stockage** | 1 To HDD | Monitoring SMART activé |
| **Système** | Debian 12 minimal (headless) | Pas d'environnement de bureau |
| **Rôle** | NAS / Backup | NFS + réception vzdump + pg_dump |

### 2.3. Topologie physique

Les deux machines sont connectées via un réseau local Ethernet. Le PC 1 (Proxmox) fait office de routeur interne grâce à pfSense déployé en machine virtuelle, tandis que le PC 2 (NAS) est positionné sur le VLAN de management pour recevoir les sauvegardes.

---

## 3. Contraintes

### 3.1. Contrainte matérielle principale : 8 Go de RAM

Avec seulement 8 Go de RAM sur le nœud Proxmox, l'optimisation mémoire constitue la contrainte technique majeure du projet. Chaque mégaoctet alloué doit être justifié. Cette contrainte impose les principes suivants :

1. Prioriser les containers LXC (10 à 50 fois plus légers qu'une VM complète)
2. Limiter le nombre de VMs à deux : pfSense (obligatoire) et Kali Linux (à la demande)
3. Configurer un swap de 4 Go sur SSD comme filet de sécurité
4. Éviter toute surallocation agressive et conserver au minimum 512 Mo de marge pour Proxmox
5. Éteindre les services non essentiels par défaut (Kali Linux notamment)

#### Budget RAM détaillé

| Composant | Type | RAM allouée | Notes |
|-----------|------|-------------|-------|
| Proxmox VE (host) | Host OS | ~1 Go | Ext4 (pas de ZFS ARC) |
| pfSense | VM | 512 Mo | Firewall + DHCP + DNS |
| Nginx Reverse Proxy | CT LXC | 256 Mo | Proxy léger |
| PostgreSQL | CT LXC | 512 Mo | Base de données de test |
| VictoriaMetrics + Grafana | CT LXC | 512 Mo | Monitoring (remplace Prometheus) |
| Forgejo | CT LXC | 384 Mo | Git self-hosted (fork Gitea) |
| Woodpecker CI | CT LXC | 384 Mo | CI/CD (fork Drone CI) |
| Docker Registry | CT LXC | 256 Mo | Registry v2 minimal |
| Kali Linux | VM | 2 Go | ⚡ Éteint par défaut |
| **Total (sans Kali)** | — | **~3,8 Go** | Mode nominal |
| **Total (avec Kali)** | — | **~5,8 Go** | Capacité maximale |
| **Marge libre** | — | **~2,2 Go** | Buffers, cache, pics |

> 💡 **Gain par rapport à la version précédente** : environ 600 Mo récupérés grâce à VictoriaMetrics (vs Prometheus) et Woodpecker CI (vs Drone CI). La marge passe de 1,6 Go à 2,2 Go.

### 3.2. Contrainte stockage

Le SSD de 250 Go du PC 1 impose un partitionnement réfléchi : environ 20 Go pour le système Proxmox, 4 Go pour le swap et le reste (~220 Go) pour les VMs et containers. Le filesystem ext4 est retenu car ZFS est inutile sur un seul disque et consomme de la RAM via l'ARC.

### 3.3. Contraintes de sécurité

- Aucun service ne doit être exposé directement sur Internet
- L'accès distant se fait exclusivement via Tailscale (VPN mesh, pas de NAT)
- Les secrets doivent être gérés via Ansible Vault, jamais en clair dans le dépôt Git
- L'authentification SSH par mot de passe est désactivée sur toute l'infrastructure
- La zone de pentest (VLAN 40) est totalement isolée des autres VLANs

### 3.4. Contraintes de temps et de budget

Le projet est réalisé sur le temps personnel, sans budget matériel supplémentaire. Le matériel existant est utilisé en l'état. Les logiciels sélectionnés sont tous open source et gratuits.

---

## 4. Exigences fonctionnelles

### 4.1. Segmentation réseau

L'infrastructure doit être segmentée en quatre zones réseau isolées via VLANs, contrôlées par un firewall centralisé. Chaque VLAN possède son propre sous-réseau et ses règles d'accès spécifiques.

| VLAN | Nom | Réseau | Passerelle | Rôle |
|------|-----|--------|------------|------|
| 10 | Management | 10.10.10.0/24 | 10.10.10.254 | Administration Proxmox, SSH, NAS |
| 20 | Services | 10.10.20.0/24 | 10.10.20.254 | Nginx, PostgreSQL, Monitoring |
| 30 | CI/CD | 10.10.30.0/24 | 10.10.30.254 | Forgejo, Woodpecker, Registry |
| 40 | DMZ | 10.10.40.0/24 | 10.10.40.254 | Kali Linux (isolée) |

#### Matrice de flux inter-VLANs

| Source | Destination | Action | Ports autorisés |
|--------|-------------|--------|-----------------|
| VLAN 10 (Management) | Tous les VLANs | AUTORISER | Tous (administration) |
| VLAN 20 (Services) | VLAN 30 (CI/CD) | AUTORISER | TCP 22, 80, 443, 3000, 8000, 9090 |
| VLAN 30 (CI/CD) | VLAN 20 (Services) | AUTORISER | TCP 22, 80, 443, 5432, 9090, 3100 |
| VLAN 40 (DMZ) | VLANs 10, 20, 30 | BLOQUER | Aucun (isolation totale) |
| VLAN 40 (DMZ) | Internet | AUTORISER | Tous (mises à jour) |
| Tous les VLANs | pfSense (DNS) | AUTORISER | UDP/TCP 53 |

### 4.2. Services applicatifs

#### 4.2.1. Reverse Proxy (Nginx)

Un reverse proxy Nginx centralise les accès HTTP/HTTPS vers l'ensemble des services web internes. Il assure le routage par nom d'hôte (virtual hosts) et le chiffrement TLS via certificats auto-signés ou une mini-CA interne.

#### 4.2.2. Base de données (PostgreSQL 16)

Une instance PostgreSQL 16 est déployée pour les besoins applicatifs des services internes. La configuration est optimisée pour un environnement contraint en mémoire. Les accès sont restreints aux VLANs autorisés via pg_hba.conf.

#### 4.2.3. Monitoring (VictoriaMetrics + Grafana)

Le monitoring centralisé repose sur VictoriaMetrics en remplacement de Prometheus, offrant une consommation mémoire réduite de 30 à 50% tout en maintenant une compatibilité PromQL complète. Grafana assure la visualisation via des dashboards préconfigurés. Node Exporter est installé sur chaque machine de l'infrastructure.

#### 4.2.4. Pipeline CI/CD

La chaîne d'intégration et de déploiement continu comprend trois composants : Forgejo (hébergement Git), Woodpecker CI (exécution des pipelines) et un Docker Registry privé (stockage des images). Forgejo est configuré en miroir bidirectionnel avec GitHub.

### 4.3. Sauvegarde et reprise d'activité

Une stratégie de sauvegarde complète est mise en place avec trois niveaux : sauvegarde des VMs/CTs via vzdump, sauvegarde des bases de données via pg_dump avec chiffrement GPG, et sauvegarde des fichiers de configuration critiques. Toutes les sauvegardes sont transférées vers le PC 2 (NAS) via rsync.

| Type de backup | Fréquence | Rétention | Chiffrement | Destination |
|----------------|-----------|-----------|-------------|-------------|
| VMs / CTs (vzdump) | Quotidien 02h00 | 7 daily + 4 weekly | Non (transport local) | PC 2 `/backup/vzdump/` |
| Bases de données | Quotidien 01h00 | 7 daily | GPG | PC 2 `/backup/databases/` |
| Configurations | Hebdomadaire dim. 03h00 | 4 weekly | Non | PC 2 `/backup/configs/` |

### 4.4. Sécurité

La sécurité est traitée en profondeur avec une approche de défense en couches : firewall périmétrique (pfSense), firewall local (UFW sur chaque machine), hardening SSH, détection d'intrusion (Fail2ban), authentification multi-facteurs sur Proxmox et accès distant via VPN (Tailscale).

### 4.5. Zone de pentest

Une machine virtuelle Kali Linux est disponible en VLAN 40 pour effectuer des tests de sécurité. Cette zone est totalement isolée des VLANs de production. La VM est éteinte par défaut et démarrée uniquement à la demande. Les résultats de scan nmap sont documentés comme preuve d'isolation.

---

## 5. Exigences techniques

### 5.1. Stack technologique retenue

| Composant | Technologie | Version | Justification |
|-----------|-------------|---------|---------------|
| Hyperviseur | Proxmox VE | 8.x (stable) | Référence open source, support LXC natif |
| Filesystem | ext4 | — | Un seul disque, ZFS inutile sans mirror |
| Firewall | pfSense CE | 2.7.x | Référence homelab, très documenté |
| Reverse Proxy | Nginx | Dernière stable | Léger, performant, très répandu |
| Base de données | PostgreSQL | 16 | Robuste, standard industrie |
| Monitoring TSDB | VictoriaMetrics | Dernière stable | Drop-in Prometheus, -30/50% RAM |
| Dashboarding | Grafana | Dernière stable | Standard industrie, riche en plugins |
| Git self-hosted | Forgejo | Dernière stable | Fork communautaire Gitea, actif |
| CI/CD | Woodpecker CI | Dernière stable | Fork open source Drone CI, actif |
| Registry | Docker Registry | v2 | Officiel, minimal |
| Automatisation | Ansible + Bash | Dernières stables | Standard DevOps, complémentaires |
| IaC (optionnel) | Terraform + bpg/proxmox | Dernière stable | Déclaratif, complément portfolio |
| VPN | Tailscale | Dernière stable | Mesh VPN, zero config NAT |
| Backup VM | vzdump + rsync | — | Natif Proxmox + transfert fiable |
| Backup BDD | pg_dump + GPG | — | Standard PostgreSQL + chiffrement |

### 5.2. Justification des choix techniques

Chaque choix technologique a été évalué au regard de la contrainte principale (8 Go de RAM), de la pérennité du projet open source, de la documentation disponible et de la pertinence pour un portfolio DevOps.

| Choix retenu | Alternative écartée | Raison de l'écart |
|--------------|---------------------|-------------------|
| Forgejo | Gitea | Fork communautaire avec gouvernance indépendante. Gitea est passé sous contrôle commercial. |
| Woodpecker CI | Drone CI | Fork open source actif. Drone est quasi-abandonné depuis le rachat par Harness. |
| VictoriaMetrics | Prometheus | Drop-in replacement, 30-50% moins gourmand en RAM/CPU, meilleure compression. |
| ext4 | ZFS | Un seul disque SSD. ZFS sans mirror n'apporte rien et consomme de la RAM (ARC). |
| pfSense CE | OPNsense / VyOS | pfSense reste la référence la plus documentée en homelab. |
| LXC natif | Docker partout | LXC est natif Proxmox, plus léger, meilleur contrôle réseau avec VLANs. |

### 5.3. Plan d'adressage IP

| Machine | VLAN | Adresse IP | Rôle |
|---------|------|------------|------|
| Proxmox VE (host) | 10 | 10.10.10.1 | Hyperviseur |
| NAS Debian (PC 2) | 10 | 10.10.10.2 | Serveur de backup |
| pfSense (WAN) | — | DHCP (internet) | Sortie internet |
| pfSense (Gateway VLAN 10) | 10 | 10.10.10.254 | Passerelle Management |
| pfSense (Gateway VLAN 20) | 20 | 10.10.20.254 | Passerelle Services |
| pfSense (Gateway VLAN 30) | 30 | 10.10.30.254 | Passerelle CI/CD |
| pfSense (Gateway VLAN 40) | 40 | 10.10.40.254 | Passerelle DMZ |
| CT Nginx | 20 | 10.10.20.10 | Reverse proxy |
| CT PostgreSQL | 20 | 10.10.20.20 | Base de données |
| CT Monitoring | 20 | 10.10.20.30 | VictoriaMetrics + Grafana |
| CT Forgejo | 30 | 10.10.30.10 | Git self-hosted |
| CT Woodpecker CI | 30 | 10.10.30.20 | CI/CD pipelines |
| CT Docker Registry | 30 | 10.10.30.30 | Registry Docker v2 |
| VM Kali Linux | 40 | 10.10.40.10 | Pentest (à la demande) |

---

## 6. Architecture cible

### 6.1. Schéma d'architecture logique

L'architecture repose sur un nœud Proxmox unique hébergeant une VM pfSense qui fait office de routeur et firewall central. Quatre VLANs segmentent le trafic. Le NAS (PC 2) est connecté au VLAN de management pour la réception des sauvegardes.

```
                        [INTERNET]
                            |
                    [PC 1 — Proxmox VE]
                            |
                [pfSense VM — Firewall/Router]
                    |               |
    ┌───────────────┼───────────────┼───────────────┐
    |               |               |               |
 VLAN 10         VLAN 20         VLAN 30         VLAN 40
 Management      Services        CI/CD           DMZ
 10.10.10.0/24   10.10.20.0/24   10.10.30.0/24   10.10.40.0/24
    |               |               |               |
 Proxmox UI      CT Nginx        CT Forgejo      VM Kali
 SSH             CT PostgreSQL   CT Woodpecker    (isolé)
 NAS (PC 2)      CT Monitoring   CT Registry
                        |
                   [Réseau local]
                        |
                [PC 2 — Debian NAS]
                   NFS / Samba
                Backup vzdump
                Backup pg_dump
                1 To stockage
```

> 📌 Un schéma détaillé sera produit au format Excalidraw ou draw.io et intégré sous `docs/diagrams/architecture-overview.png`.

### 6.2. Architecture des containers et VMs

| Machine | Type | OS | RAM | Disque | VLAN | Services |
|---------|------|----|-----|--------|------|----------|
| pfSense | VM (KVM) | FreeBSD (pfSense CE) | 512 Mo | 8 Go | WAN + Trunk | Firewall, DHCP, DNS |
| Nginx | CT LXC | Debian 12 | 256 Mo | 4 Go | 20 | Reverse proxy, TLS |
| PostgreSQL | CT LXC | Debian 12 | 512 Mo | 10 Go | 20 | BDD applicative |
| Monitoring | CT LXC | Debian 12 | 512 Mo | 15 Go | 20 | VictoriaMetrics, Grafana |
| Forgejo | CT LXC | Debian 12 | 384 Mo | 10 Go | 30 | Git, webhooks |
| Woodpecker | CT LXC | Debian 12 | 384 Mo | 10 Go | 30 | CI pipelines, Docker |
| Registry | CT LXC | Debian 12 | 256 Mo | 20 Go | 30 | Docker Registry v2 |
| Kali Linux | VM (KVM) | Kali 2024.x | 2 Go | 30 Go | 40 | Pentest, nmap, scans |

### 6.3. Architecture de sauvegarde

Les flux de sauvegarde sont unidirectionnels : du PC 1 (Proxmox) vers le PC 2 (NAS). Le transfert s'effectue via rsync sur le réseau local. Les sauvegardes de bases de données sont chiffrées avec GPG avant transfert. Une rotation automatique assure la gestion de l'espace disque sur le NAS.

Le monitoring des sauvegardes est assuré par des métriques exposées via Node Exporter (textfile collector), avec des alertes Grafana en cas de sauvegarde manquante ou d'espace disque insuffisant.

---

## 7. Plan de réalisation

Le projet est découpé en sept phases séquentielles, chacune ayant ses propres livrables et critères de validation. Chaque phase est un prérequis pour la suivante.

| Phase | Intitulé | Dépendance | Livrables principaux |
|-------|----------|------------|----------------------|
| 0 | Préparation hardware et installation | Aucune | Proxmox installé, NAS opérationnel, scripts post-install |
| 1 | Réseau et segmentation | Phase 0 | pfSense configuré, 4 VLANs actifs, règles firewall |
| 2 | Services (VLAN 20) | Phase 1 | Nginx, PostgreSQL, VictoriaMetrics + Grafana déployés |
| 3 | Pipeline CI/CD (VLAN 30) | Phase 1 | Forgejo, Woodpecker CI, Docker Registry opérationnels |
| 4 | Sécurité et hardening | Phases 2 + 3 | SSH durci, Fail2ban, UFW, MFA, Tailscale, audit Kali |
| 5 | Automatisation et IaC | Phases 2 + 3 | Ansible playbooks, scripts Bash, Terraform (optionnel) |
| 6 | Backup et Disaster Recovery | Phases 2 + 3 | Backups automatisés, procédures DR testées, RTO/RPO mesurés |

### 7.1. Phase 0 — Préparation hardware et installation

**PC 1 — Proxmox VE :**

1. Vérification et configuration du BIOS (VT-x, VT-d, désactivation Secure Boot)
2. Création de la clé USB bootable (ISO Proxmox VE 8.x)
3. Installation Proxmox VE avec ext4, partitionnement adapté, hostname `pve-lab.local`
4. Configuration post-installation (repos, mise à jour, swap, swappiness)
5. Configuration réseau bridge `vmbr0` avec VLAN-aware
6. Sécurisation initiale (clés SSH, Fail2ban)

**PC 2 — NAS Debian :**

1. Installation Debian 12 minimal (headless)
2. Installation des paquets essentiels (NFS, rsync, GPG, SMART, Fail2ban, UFW)
3. Création de l'arborescence `/backup/` et configuration des exports NFS
4. Configuration UFW et hardening SSH
5. Montage NFS dans Proxmox (storage backup)

**Livrables :** `docs/00-hardware-setup.md`, `scripts/00-proxmox-postinstall.sh`, `scripts/00-nas-setup.sh`

### 7.2. Phase 1 — Réseau et segmentation (pfSense)

1. Configuration du bridge VLAN-aware sur Proxmox (VIDs 10, 20, 30, 40)
2. Création et installation de la VM pfSense (512 Mo RAM, 8 Go disque)
3. Configuration des 4 VLANs dans pfSense avec passerelles dédiées
4. Mise en place des règles firewall inter-VLAN selon la matrice de flux
5. Activation des services DHCP, DNS Resolver (Unbound) et NAT
6. Tests de connectivité et validation de l'isolation inter-VLAN

**Livrables :** `docs/01-network-architecture.md`, `docs/diagrams/network-topology.png`, `pfsense/firewall-rules-export.xml`

### 7.3. Phase 2 — Services (VLAN 20)

1. Déploiement du CT Nginx (reverse proxy, certificats TLS)
2. Déploiement du CT PostgreSQL 16 (configuration sécurisée, tuning mémoire)
3. Déploiement du CT Monitoring (VictoriaMetrics + Grafana + Node Exporter)
4. Configuration des dashboards Grafana et des alertes
5. Tests de connectivité inter-services

**Livrables :** `ansible/roles/webserver/`, `ansible/roles/database/`, `ansible/roles/monitoring/`, `docs/02-services-deployment.md`

### 7.4. Phase 3 — Pipeline CI/CD (VLAN 30)

1. Déploiement du CT Forgejo (binaire, SQLite, miroir GitHub bidirectionnel)
2. Déploiement du CT Woodpecker CI (Server + Agent via Docker)
3. Déploiement du CT Docker Registry v2 (authentification htpasswd)
4. Configuration du webhook Forgejo vers Woodpecker CI
5. Création du pipeline de démonstration (`.woodpecker.yml`)
6. Test complet du cycle push → build → push image → notification

**Livrables :** `ansible/roles/cicd/`, `.woodpecker.yml`, `docker-compose/`, `docs/03-cicd-pipeline.md`

### 7.5. Phase 4 — Sécurité et hardening

1. Hardening SSH sur toutes les machines (port custom, clés uniquement, restrictions)
2. Déploiement de Fail2ban avec jails personnalisés (SSH, Proxmox UI, Forgejo)
3. Configuration UFW en double couche (pfSense périmètre + UFW local)
4. Installation et configuration de Tailscale (accès distant sécurisé)
5. Activation MFA (TOTP) sur Proxmox Web UI
6. Configuration de la centralisation des logs (rsyslog)
7. Déploiement et test de la VM Kali Linux (VLAN 40), scans d'isolation

**Livrables :** `ansible/roles/hardening/`, `docs/04-security-audit.md` (avec screenshots scans Kali)

### 7.6. Phase 5 — Automatisation et Infrastructure as Code

1. Écriture des scripts Bash (post-install, NAS setup, backups, health check)
2. Structuration de l'inventaire Ansible par VLAN avec group_vars
3. Développement des rôles Ansible réutilisables (common, hardening, webserver, database, monitoring, cicd, backup)
4. Création du playbook orchestrateur `site.yml`
5. Mise en place d'Ansible Vault pour les secrets
6. *(Optionnel)* Définition des ressources Terraform avec le provider bpg/proxmox

**Livrables :** `scripts/`, `ansible/` (complet), `terraform/` (optionnel), `docs/05-automation-iac.md`

### 7.7. Phase 6 — Backup et Disaster Recovery

1. Configuration des backups vzdump planifiés vers le NAS (PC 2)
2. Mise en place des backups bases de données chiffrés GPG
3. Mise en place des backups de configuration hebdomadaires
4. Exposition des métriques de backup via Node Exporter textfile
5. Configuration des alertes Grafana sur les backups
6. Test de restauration complet (CT + BDD) avec mesure du RTO et RPO
7. Documentation du plan de reprise d'activité

**Livrables :** `scripts/backup/`, `docs/06-backup-dr.md` (procédures DR, tableau RTO/RPO)

---

## 8. Livrables attendus

### 8.1. Dépôt GitHub

L'ensemble du projet est versionné sur un dépôt GitHub public structuré selon les bonnes pratiques DevOps. Le dépôt constitue le livrable principal du projet et sert de vitrine portfolio.

### 8.2. Documentation technique

| Document | Contenu |
|----------|---------|
| `00-hardware-setup.md` | Installation Proxmox et NAS, configuration hardware, post-installation |
| `01-network-architecture.md` | Schéma réseau, configuration VLANs, règles firewall pfSense |
| `02-services-deployment.md` | Déploiement Nginx, PostgreSQL, VictoriaMetrics, Grafana |
| `03-cicd-pipeline.md` | Forgejo, Woodpecker CI, Docker Registry, pipeline de démonstration |
| `04-security-audit.md` | Hardening SSH, Fail2ban, UFW, Tailscale, MFA, scans Kali |
| `05-automation-iac.md` | Scripts Bash, rôles Ansible, Terraform, Ansible Vault |
| `06-backup-dr.md` | Procédures de backup, tests de restauration, tableau RTO/RPO |
| `DECISIONS.md` | Architecture Decision Records (ADR) — justification de chaque choix technique |

### 8.3. Code d'automatisation

- Scripts Bash : post-installation, setup NAS, backups, health check
- Rôles Ansible : common, hardening, webserver, database, monitoring, cicd, backup
- Playbooks Ansible : orchestration complète de l'infrastructure
- Docker Compose : fichiers de déploiement alternatifs pour chaque service
- Terraform (optionnel) : définition déclarative des VMs et CTs Proxmox
- Pipeline CI/CD : fichier `.woodpecker.yml` de référence

### 8.4. Diagrammes

- Schéma de topologie réseau (draw.io ou Excalidraw)
- Schéma d'architecture globale
- Diagramme du budget RAM
- Screenshots : dashboards Grafana, Proxmox, pfSense, pipelines Woodpecker, scans Kali

---

## 9. Critères de validation

Chaque phase du projet est considérée comme validée lorsque l'ensemble des critères suivants sont satisfaits.

| Phase | Critères de validation (Definition of Done) |
|-------|----------------------------------------------|
| **Phase 0** | Proxmox VE accessible via Web UI. NAS joignable en SSH. NFS monté sur Proxmox. Scripts post-install exécutables sans erreur. |
| **Phase 1** | 4 VLANs actifs avec DHCP. Ping inter-VLAN conforme à la matrice de flux. VLAN 40 isolée (aucune réponse vers VLANs 10/20/30). DNS résolu via pfSense. |
| **Phase 2** | Nginx route correctement vers les services (via noms d'hôte). PostgreSQL accepte les connexions depuis les VLANs autorisés uniquement. Grafana affiche les métriques de tous les nœuds. |
| **Phase 3** | Push Git sur Forgejo déclenche un pipeline Woodpecker. L'image Docker est construite et poussée vers le Registry. Le miroir GitHub fonctionne. |
| **Phase 4** | Authentification SSH par mot de passe refusée. Fail2ban bannit après 3 tentatives. Tailscale permet l'accès distant. Scans nmap depuis Kali confirment l'isolation VLAN 40. |
| **Phase 5** | `ansible-playbook site.yml` déploie l'infrastructure complète sans erreur. Les scripts Bash s'exécutent correctement. Les secrets sont chiffrés via Vault. |
| **Phase 6** | Restauration d'un CT depuis backup vzdump réussie. Restauration d'une BDD depuis backup GPG réussie. RTO et RPO mesurés et documentés. Alertes Grafana fonctionnelles. |

---

## 10. Gestion des risques

| Risque identifié | Probabilité | Impact | Mesure de mitigation |
|------------------|-------------|--------|----------------------|
| RAM insuffisante pour faire tourner tous les services | Moyenne | Élevé | Budget RAM strict, Kali éteinte par défaut, swap 4 Go, fusion possible Forgejo+Woodpecker |
| Panne du SSD (PC 1) | Faible | Critique | Backups quotidiens sur PC 2, IaC pour reconstruire depuis zéro |
| Panne du HDD (PC 2 / NAS) | Moyenne | Élevé | Monitoring SMART, alertes préventives, rotation des backups |
| Corruption des backups chiffrés (perte clé GPG) | Faible | Critique | Stockage de la clé GPG privée hors du lab (machine perso, gestionnaire MDP) |
| Complexité excessive retardant le projet | Moyenne | Moyen | Phasage strict, phases indépendantes après Phase 1, priorisation des livrables |
| Incompatibilité matérielle avec Proxmox | Faible | Élevé | VT-x/VT-d vérifiés, hardware mainstream Intel bien supporté |
| Services trop gourmands en RAM en situation réelle | Moyenne | Moyen | Monitoring VictoriaMetrics pour détecter les dérives, tuning progressif |

---

## 11. Glossaire

| Terme | Définition |
|-------|------------|
| **CT** (Container) | Container LXC, forme de virtualisation légère native de Proxmox sans kernel dédié |
| **VM** (Virtual Machine) | Machine virtuelle complète avec son propre noyau, gérée par KVM |
| **VLAN** | Virtual Local Area Network — segmentation logique d'un réseau physique |
| **LXC** | Linux Containers — technologie de conteneurisation au niveau du système d'exploitation |
| **IaC** | Infrastructure as Code — gestion de l'infrastructure via des fichiers déclaratifs versionnés |
| **CI/CD** | Continuous Integration / Continuous Delivery — automatisation du cycle de développement |
| **RTO** | Recovery Time Objective — durée maximale acceptable pour restaurer un service |
| **RPO** | Recovery Point Objective — quantité maximale de données pouvant être perdue |
| **vzdump** | Outil natif Proxmox pour la sauvegarde des VMs et containers |
| **ADR** | Architecture Decision Record — documentation formelle des choix d'architecture |
| **TSDB** | Time Series Database — base de données optimisée pour les métriques temporelles |
| **PromQL** | Prometheus Query Language — langage de requête pour les métriques Prometheus/VictoriaMetrics |
| **GPG** | GNU Privacy Guard — outil de chiffrement et signature de données |
| **MFA** | Multi-Factor Authentication — authentification à plusieurs facteurs |

---

*Document rédigé le 10 février 2026 — Version 3.0*