# Airline Reservation System — Spring Boot 3.5.7 + JDK21

Comprehensive design, scaffolding and artifact templates to build an airline reservation microservices application. Includes service-by-service endpoints, data models, Dockerfiles, `docker-compose.yml`, Kafka topics, sample CI/CD GitHub Actions workflows, TDD/BDD/DDD guidance, testing examples, Spark ETL pipeline, sample data loaders (100 records per DB), monitoring (Prometheus/Grafana), centralized logging (ELK), and run instructions.

---

## 1. Goals & Constraints

- Spring Boot 3.5.7, JDK 21, Maven (or Gradle) build.
- Microservices: `eureka-server`, `api-gateway`, `passenger-service`, `reservation-service`, `payment-service`, `notification-service`, `rating-service`, `offer-service`, `analytics-service`, `monitoring-service`.
- Each service containerized with Docker, composed with `docker-compose.yml`.
- Databases per service: MySQL, PostgreSQL, MongoDB, Redis, Cassandra (each service uses appropriate DB as listed below but you may swap to suit requirements).
- Inter-service async messaging via Kafka.
- API Gateway: routing, load balancing, authentication/authorization (JWT + OAuth2 Resource Server), fault-tolerance (Resilience4j / Spring Cloud Circuit Breaker), request-level rate limiting.
- Swagger/OpenAPI aggregated UI (Springdoc + Gateway aggregator) to view all endpoints from single location.
- CI/CD: GitHub Actions per service + master pipeline workflow that builds, tests and deploys to local Docker Compose (or container registry). Include sample pipelines for unit, integration and E2E tests.
- Testing: TDD (JUnit 5), Mockito for unit tests, Cucumber for BDD scenarios, Selenium/Playwright for UI/acceptance E2E.
- ETL: Spark (Java/Scala) extracting from DBs, transforming and loading to `analytics-service` (Cassandra/Parquet) for reporting.
- Observability: Prometheus + Grafana + Alertmanager; centralized logs with ELK (Elasticsearch, Logstash, Kibana).

---

## 2. Tech Stack Summary

- Language: Java 21
- Framework: Spring Boot 3.5.7, Spring Cloud (compatible versions), Spring Security (JWT/OAuth2), Spring Data JPA, Spring Data MongoDB, Spring Data Redis, Spring Data Cassandra
- Messaging: Apache Kafka
- Databases: MySQL (passenger), PostgreSQL (reservation), MongoDB (offer/notification), Redis (caching/session), Cassandra (analytics/ratings), plus a RDBMS for payment as required
- API Gateway: Spring Cloud Gateway
- Service Discovery: Eureka
- Circuit breaker: Resilience4j
- OpenAPI: springdoc-openapi
- Testing: JUnit 5, Mockito, Cucumber JVM, Selenium, Playwright
- CI/CD: GitHub Actions
- ETL: Apache Spark (3.x), using Spark jobs packaged as JARs
- Observability: Prometheus, Grafana, ELK stack
- Containerization: Docker + docker-compose

---

## 3. High-level Architecture

1. `eureka-server` — service discovery
2. `api-gateway` — authn/authz, routing, swagger aggregator
3. Microservices behind gateway (each registered to Eureka)
4. Kafka brokers (topics for events)
5. Databases per microservice
6. Prometheus scraping endpoints, Grafana dashboards
7. ELK stack collecting logs via Logstash or Filebeat
8. CI/CD pipelines in GitHub Actions for each repo; master pipeline that triggers dependent workflows

---

## 4. Service-by-Service Details

### 4.1 eureka-server
**Purpose**: Service registry.
**Tech**: Spring Cloud Netflix Eureka (server).
**Ports**: 8761
**Endpoints**:
- `/eureka/apps` (Eureka API)
- `/actuator/health`
**Database**: none.
**Dockerfile**: standard Spring Boot jar run image.

