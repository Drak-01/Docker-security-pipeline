# Docker Security Pipeline

Stack locale de sécurité applicative basée sur Docker Compose : Jenkins,
SonarQube, PostgreSQL, Nginx et OWASP ZAP.

La documentation détaillée et les diagrammes sont disponibles dans
[`docs/README.md`](docs/README.md).

## Architecture

- **Jenkins** : serveur CI, accessible via Nginx; son port n'est pas publié sur l'hôte.
- **SonarQube** : analyse de code, relié à PostgreSQL et accessible via Nginx.
- **PostgreSQL** : base de données interne à SonarQube, sans port hôte publié.
- **Nginx** : reverse proxy interne aux deux sous-domaines derrière Cloudflare Tunnel.
- **OWASP ZAP** : service optionnel, activé par le profil `security-tools`.

Chaque service construit son image depuis son propre dossier : `jenkins/`,
`nginx/`, `sonarqube/` ou `owasp-zap/`. Les données persistantes sont stockées
dans des volumes Docker nommés.

## Prérequis

- Docker Engine
- Docker Compose v2 (`docker compose`)
- Linux : droits suffisants pour régler les paramètres noyau de SonarQube
- Port 9000 disponible localement pour SonarQube

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

Cloudflared tourne sur l'hôte via systemd. Dans le fichier de configuration du
tunnel existant, conserve son UUID et son fichier de credentials et configure
les règles d'ingress suivantes :

```yaml
ingress:
	- hostname: jenkins.hakilox.com
		service: http://localhost:8081
		originRequest:
			httpHostHeader: jenkins.hakilox.com
	- hostname: sonar.hakilox.com
		service: http://localhost:8082
		originRequest:
			httpHostHeader: sonar.hakilox.com
	- service: http_status:404
```

Les ports Nginx sont liés à `127.0.0.1`; Cloudflared les atteint par localhost.
Le client systemd n'a pas besoin de rejoindre le réseau Docker. Cloudflare
termine le HTTPS public; la liaison d'origine reste en HTTP local. Ne configure
pas `noTLSVerify` pour ces origines HTTP.

Dans Cloudflare Access, protège les deux sous-domaines. Pour préserver les
webhooks GitHub, exclue uniquement `/github-webhook/` de l'interaction Access
si nécessaire et active un secret de webhook dans Jenkins.

## Démarrage

Valider la configuration, construire les images et démarrer les services :

```bash
docker compose config --quiet
docker compose up -d --build
docker compose ps
```

Accès via Cloudflare Tunnel :

- Jenkins : `https://jenkins.hakilox.com`
- SonarQube : `https://sonar.hakilox.com`
- SonarQube directement : `http://127.0.0.1:9000`

Les URLs publiques sont servies en HTTPS par Cloudflare. Configure également
l'URL Jenkins (`https://jenkins.hakilox.com`) dans Jenkins et l'URL de base
SonarQube (`https://sonar.hakilox.com`) dans ses réglages d'administration pour
éviter les redirections vers un ancien domaine.

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

Cloudflared systemd et Nginx communiquent par les ports loopback 8081 et 8082;
aucun port public n'est requis sur le serveur pour le tunnel. Jenkins et
PostgreSQL ne sont pas publiés directement. SonarQube reste accessible sur
`127.0.0.1:9000` pour l'administration locale.

cp -p .env ../.env.server.backup
git restore .env
git pull --ff-only
cp -p ../.env.server.backup .env

docker compose config --quiet
docker compose build nginx-pipeline
docker compose up -d --no-build --force-recreate nginx-pipeline
docker compose port nginx-pipeline 8082
