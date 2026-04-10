# 04 — Ansible

## Vue d'ensemble

Ansible prend le relais une fois les VMs provisionnées par Terraform. Il est responsable de la **configuration complète et automatisée** de chaque machine : création des utilisateurs, installation de Docker, déploiement du service web et installation de la supervision Zabbix.

L'ensemble est structuré en **4 rôles modulaires et réutilisables**, exécutés dans l'ordre par un playbook principal.

---

## Structure des rôles

```
ansible/
├── site.yml                  # Playbook principal
├── hosts                     # Inventaire des VMs cibles
├── requirements.yml          # Dépendances Galaxy
└── roles/
    ├── users/                # Création utilisateurs + SSH + sudoers
    ├── docker/               # Installation Docker Engine
    ├── web/                  # Stack Nginx conteneurisée
    └── zabbix/               # Zabbix Agent 2 + auto-registration
```

---

## Playbook principal : `site.yml`

```yaml
- name: Base configuration
  hosts: srv_debian
  become: true
  roles:
    - role: users
      tags: ['users']
    - role: docker
      tags: ['docker']
    - role: web
      tags: ['web']
    - role: zabbix
      tags: ['zabbix']
```

Les **tags** permettent de rejouer uniquement un rôle spécifique sans relancer l'ensemble :
```bash
ansible-playbook site.yml --tags zabbix   # réinstalle seulement Zabbix
ansible-playbook site.yml --tags web      # redéploie seulement le site web
```

---

## Rôle `users` : Gestion des comptes et SSH

