# 06 — Sécurité

## Vue d'ensemble

La sécurité n'est pas une couche ajoutée après coup dans ce projet elle est **intégrée à chaque composant** dès la conception. Chaque outil du pipeline applique ses propres mesures de sécurité, formant une défense en profondeur cohérente.

---

## Synthèse des mesures par composant

| Composant | Mesure de sécurité |
|---|---|
| **GitLab CI/CD** | Accès restreint aux dépôts, historique complet des actions (pipeline, commit, logs) |
| **SSH** | Authentification par clé asymétrique uniquement, jamais par mot de passe |
| **Variables CI/CD** | Secrets chiffrés côté GitLab, masqués dans les logs, jamais en clair dans le code |
| **Proxmox API** | Token dédié avec rôle `TerraformProvisioner` — permissions minimales |
| **Code** | Aucune information sensible dans le dépôt Git (IPs, mots de passe, tokens) |
| **Ansible** | Versioning des configurations, validation `visudo`, nettoyage des fichiers temporaires |
| **Docker** | Conteneurs isolés, accès restreint à l'utilisateur défini, volumes en lecture seule |
| **Zabbix** | Monitoring des tentatives d'accès, TLS configurable (PSK) |

---

## Gestion des secrets

C'est le point le plus critique d'un pipeline CI/CD. Aucun secret ne transite en clair dans ce projet.

### Variables CI/CD GitLab chiffrées

Tous les secrets sont stockés comme **variables protégées GitLab**, masquées automatiquement dans tous les logs de pipeline :

```
SSH_PRIVATE_KEY_B64   → Clé SSH privée (encodée base64)
TFVARS_API_B64        → Variables Terraform avec token Proxmox (encodées base64)
SUDO_PASSWORD         → Mot de passe become Ansible
```

### Injection au runtime uniquement

La clé SSH est décodée **uniquement au moment de l'exécution** et n'est jamais persistée :

```bash
# Décodage au runtime dans ~/.ssh/gitlab-ci
echo "$SSH_PRIVATE_KEY_B64" | base64 -d > ~/.ssh/gitlab-ci
chmod 600 ~/.ssh/gitlab-ci
# Fichier supprimé à la fin du job (répertoire temporaire du runner)
```

### Nettoyage des fichiers sensibles

Le fichier `become` contenant le mot de passe Ansible est détruit par `shred` après usage :

```bash
# Création du fichier temporaire avec umask restrictif
umask 077
printf 'ansible_become_password: "%s"\n' "${SUDO_PASSWORD}" > /tmp/ci_vars_${CI_PIPELINE_ID}.yml

# Exécution du playbook
ansible-playbook site.yml -e "@/tmp/ci_vars_${CI_PIPELINE_ID}.yml"

# Destruction sécurisée (écrasement + suppression)
shred -u /tmp/ci_vars_${CI_PIPELINE_ID}.yml
```

> `shred` écrase le contenu du fichier avant de le supprimer, empêchant toute récupération par lecture du disque.

---

## Principe du moindre privilège

### Rôle Proxmox dédié pour Terraform

Terraform n'utilise pas le compte `root` de Proxmox. Un rôle `TerraformProvisioner` avec des permissions strictement nécessaires a été créé :

```bash
pveum role add TerraformProvisioner -privs \
  "Datastore.AllocateSpace Datastore.Audit \
   VM.Allocate VM.Audit VM.Clone \
   VM.Config.CPU VM.Config.Disk VM.Config.Memory \
   VM.Config.Network VM.Config.Options VM.Config.Cloudinit \
   VM.PowerMgmt VM.Migrate Pool.Allocate \
   Sys.Audit SDN.Use"
```

Ce que Terraform **ne peut pas** faire avec ce rôle :
- Modifier la configuration globale de Proxmox (`Sys.Modify`)
- Accéder aux VMs des autres pools
- Gérer les utilisateurs Proxmox
- Modifier le stockage ou le réseau de l'hôte

### Comptes de service dédiés

Chaque outil dispose de son propre compte de service sur sa VM :

| VM | Compte | Rôle |
|---|---|---|
| GitLab | `svc-gitlab` | Exécution GitLab uniquement |
| GitLab Runner | `svc-runner` | Exécution des jobs CI/CD |
| Terraform | `svc-terraform` | Provisioning Proxmox |
| Ansible | `svc-ansible` | Configuration des VMs |
| Zabbix | `svc-zabbix` | Supervision |

---

## Sécurité SSH

### Authentification par clé uniquement

L'ensemble des communications SSH du pipeline utilisent l'authentification par **clé asymétrique**. L'authentification par mot de passe est désactivée sur les VMs via le rôle Ansible `users` (option `harden_ssh: true`) :

```yaml
- name: Désactiver l'authentification par mot de passe SSH
  ansible.builtin.lineinfile:
    path: /etc/ssh/sshd_config
    line: 'PasswordAuthentication no'
  notify: Redémarrer SSH

- name: Forcer l'authentification par clé SSH
  ansible.builtin.lineinfile:
    path: /etc/ssh/sshd_config
    line: 'PubkeyAuthentication yes'
  notify: Redémarrer SSH
```

### Clés SSH dédiées par usage

| Paire de clés | Usage |
|---|---|
| `automation_keys` | Connexion Terraform → Proxmox |
| `ansible_keys` | Connexion Ansible → VMs déployées |
| `gitlab-ci` | Connexion Runner → VMs (injectée par CI) |
| `id_ed25519` | Connexion Ansible → VMs (via runner) |

---

## Isolation réseau

Les VMs de l'infrastructure évoluent sur un **réseau dédié isolé** (NAT), séparé du réseau physique hôte. Seuls les flux nécessaires sont ouverts :

| Flux | Port | Direction | Usage |
|---|---|---|---|
| Runner → Terraform/Ansible | 22 | Sortant | SSH CI/CD |
| Terraform → Proxmox API | 8006 | Sortant | Provisioning |
| Ansible → VMs déployées | 22 | Sortant | Configuration |
| Zabbix Agent → Serveur | 10051 | Sortant | Active checks |
| Navigateur → GitLab | 80/443 | Entrant | Interface web |

---

## Traçabilité complète

Chaque action dans le pipeline est tracée et associée à :

```
Commit Git (auteur + hash)
    └─▶ Pipeline GitLab (ID unique)
            └─▶ Workspace Terraform (ci-<pipeline_id>)
                    └─▶ VM créée (nom = debian12-<vmid>)
                                └─▶ Hôte Zabbix (nom = debian12-<vmid>)
```

En cas d'incident, il est possible de remonter depuis la VM supervisée jusqu'au commit Git qui l'a déployée.

---

## Perspectives d'évolution sécurité

| Objectif | Mise en œuvre possible |
|---|---|
| Gestion centralisée des secrets | HashiCorp Vault à la place des variables GitLab |
| Sécuriser les images Docker | Registre d'images privé + scan de vulnérabilités (Trivy) |
| Audit CI/CD | GitLab Security Dashboard + archivage des logs pipeline |
| Redondance | Cluster Proxmox + backup automatisé des VMs déployées |
| TLS Zabbix | Activation du chiffrement PSK entre agents et serveur |
