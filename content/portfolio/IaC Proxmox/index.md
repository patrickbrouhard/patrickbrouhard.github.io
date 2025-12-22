---
title: "Infrastructure automatisée Proxmox"
date: 2025-12-22
summary: "Projet IaC/DevOps automatisant la création d’un template Proxmox, la création d’une VM avec Terraform et son provisioning via Ansible (installation Docker, configuration système)."
tags: ["IaC", "DevOps", "Proxmox", "Terraform", "Ansible", "Bash", "Docker", "automation", "infrastructure", "cloud-init"]
cover: "/images/homelab-proxmox-cover.png"
draft: false
---

## Présentation du projet

Je présente un projet d’automatisation **IaC / DevOps** qui standardise la création et le provisioning d’une machine virtuelle Proxmox depuis zéro. Le pipeline combine :

1. un script Bash pour construire un template cloud-ready (cloud-init)
2. des configurations Terraform pour créer la VM à partir du template,
3. des playbooks et rôles Ansible pour le bootstrap système et l’installation de Docker / conteneurs.

L’objectif : disposer d’un workflow reproductible, traçable et idempotent pour provisionner des VM prêtes à l’emploi sur un homelab ou en plateau de test.

Ce projet est exécuté depuis une VM dédiée (Ubuntu Server).

## Objectifs et valeur ajoutée

* Réduire le temps et l’erreur humaine lors de la création de VM.
* Standardiser la création d’environnements de test et de développement.
* Fournir des artefacts réutilisables (template Proxmox, modules Terraform, rôles Ansible).
* Offrir une base modulaire pour déployer des stacks Docker (Portainer, services applicatifs, monitoring).

## Fonctionnalités principales

* Script Bash `create-proxmox-cloud-template.sh` : téléchargement d’images cloud, vérifications, conversion et création du template Proxmox.
* Terraform (`terraform/proxmox`) : provider Proxmox, définition paramétrable des VMs, clonage depuis template, configuration disques et réseau.
* Ansible (`ansible/playbooks`) : attente SSH, installation de Python3 si nécessaire, application de rôles (baseline, hardening, Docker, stack de monitoring, déploiement de stacks Docker).
* Inventaire dynamique/variables groupées et templates Jinja2 pour compose et config.
* **Gestion sécurisée des secrets avec Ansible Vault** : les variables sensibles sont chiffrées (ex. `ansible/inventory/group_vars/vault.yml`) et le mot de passe Vault est conservé hors dépôt (`~/.config/ansible/vault_pass`) pour éviter toute exposition dans Git.

## Technologies utilisées

* Proxmox VE (QEMU/KVM)
* Terraform (provider Telmate/proxmox)
* Ansible (playbooks, rôles, templates, Ansible Vault)
* Bash (script de création de template)
* Docker / docker-compose (via rôles Ansible)
* Git pour le versionning

## Compétences démontrées

* **Techniques :**

  * Infrastructure-as-Code (Terraform), configuration management (Ansible), scripting Bash avancé, intégration Docker, gestion de templates cloud-init, sécurité SSH / baselines.
  * **Sécurisation des secrets (Ansible Vault)** : chiffrement des variables sensibles, séparation code / données sensibles, stockage local du mot de passe Vault hors du dépôt.
* **Transversales :**
  * conception modulaire, automatisation, documentation, reproductibilité et bonnes pratiques Git.

---

## Extraits de code représentatifs

### Verrouillage et contrôle d’exécution (bash)

*Fichier : `scripts/create-proxmox-cloud-template.sh`*

```bash
# Empêcher deux executions simultanées
exec 200>/var/lock/create-template.lock
flock -n 200 || {
    echo "Script déjà en cours d'exécution"
    exit 1
}
```

*Mécanisme de verrouillage garantissant une exécution unique du script, indispensable pour une automatisation fiable et reproductible de la création de templates Proxmox.*

### Définition des VMs (Terraform)

*Fichier : `terraform/proxmox/main.tf`*