Ce rôle crée les comptes système, déploie les clés SSH autorisées et configure les droits sudo. Il inclut une option de **durcissement SSH** (désactivation de l'authentification par mot de passe).

### Ce qu'il fait

```yaml
# 1. Vérifie la structure de la variable 'users' avant toute action
- name: Validate 'users' variable structure
  ansible.builtin.assert:
    that:
      - users is defined
      - users | type_debug == 'list'

# 2. Crée les comptes
- name: Create user accounts
  ansible.builtin.user:
    name: "{{ item.name }}"
    shell: "{{ item.shell | default('/bin/bash') }}"
    groups: "{{ (item.groups | default([])) | join(',') }}"
  loop: "{{ users }}"

# 3. Déploie les clés SSH autorisées
- name: Install authorized_keys for each user
  ansible.posix.authorized_key:
    user: "{{ item.name }}"
    key: "{{ item.pubkey | default(lookup('file', role_path + '/files/id_rsa.pub')) }}"
  loop: "{{ users }}"

# 4. Configure sudoers sans mot de passe (validé par visudo)
- name: Configure passwordless sudo
  ansible.builtin.copy:
    dest: "/etc/sudoers.d/{{ item.name }}"
    content: "{{ item.name }} ALL=(ALL) NOPASSWD:ALL\n"
    validate: "visudo -cf %s"   # validation avant écriture
  loop: "{{ users }}"

# 5. Durcissement SSH (optionnel via harden_ssh: true)
- name: Désactiver l'authentification par mot de passe SSH
  ansible.builtin.lineinfile:
    path: /etc/ssh/sshd_config
    line: 'PasswordAuthentication no'
  when: harden_ssh | bool
  notify: Redémarrer SSH
```

### Variables par défaut

```yaml
users:
  - name: layilla
    shell: /bin/bash
    groups: ['sudo']
    append: true
    passwordless_sudo: true
harden_ssh: false
```

---

## Rôle `docker` : Installation Docker Engine

Ce rôle s'appuie sur la collection communautaire **`geerlingguy.docker`** pour installer Docker Engine, et prépare les utilisateurs définis pour l'accès Docker.

```yaml
# Crée les utilisateurs Docker si besoin (idempotent)
- name: Ensure docker users exist
  ansible.builtin.user:
    name: "{{ item }}"
    shell: /bin/bash
    create_home: true
  loop: "{{ docker_users_list }}"

# Installation via rôle Galaxy
- name: Install Docker Engine via geerlingguy.docker
  ansible.builtin.import_role:
    name: geerlingguy.docker
  vars:
    docker_apt_keyring_path: /etc/apt/keyrings/docker.gpg
    docker_users: "{{ docker_users_list }}"

# SDK Python pour les modules community.docker/*
- name: Ensure Python Docker SDK
  ansible.builtin.package:
    name: python3-docker
    state: present
```

**Dépendance Galaxy** déclarée dans `requirements.yml` :
```yaml
roles:
  - name: geerlingguy.docker
    version: 7.4.1
```

---

## Rôle `web` : Stack Nginx conteneurisée

Ce rôle déploie une **page de statut dynamique** servie par un conteneur Nginx Alpine. La page est générée par un template Jinja2 qui injecte automatiquement le hostname, la date de déploiement et la plateforme.

```yaml
# Libère le port 80 si Nginx système est présent
- name: Stop/disable system nginx (to free :80)
  ansible.builtin.service:
    name: nginx
    state: stopped
    enabled: false
  when:
    - disable_system_nginx | bool
    - "'nginx.service' in ansible_facts.services"

# Génère la page HTML via template Jinja2
- name: Deploy site index.html (Jinja2)
  ansible.builtin.template:
    src: "index.html.j2"
    dest: "{{ web_root }}/index.html"

# Lance le conteneur Nginx avec bind mount (lecture seule)
- name: Run Nginx container with bind mount
  community.docker.docker_container:
    name: "{{ container_name }}"
    image: "nginx:alpine"
    restart_policy: always
    published_ports:
      - "{{ web_port }}:80"
    volumes:
      - "{{ web_root }}:/usr/share/nginx/html:ro"
    state: started
    pull: true
```

### Variables par défaut

```yaml
web_root: /opt/web/html
web_port: 80
container_name: nginx_web
site_title: "Certification DevOps"
site_subtitle: "Déployé avec GitLab CI/CD + Terraform + Ansible + Docker"
site_host: "{{ inventory_hostname }}"
site_deployed_at: "{{ ansible_date_time.iso8601 }}"
disable_system_nginx: true
```

---

## Rôle `zabbix` : Agent 2 + auto-registration

Ce rôle installe et configure **Zabbix Agent 2** depuis le dépôt officiel, en mode actif avec auto-registration. Dès que la VM est déployée, elle s'enregistre automatiquement dans le serveur Zabbix sans intervention manuelle.

```yaml
# Ajoute le dépôt officiel Zabbix pour Debian 12
- name: Add Zabbix official repository
  ansible.builtin.apt:
    deb: "https://repo.zabbix.com/zabbix/{{ zabbix_major_version }}/debian/..."

# Installe Agent 2 (ou Agent 1 selon zabbix_use_agent2)
- name: Install Zabbix agent2
  ansible.builtin.apt:
    name: "{{ 'zabbix-agent2' if zabbix_use_agent2 else 'zabbix-agent' }}"

# Génère la configuration depuis template Jinja2
- name: Write agent configuration
  ansible.builtin.template:
    src: zabbix_agent2.conf.j2
    dest: /etc/zabbix/zabbix_agent2.conf
  notify: Restart zabbix-agent

# Ajoute zabbix au groupe docker (monitoring des conteneurs)
- name: Add zabbix user to docker group if available
  ansible.builtin.user:
    name: zabbix
    groups: docker
    append: yes
  when: docker_group_check.rc == 0

# Ouvre les ports UFW si présent (actif: 10051 sortant, passif: 10050 entrant)
- name: Ensure ufw allows Zabbix egress
  ansible.builtin.shell: ufw allow out 10051/tcp
  when: zabbix_agent_mode in ['active', 'both']
```

### Configuration Agent 2 (template Jinja2)

```ini
### Managed by Ansible ###
Server={{ zabbix_server_ip }}
ServerActive={{ zabbix_server_ip }}:{{ zabbix_server_port }}
ListenPort=10050
Hostname={{ inventory_hostname }}
HostMetadata={{ zabbix_host_metadata | default('DEVOPS_CI') }}
EnablePersistentBuffer=0
Include=/etc/zabbix/zabbix_agent2.d/*.conf
```

### Variables par défaut

```yaml
zabbix_server_port: 10051
zabbix_agent_mode: "active"       # active | passive | both
zabbix_host_metadata: "DEVOPS_CI" # clé d'auto-registration
zabbix_major_version: "7.0"
zabbix_use_agent2: true
zabbix_tls: "none"                # none | psk
```

---

## Idempotence

Tous les rôles sont **idempotents** : ils peuvent être rejoués autant de fois que nécessaire sans effet de bord. Si la ressource existe déjà dans l'état attendu, Ansible ne modifie rien — c'est le principe fondamental des modules `ansible.builtin.*`.

```
1ère exécution  →  changed: 4  (installation complète)
2ème exécution  →  changed: 0  (rien à modifier, état déjà conforme)
```

---

## Ce que ces rôles démontrent

- **Modularité** : chaque rôle est indépendant, taggué, et peut être rejoué seul
- **Réutilisabilité** : les rôles `users` et `docker` sont génériques et réutilisables sur tout projet Debian/Ubuntu
- **Validation des entrées** : assertion sur la structure des variables avant exécution
- **Sécurité** : validation `visudo` avant écriture des sudoers, durcissement SSH optionnel
- **Intégration CI/CD** : les rôles font partie du pipeline, pas d'étape post-hoc manuelle
- **Supervision intégrée** : Zabbix Agent 2 est déployé comme n'importe quel autre service, au même titre que Docker ou Nginx