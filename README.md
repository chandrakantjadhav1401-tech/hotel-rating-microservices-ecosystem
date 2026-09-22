🏨 Hotel & Rating Microservices EcosystemAn enterprise-grade, distributed hotel and rating management platform built using Java 21, Spring Boot, Spring Cloud, Resilience4j, and polyglot persistence (MySQL and PostgreSQL).This central repository brings together all 6 microservices: service orchestration, dynamic discovery, centralized Git configuration, load-balanced edge routing, and fault-tolerant patterns.📑 Repository Structure & Serviceshotel-rating-microservices-ecosystem/
├── SBMS-Config-server/     # Centralized Git-backed External Configuration (:8085)
├── ServiceRegistry_3/      # Netflix Eureka Service Discovery & Heartbeats (:8761)
├── API-GATEWAY/            # Spring Cloud Gateway edge router & reverse proxy (:8084)
├── UserService/            # Main aggregator & orchestrator service (:8081) [MySQL]
├── RatingService/          # Review & rating management service (:8083) [MySQL]
├── HotelService/           # Hotel catalog and room directory service (:8082) [PostgreSQL]
├── screenshots/            # Postman & Eureka verification screenshots
└── README.md               # Main repository documentation (this file)
🏛️ System Architecture                       +----------------------------------------------------+
                       |         SBMS Config Server (:8085)                 |
                       | (Fetches configs from Git: Microservices-Config-Server) |
                       +-------------------------+--------------------------+
                                                 |
                                                 v
                       +----------------------------------------------------+
                       |          Service Registry - Eureka (:8761)         |
                       |       (Service Discovery, Health Check & Dynamic IP)|
                       +----+--------------------+--------------------+-----+
                            |                    |                    |
                            v                    |                    |
                 +----------------------+        |                    |
                 |  API Gateway (:8084) |        |                    |
                 +----------+-----------+        |                    |
                            |                    |                    |
               +------------+------------+       |                    |
               |                         |       |                    |
               v                         v       v                    v
    +--------------------+    +--------------------+    +--------------------+
    | UserService (:8081)|    |RatingService(:8083)|    | HotelService(:8082)|
    |    (Orchestrator)  |    |     (Feedback)     |    |    (Catalog)       |
    +----------+---------+    +----------+---------+    +----------+---------+
               |                         |                         |
               v                         v                         v
     [(MySQL: college)]        [(MySQL: college)]       [(Postgres: microservice)]
🔄 End-to-End Execution Workflow[Client / Postman / Frontend]
           │
           │ 1. GET /users/user/{userId}
           ▼
+-----------------------+
|  API Gateway (:8084)  |
+-----------┬-----------+
            │
            │ 2. Routes via Eureka load balancer to USER-SERVICE
            ▼
+-----------------------+
|  UserService (:8081)  |
+-----------┬-----------+
            │
            │ 3. Fetches basic user profile from MySQL (college DB)
            │
            │ 4. Calls RatingService using Load-Balanced RestTemplate / FeignClient
            ▼
+-----------------------+
| RatingService (:8083) | ───► Queries MySQL (college DB) ──► Returns List<Rating>
+-----------┬-----------+
            │
            │ 5. Returns ratings array to UserService
            ▼
+-----------------------+
|  UserService (:8081)  |
            │
            │ 6. Iterates over ratings; calls HotelService using OpenFeign for each hotelId
            ▼
+-----------------------+
|  HotelService (:8082) | ───► Queries PostgreSQL (microservice DB) ──► Returns Hotel
+-----------┬-----------+
            │
            │ 7. Returns hotel metadata to UserService
            ▼
+-----------------------+
|  UserService (:8081)  | ───► Injects Hotel object into Rating and aggregates into User
+-----------┬-----------+
            │
            ▼
