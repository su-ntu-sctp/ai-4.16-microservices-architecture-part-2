# Lesson 4.16: Microservices Architecture - Part 2 (Practical Demo)

## Learning Objectives

By the end of this lesson, you will be able to:

1. **Build** two independent Spring Boot microservices from scratch
2. **Implement** REST-based inter-service communication using RestTemplate
3. **Deploy** and test the complete microservices system using Docker Compose
4. **Describe** Service Discovery and API Gateway as next steps beyond this foundation

---

## Prerequisites

Before starting this lesson, ensure you have:

- Completed Lesson 4.15 (Microservices Theory)
- Completed Lesson 4.6 (Docker Compose)
- Java 21 installed
- Maven installed
- Docker Desktop running
- VS Code or your preferred IDE

---

## Introduction

In Lesson 4.15, you learned microservices concepts. Today, you'll build a real microservices system!

**What you'll build:**

```
┌─────────────────┐       ┌─────────────────┐
│  User Service   │       │  Order Service  │
│   Port: 8081    │◄──────│   Port: 8082    │
│                 │  REST │                 │
│ - Create users  │       │ - Create orders │
│ - Get users     │       │ - Get orders    │
│                 │       │ - Calls User Svc│
│   [Users DB]    │       │   [Orders DB]   │
└─────────────────┘       └─────────────────┘
```

**Scenario:**
- User Service manages users
- Order Service manages orders
- When creating an order, Order Service calls User Service to validate the user exists

---

## Part 1: Project Setup

### Step 1: Create Project Directory

```bash
mkdir microservices-demo
cd microservices-demo
```

**Structure:**
```
microservices-demo/
├── user-service/
├── order-service/
└── docker-compose.yml
```

---

### Step 2: Create User Service

Go to https://start.spring.io/

**Configuration:**
- **Project:** Maven
- **Language:** Java
- **Spring Boot:** Choose the latest stable 3.x — avoid SNAPSHOT versions
- **Group:** com.example
- **Artifact:** user-service
- **Package name:** com.example.userservice
- **Packaging:** Jar
- **Java:** 21

**Dependencies:** Spring Web, Spring Data JPA, PostgreSQL Driver, Lombok

**Click "Generate"** → Download → Extract to `microservices-demo/user-service/`

> ⚠️ **Note:** When you extract the zip, it may create a nested folder (e.g., `user-service/user-service/`). Make sure `pom.xml` is directly inside `user-service/`, not in a subfolder. If nested, move all files up one level.

---

### Step 3: Create Order Service

Go to https://start.spring.io/ again with the same settings, but:
- **Artifact:** order-service
- **Package name:** com.example.orderservice

Extract to `microservices-demo/order-service/`

> ⚠️ **Note:** Same as above — ensure `pom.xml` is directly inside `order-service/`, not nested.

---

### Step 4: Verify Structure

```
microservices-demo/
├── user-service/
│   ├── src/
│   ├── pom.xml
│   └── mvnw
├── order-service/
│   ├── src/
│   ├── pom.xml
│   └── mvnw
└── (docker-compose.yml — created later)
```

---

## Part 2: Build User Service

### Step 1: Create User Entity

**File:** `user-service/src/main/java/com/example/userservice/entity/User.java`

```java
package com.example.userservice.entity;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "users")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String firstName;

    @Column(nullable = false)
    private String lastName;

    @Column(nullable = false, unique = true)
    private String email;
}
```

> ℹ️ **Note:** The table is named `users` (not `user`) because `user` is a reserved keyword in PostgreSQL.

---

### Step 2: Create User Repository

**File:** `user-service/src/main/java/com/example/userservice/repository/UserRepository.java`

```java
package com.example.userservice.repository;

import com.example.userservice.entity.User;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
}
```

---

### Step 3: Create User Service

**File:** `user-service/src/main/java/com/example/userservice/service/UserService.java`

```java
package com.example.userservice.service;

import com.example.userservice.entity.User;
import com.example.userservice.repository.UserRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Optional;

@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    public List<User> getAllUsers() {
        return userRepository.findAll();
    }

    public Optional<User> getUserById(Long id) {
        return userRepository.findById(id);
    }

    public User createUser(User user) {
        return userRepository.save(user);
    }
}
```

---

### Step 4: Create User Controller

**File:** `user-service/src/main/java/com/example/userservice/controller/UserController.java`

