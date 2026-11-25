# TP4 Spring Cloud : microservices « Calculatrice » avec Eureka, Gateway et OpenFeign

- **Matière** : Architecture MicroServices, **Filière** : MPSRCC1
- **Projet IntelliJ** : `calculatrice-spring-cloud` (Spring Boot 4.0.9, Spring Cloud 2025.1.2)
- **Nom et prénom** : *Bey Aya*

## 1. Structure et ports

```
calculatrice-spring-cloud/
|-- pom.xml                  <- pom parent
|-- eureka-service/          <- annuaire (8761)
|-- TP3-AMS/                 <- calcul-service (8081 et 8083)
|-- historique-service/      <- historique (8082)
`-- gateway-service/         <- passerelle (8888)
```

Ordre de lancement : eureka-service, calcul-service (8081 puis 8083), historique-service, gateway-service.

## 2. Partie A : questions de conception

1. **Composant qui fait la correspondance nom/adresses ?** L'annuaire Eureka Server. Les services s'y inscrivent au démarrage, envoient un battement de cœur régulier et sont retirés quand il cesse ; les clients rafraîchissent périodiquement leur copie locale.
2. **Point d'entrée unique ?** Avantage : le client ne connaît qu'une adresse (8888). Inconvénient : point de défaillance unique et goulot d'étranglement possible.
3. **Six requêtes, deux instances ?** 3 requêtes par instance (tour de rôle). Le client ne le voit pas, seules les traces des consoles le montrent.

## 3. Partie B : eureka-service

Module avec la dépendance Eureka Server, `@EnableEurekaServer`, port 8761.

![Tableau de bord Eureka vide](images/01.png)

**Question B.1.** Chaque programme a sa propre JVM, son propre Tomcat et son propre port (8761, 8081, 8082) : il n'y a pas de conflit.

## 4. Partie C : enregistrement de calcul-service

Client Eureka, springdoc et Lombok ajoutés au pom ; `application.properties` avec nom, port 8081 et adresse de l'annuaire ; trace du port dans chaque méthode de `CalculController`.

CALCUL-SERVICE UP (1) :

![Eureka : CALCUL-SERVICE UP (1)](images/04.png)

![Swagger UI de calcul-service sur le port 8081](images/02.png)

Test `GET /calcul/somme` avec `a = 3`, `b = 2` (réponse 5) et trace dans la console 8081 :

![Swagger UI : somme 3+2 = 5](images/06.png)

![Console calcul-service 8081 : trace somme 3+2](images/07.png)

### C.5 : seconde instance (port 8083)

![Eureka : CALCUL-SERVICE UP (2)](images/08.png)

![Swagger UI 8083 : produit 6x7 = 42](images/09.png)

![Console 8083 : trace produit 6*7](images/10.png)

**Question C.1.** Le nom `CALCUL-SERVICE` est commun : c'est le nom logique pour obtenir toutes les instances. L'identifiant d'instance (`calcul-service:8083`) est unique : il distingue chaque exemplaire dans l'annuaire.

**Question C.2.** Le port 8081 serait déjà occupé : Tomcat ne démarrerait pas et l'instance ne s'inscrirait pas.

### C.6 : arrêt d'une instance

Après l'arrêt de l'instance 8083 (UP (1)), puis après sa relance (UP (2)) :

![Eureka après l'arrêt de l'instance 8083](images/11.png)

![Eureka après la relance de l'instance 8083](images/12.png)

Un arrêt propre désinscrit l'instance immédiatement ; un arrêt brutal la fait retirer après 10 s sans battement de cœur.

## 5. Partie D : OpenFeign

`@EnableFeignClients`, `CalculClient` devient une interface `@FeignClient(name = "CALCUL-SERVICE")`, et `OperationController` gère `FeignException.BadRequest` (400) puis `FeignException` (503).

### D.4 : tests de historique-service

| N° | Requête | Corps | Attendu | Capture |
|---|---|---|---|---|
| 1 | `POST /operations` | somme 8+2 | 201, 10.0 | ci-dessous |
| 2 | `POST /operations` | produit 6×7 | 201, 42.0 | ci-dessous |
| 3 | `POST /operations` | division 7/2 | 201, 3.5 | ci-dessous |
| 4 | `POST /operations` | division 7/0 | 400, division par 0 | *(à ajouter)* |
| 5 | `POST /operations` | opérateur `racine` | 400, opérateur inconnu | *(à ajouter)* |
| 6 | `GET /operations` | aucun | 200, liste | *(à ajouter)* |
| 7 | `GET /operations/2` | aucun | 200, opération 2 | *(à ajouter)* |

![Requête 1 : POST somme 8+2](images/15.png)

![Requête 2 : POST produit 6x7](images/14.png)

![Requête 3 : POST division 7/2](images/16.png)

![Eureka : CALCUL-SERVICE UP (2) et HISTORIQUE-SERVICE UP (1)](images/13.png)

Traces alternées dans les consoles 8081 et 8083 : *(à ajouter)*.

**Question D.1.** Les traces alternent entre 8081 et 8083. La stratégie s'appelle le tour de rôle (round robin). *(Captures à ajouter.)*

**Question D.2.** Le nom `CALCULSERVIC` n'existe pas dans l'annuaire : LoadBalancer répond 503. Le statut obtenu est **503**, produit par le second bloc `catch (FeignException e)`. *(Capture à ajouter.)*

## 6. Partie E : gateway-service

Passerelle Spring Cloud Gateway (WebFlux, Netty) sur le port 8888, inscrite dans l'annuaire.

- **E.2** : routes statiques en YAML avec adresses fixes (l'instance 8083 est ignorée). *(captures à ajouter)*
- **E.3** : routes `lb://CALCUL-SERVICE` et `lb://HISTORIQUE-SERVICE` : les traces alternent entre 8081 et 8083. *(captures à ajouter)*
- **E.4** : mêmes routes en Java (`RouteLocator`). *(captures à ajouter)*
- **E.5** : routes dynamiques créées par le locator à partir de l'annuaire.

