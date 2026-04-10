# 03 — Terraform

## Vue d'ensemble

Terraform est l'outil de **provisioning Infrastructure as Code** du projet. Il est responsable de la création des machines virtuelles sur Proxmox VE via son API REST, en s'appuyant sur le provider `bpg/proxmox`.

Chaque VM est créée par **clonage complet** d'un template Debian 12 Cloud-Init pré-configuré, garantissant des déploiements rapides et identiques.

---

## Provider utilisé

```hcl
terraform {
  required_providers {
    proxmox = {
      source  = "bpg/proxmox"
      version = ">= 0.80.0"
    }
  }
}

provider "proxmox" {
  endpoint  = var.pm_api_url
  insecure  = var.pm_tls_insecure
  api_token = "${var.pm_token_id}=${var.pm_token_secret}"
}
```

Le provider `bpg/proxmox` a été choisi pour sa compatibilité native avec l'API REST de Proxmox VE et sa gestion du Cloud-Init. L'authentification se fait via **token API** (et non par mot de passe), conformément au principe du moindre privilège.

---

## Ressource principale : VM Proxmox

```hcl
resource "proxmox_virtual_environment_vm" "vm" {
  vm_id     = var.vm_id       # null = prochain VMID libre
  name      = var.name_suffix != "" ? "${var.vm_name}-${var.name_suffix}" : var.vm_name
  node_name = var.node_name
  pool_id   = "VM-Deploy"     # pool dédié aux VMs déployées

  started   = true

  clone {
    vm_id        = var.template_id  # ID du template Cloud-Init
    full         = true             # clone complet (pas linked clone)
    datastore_id = var.datastore_id
  }

  cpu    { cores = var.vm_cores }
  memory { dedicated = var.vm_memory }

  network_device {
    bridge = var.bridge
    model  = "virtio"
  }

  agent { enabled = true }   # qemu-guest-agent requis dans le template

  initialization {
    ip_config {
      ipv4 { address = "dhcp" }  # IP statique appliquée en post-config
    }
  }
}
```

---

## Post-configuration : IP statique via API Proxmox

Une fois la VM créée, Terraform applique une **post-configuration** via des appels directs à l'API REST Proxmox pour :

1. Renommer la VM avec son VMID (`debian12-<vmid>`) nom unique garanti
2. Assigner une IP statique calculée automatiquement : `<ip_base>.<vmid>/<prefix>`
3. Redémarrer la VM pour appliquer la configuration réseau

```hcl
resource "null_resource" "post_config" {
  depends_on = [proxmox_virtual_environment_vm.vm]

  provisioner "local-exec" {
    command = <<-CMD
      # Renommage de la VM
      curl -k -X PUT -H 'Authorization: PVEAPIToken=...' \
        --data-urlencode 'name=${vm_name}-${vmid}' \
        '${api}/nodes/${node}/qemu/${vmid}/config'

      # Attribution IP statique
      curl -k -X PUT -H 'Authorization: PVEAPIToken=...' \
        --data-urlencode 'ipconfig0=ip=${ip_base}.${vmid}/${prefix},gw=${gateway}' \
        '${api}/nodes/${node}/qemu/${vmid}/config'

      # Reboot
      curl -k -X POST -H 'Authorization: PVEAPIToken=...' \
        '${api}/nodes/${node}/qemu/${vmid}/status/reboot'
    CMD
  }
}
```

> **Pourquoi cette approche ?** L'IP statique calculée depuis le VMID (`192.168.x.<vmid>`) rend chaque VM **adressable de façon déterministe** sans gestion de DHCP ni de registre d'IPs, ce qui simplifie grandement l'inventaire Ansible dynamique.

---

## Export des outputs vers Ansible

L'IP de la VM est écrite dans un fichier local récupéré par le pipeline CI/CD pour être transmise au stage Ansible :

```hcl
resource "local_file" "vm_ip_file" {
  depends_on = [null_resource.post_config]
  content    = "${local.ip_base}.${proxmox_virtual_environment_vm.vm.vm_id}"
  filename   = "${path.module}/vm_ip.txt"
}

output "vm_id"           { value = proxmox_virtual_environment_vm.vm.vm_id }
output "vm_name_effective" { value = "${var.vm_name}-${vm.vm_id}" }
output "vm_ipv4"         { value = local_file.vm_ip_file.content }
```

---

## Variables d'entrée

Toutes les valeurs sensibles (token API, URL Proxmox) sont injectées via des **variables CI/CD GitLab chiffrées** et jamais stockées dans le dépôt.

| Variable | Type | Description |
|---|---|---|
| `pm_api_url` | string | URL de l'API Proxmox |
| `pm_token_id` | string | ID du token API (injecté par CI) |
| `pm_token_secret` | string | Secret du token (injecté par CI) |
| `pm_tls_insecure` | bool | Désactive la vérif TLS (lab) |
| `node_name` | string | Nom du nœud Proxmox cible |
| `datastore_id` | string | Stockage pour le clone |
| `template_id` | number | ID du template Cloud-Init |
| `vm_name` | string | Nom de base de la VM |
| `vm_cores` | number | Nombre de vCPU |
| `vm_memory` | number | RAM en MiB |
| `vm_id` | number | VMID fixe (`null` = prochain libre) |
| `name_suffix` | string | Suffixe unique (ex: `CI_PIPELINE_ID`) |
| `ip_mode` | string | `dhcp` ou `static` (validé) |

### Validation des variables

Les variables critiques intègrent des **règles de validation** Terraform natives :

```hcl
variable "ip_mode" {
  validation {
    condition     = contains(["dhcp", "static"], lower(trimspace(var.ip_mode)))
    error_message = "ip_mode doit être \"dhcp\" ou \"static\"."
  }
}

variable "ipv4" {
  validation {
    condition     = lower(trimspace(var.ip_mode)) != "static" || length(trimspace(var.ipv4)) > 0
    error_message = "En mode static, 'ipv4' est obligatoire."
  }
}
```

---

## Workspaces CI/CD

À chaque exécution du pipeline, un **workspace Terraform isolé** est créé avec l'ID du pipeline :

```bash
terraform workspace new "ci-${CI_PIPELINE_ID}"
```

Cela permet d'exécuter plusieurs pipelines en parallèle sans collision de state, et de conserver l'historique de chaque déploiement.

---

## Sécurité : Rôle Proxmox dédié

Terraform n'utilise pas le compte `root` de Proxmox. Un rôle `TerraformProvisioner` avec des permissions minimales a été créé :

```bash
pveum role add TerraformProvisioner -privs \
  "Datastore.AllocateSpace Datastore.Audit \
   VM.Allocate VM.Audit VM.Clone \
   VM.Config.CPU VM.Config.Disk VM.Config.Memory \
   VM.Config.Network VM.Config.Options VM.Config.Cloudinit \
   VM.PowerMgmt VM.Migrate \
   Pool.Allocate Sys.Audit SDN.Use"
```

> Principe du moindre privilège : Terraform peut créer et configurer des VMs, mais ne peut pas administrer Proxmox lui-même (pas de `Sys.Modify` global, pas d'accès aux autres pools).

---

## Ce que ce module Terraform démontre

- **IaC déclaratif** : l'état de l'infrastructure est décrit en HCL et versionné dans Git
- **State management** : Terraform gère l'état des ressources et détecte les dérives
- **Intégration API native** : communication directe avec l'API REST Proxmox via token
- **Validation des entrées** : les variables critiques sont validées avant tout apply
- **Déploiement idempotent** : relancer un apply sur une VM existante ne crée pas de doublon
- **Séparation des secrets** : aucune credential en dur, tout transite par les variables CI/CD
