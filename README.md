# 🏨 Hotel Rating Microservices Ecosystem

A backend-based **Hotel Rating Microservices Ecosystem** built using Java, Spring Boot, Spring Cloud, REST APIs, MySQL, PostgreSQL, Spring Security, JWT, and Microservices architecture.

This project is developed as a learning and portfolio project to understand how multiple independent Spring Boot services communicate with each other and how common microservices components such as Service Discovery, API Gateway, Config Server, Fault Tolerance, and Authentication work together.

---

## 📌 Project Overview

In a traditional monolithic application, most features are developed and deployed as one application.

In this project, the application is divided into multiple independent microservices.

The main services are:

- User Service
- Hotel Service
- Rating Service

The project also includes:

- Eureka Service Discovery
- Spring Cloud Config Server
- API Gateway
- OpenFeign
- Resilience4j
- JWT Authentication
- Spring Security
- MySQL
- PostgreSQL
- REST APIs

The main goal of this project is to understand how a real-world microservices-based backend application can be designed and developed using Spring Boot and Spring Cloud.

---

## 🏗️ Architecture

```text
                         ┌─────────────────────┐
                         │     Client/User     │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │    API Gateway      │
                         │      Port 8084      │
                         └──────────┬──────────┘
                                    │
                    ┌───────────────┼────────────────┐
                    │               │                │
                    ▼               ▼                ▼
             ┌────────────┐  ┌────────────┐  ┌────────────┐
             │   User     │  │   Hotel    │  │  Rating    │
             │  Service   │  │  Service   │  │  Service   │
             └─────┬──────┘  └────────────┘  └─────┬──────┘
                   │                                │
                   └──────────────┬─────────────────┘
                                  │
                                  ▼
                         ┌─────────────────┐
                         │ Eureka Server   │
                         │   Port 8761     │
                         └─────────────────┘

                         ┌─────────────────┐
                         │ Config Server   │
                         │   Port 8085     │
                         └─────────────────┘
