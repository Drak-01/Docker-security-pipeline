# Hardening

## Etape 1 : segmentation réseau

L’objectif est qu’un conteneur compromis ne puisse ni atteindre ce dont il n’a pas besoin, ni exfiltrer vers Internet. Aujourd’hui, tout est sur pipeline, qui a un accès Internet libre.

1. Matrice des flux légitimes

Il faut d’abord lister ce qui doit communiquer. Tout le reste sera interdit.

| Source                | Destination   | Port     | Raison                               |
| --------------------- | ------------- | -------- | ------------------------------------ |
| Hôte (loopback)       | nginx         | 80, 8082 | Accès utilisateur                    |
| nginx                 | jenkins       | 8080     | Reverse proxy                        |
| nginx                 | sonarqube     | 9000     | Reverse proxy                        |
| jenkins               | sonarqube     | 9000     | Envoi des analyses                   |
| jenkins               | trivy-server  | 4954     | Scans d’images                       |
| sonarqube             | sonarqube-db  | 5432     | JDBC                                 |
| jenkins, trivy-server | egress-proxy  | 3128     | Sortie Internet filtrée              |
| egress-proxy          | Internet      | 443      | Domaines en liste blanche uniquement |
| zap                   | cible du scan | Variable | DAST                                 |

2. Architecture cible

| Réseau | Interne | Membres                                               | Rôle                                |
| ------ | ------- | ----------------------------------------------------- | ----------------------------------- |
| edge   | Non     | nginx                                                 | Seul point d’entrée (ports publiés) |
| apps   | Oui     | nginx, jenkins, sonarqube, trivy-server, egress-proxy | Communication applicative           |
| db     | Oui     | sonarqube, sonarqube-db                               | Base isolée                         |
| egress | Non     | egress-proxy                                          | Seule porte vers Internet           |
| scan   | Oui     | zap + cible                                           | Isolation de l’outil offensif       |

Deux pièges à connaître :

Un conteneur uniquement sur un réseau internal ne peut pas publier de ports. C’est pourquoi nginx a son propre réseau edge non interne, en plus de apps.
Sur un réseau internal, il n’y a aucune route vers Internet. Jenkins et Trivy passeront donc par un proxy qui filtre par nom de domaine.

















