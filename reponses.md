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

### 4.0 Chargement des images dans Minikube
Mon Minikube utilise le runtime containerd, donc l option A (`eval $(minikube docker-env)`) ne fonctionne pas. J'ai utilisé l'option C : réutiliser les images construites en Partie 3.
```
PS> minikube image load movie-service:1.0.0
PS> minikube image load ticket-service:1.0.0
PS> minikube image ls | Select-String "movie|ticket"
docker.io/library/ticket-service:1.0.0
docker.io/library/movie-service:1.0.0
```

### 4.4 Déploiement et vérifications
```
PS> kubectl apply -f k8s/
namespace/cinema-exam created
configmap/movie-config created
configmap/ticket-config created
deployment.apps/movie created
service/movie created
deployment.apps/ticket created
service/ticket created

PS> kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
movie-59684459f4-2wcdh    1/1     Running   0          52s
movie-59684459f4-9kgsz    1/1     Running   0          52s
ticket-66d95c98b6-7zh2g   1/1     Running   0          52s
ticket-66d95c98b6-nqfqg   1/1     Running   0          52s

PS> kubectl get endpoints movie ticket
NAME     ENDPOINTS                           AGE
movie    10.244.0.13:8080,10.244.0.15:8080   52s
ticket   10.244.0.14:8080,10.244.0.16:8080   52s

PS> kubectl exec deploy/ticket -- wget -qO- http://movie:8080/api/movies/whoami
{"hostname":"movie-59684459f4-9kgsz","environment":"kubernetes"}

PS> kubectl exec deploy/ticket -- wget -qO- http://localhost:8080/actuator/health/readiness
{"status":"UP","components":{"movie":{"status":"UP"},"readinessState":{"status":"UP"}}}

PS> kubectl port-forward svc/ticket 8082:8080      (dans un autre terminal)
PS> curl.exe -s -X POST localhost:8082/api/tickets -H "Content-Type: application/json" -d '{"movieId":2,"seats":2}'
{"id":1,"movieId":2,"movieTitle":"Le Seigneur des Pods","seats":2,"total":24.00,"createdAt":"2026-10-08T10:53:52.294224878Z"}
```
Les 4 pods sont prêts, chaque Service a 2 endpoints, ticket joint movie par le nom du Service (`http://movie:8080`), `"environment":"kubernetes"` vient de la ConfigMap `movie-config`, et la réservation donne bien `24.00`.

**Q4.1**
`kubectl apply -f k8s/` traite les fichiers par ordre alphabétique. Les préfixes `00-`, `10-`, `20-`… garantissent l'ordre des dépendances : le namespace est créé avant les objets qui sont dedans (sinon erreur `namespaces "cinema-exam" not found`), et les ConfigMaps avant les Deployments qui les utilisent (sinon les pods seraient en `CreateContainerConfigError` au début).

**Q4.2**
C'est la startupProbe. Pendant le démarrage de Spring Boot (environ 30 s), `/actuator/health/liveness` ne répond pas encore, donc la startupProbe échoue et la readiness n'est pas encore lancée : le pod reste en 0/1. Ce n'est pas une anomalie, `RESTARTS` reste à 0 et les pods passent 1/1 dès que l'appli a démarré. On le voit dans les events :
```
PS> kubectl get events --field-selector reason=Unhealthy
movie-59684459f4-2wcdh   Startup probe failed: Get "http://10.244.0.13:8080/actuator/health/liveness": dial tcp 10.244.0.13:8080: connect: connection refused
```

**Q4.3**
Avec `imagePullPolicy: Always`, le kubelet essaierait à chaque démarrage de télécharger `movie-service:1.0.0` depuis Docker Hub (`docker.io/library/movie-service`). Or cette image n'existe que localement dans Minikube (chargée avec `minikube image load`), donc on aurait `ErrImagePull` puis `ImagePullBackOff`. `IfNotPresent` utilise l'image déjà présente sur le nœud.

## Partie 5