```hcl
resource "proxmox_vm_qemu" "vms" {
  for_each = var.vm_configs
  vmid        = each.value.vmid
  name        = each.value.name
  clone_id    = each.value.template_vmid
  full_clone  = each.value.full_clone
  network {
    id     = 0
    model  = "virtio"
    bridge = each.value.bridge
  }
  disks { ... cloudinit { storage = "local-lvm" } ... }
}
```

*Création déclarative et paramétrée des VM, clonage depuis un template cloud-init.*

### Gestion idempotente des redémarrages système (Ansible)

*Fichier : `ansible/roles/common/tasks/reboot-if-needed.yml`*

```yaml
- name: Vérifie si un reboot est nécessaire
  ansible.builtin.stat:
    path: /var/run/reboot-required
  register: reboot_required

- name: Informe qu'un reboot est nécessaire
  ansible.builtin.debug:
    msg: "Reboot requis sur {{ inventory_hostname }} après mise à jour système"
  when: reboot_required.stat.exists

- name: Reboot si nécessaire
  ansible.builtin.reboot:
    msg: "Redémarrage déclenché par Ansible après mise à jour du système"
    reboot_timeout: 600
    connect_timeout: 5
    pre_reboot_delay: 5
    post_reboot_delay: 30
    test_command: whoami
  when: reboot_required.stat.exists
  tags:
    - reboot
    - maintenance
```

*Gestion contrôlée des redémarrages pour garantir la stabilité du provisioning.*

### Utilisation d’une variable chiffrée

*Fichier : ansible/roles/notify_telegram/tasks/main.yml*

```yaml
- name: Envoi message Telegram
  ansible.builtin.uri:
    url: "https://api.telegram.org/bot{{ telegram_token }}/sendMessage"
    method: POST
    body_format: json
    body:
      chat_id: "{{ telegram_chat_id }}"
      text: "{{ telegram_message }}"
      parse_mode: "{{ telegram_parse_mode }}"
  when:
    - telegram_message | length > 0
```

*Les variables sensibles `telegram_chat_id` et surtout `telegram_token` sont fournies via un fichier chiffré (`ansible/inventory/group_vars/vault.yml`) et ne sont par conséquent jamais stockée en clair dans le dépôt.*

### orchestration modulaire de Docker (Ansible)

*Fichier : ansible/playbooks/web_docker_install.yml*

```yaml
- name: Docker - installation initiale
  hosts: web
  become: true
  gather_facts: true

  vars:
    docker_users:
      - "{{ ansible_user }}"

  tasks:
    - name: Précondition système
      import_role:
        name: common
        tasks_from: apt-update-upgrade.yml

    - name: Installation Docker
      import_role:
        name: docker
        tasks_from: install.yml

    - name: Configuration du daemon Docker
      import_role:
        name: docker
        tasks_from: daemon.yml
      vars:
        docker_restart_on_config_change: true

    - name: Post-install et validation
      import_role:
        name: docker
        tasks_from: post_install.yml
      vars:
        docker_test_hello_world: true
```

*Chaque étape (préconditions système, installation, configuration, post-install) est isolée dans des tâches dédiées, importées explicitement pour garder un contrôle fin de l’ordre d’exécution.*
Cette approche améliore la lisibilité, la réutilisabilité des rôles et la maintenabilité du provisioning.

---

## Conclusion personnelle

Ce projet m’a appris à orchestrer des outils complémentaires (Bash, Terraform, Ansible) pour obtenir une chaîne de provisioning fiable et reproductible. J’ai consolidé mes compétences en automatisation, gestion d’images cloud et modularisation d’infrastructure — des savoir-faire directement transférables en entreprise. La mise en place d’Ansible Vault m’a aussi obligé à formaliser la gestion des secrets : séparation stricte entre code et données sensibles, pratique essentielle en milieu professionnel.

### Étapes suivantes

* ajouter des rôles/playbooks pour lancer rapidement des VMs de test, de secours, ou de démonstration, notamment pour le [projet Symfony]({{< relref "portfolio/mesvoyages" >}}).
* environnements éphémères : Terraform crée → Ansible provisionne → destruction (terraform destroy).
* extraire la création de VMs en modules réutilisables.
* intégration CI/CD avec GitHub Actions + runner auto-hébergé.
* automatisations orientées usage quotidien et maintenance.

---


