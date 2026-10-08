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

## Partie 3

## Partie 4

## Partie 5

## Partie 6

## Partie 7
