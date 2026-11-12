# Documentation

Cette documentation distingue la stack actuellement décrite par Docker Compose
de l'architecture DevSecOps cible représentée dans certains assets.

Les domaines `jenkins.example.com` et `sonar.example.com` sont des exemples :
remplace-les par tes propres domaines.

## Article

| Document | Contenu |
| --- | --- |
| [Jenkins et SonarQube sur Docker pour le SAST](article-jenkins-sonarqube-sast.md) | Article d'introduction : monter le pipeline d'analyse de qualité et de sécurité du code |

## Guides

| Document | Contenu |
| --- | --- |
| [README du projet](../README.md) | Installation, configuration `.env`, démarrage, mises à jour |
| [Architecture](architecture.md) | Services déployés, flux réseau, volumes, diagrammes |
| [Sécurité réseau](note.md) | Publication des ports et accès aux interfaces d'administration |
| [SonarQube](sonarqube/readme.md) | Premier accès, GitHub App, import des projets |
| [Jenkins](Jenkins/Readme.md) | Plugin, token, webhook, job Pipeline et Jenkinsfile |

Ordre de lecture conseillé pour une première installation : README du projet,
puis SonarQube, puis Jenkins.

## Assets

| Dossier | Contenu |
| --- | --- |
| [`assets/`](assets/) | Diagrammes Mermaid de l'architecture cible et leurs rendus PNG |
| [`sonarqube/assets/`](sonarqube/assets/) | Captures du guide SonarQube |
| [`Jenkins/assets/`](Jenkins/assets/) | Captures du guide Jenkins |

Les captures ont été nettoyées des informations personnelles (noms, comptes,
favoris du navigateur) et de leurs métadonnées. Vérifie qu'aucun token, mot de
passe ou nom d'utilisateur n'est visible avant d'en ajouter de nouvelles.
