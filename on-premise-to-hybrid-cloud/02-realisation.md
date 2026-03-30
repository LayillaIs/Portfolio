# Réalisation technique

## 1. Infrastructure de base

Déploiement de l'environnement de virtualisation sous VMware avec 
configuration d'un Active Directory en haute disponibilité :

- Deux contrôleurs de domaine (DC01-MTP / DC02-MTP) avec réplication 
automatique
- Services réseau : DNS, DHCP, PKI (autorité de certification interne)
- Segmentation réseau en 4 VLANs avec filtrage pfSense
- Partages SMB sur serveur de fichiers dédié

## 2. Intégration cloud hybride

### Azure AD Connect
Synchronisation bidirectionnelle entre l'Active Directory local et 
Microsoft Entra ID :

- Création des comptes de service dédiés (on-premise + Entra ID)
- Synchronisation des identités en moins de 5 minutes
- SSO opérationnel entre environnement local et applications cloud

### Microsoft 365
Déploiement de Microsoft 365 Business Premium avec authentification 
transparente :

- Accès aux applications cloud avec les identifiants du domaine local
- Point de contrôle unique pour la gestion des identités (admin)
- Aucune rupture d'expérience utilisateur lors de la transition

## 3. Sécurisation

### MFA et accès conditionnel
- Déploiement Azure MFA (application mobile, SMS, codes de secours)
- Politiques d'accès conditionnel selon le contexte de connexion 
(localisation, appareil, risque détecté)
- Score Microsoft Secure Score : **84,85%**

### GPO et durcissement système
Application de la baseline ANSSI :

- Politique de mots de passe renforcée (12 caractères minimum, complexité)
- Désactivation des services non essentiels
- Audit de sécurité centralisé
- Restriction des droits utilisateurs

## 4. Supervision

Déploiement Zabbix avec templates Windows Server et Active Directory :

| Widget | Utilité |
|---|---|
| Network Traffic Overview | Détection d'anomalies réseau |
| System Health Summary | État des services en temps réel |
| Storage Usage | Anticipation des besoins disque |
| System Performance | Détection de surcharge serveur |
| Active Incidents | Remontée et intervention rapide |

Détection des incidents en moins de **2 minutes**.

## 5. Sauvegarde

- Sauvegardes automatiques quotidiennes via Veeam et Azure Backup
- RTO cible : 2 heures maximum
- RPO cible : 24 heures maximum
- Basculement DC01 → DC02 validé sans interruption utilisateur