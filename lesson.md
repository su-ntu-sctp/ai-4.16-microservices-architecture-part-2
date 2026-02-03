# Lesson 4.16: Microservices Architecture - Part 2 (Practical Demo)


## Learning Objectives

By the end of this lesson, you will be able to:

1. **Build** two independent microservices from scratch
2. **Implement** REST-based inter-service communication
3. **Deploy** microservices using Docker Compose
4. **Test** the complete microservices system
5. **Understand** advanced concepts (Service Discovery & API Gateway)

---

## Prerequisites

Before starting this lesson, ensure you have:

- Completed Lesson 4.15 (Microservices Theory)
- Completed Lesson 4.6 (Docker Compose)
- Java 21 installed
- Maven installed
- Docker Desktop running
- IDE (IntelliJ IDEA or VS Code)

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

**Simple, realistic, and demonstrates microservices concepts!**

---

## Part 1: Project Setup (15 minutes)

### Step 1: Create Project Directory

Create a workspace for both services:

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

**Generate Spring Boot project:**

Go to https://start.spring.io/

**Configuration:**
- **Project:** Maven
- **Language:** Java
- **Spring Boot:** 3.2.2 (or latest 3.x)
- **Group:** com.example
- **Artifact:** user-service
- **Name:** user-service
- **Package name:** com.example.userservice
- **Packaging:** Jar
- **Java:** 21

**Dependencies:**
- Spring Web
- Spring Data JPA
- PostgreSQL Driver
- Lombok

**Click "Generate"** → Download → Extract to `microservices-demo/user-service/`

---

### Step 3: Create Order Service

**Generate another Spring Boot project:**

Go to https://start.spring.io/

**Configuration:**
- **Project:** Maven
- **Language:** Java
- **Spring Boot:** 3.2.2
- **Group:** com.example
- **Artifact:** order-service
- **Name:** order-service
- **Package name:** com.example.orderservice
- **Packaging:** Jar
- **Java:** 21

**Dependencies:**
- Spring Web
- Spring Data JPA
- PostgreSQL Driver
- Lombok

**Click "Generate"** → Download → Extract to `microservices-demo/order-service/`

---

### Step 4: Verify Structure

