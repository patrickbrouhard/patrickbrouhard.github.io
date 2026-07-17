---
title: "Infrastructure automatisée Proxmox"
date: 2025-12-22
summary: "Projet homelab qui automatise la création d’un template Ubuntu cloud-init sur Proxmox, le déploiement d’une VM via Terraform, puis sa configuration via Ansible (Docker, maintenance, notifications Telegram)."
tags: ["devops", "homelab", "proxmox", "terraform", "ansible", "docker", "bash", "iac", "github-actions", "security"]
cover: 
  image: "/images/homelab-proxmox-cover.png"
  alt: "Homelab : Proxmox + Terraform + Ansible"
  caption: "Proxmox + Terraform + Ansible : automatiser la création et la configuration d’une VM"
draft: false
---

{{< mermaid >}}
flowchart TD
    A[Template Ubuntu Cloud-Init]
    B[Terraform<br/>Provisionnement VM]
    C[Inventaire dynamique<br/>hosts.ini]
    D[Ansible<br/>Configuration système]
    E[Docker<br/>Prêt à l'emploi]

    A -->|Clone le template| B
    B -->|Génère| C
    C -->|Utilisé par| D
    D -->|Installe et configure| E
{{< /mermaid >}}

## Contexte

J'ai lancé ce projet afin de pouvoir automatiser une partie de mon homelab autour de **Proxmox VE** : au lieu de créer une VM à la main, je veux pouvoir **reproduire** les étapes de bout en bout :

1) créer un **template Proxmox** à partir d’une image cloud Ubuntu (cloud-init)
2) cloner ce template et créer une VM via **Terraform** (provider Proxmox)
3) configurer la VM via **Ansible** (bootstrap, installation Docker, maintenance).

Il s'agissait pour moi de découvrir les outils DevOps, faire évoluer mon homelab et aussi préparer un futur projet lié au projet [MediatekDocuments](https://github.com/patrickbrouhard/mediatekdocuments) (le dépôt a d'ailleurs été récemment mis à jour en ce sens). 
Ce projet me sert de terrain d’expérimentation “réaliste” : des scripts et de l’IaC que je peux relancer, corriger et faire évoluer, en gardant une séparation claire entre **provisionnement** et **configuration**.

{{< button href="https://github.com/patrickbrouhard/proxmox-web-vm" target="_blank" rel="noopener" >}}
<svg xmlns="http://www.w3.org/2000/svg" 
     class="w-5 h-5 mr-2 inline-block" 
     fill="currentColor" viewBox="0 0 16 16">
<path d="M8 0C3.58 0 0 3.58 0 8a8 
           8 0 0 0 5.47 7.59c.4.07.55-.17.55-.38 
           0-.19-.01-.82-.01-1.49-2 
           .37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13
           -.28-.15-.68-.52-.01-.53.63-.01 1.08.58 
           1.23.82.72 1.21 1.87.87 
           2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95
           0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12
           0 0 .67-.21 2.2.82a7.65 7.65 0 0 1 2-.27c.68 
           0 1.36.09 2 .27 1.53-1.04 2.2-.82 
           2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 
           1.27.82 2.15 0 3.07-1.87 3.75-3.65 
           3.95.29.25.54.73.54 1.48 
           0 1.07-.01 1.93-.01 2.2 
           0 .21.15.46.55.38A8 8 0 0 0 16 
           8c0-4.42-3.58-8-8-8z"/>
</svg>
Voir sur GitHub
{{< /button >}}

---

## Objectifs

- **Gagner du temps** sur les déploiements de VM en évitant les opérations répétitives dans l’UI Proxmox. D'ailleurs, le template reste en place en permanence dans Proxmox pour pouvoir être utilisé manuellement en quelques clics, si besoin.
- **Séparer** :
  - la création / gestion des VMs (**Terraform**),
  - la configuration système et applicative (**Ansible**).
- Créer des rôles Ansible **réutilisables** (ex : rôle Docker, rôle notification Telegram).
- Mettre en place un minimum de garde-fous :
  - côté dépôt : scans GitHub Actions (secrets + IaC),
  - côté infra : attention aux tâches destructrices (maintenance Docker), et éviter certains scénarios de lock-out (sudo).

---

## Ce que fait le projet

### 1) Création d’un template Proxmox Ubuntu cloud-init (script)

