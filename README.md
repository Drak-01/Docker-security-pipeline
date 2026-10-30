# Docker Security Pipeline

Stack locale de sécurité applicative basée sur Docker Compose : Jenkins,
SonarQube, PostgreSQL, Nginx et OWASP ZAP.

La documentation détaillée et les diagrammes sont disponibles dans
[`docs/README.md`](docs/README.md).

## Architecture

- **Jenkins** : serveur CI, accessible via Nginx; son port n'est pas publié sur l'hôte.
- **SonarQube** : analyse de code, relié à PostgreSQL et accessible via Nginx.
- **PostgreSQL** : base de données interne à SonarQube, sans port hôte publié.
- **Nginx** : point d'entrée HTTP local sur `127.0.0.1:8081`, proxy vers Jenkins et SonarQube.
- **OWASP ZAP** : service optionnel, activé par le profil `security-tools`.

Chaque service construit son image depuis son propre dossier : `jenkins/`,
`nginx/`, `sonarqube/` ou `owasp-zap/`. Les données persistantes sont stockées
dans des volumes Docker nommés.

## Prérequis

- Docker Engine
- Docker Compose v2 (`docker compose`)
- Linux : droits suffisants pour régler les paramètres noyau de SonarQube
- Port 8081 disponible sur l'hôte; le port 9000 est utilisé localement par SonarQube

SonarQube utilise Elasticsearch et nécessite des paramètres noyau adaptés :

```bash
sudo tee /etc/sysctl.d/99-sonarqube.conf >/dev/null <<'EOF'
vm.max_map_count=524288
fs.file-max=131072
EOF
sudo sysctl --system
```

## Configuration

Compléter `.env` avant de démarrer la stack :

```dotenv
NGINX_HTTP_PORT=8081
SONARQUBE_HTTP_PORT=9000
SONAR_PROXY_PORT=8082
SONAR_DB_PASSWORD=remplacer_par_un_secret_unique
```

Utiliser un mot de passe long et unique pour `SONAR_DB_PASSWORD`. La même valeur
est utilisée par PostgreSQL et SonarQube. Ne pas publier ni committer un vrai
fichier `.env` contenant des secrets.

Les noms d'hôtes Nginx sont définis dans
[`nginx/conf.d/default.conf`](nginx/conf.d/default.conf) :
`jenkins.hakilox.com` et `sonar.hakilox.com`. Configurer leurs entrées DNS vers
l'adresse du serveur, ou remplacer ces noms par ceux de votre environnement.

## Démarrage

Valider la configuration, construire les images et démarrer les services :

```bash
docker compose config --quiet
docker compose up -d --build
docker compose ps
```

Accès depuis le serveur :

- Jenkins via Nginx : `http://127.0.0.1:8081`
- SonarQube directement : `http://127.0.0.1:9000`

Depuis un autre ordinateur du même réseau, ouvrez Jenkins avec
`http://IP_LOCALE_DU_SERVEUR:8081` (remplacez par l'adresse IP locale du serveur).
SonarQube est accessible sur `http://IP_LOCALE_DU_SERVEUR:8082`. Autorisez les
ports 8081 et 8082 dans le pare-feu uniquement depuis le réseau local.
N'utilisez pas le domaine `hakilox.com` pour cet accès HTTP, car le navigateur
peut forcer HTTPS.

## Utilisation

```bash
# État des conteneurs
docker compose ps

# Logs de toute la stack ou d'un service précis
docker compose logs -f
docker compose logs -f jenkins
docker compose logs -f nginx-pipeline
docker compose logs -f sonarqube

# Reconstruire les images
docker compose build

# Arrêter les services sans supprimer les données
docker compose down
```

Les données Jenkins, SonarQube et PostgreSQL restent dans des volumes nommés
après `docker compose down`. La commande `docker compose down -v` supprime aussi
ces volumes et leurs données.

### OWASP ZAP

ZAP est optionnel et n'est pas démarré par défaut. La commande actuelle met à
jour ses modules complémentaires puis se termine; elle ne lance pas de scan :

```bash
docker compose --profile security-tools run --rm zap
```

## Réseau et sécurité

PostgreSQL n'a pas de port publié. SonarQube est publié sur `127.0.0.1:9000`,
mais reste aussi joignable par Nginx sur le réseau Docker. Jenkins n'a aucun port
hôte publié. Les détails sur les bindings réseau sont dans
[`docs/note.md`](docs/note.md).

Nginx écoute dans le conteneur sur les ports 80 et 8082, publiés respectivement
sur 8081 et 8082 de toutes les interfaces du serveur. Limitez les deux ports au
réseau local avec le pare-feu. SonarQube reste lié à `127.0.0.1:9000`; le port
8082 le relaie via Nginx.