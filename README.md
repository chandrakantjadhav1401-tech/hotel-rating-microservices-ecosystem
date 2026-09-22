
# 🏨 Hotel & Rating Microservices Ecosystem

[![Java](https://img.shields.io/badge/Java-21-orange.svg)]()
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.x-brightgreen.svg)]()
[![Spring Cloud](https://img.shields.io/badge/Spring%20Cloud-2023.x-blue.svg)]()
[![Databases](https://img.shields.io/badge/Databases-MySQL%20%7C%20PostgreSQL-blueviolet.svg)]()
[![Resilience4j](https://img.shields.io/badge/Resilience4j-Fault%20Tolerant-red.svg)]()

An enterprise-grade, distributed hotel and rating management platform built using **Java 21**, **Spring Boot**, **Spring Cloud**, **Resilience4j**, and polyglot persistence (**MySQL** and **PostgreSQL**).

This central repository houses the entire 6-service microservices architecture, including service orchestration, dynamic discovery, centralized Git configuration, load-balanced edge routing, and fault-tolerant patterns.

---

## 📑 Repository Structure & Services

text
hotel-rating-microservices-ecosystem/
├── SBMS-Config-server/     # Centralized Git-backed External Configuration (:8085)
├── ServiceRegistry_3/      # Netflix Eureka Service Discovery & Heartbeats (:8761)
├── API-GATEWAY/            # Spring Cloud Gateway edge router & reverse proxy (:8084)
├── UserService/            # Main aggregator & orchestrator service (:8081) [MySQL]
├── RatingService/          # Review & rating management service (:8083) [MySQL]
├── HotelService/           # Hotel catalog and room directory service (:8082) [PostgreSQL]
├── screenshots/            # Postman & Eureka verification screenshots
└── README.md               # Main repository documentation (this file)


---

## 🏛️ System Architecture


                       +----------------------------------------------------+
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

---

## 🔄 End-to-End Execution Workflow

When a client queries user details along with hotel reviews:

[Client / Postman / Frontend]
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
            │ 3. Fetches basic user profile from MySQL (`college` DB)
            │
            │ 4. Calls RatingService using Load-Balanced RestTemplate / FeignClient
            ▼
+-----------------------+
| RatingService (:8083) | ───► Queries MySQL (`college` DB) ──► Returns List<Rating>
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
|  HotelService (:8082) | ───► Queries PostgreSQL (`microservice` DB) ──► Returns Hotel
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



---

## 🛡️ Resilience4j Fault Tolerance

Downstream communication in `UserService` is protected against cascading failures:

```java
@GetMapping("/user/{userId}")
@CircuitBreaker(name = "ratingHotelBreaker", fallbackMethod = "ratingHotelFallback")
@Retry(name = "ratingHotelService", fallbackMethod = "ratingHotelFallback")
@RateLimiter(name = "userRateLimiter", fallbackMethod = "ratingHotelFallback")
public ResponseEntity<User> getSingleUser(@PathVariable String userId) { ... }



### State Machine Lifecycle

* **CLOSED:** Normal state. Calls flow directly to downstream services.
* **OPEN:** When the failure threshold is breached, the circuit trips open. Calls fail fast immediately to prevent resource starvation, routing to `ratingHotelFallback()`.
* **HALF-OPEN:** After a waiting duration, trial requests are allowed through to evaluate downstream recovery.
* **Retry:** Automatically attempts idempotent re-executions for transient network hiccups before opening the circuit.
* **RateLimiter:** Caps burst request spikes to protect the application from overloading.

---

## 📦 Services & Port Matrix

| Service | Port | Database | Technology / Role |
| --- | --- | --- | --- |
| **SBMS-Config-Server** | `8085` | Git Backend | Externalized configuration server |
| **ServiceRegistry_3** | `8761` | Eureka In-Memory | Service discovery and registration server |
| **API-GATEWAY** | `8084` | None | Edge entry point, route predicates, reverse proxy |
| **UserService** | `8081` | MySQL (`college`) | Aggregator, orchestrator, Resilience4j host |
| **RatingService** | `8083` | MySQL (`college`) | User reviews and ratings management |
| **HotelService** | `8082` | PostgreSQL (`microservice`) | Hotel catalog and facility information |

---

## 🔌 API Endpoints Reference

### 1. API Gateway (`http://localhost:8084`)

| Method | Endpoint | Routed Service |
| --- | --- | --- |
| `GET` | `/users/user/{userId}` | `USER-SERVICE` |
| `POST` | `/users/saveUser` | `USER-SERVICE` |
| `GET` | `/users/getAllUsers` | `USER-SERVICE` |

---

### 2. UserService (`http://localhost:8081`)

| Method | Endpoint | Description |
| --- | --- | --- |
| `POST` | `/users/saveUser` | Create a new user profile |
| `GET` | `/users/getAllUsers` | Retrieve all registered users |
| `GET` | `/users/user/{userId}` | Get aggregated user details with reviews & hotels |

---

### 3. RatingService (`http://localhost:8083`)

| Method | Endpoint | Description |
| --- | --- | --- |
| `POST` | `/rating/saveRating` | Save a new rating record |
| `GET` | `/rating/getAll` | Fetch all user ratings |
| `GET` | `/rating/userId/{userId}` | Fetch ratings given by a specific user |
| `GET` | `/rating/hotelId/{hotelId}` | Fetch ratings received by a specific hotel |

---

### 4. HotelService (`http://localhost:8082`)

| Method | Endpoint | Description |
| --- | --- | --- |
| `POST` | `/hotels/saveHotel` | Register a new hotel |
| `GET` | `/hotels/byId/{hotelId}` | Fetch hotel details by ID |
| `GET` | `/hotels/allHotels` | List all available hotels |

---

## 📨 Sample Aggregated Response Payload

#### Request

```http
GET http://localhost:8084/users/user/3f93a924-0d97-4236-a487

```

#### Response (`200 OK`)

```json
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

```

---

## 📸 Testing & Verification Screenshots

Add your testing screenshots into a `/screenshots` folder in this repository:

### 1. Eureka Dashboard (`http://localhost:8761`)

All microservices registered with `UP` status:


### 2. API Gateway Routing Test

Request routed through port `8084` to `UserService`:


### 3. Aggregated Single User Output

Shows data combined from `UserService`, `RatingService`, and `HotelService`:


### 4. Resilience4j Circuit Breaker Fallback Response

Graceful fallback execution when downstream services are stopped:


---

## 🚦 Recommended Startup Sequence

Start each service in the following order to ensure dependencies and registrations resolve cleanly:

1. **SBMS-Config-server (`8085`)** — Central configuration provider.
2. **ServiceRegistry_3 (`8761`)** — Eureka server for service discovery.
3. **HotelService (`8082`)** — Ensure PostgreSQL is running and database `microservice` is created.
4. **RatingService (`8083`)** — Ensure MySQL is running and database `college` is created.
5. **UserService (`8081`)** — Connects to MySQL `college` and registers with Eureka.
6. **API-GATEWAY (`8084`)** — Gateway router and reverse proxy.

---

## 💻 Local Setup & Installation

### Prerequisites

* **JDK 21** installed
* **Maven 3.8+** installed
* **MySQL Server** running on port `3306` with database `college`
* **PostgreSQL Server** running on port `5432` with database `microservice`
* **External Config Repo:** [Microservices-Config-Server](https://github.com/chandrakantjadhav1401-tech/Microservices-Config-Server?utm_source=gemini)

### Build & Run


# Clone this central repository
git clone [https://github.com/chandrakantjadhav1401-tech/hotel-rating-microservices-ecosystem.git](https://github.com/chandrakantjadhav1401-tech/hotel-rating-microservices-ecosystem.git)
cd hotel-rating-microservices-ecosystem

# Run services individually in separate terminal tabs
cd SBMS-Config-server && mvn spring-boot:run
cd ../ServiceRegistry_3 && mvn spring-boot:run
cd ../HotelService && mvn spring-boot:run
cd ../RatingService && mvn spring-boot:run
cd ../UserService && mvn spring-boot:run
cd ../API-GATEWAY && mvn spring-boot:run

