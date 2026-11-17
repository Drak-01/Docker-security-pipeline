# Docker Security Pipeline

Stack locale de sécurité applicative basée sur Docker Compose : Jenkins,
SonarQube, PostgreSQL, Nginx et OWASP ZAP.

La documentation détaillée et les diagrammes sont disponibles dans
[`docs/README.md`](docs/README.md). Pour une introduction pas à pas, lire
l'article [Jenkins et SonarQube sur Docker pour le SAST](docs/article-jenkins-sonarqube-sast.md).

Les domaines `jenkins.example.com` et `sonar.example.com` utilisés ci-dessous
sont des exemples : les remplacer par les siens, y compris dans
[`nginx/conf.d/default.conf`](nginx/conf.d/default.conf).

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

Copier le modèle puis compléter `.env` avant de démarrer la stack :

```bash
cp .env.example .env
```

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
  - hostname: jenkins.example.com
    service: http://localhost:8081
    originRequest:
      httpHostHeader: jenkins.example.com
  - hostname: sonar.example.com
    service: http://localhost:8082
    originRequest:
      httpHostHeader: sonar.example.com
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

- Jenkins : `https://jenkins.example.com`
- SonarQube : `https://sonar.example.com`
- SonarQube directement : `http://127.0.0.1:9000`

Les URLs publiques sont servies en HTTPS par Cloudflare. Configure également
l'URL Jenkins (`https://jenkins.example.com`) dans Jenkins et l'URL de base
SonarQube (`https://sonar.example.com`) dans ses réglages d'administration pour
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

## Mise à jour du serveur

Récupérer la dernière version du dépôt en préservant le `.env` local, puis
reconstruire le service modifié (ici Nginx). `.env` n'est plus suivi par git :
la sauvegarde évite qu'un `git pull` le supprime.

```bash
cp -p .env ../.env.server.backup
git pull --ff-only
cp -p ../.env.server.backup .env

docker compose config --quiet
docker compose build nginx-pipeline
docker compose up -d --no-build --force-recreate nginx-pipeline
docker compose port nginx-pipeline 8082
```

## Mise à jour de SonarQube (9.9 LTS → Community Build)

Pour une nouvelle installation sans données à conserver, cette section ne
s'applique pas : la version de `sonarqube/Dockerfile` crée directement la base.

Les données sont dans PostgreSQL (volume `sonarqube_db`) : ne pas changer la
version majeure de `postgres`. Depuis 9.9 (`lts-community`), passer d'abord par
la 25.12 avant la version cible définie dans `sonarqube/Dockerfile`.

1. Sauvegarder la base :

```bash
docker compose stop sonarqube
docker compose exec sonarqube-db pg_dump -U sonar -d sonarqube -Fc > sonarqube_backup_$(date +%F).dump
```

2. Étape intermédiaire : mettre `FROM sonarqube:25.12.0.117093-community` dans
   `sonarqube/Dockerfile`, puis :

```bash
docker compose build sonarqube
docker compose up -d sonarqube
docker compose logs -f sonarqube
```

   Ouvrir `http://localhost:9000/setup`, cliquer sur **Upgrade** et attendre
   « Database is up-to-date ».

3. Version cible : remettre `FROM sonarqube:26.10.0.133078-community`, relancer
   les mêmes commandes, puis repasser par `/setup`.

Les plugins tiers de `extensions/plugins` peuvent bloquer le démarrage : les
retirer avant la migration. Pour revenir en arrière (image 9.9 + base restaurée) :

```bash
docker compose exec -T sonarqube-db pg_restore -U sonar -d sonarqube --clean < sonarqube_backup_AAAA-MM-JJ.dump
```


# Démarrer tous les services, y compris les outils de sécurité
docker compose --profile security-tools up -d

# Démarrer uniquement Trivy et ses dépendances
docker compose --profile security-tools up -d trivy-server

# Démarrer uniquement ZAP
docker compose --profile security-tools up -d zap