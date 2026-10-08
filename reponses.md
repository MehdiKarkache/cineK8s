# Examen CinéK8s — KARKACHE El Mehdi

## Partie 1
**Q1.1**
`MovieClient` lit la propriété Spring movie.url  (injectée par `@Value("${movie.url}")` dans son constructeur).
On la surcharge sans toucher au code avec la variable d'environnement `MOVIE_URL`, grâce au *relaxed binding* de Spring Boot (majuscules, `.` remplacé par `_`).
**Q1.2**
- (a) Film inexistant → 422 Unprocessable Entity. movie-service répond 404, mais `MovieClient` intercepte ce 404 (`HttpClientErrorException.NotFound`) et renvoie un `Optional` vide, que `TicketController` transforme en 422.
- (b) Pas assez de places → 409 Conflict.
- (c) movie-service injoignable → 503 Service Unavailable (`ResourceAccessException` : DNS, connexion refusée, timeout).
- (Remarque : une demande avec `seats < 1` renvoie 400 Bad Request.)
- 
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

(Invoke-RestMethod http://localhost:8082/actuator/health/liveness).status UP

PS> curl.exe -s -o NUL -w "%{http_code}" -X POST localhost:8082/api/tickets -H "Content-Type: application/json" -d '{"movieId":2,"seats":3}'
503
```
**Q2.1**
On ne modifie pas `application.yaml` parce que ce fichier est compilé dans le jar (puis dans l'image Docker) : ses valeurs par défaut doivent rester les mêmes pour tous les environnements. Le port 8082 est juste un besoin local (8080 est déjà pris par movie). C'est possible grâce à la configuration externalisée de Spring Boot : avec le *relaxed binding*, la variable d'environnement `SERVER_PORT` remplace la propriété `server.port`, et elle est prioritaire sur `application.yaml`.

**Q2.2**
C'est voulu parce que ticket-service lui-même fonctionne bien : seule sa dépendance (movie) est en panne. La liveness reste UP, donc Kubernetes ne le redémarre pas pour rien. La readiness DOWN dit juste « ne m'envoie plus de requêtes pour l'instant ». Quand movie revient, ticket redevient prêt tout seul. Si c'était la liveness qui tombait, les pods ticket redémarreraient en boucle sans que ça répare quoi que ce soit.

## Partie 3

## Partie 4

## Partie 5

## Partie 6

## Partie 7