### 5.1 Préparation
L'addon ingress était déjà activé. Sous Windows avec le driver Docker, `minikube ip` n'est pas joignable : j'ai lancé `minikube tunnel` dans un terminal dédié et ajouté `127.0.0.1 cinema.local` dans `C:\Windows\System32\drivers\etc\hosts` (équivalent de `/etc/hosts`).
```
PS> kubectl get pods -n ingress-nginx
NAME                                       READY   STATUS    RESTARTS       AGE
ingress-nginx-controller-d7cd8c989-j5kk9   1/1     Running   4 (109m ago)   46h
```

### 5.2 Ingress (`k8s/40-ingress.yaml`)
```
PS> kubectl describe ingress cinema
  Host          Path  Backends
  cinema.local
                /api/movies    movie:http (10.244.0.13:8080,10.244.0.15:8080)
                /api/tickets   ticket:http (10.244.0.14:8080,10.244.0.16:8080)
```

### 5.3 Tests
```
PS> (Invoke-RestMethod http://cinema.local/api/movies).title
Pod Fiction
Le Seigneur des Pods
Docker Wars
Rollback to the Future

PS> curl.exe -s -X POST http://cinema.local/api/tickets -H "Content-Type: application/json" -d '{"movieId":3,"seats":10}'
{"id":3,"movieId":3,"movieTitle":"Docker Wars","seats":10,"total":90.00,"createdAt":"2026-10-08T11:03:17.221366646Z"}

PS> 1..6 | ForEach-Object { (Invoke-RestMethod http://cinema.local/api/movies/whoami).hostname }
movie-59684459f4-2wcdh
movie-59684459f4-9kgsz
movie-59684459f4-2wcdh
movie-59684459f4-9kgsz
movie-59684459f4-2wcdh
movie-59684459f4-9kgsz

PS> curl.exe -s -o NUL -w "%{http_code}" http://cinema.local/actuator/health
404
```

**Q5.1**
Deux pods movie différents ont répondu, en alternance (`movie-59684459f4-2wcdh` et `movie-59684459f4-9kgsz`). C'est le Service `movie` qui répartit la charge : il regroupe les pods qui ont le label `app: movie` dans ses endpoints, et l'Ingress Controller nginx envoie les requêtes à tour de rôle vers ces endpoints.

**Q5.2**
Avec `pathType: Exact`, seule l'URL exacte `/api/movies` correspondrait à la règle. `GET /api/movies/1` (et aussi `/api/movies/whoami`) ne correspondrait à aucune règle, donc l'Ingress répondrait 404. Avec `Prefix`, tout ce qui commence par `/api/movies` est routé vers movie.

**Q5.3**
On obtient 404, parce qu'aucune règle de l'Ingress ne couvre `/actuator`. C'est souhaitable : les endpoints Actuator (santé avec `show-details: always`, infos internes) ne doivent pas être exposés publiquement. Les probes n'en ont pas besoin, puisque le kubelet appelle directement l'IP du pod, sans passer par l'Ingress. On expose uniquement les routes métier `/api/movies` et `/api/tickets`.

## Partie 6

### 6.1 Le service movie disparaît

**Prédictions (écrites avant de lancer les commandes)**
- (a) Les pods ticket passent en `0/1` (readiness DOWN car le check `movie` échoue), mais `RESTARTS` reste à 0 (liveness toujours OK).
- (b) `kubectl get endpoints ticket` : plus aucune adresse (`<none>`), les pods non prêts sont retirés du Service.
- (c) `GET http://cinema.local/api/tickets` : 503, renvoyé par l'Ingress parce que le Service ticket n'a plus d'endpoint.
- (d) La liveness de ticket reste `UP`.