Le script [`scripts/create-proxmox-cloud-template.sh`](https://github.com/patrickbrouhard/proxmox-web-vm/blob/master/scripts/create-proxmox-cloud-template.sh) automatise la création d’un template Proxmox à partir d’une image Ubuntu cloud :

- téléchargement + vérification SHA256 ;
- customization (`virt-customize`) + nettoyage (`virt-sysprep`)
- création de VM Proxmox avec `qm`, ajout cloud-init, EFI/OVMF, réseau
- conversion de la VM en template.

Le script est prévu pour être lancé directement sur un hôte Proxmox. (il vérifie notamment la présence de `qm`, `pvesm`, etc.) et inclut un verrou (`/var/lock/create-template.lock`) pour éviter les exécutions concurrentes.
Dans les deux cas, j'ai retrouvé des concepts classiques en développement logiciel : fail-fast (on sort le plus vite possible sous certaines conditions) et mutex pour éviter les conflits d'exécution.

### 2) Provisionnement des VMs via Terraform (Proxmox provider)

Le dossier [`terraform/proxmox/`](https://github.com/patrickbrouhard/proxmox-web-vm/terraform/proxmox) contient la configuration Terraform :

- ressource `proxmox_vm_qemu` itérée via `for_each` sur une map `vm_configs`
- configuration cloud-init (user, réseau, clé SSH), disque, réseau virtio, agent, etc.  
  (fichiers clés : `main.tf`, `variables.tf`, `vms.auto.tfvars`)

Les secrets et paramètres sensibles ne sont pas committés : un exemple est fourni via  
[`terraform/proxmox/credentials.auto.tfvars.example`](https://github.com/patrickbrouhard/proxmox-web-vm/blob/master/terraform/proxmox/credentials.auto.tfvars.example) (à copier en `credentials.auto.tfvars` local).

### 3) Génération d’inventaire Ansible depuis Terraform

Terraform génère automatiquement un inventaire Ansible YAML via  
[`terraform/proxmox/ansible_inventory.tf`](https://github.com/patrickbrouhard/proxmox-web-vm/blob/master/terraform/proxmox/ansible_inventory.tf), ce qui évite de "tenir à la main" l’IP / user de la VM.

Le fichier généré est :  
[`ansible/inventory/web.generated.yml`](https://github.com/patrickbrouhard/proxmox-web-vm/blob/master/ansible/inventory/web.generated.yml)

### 4) Configuration Ansible : bootstrap + Docker + maintenance + notifications

- bootstrap d’une VM cloud-init :  
  [`ansible/playbooks/bootstrap-linux-cloudinit.yml`](https://github.com/patrickbrouhard/proxmox-web-vm/blob/master/ansible/playbooks/bootstrap-linux-cloudinit.yml)  
  (attente SSH, attente cloud-init, installation de python3 si nécessaire)
- installation Docker :  
  [`ansible/playbooks/web_docker_install.yml`](https://github.com/patrickbrouhard/proxmox-web-vm/blob/master/ansible/playbooks/web_docker_install.yml)  
  basé sur le rôle [`ansible/roles/docker/`](https://github.com/patrickbrouhard/proxmox-web-vm/blob/master/ansible/roles/docker/tasks/install.yml)
- maintenance Docker “safe by default” :  
  [`ansible/playbooks/docker_maintenance.yml`](https://github.com/patrickbrouhard/proxmox-web-vm/blob/master/ansible/playbooks/docker_maintenance.yml)  
  + rôle docker `tasks/maintenance.yml`
- notifications Telegram (utile en cas d’échec bootstrap, testable et secret géré via Ansible Vault) :  
  rôle [`ansible/roles/notify_telegram/`](https://github.com/patrickbrouhard/proxmox-web-vm/blob/master/ansible/roles/notify_telegram/tasks/main.yml)  
  + playbook de test [`ansible/playbooks/test-role-notify_telegram.yml`](https://github.com/patrickbrouhard/proxmox-web-vm/blob/master/ansible/playbooks/test-role-notify_telegram.yml)

---

## Technologies utilisées

- **Proxmox VE** (API + CLI `qm`, `pvesm` dans le script de template)
- **Terraform** (>= 1.7.0) + provider **Telmate/proxmox**
- **Ansible**
  - collections : `community.docker`, `community.general`, `ansible.posix` (voir `ansible/requirements.yml`)
- **Docker** (installation, configuration, maintenance)
- **Bash** (orchestration et scripting)
- **GitHub Actions**
  - `gitleaks` (scan secrets)
  - `terrascan` (scan IaC Terraform)
- **YAML / Jinja2** (playbooks, inventory, templates Ansible)
- **Ansible Vault** pour la gestion sécurisée des secrets (ex : token Telegram)
- **Git** pour le versioning

---

## Compétences mises en évidence

- **Automatisation de bout en bout (local)** : `scripts/create-vm-web.sh` enchaîne Terraform puis Ansible.
- **Infrastructure as Code** : `terraform/proxmox/main.tf` + `variables.tf` + `vms.auto.tfvars`.
- **Intégration Terraform → Ansible** : génération d’inventaire via `ansible_inventory.tf`.
- **Gestion de configuration** : rôle Docker structuré (install/config/post-install/maintenance) + template `daemon.json.j2`.
- **Sécurité côté repo** : workflows GitHub Actions `gitleaks.yml` et `terrascan.yml`.
- **Sécurité côté système (début)** : rôle `ssh_hardening` avec asserts (éviter un lock-out lors de la config sudo).

> Note : le rôle `ssh_hardening` est volontairement minimal pour l’instant.

---

## Extraits de code (représentatifs)

### 1) Orchestration Terraform → Ansible (script)

Fichier : `scripts/create-vm-web.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

command -v terraform >/dev/null || { echo "Terraform manquant"; exit 1; }
command -v ansible-playbook >/dev/null || { echo "Ansible manquant"; exit 1; }

ROOT_DIR="$(cd "$(dirname "$0")/.." && pwd)"

cd "$ROOT_DIR/terraform/proxmox"

echo "=== Terraform init ==="
terraform init -upgrade

echo "=== Terraform validate ==="
terraform validate

echo "=== Terraform apply ==="
terraform apply -auto-approve

cd "$ROOT_DIR/ansible"

echo "=== Ansible bootstrap ==="
ansible-playbook playbooks/bootstrap-linux-cloudinit.yml --limit web

echo "=== Ansible provisioning ==="
ansible-playbook playbooks/web_docker_install.yml --limit web
```

---

### 2) Inventaire Ansible généré par Terraform (pont entre outils)

Fichier : `terraform/proxmox/ansible_inventory.tf`

```hcl
locals {
  web_hosts = {
    for _, vm in var.vm_configs :
    vm.name => {
      ansible_host = regex("ip=([0-9.]+)", vm.ipconfig0)[0]
      ansible_user = vm.ciuser
    }
  }

  ansible_inventory_web = {
    web = {
      hosts = local.web_hosts
    }
  }
}

resource "local_file" "ansible_inventory_web" {
  filename = "${path.module}/../../ansible/inventory/web.generated.yml"
  content  = yamlencode(local.ansible_inventory_web)
}
```

On évite une étape manuelle (maintenir l’inventaire) et on réduit le risque de divergence entre “ce que Terraform a créé” et “ce qu’Ansible cible”.

---

### 3) Maintenance Docker “safe by default” (opt-in pour actions destructrices)

Fichier : `ansible/roles/docker/tasks/maintenance.yml`

```yaml
- name: Supprimer (prune) (images, containers, volumes, builder cache) selon variables
  community.docker.docker_prune:
    containers: "{{ docker_prune_containers | default(false) }}"
    images: "{{ docker_prune_images | default(false) }}"
    networks: false
    volumes: "{{ docker_prune_volumes | default(false) }}"
    builder_cache: "{{ docker_prune_builder_cache | default(false) }}"
    images_filters:
      dangling: false
  when: >
    (docker_prune_containers | default(false)) or
    (docker_prune_images | default(false)) or
    (docker_prune_volumes | default(false)) or
    (docker_prune_builder_cache | default(false))
  tags:
    - docker
    - docker_maintenance
    - docker_prune
    - "{{ docker_danger_tag }}"
```

Montre une approche prudente côté ops : par défaut on ne fait rien de destructeur sans action explicite (variables + tags).

---

## CI / sécurité côté dépôt (GitHub Actions)

- Scan de secrets :  
  workflow [`/.github/workflows/gitleaks.yml`](https://github.com/patrickbrouhard/proxmox-web-vm/blob/master/.github/workflows/gitleaks.yml)
- Scan IaC Terraform :  
  workflow [`/.github/workflows/terrascan.yml`](https://github.com/patrickbrouhard/proxmox-web-vm/blob/master/.github/workflows/terrascan.yml) (en mode `only_warn: true`)

Je l’ai gardé volontairement simple : l’objectif est surtout d’avoir un minimum de “feedback automatique” quand je modifie l’IaC ou que je pousse du code.

---

## Conclusion personnelle

Ce projet a été l'occasion pour moi de découvrir et pratiquer des outils clés du monde DevOps (Terraform, Ansible, Docker) dans un contexte concret de homelab (que j'utilise quotidiennement) avec une logique d’automatisation de bout en bout. 
J’ai particulièrement apprécié le fait de pouvoir faire en un clic ce qui autrefois nécessitait plusieurs étapes parfois douloureuses et longues, et aussi l'idée de fonctionner en déclaratif (IaC, “décrire l’état attendu”).

Même si le sujet était nouveau pour moi, le processus en lui-même était en réalité très proche de ce que je faisais déjà en développement logiciel "standard" : écrire du code (scripts, IaC), tester, corriger, faire évoluer avec une logique d’amélioration continue, etc.

Même si le périmètre reste volontairement limité (une VM “web” et des rôles centrés Docker/maintenance/notifications), il reflète des sujets qu'on retrouve souvent dans des contextes réels : bootstrap de machines cloud-init, inventaire, idempotence, configuration système, et garde-fous.

---

## Pistes d’amélioration

- Ajouter des tests de rôles Ansible avec **Molecule**. Ca serait l'étape suivante logique du point de vue d'un développeur.
- Étendre `ansible/roles/ssh_hardening/` au-delà de la gestion de clés/sudo.
- Continuer à enrichir `vm_configs` côté Terraform pour gérer plusieurs VMs et cas d’usage, en gardant la génération d’inventaire alignée. Je pense en particulier aux VMs que je garde déjà en permanence sur mon Homelab et qui contiennent par exemples du monitoring (ex : Grafana).
- Structurer davantage la maintenance (ex : taguer systématiquement les playbooks/tâches, mieux documenter, cron jobs, etc.).
- Créer des rôles Ansible pour d’autres usages que Docker (ex : monitoring, backup, etc.) et les intégrer dans le processus de provisioning.
- définir desenvironnements éphémères : Terraform crée → Ansible provisionne → destruction (terraform destroy).

