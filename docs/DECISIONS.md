# 📐 Architecture Decision Records (ADR)

> Ce document recense les décisions techniques majeures prises au cours du projet Proxmox Secure Lab. Chaque ADR explique le contexte, les alternatives évaluées, la décision retenue et ses conséquences.
>
> Format inspiré de [Michael Nygard's ADR template](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions).

---

## ADR-001 — Utilisation de l'IA comme outil de développement

**Date** : 10 février 2026
**Statut** : Accepté

### Contexte

Les outils d'intelligence artificielle générative ont atteint un niveau de maturité qui en fait des assistants de développement crédibles. Dans le cadre d'un projet portfolio destiné à démontrer des compétences DevOps, la question se pose de l'intégration de ces outils dans le workflow de travail et de la transparence à adopter vis-à-vis de leur utilisation.

### Outils retenus

| Outil | Usage principal | Contexte d'utilisation |
|-------|----------------|----------------------|
| **Claude.ai** (chat web) | Conception, brainstorming, rédaction | Phase de réflexion et de planification : rédaction du cahier des charges, choix d'architecture, structuration du projet, résolution de problèmes complexes. |
| **Claude for VS Code** (extension) | Assistance au code en contexte | Développement quotidien : autocomplétion intelligente, explication de code, refactoring, aide à la rédaction de playbooks Ansible et scripts Bash, documentation inline. |
| **Claude Code** (CLI) | Automatisation et tâches agentic | Tâches d'envergure depuis le terminal : génération de fichiers de configuration, scaffolding de rôles Ansible, debugging d'erreurs complexes, revue de code automatisée. |

### Philosophie d'utilisation

L'IA est utilisée comme un **multiplicateur de productivité**, pas comme un substitut à la compréhension. Les principes suivants encadrent son utilisation :

1. **Comprendre avant d'appliquer** — Chaque suggestion de l'IA est lue, comprise et validée avant d'être intégrée. Le code copié-collé sans compréhension est interdit.
2. **L'humain décide** — Les choix d'architecture, les arbitrages techniques et les compromis restent des décisions humaines. L'IA propose, l'auteur dispose.
3. **Transparence totale** — L'utilisation de l'IA est documentée ouvertement (ce document, section README). Aucune tentative de masquer l'assistance IA.
4. **Validation systématique** — Tout code généré est testé, relu et adapté au contexte spécifique du projet. L'IA ne connaît pas l'état réel de l'infrastructure.
5. **Apprentissage actif** — L'IA est aussi utilisée comme outil pédagogique : demander des explications, explorer des alternatives, comprendre les implications d'un choix.

### Workflow concret

```
Phase de conception     →  Claude.ai (chat)
  Brainstorming, CDC, architecture, ADRs

Phase de développement  →  Claude for VS Code (extension)
  Écriture de code, playbooks, scripts, docs

Tâches d'automatisation →  Claude Code (CLI)
  Scaffolding, génération batch, debugging

Validation              →  Humain (toujours)
  Test, relecture, adaptation, décision finale
```

### Alternatives envisagées

| Alternative | Raison de l'écart |
|-------------|------------------|
| Ne pas utiliser d'IA | Contre-productif en 2026. L'IA est un outil standard dans l'industrie. Ne pas l'utiliser serait comme refuser d'utiliser Stack Overflow en 2015. |
| GitHub Copilot | Bonne alternative, mais l'écosystème Claude (chat + extension + CLI) offre une cohérence d'expérience et une qualité de raisonnement supérieure pour les tâches de conception et d'architecture. |
| Utiliser l'IA sans le documenter | Manque de transparence inacceptable dans un contexte portfolio. La valeur du projet réside aussi dans la capacité à expliquer ses choix et sa méthode de travail. |

### Conséquences

**Positives :**
- Productivité accrue sur la documentation, le scaffolding et le debugging
- Meilleure qualité de code grâce à la revue assistée
- Exploration plus large des alternatives techniques
- Démonstration d'une compétence recherchée en entreprise (savoir utiliser l'IA efficacement)

**Négatives / Risques :**
- Risque de dépendance si l'on ne vérifie pas systématiquement la compréhension
- Les suggestions IA peuvent être incorrectes ou inadaptées au contexte spécifique
- Nécessité de toujours valider contre la documentation officielle

**Mitigation :**
- Chaque bloc de code généré par l'IA est testé en conditions réelles sur l'infrastructure
- Les choix d'architecture sont documentés avec leur raisonnement (ces ADRs)
- Le journal de bord du projet (CHANGELOG) retrace l'évolution réelle du travail

---

## ADR-002 — Forgejo plutôt que Gitea

**Date** : 10 février 2026
**Statut** : Accepté

### Contexte