**Observations**
```
PS> kubectl scale deploy/movie --replicas=0
deployment.apps/movie scaled

PS> kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
ticket-66d95c98b6-7zh2g   0/1     Running   0          17m
ticket-66d95c98b6-nqfqg   0/1     Running   0          17m

PS> kubectl get endpoints ticket
NAME     ENDPOINTS   AGE
ticket               17m

PS> curl.exe -si http://cinema.local/api/tickets | Select-Object -First 1
HTTP/1.1 503 Service Temporarily Unavailable

PS> kubectl get events --field-selector reason=Unhealthy -o custom-columns=OBJET:.involvedObject.name,MESSAGE:.message
ticket-66d95c98b6-7zh2g   Readiness probe failed: HTTP probe failed with statuscode: 503
ticket-66d95c98b6-nqfqg   Readiness probe failed: HTTP probe failed with statuscode: 503

PS> kubectl exec deploy/ticket -- wget -qO- http://localhost:8080/actuator/health/liveness
{"status":"UP"}
```
Retour à la normale, sans rien toucher à ticket :
```
PS> kubectl scale deploy/movie --replicas=2
PS> kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
movie-59684459f4-5l67c    1/1     Running   0          23s
movie-59684459f4-vtfxt    1/1     Running   0          23s
ticket-66d95c98b6-7zh2g   1/1     Running   0          18m
ticket-66d95c98b6-nqfqg   1/1     Running   0          18m

PS> kubectl get endpoints ticket
NAME     ENDPOINTS                           AGE
ticket   10.244.0.14:8080,10.244.0.16:8080   18m
```
Les 4 prédictions sont vérifiées. Ticket est redevenu prêt environ 5 s après movie, avec `RESTARTS` toujours à 0, et `/api/tickets` répond de nouveau 200.

**Q6.1**
1. movie est à 0 réplica, donc le `MovieHealthIndicator` de ticket n'arrive plus à joindre `http://movie:8080` : `/actuator/health/readiness` de ticket passe DOWN (HTTP 503).
2. La readinessProbe de ticket échoue plusieurs fois de suite (`Readiness probe failed: … statuscode: 503`), donc Kubernetes marque les pods ticket NotReady (0/1).
3. Les pods NotReady sont retirés des endpoints du Service ticket : `kubectl get endpoints ticket` est vide.
4. L'Ingress n'a plus aucun backend pour `/api/tickets`, donc il répond lui-même 503 Service Temporarily Unavailable, une erreur propre au lieu d'erreurs applicatives.

`RESTARTS` est resté à 0 parce que la liveness (et la startup) de ticket ne dépend pas de movie : ticket lui-même fonctionne, sa liveness reste UP, donc le kubelet n'a aucune raison de redémarrer le conteneur. La dépendance est seulement dans la readiness (Q1.3).

### 6.2 Mission dépannage (`broken/ticket-debug.yaml`)

| # | Statut observé | Commande de diagnostic | Cause exacte | Correction apportée |
|---|----------------|------------------------|--------------|---------------------|
| 1 | `ErrImagePull` puis `ImagePullBackOff` | `kubectl describe pod <pod>` (Events) | `imagePullPolicy: Always` : le kubelet essaie de télécharger `docker.io/library/ticket-service:1.0.0` (« pull access denied, repository does not exist »), alors que l'image n'existe que dans Minikube | `imagePullPolicy: IfNotPresent` |
| 2 | `CreateContainerConfigError` | `kubectl describe pod <pod>` (Events) + `kubectl get cm` | `configmap "ticket-configmap" not found` : la ConfigMap s'appelle `ticket-config` | `configMapRef.name: ticket-config` |
| 3 | `Running` mais `0/1` en permanence | `kubectl describe pod <pod>` (Events) + `kubectl logs <pod>` | `Readiness probe failed: … :8081 … connection refused` : la probe vise le port 8081, alors que l'appli écoute sur 8080 (`Tomcat started on port 8080`) | readinessProbe sur `port: http` (le port nommé = 8080) |

Détail des messages relevés :
```
1) Failed to pull image "ticket-service:1.0.0": … "docker.io/library/ticket-service:1.0.0": pull access denied, repository does not exist or may require authorization
2) Error: configmap "ticket-configmap" not found
3) Readiness probe failed: Get "http://10.244.0.21:8081/actuator/health/readiness": dial tcp 10.244.0.21:8081: connect: connection refused
```
Résultat après les 3 corrections (uniquement dans le YAML, pas dans le code Java) :
```
PS> kubectl get pods -l app=ticket-debug
NAME                            READY   STATUS    RESTARTS   AGE
ticket-debug-77fbf56679-h7ct6   1/1     Running   0          21s

PS> kubectl delete -f broken/ticket-debug.yaml
deployment.apps "ticket-debug" deleted from cinema-exam namespace
```
Les erreurs apparaissent bien l'une après l'autre : l'image doit d'abord être récupérée, puis la config du conteneur (ConfigMap) est construite au démarrage, et la readiness n'est testée qu'une fois le conteneur lancé.

