# Note technique : sécuriser les ports d'administration

## Adresse de publication des ports

Dans une règle Compose, `127.0.0.1:8080:8080` publie le port uniquement sur
l'interface locale du serveur. `0.0.0.0:8080:8080` (ou `8080:8080`) le rend
accessible sur toutes les interfaces réseau, sous réserve des règles du pare-feu.

## Configuration de cette stack

| Service | Publication | Accès |
| --- | --- | --- |
| Jenkins | Aucun port hôte | Via Nginx sur le port 80 |
| Nginx | `8081:80` | Port 8081 de l'hôte vers le port 80 du conteneur |
| SonarQube | `127.0.0.1:9000:9000` | Depuis le serveur uniquement |
| PostgreSQL | Aucun port hôte | Réseau Docker uniquement |
| OWASP ZAP | Aucun port hôte | Profil optionnel, réseau Docker uniquement |

Jenkins n'est pas publié sur le port 8080 de l'hôte. Nginx le joint avec le nom
de service Compose `jenkins` sur le réseau `pipeline`. Depuis l'extérieur,
ouvrez `http://<IP-du-serveur>:8081/`. Depuis le serveur, SonarQube est accessible sur
`http://127.0.0.1:9000`.

Le port 443 n'est pas activé tant qu'un certificat TLS et sa configuration Nginx
ne sont pas installés. Ne publiez pas ce port comme HTTPS sans configurer TLS.

## Proxy Jenkins

Nginx et Jenkins tournent tous les deux dans Docker. Il faut donc utiliser le nom
du service `jenkins` comme adresse amont, pas `127.0.0.1` (qui désignerait le
conteneur Nginx lui-même).