Le projet nécessite un serveur Git self-hosted léger pour héberger le code et déclencher les pipelines CI/CD. Gitea et Forgejo sont les deux principales options dans cette catégorie.

### Décision

Forgejo est retenu comme serveur Git self-hosted.

### Justification

Forgejo est un fork communautaire de Gitea créé fin 2022 suite au rachat de Gitea par une entité commerciale (Gitea Ltd). Depuis 2023, Forgejo a :
- une gouvernance indépendante et transparente (sous l'égide de Codeberg e.V.)
- un rythme de développement actif avec des fonctionnalités propres
- une compatibilité descendante avec Gitea (migration simple)
- une philosophie résolument open source sans version "Enterprise" fermée

Gitea reste fonctionnel mais son orientation commerciale crée une incertitude sur la pérennité de la version communautaire.

### Conséquences

- API et webhooks compatibles avec l'écosystème Gitea existant
- Woodpecker CI supporte nativement Forgejo
- Configuration avec SQLite pour économiser la RAM (pas besoin d'un PostgreSQL dédié)
- Migration vers Gitea possible si nécessaire (fork, donc compatibilité)

---

## ADR-003 — VictoriaMetrics plutôt que Prometheus

**Date** : 10 février 2026
**Statut** : Accepté

### Contexte

Le monitoring de l'infrastructure nécessite une base de données de métriques temporelles (TSDB). Prometheus est le standard de fait, mais la contrainte de 8 Go de RAM impose d'évaluer des alternatives plus légères.

### Décision

VictoriaMetrics (single-node) est retenu en remplacement de Prometheus.

### Justification

- **Drop-in replacement** : VictoriaMetrics expose une API 100% compatible Prometheus. Les dashboards Grafana, les règles d'alerte et les requêtes PromQL fonctionnent sans modification.
- **Consommation mémoire** : 30 à 50% de RAM en moins par rapport à Prometheus pour un volume de métriques équivalent.
- **Meilleure compression** : les données stockées occupent moins d'espace disque, pertinent sur un SSD de 250 Go.
- **Simplicité** : un seul binaire à déployer, pas de dépendances externes.

### Chiffres concrets

| Métrique | Prometheus | VictoriaMetrics | Gain |
|----------|-----------|-----------------|------|
| RAM estimée (homelab) | ~768 Mo | ~400-512 Mo | ~256 Mo |
| Binaire | ~90 Mo | ~15 Mo | -83% |
| Compression données | ~1.3 bytes/sample | ~0.7 bytes/sample | ~46% |

### Conséquences

- Le CT Monitoring est dimensionné à 512 Mo (VictoriaMetrics + Grafana + Node Exporter)
- Tous les dashboards Grafana communautaires (Node Exporter Full, PostgreSQL) fonctionnent tels quels
- La marge RAM globale passe de 1,6 Go à 2,2 Go grâce à ce choix

---

## ADR-004 — Woodpecker CI plutôt que Drone CI

**Date** : 10 février 2026
**Statut** : Accepté

### Contexte

Le projet nécessite un moteur CI/CD léger, compatible avec Forgejo, capable de construire des images Docker et d'exécuter des pipelines définis en YAML.

### Décision

Woodpecker CI est retenu comme moteur CI/CD.

### Justification

Woodpecker CI est un fork open source de Drone CI créé suite au rachat de Drone par Harness en 2020. Depuis :
- Drone CI n'a pratiquement plus de mises à jour communautaires
- Woodpecker CI est activement développé avec des releases régulières
- La syntaxe YAML des pipelines est compatible (migration simple depuis Drone)
- Le support natif de Forgejo est intégré
- L'architecture Server + Agent est légère et adaptée à un homelab

### Conséquences

- Pipeline YAML compatible avec la syntaxe Drone existante
- Docker-in-Docker requis dans le CT (activation des features nesting + keyctl)
- Consommation mémoire estimée à ~384 Mo pour le Server + Agent
- Webhook natif depuis Forgejo, configuration straightforward

---

## ADR-005 — ext4 plutôt que ZFS

**Date** : 10 février 2026
**Statut** : Accepté

### Contexte

Proxmox VE propose plusieurs filesystems à l'installation : ext4, ZFS et XFS. Le choix du filesystem impacte les performances, la consommation de RAM et les fonctionnalités disponibles.

### Décision

ext4 est retenu comme filesystem pour le nœud Proxmox.

### Justification

Le PC 1 dispose d'un seul SSD de 250 Go. ZFS apporte des fonctionnalités avancées (snapshots, checksums, compression, RAID-Z) qui n'ont de sens que dans un contexte multi-disques :
- **Pas de mirror possible** avec un seul disque — la protection contre la corruption de données est limitée
- **L'ARC (Adaptive Replacement Cache)** de ZFS consomme de la RAM qui est notre ressource la plus contrainte
- **ext4** est éprouvé, stable, et n'a aucun overhead mémoire significatif

### Conséquences

- Pas de snapshots natifs ZFS (compensé par vzdump pour les backups)
- Pas de compression transparente (impact négligeable sur un SSD)
- Économie de ~500 Mo à 1 Go de RAM (ARC ZFS) disponible pour les VMs et CTs

---

## ADR-006 — LXC natif plutôt que Docker partout

**Date** : 10 février 2026
**Statut** : Accepté

### Contexte

Proxmox VE supporte nativement deux types de virtualisation : les VMs KVM et les containers LXC. Docker est également une option pour déployer des services. Le choix entre ces technologies impacte la consommation de ressources, la gestion réseau et la complexité opérationnelle.

### Décision

Les containers LXC natifs Proxmox sont utilisés en priorité pour tous les services. Docker est utilisé uniquement à l'intérieur des CTs qui en ont besoin (Woodpecker CI, Docker Registry).

### Justification

- **LXC est natif Proxmox** : gestion intégrée via l'UI et l'API, snapshots, backups vzdump, migration
- **10 à 50 fois plus léger qu'une VM** : pas de kernel dédié, partage du kernel host
- **Meilleur contrôle réseau** : intégration native avec les VLANs Proxmox, configuration réseau par CT
- **Docker dans LXC** : quand Docker est nécessaire (Woodpecker, Registry), il tourne dans un CT avec les features nesting et keyctl activées — le meilleur des deux mondes

### Conséquences

- Chaque service est isolé dans son propre CT avec son propre réseau
- Les VLANs sont gérés au niveau Proxmox, pas au niveau Docker (plus propre)
- Seules 2 VMs sont nécessaires : pfSense (FreeBSD, pas de support LXC) et Kali (kernel complet nécessaire)
- Docker est confiné aux CTs qui en ont explicitement besoin

---

## ADR-007 — Exegol (Docker) au lieu de Kali Linux (VM)

**Date** : 12 février 2026
**Statut** : Accepté

### Contexte

L'architecture initiale prévoyait une VM Kali Linux (2 Go RAM) isolée sur le VLAN 40 (DMZ) pour les tests de sécurité. Avec seulement 8 Go de RAM sur le PC 1, ces 2 Go représentent 25% des ressources totales pour un outil utilisé ponctuellement.

### Décision

Remplacer la VM Kali par Exegol, un environnement de pentest français basé sur Docker, lancé à la demande depuis un container LXC dédié.

### Alternatives considérées

| Alternative | Description | Raison de l'écart |
|-------------|-------------|-------------------|
| Kali Linux VM | 2 Go RAM, noyau complet, VLAN 40 isolé | Isolation parfaite mais trop gourmand en RAM pour un usage ponctuel |
| Exegol Docker | Quelques centaines de Mo, images spécialisées : light/full/web/AD/OSINT | Léger, démarrage rapide, outils identiques, projet open source français actif |
| Outils installés manuellement | nmap, nikto, etc. sur un LXC | Pas de cohérence, maintenance difficile |

### Conséquences

**Gains :**
- ~1.5 à 2 Go de RAM libérés, suppression du VLAN 40 dédié
- Images spécialisées (web, AD, OSINT) au lieu d'un OS monolithique
- Reproductibilité totale (`docker pull` = même environnement à chaque fois)

**Compromis :**
- L'isolation réseau doit être gérée via la configuration Docker (`--network`) au lieu d'un VLAN physique dédié. **Mitigation** : créer un réseau Docker isolé avec des règles iptables strictes
- Docker doit être installé sur le lab (dépendance supplémentaire)

**Risques :**
- Certains outils bas niveau (injection de paquets, scan ARP) peuvent nécessiter `--privileged` ou `--net=host`, ce qui réduit l'isolation

---

## Index des ADR

| ID | Titre | Statut | Date |
|----|-------|--------|------|
| ADR-001 | Utilisation de l'IA comme outil de développement | ✅ Accepté | 2026-02-10 |
| ADR-002 | Forgejo plutôt que Gitea | ✅ Accepté | 2026-02-10 |
| ADR-003 | VictoriaMetrics plutôt que Prometheus | ✅ Accepté | 2026-02-10 |
| ADR-004 | Woodpecker CI plutôt que Drone CI | ✅ Accepté | 2026-02-10 |
| ADR-005 | ext4 plutôt que ZFS | ✅ Accepté | 2026-02-10 |
| ADR-006 | LXC natif plutôt que Docker partout | ✅ Accepté | 2026-02-10 |
| ADR-007 | Exegol (Docker) au lieu de Kali Linux (VM) | ✅ Accepté | 2026-02-12 |