### 6.3 Changer la configuration sans rebuild
```
PS> kubectl apply -f k8s/10-config.yaml
configmap/movie-config configured
configmap/ticket-config unchanged

PS> curl.exe -s http://cinema.local/api/movies/whoami
{"hostname":"movie-59684459f4-5l67c","environment":"kubernetes"}

PS> kubectl rollout restart deploy/movie
PS> kubectl rollout status deploy/movie
deployment "movie" successfully rolled out

PS> curl.exe -s http://cinema.local/api/movies/whoami
{"environment":"production","hostname":"movie-67fd5fbdfc-bmrxl"}
```

**Q6.3**
La ConfigMap est injectée avec `envFrom`, donc sous forme de variables d'environnement. Elles sont lues une seule fois, au démarrage du conteneur : modifier la ConfigMap ne change rien dans les pods déjà lancés, d'où `"kubernetes"` juste après le `apply`. C'est le `kubectl rollout restart` qui a rendu la modification effective : il recrée les pods (rolling update), et les nouveaux pods lisent la nouvelle valeur `production` au démarrage.

## Partie 7

**Q7.1**
1. Le pod ticket demande au DNS l'adresse de `movie`. Son `/etc/resolv.conf` pointe vers CoreDNS (le DNS du cluster) et contient le domaine de recherche `cinema-exam.svc.cluster.local`, donc `movie` est complété en `movie.cinema-exam.svc.cluster.local`.
2. CoreDNS répond avec la ClusterIP du Service `movie` : une IP virtuelle et stable, qui ne correspond à aucun pod.
3. Ticket envoie la requête HTTP vers ClusterIP:8080. Les règles réseau installées par kube-proxy sur le nœud interceptent ce trafic et le redirigent vers un des endpoints du Service, c'est-à-dire l'IP:8080 d'un pod `app: movie` prêt (choisi au hasard parmi les pods Ready).
4. Le pod movie choisi reçoit `GET /api/movies/1` sur son port `http` (8080) et répond. Si un pod movie disparaît ou n'est pas prêt, il n'est plus dans les endpoints et ne reçoit plus de requêtes.

Vérification dans un pod ticket :
```
PS> kubectl exec deploy/ticket -- cat /etc/resolv.conf
search cinema-exam.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10

PS> kubectl exec deploy/ticket -- nslookup movie.cinema-exam.svc.cluster.local
Name:    movie.cinema-exam.svc.cluster.local
Address: 10.109.96.41

PS> kubectl get svc movie
NAME    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
movie   ClusterIP   10.109.96.41   <none>        8080/TCP   59m
```
`10.96.0.10` est l'IP du Service `kube-dns` (CoreDNS), et le nom `movie` est bien résolu en ClusterIP du Service movie.

**Q7.2**
```
PS> 1..4 | ForEach-Object { curl.exe -s -X POST http://cinema.local/api/tickets -H "Content-Type: application/json" -d '{"movieId":3,"seats":1}' }
{"id":1,"movieId":3,"movieTitle":"Docker Wars","seats":1,"total":9.00,…}
{"id":4,"movieId":3,"movieTitle":"Docker Wars","seats":1,"total":9.00,…}
{"id":5,"movieId":3,"movieTitle":"Docker Wars","seats":1,"total":9.00,…}
{"id":6,"movieId":3,"movieTitle":"Docker Wars","seats":1,"total":9.00,…}

PS> (appel répété de GET http://cinema.local/api/tickets, nombre de tickets renvoyés)
6 tickets : ids = 1,2,3,4,5,6
6 tickets : ids = 1,2,3,4,5,6
1 tickets : ids = 1
1 tickets : ids = 1
1 tickets : ids = 1
6 tickets : ids = 1,2,3,4,5,6

PS> kubectl delete pod -l app=ticket
PS> curl.exe -s http://cinema.local/api/tickets
[]
```
Le nombre varie parce que les réservations sont stockées en mémoire, dans une liste propre à chaque instance (`tickets` dans `TicketController`). Il y a 2 pods ticket, chacun avec sa propre liste, et le Service répartit les requêtes entre eux : selon le pod qui répond, on voit 1 ou 6 réservations (on voit aussi que les ids se répètent, chaque pod a son propre compteur). Quand on supprime les pods ticket, les nouveaux pods repartent de zéro : toutes les réservations sont perdues (`[]`).
La solution architecturale est de rendre ticket-service sans état : stocker les réservations dans une base de données externe partagée (par exemple PostgreSQL, avec un volume persistant PVC), à laquelle tous les pods se connectent. Les pods deviennent alors interchangeables et jetables, et les données survivent aux redémarrages et au scaling.

