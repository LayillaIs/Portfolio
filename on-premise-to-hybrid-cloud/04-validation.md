# Validation et retour d'expérience

## Résultats des tests

### Performance et disponibilité

| Indicateur | Objectif | Résultat |
|---|---|---|
| Disponibilité de l'infrastructure | 99,5% | **99,8%** ✅ |
| Synchronisation Entra ID | < 30 min | **< 5 min** ✅ |
| Temps d'authentification moyen | < 3 sec | **1,2 sec** ✅ |
| Détection d'incident Zabbix | < 5 min | **< 2 min** ✅ |

### Haute disponibilité
Basculement DC01 → DC02 simulé sans interruption de service côté 
utilisateurs. Reprise automatique au retour du contrôleur principal 
validée.

### SSO et Microsoft 365
Authentification unique opérationnelle sur poste joint au domaine, 
accès M365 et gestion des mots de passe expirés — aucune friction 
utilisateur observée.

### Sauvegardes
Sauvegardes automatiques quotidiennes exécutées sans erreur sur toute 
la durée du projet.

## Difficultés rencontrées

**Connectivité pfSense / Azure AD Connect**
La synchronisation hybride a nécessité plusieurs ajustements des règles 
de filtrage pfSense avant d'obtenir une communication stable entre 
l'environnement on-premise et le tenant Azure.

**Initialisation d'Azure AD Connect**
La configuration des comptes de service (un côté AD local, un côté 
Entra ID) avec leurs rôles bien distincts a posé des difficultés 
initiales. Solution retenue : rédaction d'un runbook détaillé avec 
points de contrôle à chaque étape clé.

**Gestion du temps**
Certaines phases ont nécessité une réorganisation pour prioriser les 
fonctionnalités critiques et paralléliser les tâches compatibles 
(tests en parallèle des installations suivantes).

**Montée en compétence Azure**
Première expérience hands-on sur Microsoft Azure. Approche retenue : 
snapshots systématiques, sauvegardes fréquentes et documentation en 
continu pour sécuriser chaque expérimentation.