```java
package com.example.userservice.controller;

import com.example.userservice.entity.User;
import com.example.userservice.service.UserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/users")
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping
    public ResponseEntity<List<User>> getAllUsers() {
        List<User> users = userService.getAllUsers();
        return new ResponseEntity<>(users, HttpStatus.OK);
    }

    @GetMapping("/{id}")
    public ResponseEntity<User> getUserById(@PathVariable Long id) {
        return userService.getUserById(id)
                .map(user -> new ResponseEntity<>(user, HttpStatus.OK))
                .orElse(new ResponseEntity<>(HttpStatus.NOT_FOUND));
    }

    @PostMapping
    public ResponseEntity<User> createUser(@RequestBody User user) {
        User createdUser = userService.createUser(user);
        return new ResponseEntity<>(createdUser, HttpStatus.CREATED);
    }
}
```

---

### Step 5: Configure User Service

**File:** `user-service/src/main/resources/application.properties`

```properties
# Server configuration
server.port=${SERVER_PORT:8081}

# Database configuration
spring.datasource.url=${DATABASE_URL:jdbc:postgresql://localhost:5432/userdb}
spring.datasource.username=${DATABASE_USERNAME:postgres}
spring.datasource.password=${DATABASE_PASSWORD:postgres}

# JPA configuration
spring.jpa.hibernate.ddl-auto=update
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.show-sql=true

# Application name
spring.application.name=user-service
```

> ℹ️ The `${VARIABLE:default}` syntax means: use the environment variable if set, otherwise use the default. When running locally, defaults apply. When running via Docker Compose, values are provided in `docker-compose.yml`.

---

### Step 6: Create Dockerfile for User Service

**File:** `user-service/Dockerfile`

```dockerfile
FROM eclipse-temurin:21-jdk-alpine
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
CMD ["java", "-jar", "app.jar"]
```

---

### Step 7: Create .dockerignore for User Service

**File:** `user-service/.dockerignore`

```
target/classes/
target/test-classes/
target/generated-sources/
target/maven-status/
.git
.gitignore
.idea/
.vscode/
*.md
```

---

## ✅ Test User Service Locally

Before building Order Service, verify User Service works correctly.

### Step 1: Create the local database

Spring Boot auto-creates **tables**, but the **database** must exist first. Create `userdb` using DBeaver or terminal:

```bash
sudo service postgresql start
psql -U postgres -c "CREATE DATABASE userdb;"
```

### Step 2: Run User Service

```bash
cd microservices-demo/user-service
mvn spring-boot:run
```

Wait for: `Started UserServiceApplication in X.XXX seconds`

### Step 3: Test endpoints in Postman

**Create a user** — `POST http://localhost:8081/api/users`
```json
{ "firstName": "John", "lastName": "Doe", "email": "john@example.com" }
```
Expected: `201 Created` with user object including `id`

**Get all users** — `GET http://localhost:8081/api/users` → `200 OK`

**Get user by ID** — `GET http://localhost:8081/api/users/1` → `200 OK`

**Get non-existent user** — `GET http://localhost:8081/api/users/999` → `404 Not Found`

✅ Once all endpoints work, stop the service (`Ctrl+C`) and move to Order Service.

---

## Part 3: Build Order Service

### Step 1: Create Order Entity

**File:** `order-service/src/main/java/com/example/orderservice/entity/Order.java`

```java
package com.example.orderservice.entity;

import jakarta.persistence.*;
import lombok.*;

import java.time.LocalDateTime;

@Entity
@Table(name = "orders")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private Long userId;

    @Column(nullable = false)
    private String productName;

    @Column(nullable = false)
    private Integer quantity;

    @Column(nullable = false)
    private Double totalPrice;

    @Column(nullable = false)
    private String status;

    @Column(nullable = false)
    private LocalDateTime orderDate;

    @PrePersist
    public void prePersist() {
        if (orderDate == null) {
            orderDate = LocalDateTime.now();
        }
        if (status == null) {
            status = "PENDING";
        }
    }
}
```

---

### Step 2: Create User DTO

**File:** `order-service/src/main/java/com/example/orderservice/dto/UserDTO.java`

```java
package com.example.orderservice.dto;

import lombok.*;

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class UserDTO {
    private Long id;
    private String firstName;
    private String lastName;
    private String email;
}
```

---

### Step 3: Create Order Repository

**File:** `order-service/src/main/java/com/example/orderservice/repository/OrderRepository.java`

```java
package com.example.orderservice.repository;

import com.example.orderservice.entity.Order;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByUserId(Long userId);
}
```

---

### Step 4: Create Order Service

**File:** `order-service/src/main/java/com/example/orderservice/service/OrderService.java`