Your directory should look like:

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
└── (docker-compose.yml - we'll create this later)
```

---

## Part 2: Build User Service (40 minutes)

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

## Part 3: Build Order Service (45 minutes)

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

### Step 2: Create User DTO (for API responses)

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

### Step 4: Create Order Service (with User Service call)

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
        // Validate user exists by calling User Service
        Long userId = order.getUserId();
        String url = userServiceUrl + "/api/users/" + userId;
        
        try {
            UserDTO user = restTemplate.getForObject(url, UserDTO.class);
            
            if (user == null) {
                throw new RuntimeException("User not found with ID: " + userId);
            }
            
            // User exists, save order
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
    public ResponseEntity<Order> createOrder(@RequestBody Order order) {
        try {
            Order createdOrder = orderService.createOrder(order);
            return new ResponseEntity<>(createdOrder, HttpStatus.CREATED);
        } catch (Exception e) {
            return new ResponseEntity<>(HttpStatus.BAD_REQUEST);
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

## Part 4: Docker Compose Configuration (20 minutes)

### Step 1: Create docker-compose.yml

**File:** `microservices-demo/docker-compose.yml`

```yaml
version: "3.9"

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

---

### Understanding the Configuration

**Key Points:**

1. **Two Services:**
   - `user-service` runs on port 8081 (host) → 8080 (container)
   - `order-service` runs on port 8082 (host) → 8080 (container)

2. **Two Databases:**
   - `user-db` on port 5433 (host) → 5432 (container)
   - `order-db` on port 5434 (host) → 5432 (container)

3. **Service Communication:**
   - Order Service calls: `http://user-service:8080/api/users/{id}`
   - Uses container name (`user-service`), not `localhost`

4. **Network:**
   - All services on same network: `microservices-network`
   - Enables service-to-service communication

5. **Volumes:**
   - Data persists across container restarts
   - Separate volumes for each database

---

## Part 5: Build and Deploy (25 minutes)

### Step 1: Build User Service

```bash
cd microservices-demo/user-service
mvn clean package -DskipTests
```

**Verify JAR created:**
```bash
ls -lh target/*.jar
```

Should see: `user-service-0.0.1-SNAPSHOT.jar`

---

### Step 2: Build Order Service

```bash
cd ../order-service
mvn clean package -DskipTests
```

**Verify JAR created:**
```bash
ls -lh target/*.jar
```

Should see: `order-service-0.0.1-SNAPSHOT.jar`

---

### Step 3: Start All Services

```bash
cd ..
docker compose up -d --build
```

**What happens:**
1. Builds Docker images for both services
2. Starts PostgreSQL databases
3. Starts User Service
4. Starts Order Service
5. Creates network

**Expected output:**
```
[+] Running 5/5
 ✔ Network microservices-demo_microservices-network  Created
 ✔ Container user-db                                  Started
 ✔ Container order-db                                 Started
 ✔ Container user-service                             Started
 ✔ Container order-service                            Started
```

---

### Step 4: Verify All Containers Running

```bash
docker compose ps
```

**Expected output:**
```
NAME            IMAGE                     STATUS         PORTS
user-service    microservices-demo-user   Up 30 seconds  0.0.0.0:8081->8080/tcp
order-service   microservices-demo-order  Up 30 seconds  0.0.0.0:8082->8080/tcp
user-db         postgres:15               Up 35 seconds  0.0.0.0:5433->5432/tcp
order-db        postgres:15               Up 35 seconds  0.0.0.0:5434->5432/tcp
```

All should show `Up`!

---

### Step 5: Check Service Logs

**User Service logs:**
```bash
docker compose logs user-service
```

**Look for:**
```
Started UserServiceApplication in X.XXX seconds
Tomcat started on port 8080
```

**Order Service logs:**
```bash
docker compose logs order-service
```

**Look for:**
```
Started OrderServiceApplication in X.XXX seconds
Tomcat started on port 8080
```

---

## Part 6: Testing the Microservices (30 minutes)

### Step 1: Test User Service

**Create a user:**

```bash
curl -X POST http://localhost:8081/api/users \
  -H "Content-Type: application/json" \
  -d '{
    "firstName": "John",
    "lastName": "Doe",
    "email": "john.doe@example.com"
  }'
```

**Expected response:**
```json
{
  "id": 1,
  "firstName": "John",
  "lastName": "Doe",
  "email": "john.doe@example.com"
}
```

**Create more users:**

```bash
curl -X POST http://localhost:8081/api/users \
  -H "Content-Type: application/json" \
  -d '{
    "firstName": "Jane",
    "lastName": "Smith",
    "email": "jane.smith@example.com"
  }'

curl -X POST http://localhost:8081/api/users \
  -H "Content-Type: application/json" \
  -d '{
    "firstName": "Bob",
    "lastName": "Johnson",
    "email": "bob.johnson@example.com"
  }'
```

---

### Step 2: Get All Users

```bash
curl http://localhost:8081/api/users
```

**Expected response:**
```json
[
  {
    "id": 1,
    "firstName": "John",
    "lastName": "Doe",
    "email": "john.doe@example.com"
  },
  {
    "id": 2,
    "firstName": "Jane",
    "lastName": "Smith",
    "email": "jane.smith@example.com"
  },
  {
    "id": 3,
    "firstName": "Bob",
    "lastName": "Johnson",
    "email": "bob.johnson@example.com"
  }
]
```

---

### Step 3: Get User by ID

```bash
curl http://localhost:8081/api/users/1
```

**Expected response:**
```json
{
  "id": 1,
  "firstName": "John",
  "lastName": "Doe",
  "email": "john.doe@example.com"
}
```

---

### Step 4: Test Order Service (Creates Order + Validates User)

**Create order for existing user (userId=1):**

```bash
curl -X POST http://localhost:8082/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "userId": 1,
    "productName": "Laptop",
    "quantity": 1,
    "totalPrice": 1299.99
  }'
```

**Expected response:**
```json
{
  "id": 1,
  "userId": 1,
  "productName": "Laptop",
  "quantity": 1,
  "totalPrice": 1299.99,
  "status": "PENDING",
  "orderDate": "2026-02-03T12:30:45"
}
```

**This proves Order Service successfully called User Service to validate user!**

---

### Step 5: Create More Orders

```bash
curl -X POST http://localhost:8082/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "userId": 2,
    "productName": "Mouse",
    "quantity": 2,
    "totalPrice": 49.98
  }'

curl -X POST http://localhost:8082/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "userId": 1,
    "productName": "Keyboard",
    "quantity": 1,
    "totalPrice": 79.99
  }'
```

---

### Step 6: Get All Orders

```bash
curl http://localhost:8082/api/orders
```

**Expected response:**
```json
[
  {
    "id": 1,
    "userId": 1,
    "productName": "Laptop",
    "quantity": 1,
    "totalPrice": 1299.99,
    "status": "PENDING",
    "orderDate": "2026-02-03T12:30:45"
  },
  {
    "id": 2,
    "userId": 2,
    "productName": "Mouse",
    "quantity": 2,
    "totalPrice": 49.98,
    "status": "PENDING",
    "orderDate": "2026-02-03T12:31:12"
  },
  {
    "id": 3,
    "userId": 1,
    "productName": "Keyboard",
    "quantity": 1,
    "totalPrice": 79.99,
    "status": "PENDING",
    "orderDate": "2026-02-03T12:31:45"
  }
]
```

---

### Step 7: Get Orders by User

```bash
curl http://localhost:8082/api/orders/user/1
```

**Expected response:**
```json
[
  {
    "id": 1,
    "userId": 1,
    "productName": "Laptop",
    "quantity": 1,
    "totalPrice": 1299.99,
    "status": "PENDING",
    "orderDate": "2026-02-03T12:30:45"
  },
  {
    "id": 3,
    "userId": 1,
    "productName": "Keyboard",
    "quantity": 1,
    "totalPrice": 79.99,
    "status": "PENDING",
    "orderDate": "2026-02-03T12:31:45"
  }
]
```

---

### Step 8: Test Error Handling (Invalid User)

**Try to create order for non-existent user:**

```bash
curl -X POST http://localhost:8082/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "userId": 999,
    "productName": "Monitor",
    "quantity": 1,
    "totalPrice": 299.99
  }'
```

**Expected response:**
- HTTP 400 Bad Request
- Order NOT created

**This proves:**
- Order Service called User Service
- User Service returned 404
- Order Service rejected the order

**Inter-service validation working!** ✅

---

### Step 9: View Service Logs

**Check Order Service logs to see User Service call:**

```bash
docker compose logs order-service | grep "User not found"
```

Should see error message about user 999 not found.

---

## Part 7: Understanding What You Built (15 minutes)

### Architecture Recap

```
┌──────────────────────────────────────────────────────┐
│         Microservices System Running                  │
└──────────────────────────────────────────────────────┘

                     Docker Compose Network
    ┌────────────────────────────────────────────────┐
    │                                                │
    │  ┌─────────────────┐    ┌─────────────────┐  │
    │  │  User Service   │    │  Order Service  │  │
    │  │   Port: 8081    │◄───│   Port: 8082    │  │
    │  │                 │REST│                 │  │
    │  │  - Create user  │    │  - Create order │  │
    │  │  - Get users    │    │  - Validate user│  │
    │  └────────┬────────┘    └────────┬────────┘  │
    │           │                      │            │
    │      ┌────▼────┐            ┌───▼────┐       │
    │      │ User DB │            │Order DB│       │
    │      │ :5433   │            │ :5434  │       │
    │      └─────────┘            └────────┘       │
    │                                                │
    └────────────────────────────────────────────────┘
             Accessible from localhost
```

---

### What You Demonstrated

**1. Independent Services**
- User Service runs independently
- Order Service runs independently
- Each has own database
- Each can be scaled separately

**2. REST Communication**
- Order Service calls User Service via HTTP
- Uses container name (`user-service`) for discovery
- Validates user before creating order

**3. Data Independence**
- User Service owns user data
- Order Service owns order data
- No shared database

**4. Docker Compose Orchestration**
- All services defined in one file
- Started with single command
- Automatic networking
- Data persistence with volumes

---

### Key Microservices Concepts Applied

**✅ Service Decomposition**
- Separate services for users and orders
- Clear boundaries

**✅ Independent Deployment**
- Can update User Service without touching Order Service
- Each service has own Dockerfile

**✅ Inter-Service Communication**
- REST API calls between services
- JSON data exchange

**✅ Decentralized Data**
- Each service has own database
- No shared tables

**✅ Configuration Management**
- Environment variables in docker-compose.yml
- Different configs per service

---

## Part 8: Advanced Concepts (20 minutes)

### Service Discovery (Concept)

**Current Limitation:**

Order Service has hardcoded URL:
```
USER_SERVICE_URL=http://user-service:8080
```

**Problem in production:**
- User Service may have multiple instances
- IPs change dynamically
- Need to know which instance to call

---

### What is Service Discovery?

**Service Discovery** automatically finds available service instances.

```
┌─────────────────────────────────────────────────┐
│          Service Discovery Pattern               │
└─────────────────────────────────────────────────┘

                    ┌──────────────┐
                    │   Service    │
                    │   Registry   │
                    │  (Eureka)    │
                    └──────┬───────┘
                           │
        ┌──────────────────┼──────────────────┐
        │ Register         │         Query    │
        │                  │                  │
   ┌────▼────┐        ┌───▼────┐        ┌───▼────┐
   │User Svc │        │User Svc│        │Order   │
   │Instance1│        │Instance2│        │Service │
   └─────────┘        └────────┘        └────────┘
     :8081              :8082          Calls any
                                       available
                                       instance

How it works:
1. User Service instances register with Service Registry
2. Order Service queries registry for User Service
3. Registry returns available instances
4. Order Service calls one of them (load balanced)
```

**Benefits:**
- ✅ Dynamic instance discovery
- ✅ Automatic load balancing
- ✅ Fault tolerance (if one instance down, call another)

**Popular Tools:**
- Netflix Eureka (Spring Cloud)
- Consul
- etcd
- Kubernetes Service Discovery

---

### Service Discovery with Eureka (Example)

**How it would work:**

**1. Setup Eureka Server:**
```yaml
# Eureka Server
eureka-server:
  image: springcloud/eureka
  ports:
    - "8761:8761"
```

**2. User Service registers:**
```properties
# User Service
eureka.client.service-url.defaultZone=http://eureka-server:8761/eureka
eureka.instance.instance-id=${spring.application.name}:${random.value}
```

**3. Order Service queries Eureka:**
```java
// Instead of hardcoded URL
@LoadBalanced
@Bean
public RestTemplate restTemplate() {
    return new RestTemplate();
}

// Call using service name (Eureka resolves to actual instance)
String url = "http://user-service/api/users/" + userId;
User user = restTemplate.getForObject(url, User.class);
```

**Reference:**
https://spring.io/guides/gs/service-registration-and-discovery/

---

### API Gateway (Concept)

**Current Limitation:**

Clients must know all service URLs:
```
User Service:  http://localhost:8081/api/users
Order Service: http://localhost:8082/api/orders
```

**Problem:**
- Multiple entry points
- CORS issues
- Authentication for each service
- Client knows internal architecture

---

### What is an API Gateway?

**API Gateway** = Single entry point for all services.

```
┌─────────────────────────────────────────────────┐
│             API Gateway Pattern                  │
└─────────────────────────────────────────────────┘

                  ┌──────────────┐
     Client ─────>│ API Gateway  │
    (Browser)     │   :8080      │
                  └──────┬───────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        │ Route          │ Route          │
        │ /users         │ /orders        │
        │                │                │
   ┌────▼────┐      ┌───▼────┐      ┌───▼────┐
   │User Svc │      │Order   │      │Payment │
   │:8081    │      │Service │      │Service │
   └─────────┘      │:8082   │      │:8083   │
                    └────────┘      └────────┘

Client makes ONE call:
- GET http://api-gateway:8080/users → routes to User Service
- POST http://api-gateway:8080/orders → routes to Order Service
- POST http://api-gateway:8080/payments → routes to Payment Service
```

**Benefits:**
- ✅ Single entry point
- ✅ Centralized authentication
- ✅ Rate limiting
- ✅ Request routing
- ✅ Load balancing
- ✅ Hide internal structure

**Popular Tools:**
- Spring Cloud Gateway
- Netflix Zuul
- Kong
- AWS API Gateway
- Azure API Management

---

### API Gateway with Spring Cloud Gateway (Example)

**How it would work:**

**1. Add API Gateway service:**
```yaml
# docker-compose.yml
api-gateway:
  image: api-gateway
  ports:
    - "8080:8080"
  environment:
    USER_SERVICE_URL: http://user-service:8081
    ORDER_SERVICE_URL: http://order-service:8082
```

**2. Configure routing:**
```java
// API Gateway Configuration
@Configuration
public class GatewayConfig {
    
    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            // Route /users/** to User Service
            .route("user-service", r -> r
                .path("/users/**")
                .uri("http://user-service:8081"))
            
            // Route /orders/** to Order Service
            .route("order-service", r -> r
                .path("/orders/**")
                .uri("http://order-service:8082"))
            
            .build();
    }
}
```

**3. Client calls only API Gateway:**
```bash
# All calls go through API Gateway
curl http://localhost:8080/users
curl http://localhost:8080/orders
```

**Reference:**
https://spring.io/projects/spring-cloud-gateway

---

### Why We Didn't Implement These

**Service Discovery & API Gateway are:**
- ✅ Important for production systems
- ✅ Necessary for large-scale microservices
- ❌ Too complex for beginners
- ❌ Require additional infrastructure

**What you built today:**
- ✅ Demonstrates core microservices concepts
- ✅ Shows inter-service communication
- ✅ Realistic for learning
- ✅ Foundation for advanced topics

**Next steps:**
- Master these basics first
- Then learn Service Discovery
- Then learn API Gateway
- Build incrementally!

---

## Part 9: Clean Up and Summary (10 minutes)

### Stop All Services

```bash
docker compose down
```

**Removes:**
- ✅ All containers
- ✅ Network
- ✅ Keeps volumes (data persists)

### Remove Everything Including Data

```bash
docker compose down -v
```

**Removes:**
- ✅ Containers
- ✅ Network
- ✅ Volumes (data deleted!)

### Restart Services Later

```bash
# Build and start
docker compose up -d --build

# Or just start (if already built)
docker compose up -d
```

---

## Summary

### What You Built

**Two Independent Microservices:**

**User Service:**
- ✅ Manages users (CRUD)
- ✅ Exposes REST API
- ✅ Own database
- ✅ Independent deployment

**Order Service:**
- ✅ Manages orders (CRUD)
- ✅ Calls User Service (inter-service communication)
- ✅ Own database
- ✅ Independent deployment

**Complete System:**
- ✅ Docker Compose orchestration
- ✅ Multiple databases
- ✅ Network communication
- ✅ Data persistence

---

### Key Takeaways

**1. Microservices in Practice**
- Each service is independent Spring Boot application
- Services communicate via REST APIs
- Docker Compose simplifies orchestration

**2. Inter-Service Communication**
- RestTemplate for HTTP calls
- Service names for discovery (via Docker network)
- Error handling for service failures

**3. Data Independence**
- Each service owns its data
- No shared database
- Data duplication acceptable

**4. Configuration Management**
- Environment variables in Docker Compose
- Different configs per service
- Externalized configuration

**5. Advanced Concepts**
- Service Discovery for dynamic instance location
- API Gateway for single entry point
- Production systems need these

---

### Microservices Benefits Demonstrated

**✅ Independent Deployment**
- Update User Service without touching Order Service

**✅ Technology Flexibility**
- Could rewrite User Service in Python
- Order Service wouldn't change

**✅ Scalability**
- Scale User Service independently
- ```yaml
  user-service:
    deploy:
      replicas: 3  # Run 3 instances
  ```

**✅ Fault Isolation**
- If User Service down, Order Service can handle gracefully

---

### Challenges You Experienced

**❌ Complexity**
- Two projects instead of one
- More configuration files
- More moving parts

**❌ Testing Complexity**
- Must test both services
- Must test integration

**❌ Network Latency**
- HTTP calls slower than method calls
- User validation adds latency to order creation

**❌ Operations**
- More containers to manage
- More logs to check
- More potential failure points

---

### When to Use Microservices

**Use Microservices for:**
- ✅ Large applications
- ✅ Large teams
- ✅ Different scaling needs
- ✅ Technology diversity
- ✅ Frequent deployments

**Don't Use for:**
- ❌ Small applications
- ❌ Small teams
- ❌ MVPs/Prototypes
- ❌ Simple requirements

**Remember:** Start with monolith, evolve to microservices when needed!

---

## Additional Resources

### Documentation
- [Spring Boot Microservices Guide](https://spring.io/microservices)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Spring Cloud Documentation](https://spring.io/projects/spring-cloud)

### Books
- "Building Microservices" by Sam Newman
- "Microservices Patterns" by Chris Richardson

### Videos
- [Microservices Tutorial](https://www.youtube.com/results?search_query=spring+boot+microservices+tutorial)
- [Docker Compose with Spring Boot](https://www.youtube.com/results?search_query=docker+compose+spring+boot+microservices)

### GitHub Examples
- [Spring Boot Microservices Example](https://github.com/search?q=spring+boot+microservices+example)

---


**You now understand:**
- ✅ How microservices work in practice
- ✅ Inter-service REST communication
- ✅ Docker Compose orchestration
- ✅ Service decomposition
- ✅ Advanced concepts (Service Discovery, API Gateway)

**You're ready to build larger microservices systems!** 
