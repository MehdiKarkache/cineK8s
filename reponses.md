# Examen CinéK8s — KARKACHE El Mehdi

## Partie 1
**Q1.1**
`MovieClient` lit la propriété Spring `movie.url` (injectée par `@Value("${movie.url}")` dans son constructeur).
On la surcharge sans toucher au code avec la variable d'environnement `MOVIE_URL`, grâce au *relaxed binding* de Spring Boot (majuscules, `.` remplacé par `_`).

**Q1.2**
- (a) Film inexistant → 422 Unprocessable Entity. movie-service répond 404, mais `MovieClient` intercepte ce 404 (`HttpClientErrorException.NotFound`) et renvoie un `Optional` vide, que `TicketController` transforme en 422.
- (b) Pas assez de places → 409 Conflict.
- (c) movie-service injoignable → 503 Service Unavailable (`ResourceAccessException` : DNS, connexion refusée, timeout).
- (Remarque : une demande avec `seats < 1` renvoie 400 Bad Request.)

**Q1.3**

Ligne complétée dans `ticket-service/src/main/resources/application.yaml` :
```yaml
include: readinessState,movie
```
(`movie` est le nom du bean déclaré par `@Component("movie")` sur `MovieHealthIndicator`.)

Si movie-service tombe, une readiness en échec retire seulement les Pods ticket des Endpoints du Service : ils ne reçoivent plus de trafic, le client obtient un 503 propre, et ils redeviennent prêts tout seuls quand movie revient, sans redémarrage.
Dans la liveness, la même panne ferait redémarrer en boucle des Pods ticket parfaitement sains (CrashLoopBackOff), ce qui ne répare rien et propage la panne : une dépendance externe relève donc de la readiness, jamais de la liveness.

**Q1.4**

| Endpoint | Probe(s) | Conséquence si la probe échoue |
|----------|----------|--------------------------------|
| /actuator/health/liveness | startupProbe et livenessProbe | Kubernetes redémarre le conteneur (RESTARTS augmente). La startupProbe sert seulement au démarrage : tant qu'elle n'est pas OK, les deux autres probes ne sont pas lancées. |
| /actuator/health/readiness | readinessProbe | Le pod passe en 0/1 et il est retiré des endpoints du service, donc il ne reçoit plus de requêtes. Par contre il n'est pas redémarré. |

shutdown: graceful : pendant un rolling update, quand un ancien pod est arrêté, Spring termine les requêtes en cours avant de s'éteindre, comme ça les utilisateurs n'ont pas d'erreur.


## Partie 2

Remarque : dans le code fourni, `application.yaml` met movie-service sur le port 8085 et ticket-service sur 8086, alors que l'énoncé suppose 8080 (et `movie.url` pointe sur `localhost:8080`). Pour ne pas modifier le code, j'ai surchargé le port au lancement : movie avec `--server.port=8080`, ticket avec `SERVER_PORT=8082`.

### 2.1
```
POST /api/tickets
{"id":1,"movieId":2,"movieTitle":"Le Seigneur des Pods","seats":3,"total":36.00,"createdAt":"2026-10-08T10:27:23.463253400Z"}

/actuator/health/readiness
{"status":"UP","components":{"movie":{"status":"UP"},"readinessState":{"status":"UP"}}}
```

### 2.2 (movie-service arrêté)
```
/actuator/health/readiness
{"status":"DOWN","components":{"movie":{"status":"DOWN","details":{"error":"I/O error on GET request for \"http://localhost:8080/actuator/health/liveness\": Connection refused: getsockopt"}},"readinessState":{"status":"UP"}}}

/actuator/health/liveness
UP

POST /api/tickets (code HTTP)
503
```