[Client Receives Complete Aggregated Response]
🛡️ Resilience4j Fault ToleranceDownstream communication in UserService is protected against cascading failures:@GetMapping("/user/{userId}")
@CircuitBreaker(name = "ratingHotelBreaker", fallbackMethod = "ratingHotelFallback")
@Retry(name = "ratingHotelService", fallbackMethod = "ratingHotelFallback")
@RateLimiter(name = "userRateLimiter", fallbackMethod = "ratingHotelFallback")
public ResponseEntity<User> getSingleUser(@PathVariable String userId) { ... }
Circuit Breaker: Transitions across CLOSED, OPEN, and HALF-OPEN states to isolate failing services.Retry: Automatically re-executes calls on transient network failures.Rate Limiter: Guards against traffic spikes.Fallback: Gracefully returns the profile with empty or partial reviews when dependencies fail.📦 Services & Port MatrixServicePortDatabaseTechnology / RoleSBMS-Config-Server8085Git BackendExternalized configuration serverServiceRegistry_38761Eureka In-MemoryService discovery and registration serverAPI-GATEWAY8084NoneEdge entry point, route predicates, reverse proxyUserService8081MySQL (college)Aggregator, orchestrator, Resilience4j hostRatingService8083MySQL (college)User reviews and ratings managementHotelService8082PostgreSQL (microservice)Hotel catalog and facility information🔌 API Endpoints Reference1. API Gateway (http://localhost:8084)GET /users/user/{userId} -> Routed to USER-SERVICEPOST /users/saveUser -> Routed to USER-SERVICEGET /users/getAllUsers -> Routed to USER-SERVICE2. UserService (http://localhost:8081)POST /users/saveUser -> Create a new user profileGET /users/getAllUsers -> Retrieve all registered usersGET /users/user/{userId} -> Get aggregated user details with reviews & hotels3. RatingService (http://localhost:8083)POST /rating/saveRating -> Save a new rating recordGET /rating/getAll -> Fetch all user ratingsGET /rating/userId/{userId} -> Fetch ratings given by a specific userGET /rating/hotelId/{hotelId} -> Fetch ratings received by a specific hotel4. HotelService (http://localhost:8082)POST /hotels/saveHotel -> Register a new hotelGET /hotels/byId/{hotelId} -> Fetch hotel details by IDGET /hotels/allHotels -> List all available hotels📨 Sample Aggregated Response PayloadGET http://localhost:8084/users/user/3f93a924-0d97-4236-a487
{
  "userId": "3f93a924-0d97-4236-a487",
  "name": "chandu",
  "email": "chandu@gmail.com",
  "about": "we are expert in computer design i design the business related designs",
  "ratings": [
    {
      "ratingId": "90f2f03b-7c28-42b3-bc01",
      "userId": "3f93a924-0d97-4236-a487",
      "hotelId": "80fbd12b-3f2b-4e07-a143",
      "rating": 8,
      "feedback": "excellent",
      "hotel": {
        "id": "80fbd12b-3f2b-4e07-a143",
        "name": "hoti mahal",
        "location": "sambhajinagar",
        "about": "this is good place"
      }
    },
    {
      "ratingId": "e330e2b1-45f0-47fb-9061",
      "userId": "3f93a924-0d97-4236-a487",
      "hotelId": "4112b5d8-0d78-4d1e-ada0",
      "rating": 6,
      "feedback": "this is good for friends",
      "hotel": {
        "id": "4112b5d8-0d78-4d1e-ada0",
        "name": "shri ganesha",
        "location": "pune",
        "about": "24x7 services"
      }
    }
  ]
}
📸 Testing & Verification ScreenshotsStore test screenshots inside a /screenshots folder at the root of the repository:1. Eureka Dashboard (http://localhost:8761)2. API Gateway Routing Test3. Aggregated Single User Output4. Resilience4j Circuit Breaker Fallback Response🚦 Recommended Startup SequenceSBMS-Config-server (8085)ServiceRegistry_3 (8761)HotelService (8082)RatingService (8083)UserService (8081)API-GATEWAY (8084)💻 Local Setup & InstallationPrerequisitesJDK 21Maven 3.8+MySQL on port 3306 (college database)PostgreSQL on port 5432 (microservice database)Remote Config Repo: Microservices-Config-ServerRun Servicesgit clone https://github.com/chandrakantjadhav1401-tech/hotel-rating-microservices-ecosystem.git
cd hotel-rating-microservices-ecosystem

cd SBMS-Config-server && mvn spring-boot:run
cd ../ServiceRegistry_3 && mvn spring-boot:run
cd ../HotelService && mvn spring-boot:run
cd ../RatingService && mvn spring-boot:run
cd ../UserService && mvn spring-boot:run
cd ../API-GATEWAY && mvn spring-boot:run
