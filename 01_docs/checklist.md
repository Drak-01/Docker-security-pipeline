# 1. Syntaxe, depuis le conteneur (ne valide pas la résolution DNS, c'est voulu)
docker compose exec nginx-pipeline nginx -t

# 2. Host inconnu : doit être coupé (curl: Empty reply / 52)
curl -si -m 5 http://127.0.0.1:8081/

# 3. Bon Host : doit renvoyer 200/403 de Jenkins
curl -sI -m 5 -H 'Host: jenkins.hakilox.com' http://127.0.0.1:8081/login | head -n1

# 4. SonarQube via nginx
curl -s -m 5 -H 'Host: sonar.hakilox.com' http://127.0.0.1:8082/api/system/status

# 5. Santé nginx
docker compose exec nginx-pipeline wget -qO- http://127.0.0.1:80/healthz

# 6. Rate limit : les premières requêtes passent, puis 429
for i in $(seq 1 20); do
  curl -s -o /dev/null -w '%{http_code}\n' -X POST -m 5 \
    -H 'Host: sonar.hakilox.com' http://127.0.0.1:8082/api/authentication/login -d 'login=x&password=x'
done

# 7. Résilience : recrée Jenkins et vérifie qu'il n'y a plus de 502 durable
docker compose up -d --force-recreate jenkins
sleep 60
curl -sI -m 5 -H 'Host: jenkins.hakilox.com' http://127.0.0.1:8081/login | head -n1