### 4.2 api-gateway
**Purpose**: Single entrypoint, authentication/authorization, swagger aggregation, rate-limiting, load-balancing, routing, fault-tolerance.
**Tech**: Spring Cloud Gateway, Spring Security (JWT Resource Server + optional OAuth2 authorization integration), Springdoc OpenAPI aggregator, Resilience4j.
**Ports**: 8080
**Endpoints** (examples):
- `POST /auth/login` — issue JWT (delegates to an `auth` sub-service or integrated with Keycloak — you may use an authentication microservice or Keycloak container)
- Proxy routes: `/passengers/**` -> passenger-service, `/reservations/**` -> reservation-service, etc.
- `/openapi` — aggregated swagger UI (springdoc configuration)
- `/actuator/prometheus`

### 4.3 passenger-service
**Purpose**: Passenger CRUD, profile, document verification.
**Tech**: Spring WebFlux or MVC + Spring Data JPA + MySQL.
**DB**: MySQL
**Example Endpoints**:
- `GET /api/passengers` — list (with pagination)
- `GET /api/passengers/{id}`
- `POST /api/passengers` — create
- `PUT /api/passengers/{id}`
- `DELETE /api/passengers/{id}`
- `GET /api/passengers/{id}/reservations` — fetch reservations via reservation service (Feign or Kafka request/response)
**Events Published**:
- `passenger.created`, `passenger.updated`, `passenger.deleted` (Kafka topics)

### 4.4 reservation-service
**Purpose**: Flight search, seat allocation, booking management.
**Tech**: Spring Boot + PostgreSQL + JPA
**DB**: PostgreSQL
**Endpoints**:
- `GET /api/reservations` — list
- `GET /api/reservations/{id}`
- `POST /api/reservations` — create reservation (transactional — communicates with payment-service)
- `POST /api/reservations/{id}/confirm` — confirm booking
- `POST /api/reservations/{id}/cancel` — cancel
**Domain Events**:
- `reservation.created`, `reservation.confirmed`, `reservation.cancelled` (Kafka)

### 4.5 payment-service
**Purpose**: Payment processing, refunds, transaction history.
**Tech**: Spring Boot + JPA + MySQL (or Postgres)
**DB**: MySQL (or Postgres)
**Endpoints**:
- `POST /api/payments` — initiate payment
- `GET /api/payments/{id}`
- `POST /api/payments/{id}/refund`
**Integration**:
- Communicates with external payment gateway using a mocked adapter for test environment.
- Subscribes to `reservation.created` events to charge customer.
- Publishes `payment.succeeded`, `payment.failed`.

### 4.6 notification-service
**Purpose**: Email/SMS/push notifications.
**Tech**: Spring Boot + MongoDB (for message persistence & templates)
**DB**: MongoDB
**Endpoints**:
- `POST /api/notifications/send` — send notification
- `GET /api/notifications/{id}`
**Integration**:
- Subscribes to Kafka events: `reservation.confirmed`, `payment.succeeded`, `reservation.cancelled`.

### 4.7 rating-service
**Purpose**: Passenger ratings for flights or services; analytics input.
**Tech**: Spring Boot + Cassandra (time-series friendly)
**DB**: Cassandra
**Endpoints**:
- `POST /api/ratings` — submit rating
- `GET /api/ratings?flightId=...` — list ratings
**Events**:
- `rating.submitted` published to Kafka

### 4.8 offer-service
**Purpose**: Promotions, coupon codes, dynamic discounts.
**Tech**: Spring Boot + MongoDB
**DB**: MongoDB
**Endpoints**:
- `GET /api/offers` — list
- `POST /api/offers` — create
- `POST /api/offers/{id}/apply` — validate and apply offer to reservation

### 4.9 analytics-service
**Purpose**: Batch and streaming analytics, reporting.
**Tech**: Spark jobs + Cassandra/Parquet stores + REST microservice for reports
**DB**: Cassandra (or read-optimized store)
**Endpoints**:
- `GET /api/reports/daily-sales`
- `GET /api/reports/top-routes`
**ETL**:
- Spark jobs consume Kafka topics and DB snapshots, transform and load aggregates into Cassandra.