![Swagger UI de la passerelle : définition calcul-service et zone Servers](images/21.png)

![Requête somme via la passerelle](images/19.png)

![Description OpenAPI de calcul-service via la passerelle](images/20.png)

![Routes dynamiques de la passerelle](images/22.png)

**Question E.1.** La passerelle ajoute les en-têtes `X-Forwarded-*` ; avec `server.forward-headers-strategy=framework`, historique-service reconstruit l'URL du `Location` à partir de l'adresse du client (port 8888).

**Question E.2.** E.3 : URI propres (`/calcul/somme`), mais il faut modifier la passerelle à chaque nouveau service. E.5 : aucune configuration (route automatique), mais le nom du service apparaît dans l'URI (`/calcul-service/calcul/somme`).

**Question E.3.** La passerelle est aussi inscrite dans Eureka (`GATEWAY-SERVICE`), donc le locator crée une route vers elle-même. Peu gênant dans le TP, mais à éviter en production (elle exposerait ses propres endpoints).

## 7. Questions de synthèse

1. **Étapes du tableau A.3** : nombre de programmes (B, C.5, E.1) ; dépendances (C.1, C.7, E.1) ; adresse du service (C.7, D.2) ; appel entre services (D.2) ; erreurs d'appel (D.3, D.4) ; point d'entrée (E.2, E.3) ; plusieurs instances (C.5, D.4, E.3) ; serveur embarqué (E.1, Netty) ; arrêt d'un service (C.6, E.5) ; organisation du code (B.0, C.0) ; outil de test (C.4, E.2) ; classes de données (C.1 bis, C.7).
2. **Trajet de `POST :8888/historique-service/operations`** : client → passerelle (route `/historique-service/**`, instance 8082 trouvée dans la copie locale de l'annuaire, préfixe retiré) → historique-service (`creer`, appel Feign vers `CALCUL-SERVICE`, instance 8081 ou 8083) → calcul-service répond `3` → historique-service enregistre et répond 201 → passerelle → client. Trois requêtes HTTP, trois réponses.
3. **Trois LoadBalancer** : gateway-service (`CALCUL-SERVICE`, `HISTORIQUE-SERVICE`), historique-service (`CALCUL-SERVICE`), calcul-service (bibliothèque présente, mais aucun nom résolu).
4. **Dépendances** : client Eureka dans les trois (inscription, et découverte pour historique et passerelle) ; OpenFeign seulement dans historique-service (il appelle calcul-service) ; Gateway seulement dans gateway-service.
5. **Adresse en dur** : un seul exemplaire de calcul-service est appelé quel que soit le nombre d'instances, et un changement d'adresse oblige à reconfigurer. Résolu par Eureka (annuaire), le client Eureka, LoadBalancer (tour de rôle), OpenFeign (appel par nom) et la passerelle (`lb://`).
6. **Deux instances de historique-service** : chaque instance a sa mémoire, donc le GET ne renvoie qu'une partie des opérations. Principe mis en défaut : service sans état (stateless). Correction : base de données partagée.
7. **Sans IntelliJ** : `mvn clean package -DskipTests` à la racine, puis `java -jar` pour eureka-service, calcul-service, calcul-service avec `--server.port=8083`, historique-service et gateway-service, dans cet ordre.
8. **`@GetMapping("/calcul/{operateur}")`** : dans le contrôleur, il décrit une requête reçue ; dans l'interface Feign, il décrit une requête envoyée.
9. **Modules et indépendance** : l'indépendance reste entière (JVM, port, `.jar` et cycle de vie propres) ; le regroupement n'est qu'un confort de développement.
10. **Zone « Servers »** : elle vient des en-têtes `X-Forwarded-*` (et `X-Forwarded-Prefix` en E.5) lus grâce à `server.forward-headers-strategy=framework`. Sans cela, Swagger afficherait l'adresse directe du microservice et contournerait la passerelle.
11. **Lombok** : les méthodes générées sont dans `Operation.class` (`target/classes`). Lombok n'agit qu'à la compilation, donc il peut être exclu du `.jar`.

## 8. Remarques sur la configuration

- L'annuaire doit être lancé en premier, sinon `Connection refused` vers `localhost:8761`.
- Un caractère en trop après l'accolade fermante du corps JSON dans Swagger (un `*` copié) provoque un `400 Bad Request` standard de Spring.
- Spring Cloud Gateway 5 : préfixe `spring.cloud.gateway.server.webflux` ; `management.endpoint.gateway.access=read-only` pour afficher `/actuator/gateway/routes` ; `trusted-proxies` (valeur `.*` en local) pour que les en-têtes `X-Forwarded-*` soient transmis.

