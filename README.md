# 🔒 Proxmox Secure Lab

> Environnement de test sécurisé et segmenté sur Proxmox VE, déployé sur deux laptops recyclés — Infrastructure as Code.

---

## 🎯 Objectif

Concevoir, déployer et documenter un **homelab professionnel** simulant un environnement de production avec segmentation réseau, CI/CD, monitoring, hardening sécurité et disaster recovery.

Ce projet sert à la fois de **plateforme d'apprentissage** et de **projet portfolio** dans le cadre d'une recherche d'alternance en Administration Système & DevOps.

---

## 💻 Hardware

| Machine | CPU | RAM | Stockage | Rôle |
|---------|-----|-----|----------|------|
| **PC 1** | Intel i5-8265U (4C/8T) | 8 Go DDR4 | 250 Go SSD | Proxmox VE — Hyperviseur |
| **PC 2** | Intel i5-5200U (2C/4T) | 8 Go DDR3L | 1 To HDD | Debian 13 — NAS / Backup |

---

## 🏗️ Architecture

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
                 CT Monitoring   CT Registry
                        |
                   [Réseau local]
                        |
                [PC 2 — Debian NAS]
                   NFS / Samba
                Backup vzdump
                Backup pg_dump
```

| VLAN | Réseau | Rôle | Services |
|------|--------|------|----------|
| 10 | 10.10.10.0/24 | Management | Proxmox UI, SSH, NAS |
| 20 | 10.10.20.0/24 | Services | Nginx, PostgreSQL, Grafana |
| 30 | 10.10.30.0/24 | CI/CD | Forgejo, Woodpecker CI, Registry |
| 40 | 10.10.40.0/24 | DMZ | Kali Linux (isolé) |

---

## 🛠️ Stack technique

| Catégorie | Technologie |
|-----------|-------------|
| **Hyperviseur** | Proxmox VE 9.x (ext4) |
| **Firewall** | pfSense CE (VM, 512 Mo) |
| **Automatisation** | Ansible + Bash + Terraform (optionnel) |
| **CI/CD** | Forgejo + Woodpecker CI + Docker Registry v2 |
| **Monitoring** | VictoriaMetrics + Grafana + Node Exporter |
| **Backup** | vzdump → rsync chiffré GPG → NAS (PC 2) |
| **Sécurité** | VLANs, pfSense, Fail2ban, UFW, Tailscale, MFA |
| **Outils IA** | Claude Code, Claude.ai, Claude for VS Code |

---

## ⚡ Optimisation 8 Go RAM

Architecture optimisée avec **7 containers LXC + 2 VMs** pour tenir dans 8 Go :

| Composant | Type | RAM |
|-----------|------|-----|
| Proxmox VE (host) | Host | ~1 Go |
| pfSense | VM | 512 Mo |
| Nginx | CT LXC | 256 Mo |
| PostgreSQL | CT LXC | 512 Mo |
| VictoriaMetrics + Grafana | CT LXC | 512 Mo |
| Forgejo | CT LXC | 384 Mo |
| Woodpecker CI | CT LXC | 384 Mo |
| Docker Registry | CT LXC | 256 Mo |
| Kali Linux | VM | 2 Go *(éteint par défaut)* |
| **Total sans Kali** | | **~3,8 Go** |
| **Marge libre** | | **~2,2 Go** |

Détail complet : voir [docs/00-hardware-setup.md](docs/00-hardware-setup.md)

---

## 🚀 Déploiement rapide

```bash
# 1. Post-installation Proxmox
bash scripts/00-proxmox-postinstall.sh

# 2. Setup NAS (PC 2)
bash scripts/00-nas-setup.sh

# 3. Déployer toute l'infrastructure
cd ansible/
ansible-playbook -i inventory/hosts.ini site.yml --ask-vault-pass
```

---

## 📚 Documentation

| Phase | Document | Contenu |
|-------|----------|---------|
| 0 | [Hardware Setup](docs/00-hardware-setup.md) | Installation Proxmox + NAS |
| 1 | [Network](docs/01-network-architecture.md) | VLANs, pfSense, firewall rules |
| 2 | [Services](docs/02-services-deployment.md) | Nginx, PostgreSQL, Monitoring |
| 3 | [CI/CD](docs/03-cicd-pipeline.md) | Forgejo, Woodpecker, Registry |
| 4 | [Security](docs/04-security-audit.md) | Hardening, scans, isolation |
| 5 | [Automation](docs/05-automation-iac.md) | Ansible, scripts, Terraform |
| 6 | [Backup & DR](docs/06-backup-dr.md) | Procédures, RTO/RPO, tests |

---

## 📐 Architecture Decision Records

Les choix techniques sont documentés et justifiés dans [docs/DECISIONS.md](docs/DECISIONS.md) :

- **ADR-001** : Utilisation de l'IA comme outil de développement
- **ADR-002** : Forgejo plutôt que Gitea
- **ADR-003** : VictoriaMetrics plutôt que Prometheus
- **ADR-004** : Woodpecker CI plutôt que Drone CI
- **ADR-005** : ext4 plutôt que ZFS
- **ADR-006** : LXC natif plutôt que Docker partout

---

## 🤖 Transparence IA

Ce projet utilise des outils d'IA (Claude Code, Claude.ai, Claude for VS Code) comme assistants de développement. Chaque décision technique, ligne de code et choix d'architecture reste sous le contrôle et la validation de l'auteur. Cette approche est documentée de manière transparente dans [ADR-001](docs/DECISIONS.md#adr-001--utilisation-de-lia-comme-outil-de-développement).

---

## 📦 Structure du projet

```
proxmox-secure-lab/
├── README.md
├── LICENSE (MIT)
├── CHANGELOG.md
├── .gitignore
├── docs/
│   ├── 00-hardware-setup.md
│   ├── 01-network-architecture.md
│   ├── 02-services-deployment.md
│   ├── 03-cicd-pipeline.md
│   ├── 04-security-audit.md
│   ├── 05-automation-iac.md
│   ├── 06-backup-dr.md
│   ├── DECISIONS.md
│   └── diagrams/
├── scripts/
│   ├── 00-proxmox-postinstall.sh
│   ├── 00-nas-setup.sh
│   ├── backup/
│   └── health-check.sh
├── ansible/
│   ├── inventory/
│   ├── playbooks/
│   ├── roles/
│   └── site.yml
├── docker-compose/
├── terraform/ (optionnel)
├── pfsense/
└── .woodpecker.yml
```

---

## 🏷️ Versioning

| Tag | Étape |
|-----|-------|
| `v0.0-init` | Structure du projet, CDC, README |
| `v0.1-network` | Phase 1 — Réseau & pfSense |
| `v0.2-services` | Phase 2 — Services VLAN 20 |
| `v0.3-cicd` | Phase 3 — Pipeline CI/CD |
| `v0.4-security` | Phase 4 — Hardening |
| `v0.5-automation` | Phase 5 — IaC complète |
| `v1.0-complete` | Toutes les phases validées |

---

## 👤 Auteur

**Tom Daluzeau** — Alternant Administrateur Systèmes & DevOps

- LinkedIn : [Tom Daluzeau](https://www.linkedin.com/in/daluzeautom/)

---

## 📄 Licence

Ce projet est sous licence [MIT](LICENSE).