### 4.10 monitoring-service
**Purpose**: Aggregates metrics, custom health checks, dashboard meta.
**Tech**: Spring Boot actuator endpoints, Prometheus metrics exporter, Grafana dashboard provisioning.
**Endpoints**:
- `/actuator/prometheus`

---

## 5. Sample Data Models (condensed)

### passenger-service (MySQL)
```sql
CREATE TABLE passengers (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  first_name VARCHAR(100),
  last_name VARCHAR(100),
  email VARCHAR(200) UNIQUE,
  phone VARCHAR(50),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### reservation-service (Postgres)
```sql
CREATE TABLE reservations (
  id UUID PRIMARY KEY,
  passenger_id BIGINT,
  flight_number VARCHAR(50),
  departure TIMESTAMP,
  arrival TIMESTAMP,
  seat VARCHAR(10),
  status VARCHAR(20),
  created_at TIMESTAMP DEFAULT now()
);
```

### payment-service (MySQL)
```sql
CREATE TABLE payments (
  id UUID PRIMARY KEY,
  reservation_id UUID,
  amount DECIMAL(10,2),
  currency VARCHAR(10),
  status VARCHAR(20),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### notification-service (MongoDB)
Document example:
```json
{
  "_id": "uuid",
  "recipient": "siva@example.com",
  "type":"email",
  "subject":"Booking Confirmed",
  "body":"Your reservation ...",
  "sent": false,
  "createdAt": "2025-11-20T00:00:00Z"
}
```

### rating-service (Cassandra)
```cql
CREATE TABLE ratings (
  flight_id text,
  rating_time timestamp,
  passenger_id bigint,
  rating int,
  comment text,
  PRIMARY KEY (flight_id, rating_time)
) WITH CLUSTERING ORDER BY (rating_time DESC);
```

### offer-service (MongoDB)
Document example: coupon code, percentage, expiry.

---

## 6. Kafka Topics (recommended)
- `passenger.events` (creates/updates)
- `reservation.events` (created/confirmed/cancelled)
- `payment.events` (succeeded/failed)
- `notification.events` (requests for notifications)
- `rating.events`
- `offer.events`
- `analytics.raw` (raw events for Spark)

---

## 7. Dockerfile Template (use per service)

```dockerfile
# stage 1: build
FROM maven:3.9.4-eclipse-temurin-21 as build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn -B -DskipTests package

# stage 2: runtime
FROM eclipse-temurin:21-jre
ARG JAR_FILE=target/*.jar
COPY --from=build /app/${JAR_FILE} app.jar
EXPOSE 8080
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```

Adjust port / jar name per service. For smaller images use `jib` or `spring-boot:build-image`.

---

## 8. docker-compose.yml (high-level excerpt)

This compose groups services, Kafka, Zookeeper, DBs, ELK, Prometheus, Grafana.

```yaml
version: '3.8'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
  kafka:
    image: confluentinc/cp-kafka:7.4.0
    depends_on: [zookeeper]
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092

  eureka-server:
    build: ./eureka-server
    ports: ['8761:8761']
    environment:
      - SPRING_PROFILES_ACTIVE=default
    depends_on: [kafka]

  api-gateway:
    build: ./api-gateway
    ports: ['8080:8080']
    depends_on: [eureka-server]

  passenger-service:
    build: ./passenger-service
    ports: ['8081:8081']
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/passengerdb
    depends_on: [mysql, eureka-server, kafka]

  mysql:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: passengerdb
    ports: ['3306:3306']

  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: postgres
    ports: ['5432:5432']

  mongodb:
    image: mongo:6
    ports: ['27017:27017']

  cassandra:
    image: cassandra:4.1
    ports: ['9042:9042']

  prometheus:
    image: prom/prometheus
    ports: ['9090:9090']
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana
    ports: ['3000:3000']

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.10.0
    environment:
      discovery.type: single-node
    ports: ['9200:9200']

  kibana:
    image: docker.elastic.co/kibana/kibana:8.10.0
    ports: ['5601:5601']

  logstash:
    image: docker.elastic.co/logstash/logstash:8.10.0
    ports: ['5044:5044']

networks:
  default:
    driver: bridge
```

> Note: For production, consider Kubernetes and separate hostnames, secrets and volumes.

---

## 9. Swagger (OpenAPI) Aggregation

- Each service includes `springdoc-openapi` dependency and exposes `/v3/api-docs`.
- API Gateway aggregates downstream docs using configuration so `GET /openapi` returns aggregated docs; an admin endpoint in the gateway can fetch `/v3/api-docs` from registered services and merge into a single OpenAPI document served at `/openapi`.
- Use `springdoc-openapi-ui` in the gateway to serve Swagger UI.

Implementation note: Provide a simple `OpenApiAggregator` component in the gateway that uses `DiscoveryClient` to find services and fetch their `/v3/api-docs` and merge with `swagger-core` utilities.

---

## 10. Testing Strategy & Examples

### Unit tests (JUnit 5 + Mockito)
- Each domain, repository and service class has unit tests; mocking external dependencies (KafkaTemplate, Repositories).
- Example test skeleton (JUnit 5 + Mockito):

```java
@SpringBootTest
class PassengerServiceTest {
  @MockBean PassengerRepository repo;
  @Autowired PassengerService service;

  @Test
  void createPassenger_callsSave() {
    Passenger p = new Passenger(null, "Siva", "Kumar", "siva@example.com");
    when(repo.save(any())).thenReturn(new Passenger(1L, "Siva", "Kumar", "siva@example.com"));
    var res = service.create(p);
    assertNotNull(res.getId());
    verify(repo).save(any());
  }
}
```

### Integration tests (Spring Boot Test + Testcontainers)
- Use Testcontainers for MySQL/Postgres/Kafka to run integration tests in CI.

### BDD (Cucumber)
- Feature file example: `booking.feature` with scenarios: `Successful booking`, `Payment failure`, `Offer applied`.

### E2E (Selenium / Playwright)
- Playwright to drive API Gateway UI flows. Use headless mode in CI.

---

## 11. Sample GitHub Actions (per-service workflow)

`.github/workflows/ci.yml` (per microservice):

```yaml
name: CI
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
      - name: Build and test
        run: mvn -B -DskipTests=false test package
      - name: Build Docker image
        run: |
          docker build -t ghcr.io/${{ github.repository_owner }}/${{ github.repository }}:latest .
      - name: Push image
        uses: docker/build-push-action@v5
        with:
          push: false # set true if pushing to registry
```

### Master pipeline (orchestrator)
- Use `workflow_dispatch` or `workflow_run` to trigger dependent services. Build images for each service and run `docker-compose up --build` in a runner that has Docker-in-Docker or push to registry and deploy to environment.

---

## 12. Spark ETL Pipeline

- Create Spark jobs that consume `analytics.raw` Kafka topic and/or read DB snapshots.
- Transform: compute daily sales, bookings by route, average payment times, rating aggregates.
- Load: write aggregated tables into Cassandra or Parquet files consumed by analytics-service.

Sample Spark job pseudocode (Scala/Java):

```scala
val spark = SparkSession.builder.appName("airline-etl").getOrCreate()
val kafkaDf = spark
  .readStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "kafka:9092")
  .option("subscribe", "analytics.raw")
  .load()

// parse JSON events, perform windowed aggregations, write to Cassandra
```

---

## 13. Sample Data Loaders (100 records per DB)

Provide scripts per DB. Examples:

### MySQL (passenger) — `sql/seed_passengers.sql`
```sql
INSERT INTO passengers (first_name, last_name, email, phone)
VALUES
('Siva','Kumar','siva1@example.com','+919000000001'),
('Siva','Kumar','siva2@example.com','+919000000002'),
... -- repeat to 100 rows
;
```

### Postgres (reservations) — `sql/seed_reservations.sql`
- Create 100 reservations with random UUIDs, different flights.

### MongoDB (offers/notifications) — `mongo/seed_offers.js`
```js
const offers = [], now = new Date();
for(let i=1;i<=100;i++) offers.push({ code: `OFF${i}`, discount: Math.floor(Math.random()*50)+5, expiry: new Date(now.getTime()+1000*60*60*24*30) });
db.offers.insertMany(offers)
```

### Cassandra (ratings) — `cassandra/seed_ratings.cql`
- Generate 100 INSERT statements using a small script.

### Redis
- Use a Redis CLI script to set sample keys for caching.

---

## 14. CI/CD: Testing Matrix & Environment

- Matrix: Java 21; run unit tests, integration (Testcontainers), BDD (Cucumber), and E2E (Playwright).
- Cache Maven dependencies, Docker layered cache in GitHub Actions.
- Optionally sign artifacts and push images to GitHub Container Registry.

---

## 15. Observability & Logging

- Each Spring Boot app exposes `/actuator/prometheus` and standard metrics.
- Prometheus scrapes endpoints; Grafana dashboard connects to Prometheus.
- Configure Spring Boot logging to ship logs to Logstash or write to file and use Filebeat.
- Example `logstash.conf` to accept beats and forward to Elasticsearch.

---

## 16. Security

- Use JWT tokens signed with private key; keys stored in secrets manager.
- API Gateway validates JWT and forwards user principal via headers.
- Each service checks scopes/roles on sensitive endpoints.
- Use TLS for production, secrets management (Vault or cloud secret), limit service account privileges.

---

## 17. Implementation Notes: TDD, BDD, DDD

- **DDD**: Identify bounded contexts — passengers, reservations, payments, offers, ratings, notifications, analytics.
- Each service implements domain model, repositories, domain events (Kafka), and an application layer with use-cases.
- **TDD**: write failing unit tests first for domain services and behavior, then implement.
- **BDD**: define critical business flows in Gherkin and drive acceptance tests (Cucumber).

---

## 18. Example Files & Snippets

### application.yml (passenger-service)
```yaml
server:
  port: 8081
spring:
  datasource:
    url: jdbc:mysql://mysql:3306/passengerdb
    username: root
    password: root
  jpa:
    hibernate:
      ddl-auto: update
management:
  endpoints:
    web:
      exposure:
        include: "health,info,prometheus"
```

### Kafka Producer (Spring)
```java
@Service
public class PassengerEventPublisher {
  private final KafkaTemplate<String, String> kafka;
  public PassengerEventPublisher(KafkaTemplate<String,String> kafka) { this.kafka = kafka; }
  public void publishCreated(Passenger p) {
    kafka.send("passenger.events", p.getId().toString(), /* serialize to JSON */ serialize(p));
  }
}
```

### Kafka Consumer example
```java
@KafkaListener(topics = "reservation.events")
public void handleReservationEvent(String payload) { /* parse and handle */ }
```

---

## 19. Run & Develop Locally (quick start)

1. Clone each microservice into separate folders or a mono-repo with submodules.
2. Ensure `docker-compose.yml` at repo root contains all dependencies.
3. Build images locally: `mvn -T 1C -DskipTests package` then `docker-compose up --build`.
4. Seed databases: run provided seed scripts in `sql/`, `mongo/`, `cassandra/` folders.
5. Open API Gateway `http://localhost:8080` and Eureka `http://localhost:8761`.
6. Check Prometheus at `http://localhost:9090` and Grafana at `http://localhost:3000`.

---

## 20. Next steps & Deliverables you can request

- Full starter code for one service (recommended: `passenger-service`) including controllers, service layer, repository, tests, Dockerfile and GitHub Actions workflow.
- Aggregated `docker-compose.yml` fully populated and tested locally.
- OpenAPI aggregator code for the gateway.
- Spark ETL job code (Scala/Java) with sample Kafka consumer and Cassandra writer.
- Seed scripts for all DBs with 100 records each (automated generator script).
- CI/CD master workflow that orchestrates builds and runs integration tests.

---

### License & Comments
This document is a scaffold—templates, examples and references. For production hardening: secrets management, TLS, observability tuning, horizontal scaling, container orchestration (K8s) and cost considerations are required.


<!-- End of document -->