```java
package com.example.orderservice.service;

import com.example.orderservice.dto.UserDTO;
import com.example.orderservice.entity.Order;
import com.example.orderservice.repository.OrderRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

import java.util.List;
import java.util.Optional;

@Service
public class OrderService {

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private RestTemplate restTemplate;

    @Value("${user.service.url}")
    private String userServiceUrl;

    public List<Order> getAllOrders() {
        return orderRepository.findAll();
    }

    public Optional<Order> getOrderById(Long id) {
        return orderRepository.findById(id);
    }

    public List<Order> getOrdersByUserId(Long userId) {
        return orderRepository.findByUserId(userId);
    }

    public Order createOrder(Order order) {
        Long userId = order.getUserId();
        String url = userServiceUrl + "/api/users/" + userId;

        try {
            UserDTO user = restTemplate.getForObject(url, UserDTO.class);

            // Note: In practice, when User Service returns 404, RestTemplate throws
            // HttpClientErrorException.NotFound rather than returning null.
            // The catch block below handles this correctly.
            // The null check is kept here as a safety net for other edge cases.
            if (user == null) {
                throw new RuntimeException("User not found with ID: " + userId);
            }

            return orderRepository.save(order);

        } catch (Exception e) {
            throw new RuntimeException("Failed to validate user: " + e.getMessage());
        }
    }
}
```

---

### Step 5: Create RestTemplate Configuration

**File:** `order-service/src/main/java/com/example/orderservice/config/AppConfig.java`

```java
package com.example.orderservice.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;

@Configuration
public class AppConfig {

    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

> ⚠️ **Important:** `AppConfig.java` must be placed at:
> `src/main/java/com/example/orderservice/config/AppConfig.java`
>
> If placed outside `src/`, Spring Boot will not detect it and the application will fail with:
> `Field restTemplate ... required a bean of type 'RestTemplate' that could not be found.`

---

### Step 6: Create Order Controller

**File:** `order-service/src/main/java/com/example/orderservice/controller/OrderController.java`

```java
package com.example.orderservice.controller;

import com.example.orderservice.entity.Order;
import com.example.orderservice.service.OrderService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @Autowired
    private OrderService orderService;

    @GetMapping
    public ResponseEntity<List<Order>> getAllOrders() {
        List<Order> orders = orderService.getAllOrders();
        return new ResponseEntity<>(orders, HttpStatus.OK);
    }

    @GetMapping("/{id}")
    public ResponseEntity<Order> getOrderById(@PathVariable Long id) {
        return orderService.getOrderById(id)
                .map(order -> new ResponseEntity<>(order, HttpStatus.OK))
                .orElse(new ResponseEntity<>(HttpStatus.NOT_FOUND));
    }

    @GetMapping("/user/{userId}")
    public ResponseEntity<List<Order>> getOrdersByUserId(@PathVariable Long userId) {
        List<Order> orders = orderService.getOrdersByUserId(userId);
        return new ResponseEntity<>(orders, HttpStatus.OK);
    }

    @PostMapping
    public ResponseEntity<?> createOrder(@RequestBody Order order) {
        try {
            Order createdOrder = orderService.createOrder(order);
            return new ResponseEntity<>(createdOrder, HttpStatus.CREATED);
        } catch (Exception e) {
            return new ResponseEntity<>("User not found with ID: " + order.getUserId(), HttpStatus.BAD_REQUEST);
        }
    }
}
```

---

### Step 7: Configure Order Service

**File:** `order-service/src/main/resources/application.properties`

```properties
# Server configuration
server.port=${SERVER_PORT:8082}

