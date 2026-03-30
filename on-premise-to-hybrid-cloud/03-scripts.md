# Scripts PowerShell

L'automatisation a été privilégiée dès le départ pour garantir des 
déploiements reproductibles et limiter les erreurs humaines.

## Scripts de déploiement

| Script | Description |
|---|---|
| `GPO.ps1` | Configuration des GPO : mots de passe, audit, journalisation, droits utilisateurs, pare-feu |
| `PKI.ps1` | Déploiement de l'autorité de certification interne |
| `SMB.ps1` | Configuration des partages SMB sur le serveur de fichiers |
| `Share-VeeamAgent.ps1` | Création du partage réseau pour le déploiement de l'agent Veeam |
| `Backup-Infrastructure.ps1` | Automatisation des sauvegardes quotidiennes |

## Scripts de supervision

| Script | Description |
|---|---|
| `Monitor-HostResources.ps1` | Surveillance des ressources système avec alerte automatique au-delà de 85% d'utilisation |
| `monitoring-des-DC.ps1` | Supervision spécifique des contrôleurs de domaine |

## Philosophie

Chaque script a été conçu pour être :
- **Idempotent** : exécutable plusieurs fois sans effet de bord
- **Documenté** : commentaires intégrés pour la maintenabilité
- **Réutilisable** : paramétrable pour d'autres contextes que le projet pilote