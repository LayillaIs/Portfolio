# Hybrid AD Migration — ApexTech Solutions

Migration d'une infrastructure Active Directory on-premise vers une architecture hybride Azure/Entra ID pour une PME de 50 collaborateurs.

## Contexte

ApexTech Solutions, entreprise fictive de développement logiciel, opérait sur une infrastructure AD traditionnelle présentant plusieurs limites : absence de MFA, pas de SSO, coûts d'infrastructure croissants et aucune mobilité pour les utilisateurs.

**Objectif :** moderniser la gestion des identités et renforcer la sécurité, sans rupture de service.

## Stack technique

| Domaine | Technologies |
|---|---|
| Virtualisation | VMware Workstation |
| Système | Windows Server 2022, Debian 12 |
| Identité | Active Directory, Microsoft Entra ID, Azure AD Connect |
| Collaboration | Microsoft 365 Business Premium |
| Sécurité | MFA, Accès conditionnel, GPO (baseline ANSSI), pfSense |
| Sauvegarde | Veeam, Azure Backup |
| Supervision | Zabbix |
| Automatisation | PowerShell |

## Résultats

✅ Disponibilité mesurée : **99,8%** (seuil requis : 99,5%)
✅ Synchronisation Entra ID : **< 5 min** (tolérance : 30 min)
✅ Temps d'authentification moyen : **1,2 seconde**
✅ Sauvegardes automatiques quotidiennes sans erreur
✅ Basculement domaine contrôleur DC01 → DC02 sans interruption utilisateur

## Structure du projet

- [`01-architecture.md`](./01-architecture.md) — Architecture cible et 
choix retenus
- [`02-realisation.md`](./02-realisation.md) — Déploiement AD, Entra ID, 
M365, sécurité, supervision
- [`03-scripts.md`](./03-scripts.md) — Scripts PowerShell développés
- [`04-validation.md`](./04-validation.md) — Tests, résultats et 
difficultés rencontrées