**Q2.1**
On ne modifie pas `application.yaml` parce que ce fichier est compilé dans le jar (puis dans l'image Docker) : ses valeurs par défaut doivent rester les mêmes pour tous les environnements. Le port 8082 est juste un besoin local (8080 est déjà pris par movie). C'est possible grâce à la configuration externalisée de Spring Boot : avec le *relaxed binding*, la variable d'environnement `SERVER_PORT` remplace la propriété `server.port`, et elle est prioritaire sur `application.yaml`.

**Q2.2**
C'est voulu parce que ticket-service lui-même fonctionne bien : seule sa dépendance (movie) est en panne. La liveness reste UP, donc Kubernetes ne le redémarre pas pour rien. La readiness DOWN dit juste « ne m'envoie plus de requêtes pour l'instant ». Quand movie revient, ticket redevient prêt tout seul. Si c'était la liveness qui tombait, les pods ticket redémarreraient en boucle sans que ça répare quoi que ce soit.

## Partie 3

Remarque : comme le code écoute sur 8085/8086, j'ai ajouté `ENV SERVER_PORT=8080` dans le Dockerfile. L'image écoute donc vraiment sur le port déclaré par `EXPOSE 8080`, sans modifier `application.yaml` (relaxed binding), et le même Dockerfile sert pour les deux services.

### 3.1
```
PS> docker images
REPOSITORY:TAG                        IMAGE ID       SIZE
movie-service:1.0.0                   4def387c08a2   331MB
ticket-service:1.0.0                  28f709258f5a   331MB

PS> docker image ls movie-service
IMAGE                 ID             DISK USAGE   CONTENT SIZE
movie-service:1.0.0   4def387c08a2        331MB         95.2MB

PS> docker run --rm --entrypoint id movie-service:1.0.0
uid=10001(spring) gid=101(spring) groups=101(spring)
```
L'image finale ne contient que le JRE et le jar (331 Mo sur disque, 95 Mo compressés), loin des ~700 Mo d'une image avec Maven et le JDK. Elle tourne avec l'utilisateur `spring` (uid 10001), pas en root.

### 3.2
```
PS> docker compose ps
NAME               IMAGE                  STATUS                   PORTS
cinek8s-movie-1    movie-service:1.0.0    Up 9 seconds (healthy)   0.0.0.0:8080->8080/tcp
cinek8s-ticket-1   ticket-service:1.0.0   Up 3 seconds             0.0.0.0:8082->8080/tcp

PS> curl.exe -s localhost:8080/api/movies/whoami
{"environment":"compose","hostname":"f616e7765ad1"}

PS> curl.exe -s -X POST localhost:8082/api/tickets -H "Content-Type: application/json" -d '{"movieId":1,"seats":2}'
{"id":1,"movieId":1,"movieTitle":"Pod Fiction","seats":2,"total":21.00,"createdAt":"2026-10-08T10:49:55.654452647Z"}
```
`"environment":"compose"` montre que la variable `MOVIE_ENVIRONMENT` du compose a bien surchargé `application.yaml`. Ticket joint movie par le nom du service Compose (`http://movie:8080`).

**Q3.1**
Docker met chaque instruction en cache (une couche par instruction). En copiant `pom.xml` seul puis en faisant `dependency:go-offline`, le téléchargement des dépendances Maven est dans une couche qui ne change que si le `pom.xml` change. Si je modifie une seule ligne de Java, seules les couches à partir de `COPY src` sont refaites : Docker recompile le code mais ne retélécharge pas les dépendances, donc le build est beaucoup plus rapide.

**Q3.2**
Avec `-XX:MaxRAMPercentage=75`, la JVM calcule la taille de son heap à partir de la limite mémoire du conteneur (75 % de la limite). La même image s'adapte donc toute seule si on change `limits.memory` dans Kubernetes. Avec `-Xmx512m`, la valeur est fixe : si la limite du conteneur est 512Mi, heap + métaspace + threads dépassent la limite et le conteneur est tué (OOMKilled). Et si on augmente la limite, la JVM ne l'utilise pas.

**Q3.3**
Dans Kubernetes, les pods ticket peuvent démarrer avant movie, mais ce n'est pas grave. Ils démarrent normalement (la liveness est OK), mais leur readiness est DOWN parce que le check `movie` échoue : ils restent en 0/1 et ne sont pas ajoutés aux endpoints du Service, donc ils ne reçoivent pas de trafic. Dès que les pods movie sont prêts, la readiness de ticket passe UP toute seule et ils reçoivent du trafic. C'est la readiness qui remplace le `depends_on` de Compose.

## Partie 4

## Partie 5

## Partie 6

## Partie 7