**Q7.3**
```
PS> kubectl delete pod/movie-67fd5fbdfc-bmrxl
PS> kubectl get pods -l app=movie
NAME                     READY   STATUS    RESTARTS   AGE
movie-67fd5fbdfc-szdbw   1/1     Running   0          37m
movie-67fd5fbdfc-zrh44   0/1     Running   0          1s
(quelques secondes plus tard)
movie-67fd5fbdfc-szdbw   1/1     Running   0          37m
movie-67fd5fbdfc-zrh44   1/1     Running   0          6s
```
Un nouveau pod (`movie-67fd5fbdfc-zrh44`, avec un nouveau nom et une nouvelle IP) est recréé immédiatement : le ReplicaSet du Deployment voit 1 pod au lieu des 2 demandés et corrige l'écart (boucle de réconciliation). Pendant ce temps, l'autre pod continue de répondre, donc pas de coupure. Avec un `kind: Pod` « nu », personne ne l'aurait recréé : le pod aurait disparu définitivement, et on aurait aussi perdu le scaling (`replicas`), le rolling update et le rollback.

## Bonus

### B1 — Durcir le Deployment movie
J'ai ajouté au conteneur `movie` (dans `k8s/20-movie.yaml`) un `securityContext` : `runAsNonRoot: true`, `runAsUser: 10001`, `allowPrivilegeEscalation: false`, `capabilities.drop: ["ALL"]` et `readOnlyRootFilesystem: true`.

Premier essai avec seulement le `securityContext` : le nouveau pod plante, mais les anciens pods continuent de servir (rolling update).
```
PS> kubectl get pods -l app=movie
NAME                     READY   STATUS             RESTARTS      AGE
movie-67fd5fbdfc-szdbw   1/1     Running            0             57m
movie-67fd5fbdfc-zrh44   1/1     Running            0             20m
movie-7ff589bbfb-mvdj6   0/1     CrashLoopBackOff   2 (14s ago)   40s

PS> kubectl logs movie-7ff589bbfb-mvdj6
org.springframework.context.ApplicationContextException: Unable to start web server
Caused by: org.springframework.boot.web.server.WebServerException: Unable to create tempDir. java.io.tmpdir is set to /tmp
```
Tomcat a besoin d ecrire un dossier temporaire dans `/tmp`, ce qui est impossible avec une racine en lecture seule. J'ai donc ajouté un volume `emptyDir` monté sur `/tmp` (`volumes` au niveau du pod, `volumeMounts` au niveau du conteneur).

Vérification :
```
PS> kubectl get pods -l app=movie
NAME                    READY   STATUS    RESTARTS   AGE
movie-f747689d6-8n5r2   1/1     Running   0          16s
movie-f747689d6-rxwjr   1/1     Running   0          9s

PS> kubectl exec deploy/movie -- id
uid=10001(spring) gid=101(spring) groups=101(spring)

PS> kubectl exec deploy/movie -- touch /test
touch: cannot touch '/test': Read-only file system

PS> kubectl exec deploy/movie -- sh -c 'touch /tmp/ok && echo /tmp inscriptible'
/tmp inscriptible
```
Le conteneur tourne avec l'uid 10001, la racine est en lecture seule, seul `/tmp` (emptyDir) est inscriptible, et les pods sont `1/1`. L'application répond toujours normalement via `cinema.local`.