# Database configuration
spring.datasource.url=${DATABASE_URL:jdbc:postgresql://localhost:5432/orderdb}
spring.datasource.username=${DATABASE_USERNAME:postgres}
spring.datasource.password=${DATABASE_PASSWORD:postgres}

# JPA configuration
spring.jpa.hibernate.ddl-auto=update
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.show-sql=true

# Application name
spring.application.name=order-service

# User Service URL
user.service.url=${USER_SERVICE_URL:http://localhost:8081}
```

---

### Step 8: Create Dockerfile for Order Service

**File:** `order-service/Dockerfile`

```dockerfile
FROM eclipse-temurin:21-jdk-alpine
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
CMD ["java", "-jar", "app.jar"]
```

---

### Step 9: Create .dockerignore for Order Service

**File:** `order-service/.dockerignore`

```
target/classes/
target/test-classes/
target/generated-sources/
target/maven-status/
.git
.gitignore
.idea/
.vscode/
*.md
```

---

## Part 4: Docker Compose Configuration

### Create docker-compose.yml

**File:** `microservices-demo/docker-compose.yml`

```yaml
services:
  # User Service
  user-service:
    build: ./user-service
    container_name: user-service
    ports:
      - "8081:8080"
    environment:
      SERVER_PORT: 8080
      DATABASE_URL: jdbc:postgresql://user-db:5432/userdb
      DATABASE_USERNAME: postgres
      DATABASE_PASSWORD: postgres
    depends_on:
      - user-db
    networks:
      - microservices-network

  # Order Service
  order-service:
    build: ./order-service
    container_name: order-service
    ports:
      - "8082:8080"
    environment:
      SERVER_PORT: 8080
      DATABASE_URL: jdbc:postgresql://order-db:5432/orderdb
      DATABASE_USERNAME: postgres
      DATABASE_PASSWORD: postgres
      USER_SERVICE_URL: http://user-service:8080
    depends_on:
      - order-db
      - user-service
    networks:
      - microservices-network

  # User Database
  user-db:
    image: postgres:15
    container_name: user-db
    ports:
      - "5433:5432"
    environment:
      POSTGRES_DB: userdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - user-db-data:/var/lib/postgresql/data
    networks:
      - microservices-network

  # Order Database
  order-db:
    image: postgres:15
    container_name: order-db
    ports:
      - "5434:5432"
    environment:
      POSTGRES_DB: orderdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - order-db-data:/var/lib/postgresql/data
    networks:
      - microservices-network

volumes:
  user-db-data:
  order-db-data:

networks:
  microservices-network:
    driver: bridge
```

### Understanding the Configuration

**Key Points:**

1. **Two Services:** `user-service` on port 8081, `order-service` on port 8082 — both map to container port 8080.
2. **Two Databases:** `user-db` on port 5433, `order-db` on port 5434 — avoid conflict with your local PostgreSQL on 5432.
3. **Service Communication:** Order Service uses `http://user-service:8080` — the container name, not `localhost`.
4. **Network:** All services share `microservices-network` for inter-container communication.
5. **Volumes:** Data persists across container restarts via named volumes.

---

## Part 5: Build and Deploy

### Step 1: Build User Service

```bash
cd microservices-demo/user-service
mvn clean package -DskipTests
ls -lh target/*.jar
```

Should see: `user-service-0.0.1-SNAPSHOT.jar`

---

### Step 2: Build Order Service

```bash
cd ../order-service
mvn clean package -DskipTests
ls -lh target/*.jar
```

Should see: `order-service-0.0.1-SNAPSHOT.jar`

---

### Step 3: Start All Services

```bash
cd ..
docker compose up -d --build
```

**Expected output:**
```
[+] Running 7/7
 ✔ Network microservices-demo_microservices-network  Created
 ✔ Container user-db                                  Created
 ✔ Container order-db                                 Created
 ✔ Container user-service                             Created
 ✔ Container order-service                            Created
```

> ⚠️ **VS Code users:** If running Docker Compose via VS Code task, you may see:
> `Terminal will be reused by tasks, press any key to close it.`
> **Do NOT press any key** — containers are still running in the background. Open a **new terminal** to continue.

---

### Step 4: Verify All Containers Running

```bash
docker compose ps
```

All four containers should show `Up`. If any show `Exit`, check the logs before proceeding.

---

### Step 5: Check Service Logs

```bash
docker compose logs user-service
docker compose logs order-service
```

Look for `Started ...Application in X.XXX seconds` in each.

---

## Part 6: Testing the Microservices **(30 minutes)**

### Viewing Data in DBeaver

The Docker databases run on different ports than your local PostgreSQL. Add two new connections in DBeaver:

**user-db:** Host `localhost`, Port `5433`, Database `userdb`, Username/Password `postgres`

**order-db:** Host `localhost`, Port `5434`, Database `orderdb`, Username/Password `postgres`

> ℹ️ These connections only work while Docker containers are running.

---

### Testing with Postman

Create a new collection called **Microservices Demo**.

#### User Service

**Create users** — `POST http://localhost:8081/api/users`
```json
{ "firstName": "John", "lastName": "Doe", "email": "john.doe@example.com" }
{ "firstName": "Jane", "lastName": "Smith", "email": "jane.smith@example.com" }
{ "firstName": "Bob", "lastName": "Johnson", "email": "bob.johnson@example.com" }
```

**Get all users** — `GET http://localhost:8081/api/users`

**Get user by ID** — `GET http://localhost:8081/api/users/1`

---

#### Order Service

**Create order (valid user)** — `POST http://localhost:8082/api/orders`
```json
{ "userId": 1, "productName": "Laptop", "quantity": 1, "totalPrice": 1299.99 }
```
Expected: `201 Created` — this proves Order Service successfully called User Service! ✅

**Create more orders:**
```json
{ "userId": 2, "productName": "Mouse", "quantity": 2, "totalPrice": 49.98 }
{ "userId": 1, "productName": "Keyboard", "quantity": 1, "totalPrice": 79.99 }
```

**Get all orders** — `GET http://localhost:8082/api/orders`

**Get orders by user** — `GET http://localhost:8082/api/orders/user/1`

**Test invalid user** — `POST http://localhost:8082/api/orders`
```json
{ "userId": 999, "productName": "Monitor", "quantity": 1, "totalPrice": 299.99 }
```
Expected: `400 Bad Request` — proves inter-service validation is working! ✅

---

### Testing with curl

```bash
# Create a user
curl -X POST http://localhost:8081/api/users \
  -H "Content-Type: application/json" \
  -d '{"firstName": "John", "lastName": "Doe", "email": "john.doe@example.com"}'

# Get all users
curl http://localhost:8081/api/users

# Create order for valid user
curl -X POST http://localhost:8082/api/orders \
  -H "Content-Type: application/json" \
  -d '{"userId": 1, "productName": "Laptop", "quantity": 1, "totalPrice": 1299.99}'

# Test invalid user
curl -v -X POST http://localhost:8082/api/orders \
  -H "Content-Type: application/json" \
  -d '{"userId": 999, "productName": "Monitor", "quantity": 1, "totalPrice": 299.99}'

# Get orders by user
curl http://localhost:8082/api/orders/user/1
```

---

## Part 7: Understanding What You Built

### Architecture Recap

```
                     Docker Compose Network
    ┌────────────────────────────────────────────────┐
    │                                                │
    │  ┌─────────────────┐    ┌─────────────────┐  │
    │  │  User Service   │    │  Order Service  │  │
    │  │   Port: 8081    │◄───│   Port: 8082    │  │
    │  │                 │REST│                 │  │
    │  └────────┬────────┘    └────────┬────────┘  │
    │           │                      │            │
    │      ┌────▼────┐            ┌───▼────┐       │
    │      │ User DB │            │Order DB│       │
    │      │ :5433   │            │ :5434  │       │
    │      └─────────┘            └────────┘       │
    └────────────────────────────────────────────────┘
```

### Key Microservices Concepts Applied

- ✅ **Service Decomposition** — separate services with clear boundaries
- ✅ **Independent Deployment** — each service has its own Dockerfile
- ✅ **Inter-Service Communication** — REST via RestTemplate
- ✅ **Decentralized Data** — each service has its own database
- ✅ **Configuration Management** — environment variables in Docker Compose

---

## Part 8: Advanced Concepts (Awareness Only)

### Service Discovery

**Current limitation:** Order Service has a hardcoded URL `http://user-service:8080`. In production with multiple instances, IPs change dynamically.

**Service Discovery** (e.g. Netflix Eureka, Consul) automatically finds available service instances. Services register themselves and others query the registry to find them.

> ℹ️ Not required to implement — awareness only. Reference: https://spring.io/guides/gs/service-registration-and-discovery/

---

### API Gateway

**Current limitation:** Clients must know all service URLs separately.

**API Gateway** provides a single entry point that routes requests to the correct service.

> ℹ️ Not required to implement — awareness only. Reference: https://spring.io/projects/spring-cloud-gateway

---

### Why We Didn't Implement These

Service Discovery and API Gateway are important for production systems but require additional infrastructure and are too complex for this foundation lesson. Master the basics first, then explore these when needed.

---

## Part 9: Clean Up

```bash
# Stop containers and remove network (keep data)
docker compose down

# Stop and remove everything including data volumes
docker compose down -v

# Restart later
docker compose up -d --build
```

---

## Summary

### What You Built

- **User Service** — independent Spring Boot app managing users with its own database
- **Order Service** — independent Spring Boot app managing orders, calling User Service for validation
- **Docker Compose** — orchestrates both services, both databases, and a shared network

### Key Takeaways

- Each microservice is an independent Spring Boot application with its own database
- `RestTemplate` enables synchronous HTTP calls between services
- Docker Compose container names (e.g. `user-service`) act as hostnames within the network
- Environment variables externalize configuration — no hardcoding, easy to change per environment
- Service Discovery and API Gateway are the natural next steps for production systems

---

## Additional Resources

- [Spring Boot Microservices Guide](https://spring.io/microservices)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Spring Cloud Documentation](https://spring.io/projects/spring-cloud)
- "Building Microservices" by Sam Newman
- "Microservices Patterns" by Chris Richardson

---

END