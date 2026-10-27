# Lesson 4.16: Microservices Architecture - Part 2 (Practical)

## Learning Objectives

By the end of this lesson, you will be able to:

1. **Containerize** two Spring Boot microservices with multi-stage Dockerfiles
2. **Run and test** a multi-service system locally with Docker Compose, using service names, environment variables and healthchecks
3. **Publish** each service's image to Docker Hub
4. **Deploy** the services to Railway from Docker Hub and connect them over Railway's private network
5. **Describe** Service Discovery and API Gateway as next steps beyond this foundation

---

## Prerequisites

Before starting this lesson, ensure you have:

- Completed Lesson 4.15 (Microservices Architecture - Part 1)
- Completed Lesson 4.4 (multi-stage Dockerfiles), Lesson 4.6 (Docker Compose) and Lesson 4.10 (Docker Hub + Railway)
- Docker Desktop installed and running
- A Docker Hub account
- A Railway account (from Lesson 4.10)
- Postman (or `curl` in your terminal)

> **No Java or Maven needed on your computer today.** The multi-stage Dockerfiles you write run Maven inside Docker.

---

## Introduction

In Lesson 4.15 you learned the theory of microservices. Today you'll **ship** a real microservices system - first on your own computer, then live on Railway.

You are the **DevOps engineer** on this team. The developers have already written two Spring Boot services. Your job is to package, run, publish and deploy them.

**What you'll deploy:**

```
┌─────────────────┐   GET /api/users/{id}   ┌─────────────────┐
│  User Service   │◄────────────────────────│  Order Service  │
│                 │          REST           │                 │
│ - Create users  │                         │ - Create orders │
│ - Get users     │                         │ - Get orders    │
│                 │                         │ - Checks the    │
│                 │                         │   user exists   │
│   [H2 userdb]   │                         │  [H2 orderdb]   │
└─────────────────┘                         └─────────────────┘
```

**Scenario:**
- User Service manages users
- Order Service manages orders
- When an order is created, Order Service calls User Service to check the user exists

**Where it runs today:**

| | Local (Parts 2-4) | Live (Parts 5-6) |
|---|---|---|
| Images built by | Docker Compose (`build:`) | You (`docker build` + `docker push`) |
| Images stored in | Your computer | Docker Hub |
| Containers run by | Docker Compose | Railway |
| Order Service finds User Service at | `http://user-service:8080` | `http://user-service.railway.internal:8080` |

---

## Part 1: Get the Project

### Step 1: Download the Starter Project

For Windows users, use the WSL terminal only. For macOS users, use your preferred terminal.

```sh
cd ~
cd playground
curl -L -O https://github.com/aie-devops/microservices-demo/archive/refs/tags/v4.16.0.zip
unzip v4.16.0.zip
cd microservices-demo-4.16.0
```

Open the folder in VS Code:

```sh
code .
```

---

### Step 2: Explore the Structure

```
microservices-demo-4.16.0/
├── README.md
├── user-service/
│   ├── pom.xml
│   └── src/main/
│       ├── java/com/example/userservice/
│       │   ├── controller/UserController.java
│       │   ├── entity/User.java
│       │   ├── repository/UserRepository.java
│       │   └── service/UserService.java
│       └── resources/application.properties
└── order-service/
    ├── pom.xml
    └── src/main/
        ├── java/com/example/orderservice/
        │   ├── config/AppConfig.java          <- RestTemplate with timeouts
        │   ├── controller/OrderController.java
        │   ├── dto/UserDTO.java
        │   ├── entity/Order.java
        │   ├── exception/...
        │   ├── repository/OrderRepository.java
        │   └── service/OrderService.java       <- calls User Service
        └── resources/application.properties
```

Two **independent** Spring Boot applications - each has its own `pom.xml`, its own code and its own database. There are **no Dockerfiles and no Docker Compose file** yet. Adding those is your job today.

---

### Step 3: Read the Parts That Matter for DevOps

You don't need to understand every line of Java. Focus on **how the services are configured** and **how they talk to each other**.

**`order-service/src/main/resources/application.properties`**

```properties
# Port inside the container. Override with the SERVER_PORT environment variable.
server.port=${SERVER_PORT:8080}

# H2 in-memory database - owned by this service only.
spring.datasource.url=jdbc:h2:mem:orderdb

# Where to find the User Service. Set the USER_SERVICE_URL environment variable.
user.service.url=${USER_SERVICE_URL:http://localhost:8081}
```

**`order-service/.../service/OrderService.java`** (the important part)

```java
// Synchronous REST call: GET {USER_SERVICE_URL}/api/users/{id}
private void validateUserExists(Long userId) {
    String url = userServiceUrl + "/api/users/" + userId;
    try {
        restTemplate.getForObject(url, UserDTO.class);
    } catch (HttpClientErrorException.NotFound e) {
        throw new UserNotFoundException(userId);              // -> 400 Bad Request
    } catch (ResourceAccessException e) {
        throw new UserServiceUnavailableException(userServiceUrl);  // -> 503 Service Unavailable
    }
}
```

**Configuration you will set as a DevOps engineer:**

| Environment variable | Service | Default | What it does |
|---|---|---|---|
| `SERVER_PORT` | both | `8080` | Port the service listens on inside its container |
| `USER_SERVICE_URL` | order-service | `http://localhost:8081` | Where Order Service sends its REST calls |

**Endpoints:**

| Service | Endpoint | Purpose |
|---|---|---|
| user-service | `GET /api/users` | List users |
| user-service | `GET /api/users/{id}` | Get one user (Order Service calls this) |
| user-service | `POST /api/users` | Create a user |
| order-service | `GET /api/orders` | List orders |
| order-service | `GET /api/orders/user/{userId}` | Orders for one user |
| order-service | `POST /api/orders` | Create an order (checks the user first) |
| both | `GET /actuator/health` | Health check - returns `{"status":"UP"}` |

> **Why H2?** H2 is an **in-memory** database that runs inside each service - no database server to set up. Each service still owns its own data (`userdb` and `orderdb`), which is the "database per service" rule from Lesson 4.15. The trade-off: **all data is lost whenever a service restarts.** A real system would give each service its own persistent database.

---

## Part 2: Write the Dockerfiles

Each microservice is built and shipped as its **own image**. You'll use the multi-stage pattern from Lesson 4.4: stage 1 builds the JAR with Maven, stage 2 runs it on a small Java runtime image.

### Step 1: Create the User Service Dockerfile

**File:** `user-service/Dockerfile`

```dockerfile
# Stage 1: Build the JAR with Maven
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: Run the JAR on a small Java runtime image
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
CMD ["java", "-jar", "app.jar"]
```

| Line | What it does |
|---|---|
| `FROM maven:3.9-eclipse-temurin-21 AS build` | Stage 1 has Maven and Java 21 - used only to build |
| `COPY pom.xml .` / `COPY src ./src` | Copy only what Maven needs |
| `RUN mvn clean package -DskipTests` | Build the JAR inside Docker |
| `FROM eclipse-temurin:21-jre-alpine` | Stage 2 starts fresh from a small **JRE** image (no Maven, no compiler) |
| `COPY --from=build ...` | Take just the JAR from stage 1 |
| `EXPOSE 8080` | Document the port the service listens on |

---

### Step 2: Create the User Service .dockerignore

**File:** `user-service/.dockerignore`

```
target/
.git
.idea/
.vscode/
*.md
```

> Because Maven builds the JAR **inside** Docker, we can ignore the whole `target/` folder - unlike Lesson 4.4's single-stage Dockerfile, which needed `target/*.jar`.

---

### Step 3: Create the Order Service Dockerfile and .dockerignore

The Order Service is also a Spring Boot app built with Maven, so the Dockerfile is **exactly the same**.

**File:** `order-service/Dockerfile`

```dockerfile
# Stage 1: Build the JAR with Maven
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: Run the JAR on a small Java runtime image
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
CMD ["java", "-jar", "app.jar"]
```

**File:** `order-service/.dockerignore`

```
target/
.git
.idea/
.vscode/
*.md
```

> **Think about it:** both services listen on port **8080** inside their containers. Why doesn't that clash? (Each container has its own network namespace - you'll map them to different **host** ports in Part 3.)

---

### Step 4: Check Your Files

```
microservices-demo-4.16.0/
├── user-service/
│   ├── Dockerfile        ✓
│   ├── .dockerignore     ✓
│   ├── pom.xml
│   └── src/
└── order-service/
    ├── Dockerfile        ✓
    ├── .dockerignore     ✓
    ├── pom.xml
    └── src/
```

---

## Part 3: Docker Compose

One `docker build` + `docker run` per service gets tedious fast. Docker Compose builds and runs the **whole system** with one command.

### Step 1: Create docker-compose.yml

**File:** `docker-compose.yml` (in the project root, next to the two service folders)

```yaml
services:
  user-service:
    build: ./user-service
    container_name: user-service
    ports:
      - "8081:8080"
    environment:
      SERVER_PORT: 8080
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8080/actuator/health"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 30s

  order-service:
    build: ./order-service
    container_name: order-service
    ports:
      - "8082:8080"
    environment:
      SERVER_PORT: 8080
      USER_SERVICE_URL: http://user-service:8080
    depends_on:
      user-service:
        condition: service_healthy
```

---

### Step 2: Understand the Configuration

| Setting | What it does |
|---|---|
| `build: ./user-service` | Build the image from the Dockerfile in that folder |
| `ports: "8081:8080"` | Host port **8081** → container port **8080** (order-service uses **8082**) |
| `SERVER_PORT: 8080` | The port Spring Boot listens on inside the container |
| `USER_SERVICE_URL: http://user-service:8080` | Order Service calls User Service by its **service name** and **container port** |
| `healthcheck` | Docker calls `/actuator/health` until the service reports `UP` (`wget` is built into the Alpine image) |
| `depends_on: condition: service_healthy` | Start order-service only once user-service is **healthy** - same idea as the database healthcheck in Lesson 4.8 |

**Two kinds of address - don't mix them up:**

```
Your computer (Postman, browser)            Inside the Compose network
──────────────────────────────              ──────────────────────────────
http://localhost:8081  ──► user-service     order-service ──► http://user-service:8080
http://localhost:8082  ──► order-service
```

- From **your computer**, use `localhost` and the **host** ports (8081, 8082)
- From **one container to another**, use the **service name** and the **container** port (8080). Inside a container, `localhost` means *that container itself*.

> **No `networks:` section?** Docker Compose automatically creates a network for the project and registers each service name in its DNS. That's why `user-service` works as a hostname.
>
> **No databases or volumes?** Each service runs its own H2 database inside its container.

---

### Step 3: Build and Start Everything

From the project root:

```bash
docker compose up --detach --build
```

| Part of the command | What it does |
|---|---|
| `docker compose up` | Creates and starts every service in `docker-compose.yml`, plus a network so the services can reach each other by name |
| `--build` | Rebuilds both images from their Dockerfiles before starting |
| `--detach` | Runs the containers in the background and gives you your terminal back (`-d` is the short form) |

**Why `--build`?** Without it, Compose builds an image only the **first** time. After that it reuses the existing image, so changes to the Java code or a Dockerfile are silently ignored and you'd be running an old version. With `--build`, what runs always matches the code in the folder - and Docker's layer cache keeps unchanged steps fast.

**Why `--detach`?** Without it, Compose streams the logs of every container into your terminal and stops them all when you press `Ctrl+C`.

The first build takes a few minutes while Maven downloads dependencies for both services.

> ⚠️ **VS Code users:** if you see `Terminal will be reused by tasks, press any key to close it.`, **don't press any key** - the containers keep running. Open a new terminal.

---

### Step 4: Check the Containers

```bash
docker compose ps
```

Expected:

```
NAME            IMAGE                                STATUS
order-service   microservices-demo-4160-order-service   Up 20 seconds
user-service    microservices-demo-4160-user-service    Up 45 seconds (healthy)
```

- `user-service` shows **(healthy)** because it has a healthcheck
- `order-service` started only **after** user-service became healthy

Check the logs:

```bash
docker compose logs user-service
docker compose logs order-service
```

Look for `Started UserServiceApplication` and `Started OrderServiceApplication`.

> **Tip:** add `--follow` to keep watching new log lines as they arrive, e.g. `docker compose logs --follow order-service`. Press `Ctrl+C` to stop watching - the container keeps running.

---

## Part 4: Test the System Locally

### Step 1: Create Users

**Postman:** create a collection called **Microservices Demo** and send `POST http://localhost:8081/api/users` with a JSON body. Or use `curl`:

```bash
curl -X POST http://localhost:8081/api/users \
  -H "Content-Type: application/json" \
  -d '{"firstName": "John", "lastName": "Doe", "email": "john.doe@example.com"}'

curl -X POST http://localhost:8081/api/users \
  -H "Content-Type: application/json" \
  -d '{"firstName": "Jane", "lastName": "Smith", "email": "jane.smith@example.com"}'

curl http://localhost:8081/api/users
```

Expected: `201 Created` for each user, then a list with users `1` and `2`.

---

### Step 2: Create Orders

```bash
# Valid user
curl -i -X POST http://localhost:8082/api/orders \
  -H "Content-Type: application/json" \
  -d '{"userId": 1, "productName": "Laptop", "quantity": 1, "totalPrice": 1299.99}'

# Another order
curl -i -X POST http://localhost:8082/api/orders \
  -H "Content-Type: application/json" \
  -d '{"userId": 2, "productName": "Mouse", "quantity": 2, "totalPrice": 49.98}'

# User that doesn't exist
curl -i -X POST http://localhost:8082/api/orders \
  -H "Content-Type: application/json" \
  -d '{"userId": 999, "productName": "Monitor", "quantity": 1, "totalPrice": 299.99}'

# Orders for user 1
curl http://localhost:8082/api/orders/user/1
```

| Request | Expected | What it proves |
|---|---|---|
| Order for user 1 | `201 Created` | Order Service called User Service and found the user ✅ |
| Order for user 999 | `400 Bad Request` - `User not found with ID: 999` | User Service answered `404`, Order Service rejected the order ✅ |

> `-i` prints the HTTP status line so you can see the status code.

---

### 🧑‍💻 Activity: Break It on Purpose **(10 minutes)**

In Lesson 4.15 you learned that a **synchronous** call fails when the service it depends on is down. Let's see it happen.

1. Stop **only** the User Service (the container is stopped, not removed):
   ```bash
   docker compose stop user-service
   ```
2. Try to create an order for user 1 (same `curl` as above). What status code do you get? What does the message say?
3. Can you still list orders with `GET http://localhost:8082/api/orders`? Why?
4. Start the User Service again:
   ```bash
   docker compose start user-service
   ```
5. Create an order for user 1 again. What happens now - and why?
   <details>
   <summary>Hint</summary>

   User Service keeps its data in an **in-memory** H2 database. When it restarted, its database started empty - user 1 no longer exists. Create the user again, then retry the order.

   </details>

**Discuss with a partner:**
- Which parts of the system kept working while User Service was down?
- How could the timeouts in `AppConfig.java` protect Order Service?
- How would an **asynchronous** (event-driven) design change what happens?

---

## Part 5: Publish the Images to Docker Hub

Docker Compose built the images **on your computer**. Railway can't see them there - it pulls images from a **registry**. You'll push both images to Docker Hub, just like devops-demo in Lesson 4.10.

```
Your computer                 Docker Hub                                Railway
docker build + push  ───►  YOUR_USERNAME/user-service:v4.16.0   ───►  user-service
                           YOUR_USERNAME/order-service:v4.16.0  ───►  order-service
```

### Step 1: Log In

```bash
docker login
```

---

### Step 2: Build and Push Both Images

From the project root (replace `YOUR_DOCKERHUB_USERNAME`):

```bash
docker build --platform linux/amd64 -t YOUR_DOCKERHUB_USERNAME/user-service:v4.16.0 ./user-service
docker push YOUR_DOCKERHUB_USERNAME/user-service:v4.16.0

docker build --platform linux/amd64 -t YOUR_DOCKERHUB_USERNAME/order-service:v4.16.0 ./order-service
docker push YOUR_DOCKERHUB_USERNAME/order-service:v4.16.0
```

| Part of the command | Why |
|---|---|
| `--platform linux/amd64` | Railway runs on `amd64` servers. Without it, an Apple Silicon Mac builds an ARM image that crashes with `exec format error` |
| `-t YOUR_DOCKERHUB_USERNAME/user-service:v4.16.0` | One Docker Hub **repository per service**, tagged with the lesson version |
| `./user-service` | The build context - the folder with that service's Dockerfile |

> On Apple Silicon these builds are slower because Docker emulates `amd64`.

---

### Step 3: Check Docker Hub

1. Go to [https://hub.docker.com](https://hub.docker.com)
2. You should see **two** repositories: `user-service` and `order-service`, each with tag `v4.16.0`
3. Check both are **Public** (Railway can't pull private images without extra setup)

<details>
<summary>Build or push not working? Use the instructor's images instead.</summary>

If you can't build or push your own images in time, use the instructor's public images in Part 6:

```
jsstrn/user-service:v4.16.0
jsstrn/order-service:v4.16.0
```

They are built from the same `v4.16.0` code.

</details>

---

## Part 6: Deploy to Railway

On Railway, one **project** holds the whole system and each microservice is its own **service**. Railway doesn't run your `docker-compose.yml` - you recreate the same setup in the dashboard.

| Docker Compose (local) | Railway (live) |
|---|---|
| `docker-compose.yml` | A Railway **project** |
| `services: user-service:` | A Railway **service** named `user-service` |
| `build: ./user-service` | **Source image** `YOUR_USERNAME/user-service:v4.16.0` from Docker Hub |
| `environment:` | The service's **Variables** tab |
| `ports: "8081:8080"` | **Generate Domain** with port `8080` (public HTTPS URL) |
| Service name as hostname (`user-service`) | **Private network** hostname `user-service.railway.internal` |

### Step 1: Deploy the User Service

1. Go to [https://railway.app](https://railway.app) and click **"New Project"**
2. Select **"Deploy from Docker Image"**
3. Enter your image:
   ```
   YOUR_DOCKERHUB_USERNAME/user-service:v4.16.0
   ```
4. Click **"Deploy"**
5. If you see a warning that the project is temporary, click **"Claim Project"**

**Check the service name:** click the service, open **Settings** and make sure the **service name** is exactly `user-service`. Rename it if needed - the name becomes its private hostname.

**Generate a public domain:**

1. In **Settings → Networking → Public Networking**, click **"Generate Domain"**
2. **Port:** `8080`
3. Click **"Generate Domain"**

**Test it** (replace with your domain):

```bash
curl https://user-service-production-xxxx.up.railway.app/actuator/health
curl -X POST https://user-service-production-xxxx.up.railway.app/api/users \
  -H "Content-Type: application/json" \
  -d '{"firstName": "John", "lastName": "Doe", "email": "john.doe@example.com"}'
```

Expected: `{"status":"UP"}`, then `201 Created`.

---

### Step 2: Deploy the Order Service in the Same Project

1. In the **same project**, click **"+ Create"** (or right-click the canvas) → **"Docker Image"**
2. Enter:
   ```
   YOUR_DOCKERHUB_USERNAME/order-service:v4.16.0
   ```
3. Check the service name is `order-service`

**Tell Order Service where to find User Service:**

1. Click the **order-service** service → **Variables** tab
2. Add a new variable:

| Name | Value |
|---|---|
| `USER_SERVICE_URL` | `http://user-service.railway.internal:8080` |

3. Click **"Deploy"** (or **"Apply changes"**) to redeploy with the variable

| Part of the URL | Why |
|---|---|
| `http://` | Private network traffic stays inside Railway and is already encrypted - no HTTPS needed |
| `user-service.railway.internal` | Railway's private DNS name: `<service-name>.railway.internal` |
| `:8080` | The port User Service listens on inside its container |

**Generate a public domain** for order-service (port `8080`), the same way as Step 1.

---

### Step 3: Test the Live System

Replace the two domains with your own:

```bash
# Valid user (created in Step 1)
curl -i -X POST https://order-service-production-xxxx.up.railway.app/api/orders \
  -H "Content-Type: application/json" \
  -d '{"userId": 1, "productName": "Laptop", "quantity": 1, "totalPrice": 1299.99}'

# User that doesn't exist
curl -i -X POST https://order-service-production-xxxx.up.railway.app/api/orders \
  -H "Content-Type: application/json" \
  -d '{"userId": 999, "productName": "Monitor", "quantity": 1, "totalPrice": 299.99}'
```

Expected: `201 Created`, then `400 Bad Request`. Your microservices are talking to each other **live on the internet** - over Railway's private network. ✅

Open the **Deployments → View logs** tab of each service to see the requests arrive.

> **Remember: H2 is in-memory.** Every redeploy or restart of user-service empties its database. If an order for user 1 suddenly returns `400`, create the user again.

---

### Alternative Platform: Render

[Render](https://render.com) can also deploy images from Docker Hub (**+ New → Web Service → Existing Image**). The steps are similar, with some differences on the free plan:

| | Railway | Render (free plan) |
|---|---|---|
| Deploys images from Docker Hub | ✅ | ✅ |
| Runs `docker-compose.yml` | ❌ - one service at a time | ❌ - one service at a time |
| Private network between services | ✅ `user-service.railway.internal` | ❌ free web services can't **receive** private traffic |
| `USER_SERVICE_URL` | `http://user-service.railway.internal:8080` | User Service's **public** URL, e.g. `https://user-service-xxxx.onrender.com` |
| Idle services | Keep running (uses trial credit) | Spin down after 15 min - the first request takes about a minute to wake up |

On Render's free plan, wake User Service up (open its URL) **before** creating an order, or the first call from Order Service may time out with `503`.

---

## Part 7: Understanding What You Built

### Architecture Recap

```
            Local                                         Railway project
┌─────────────────────────────────────┐     ┌──────────────────────────────────────────┐
│  Docker Compose network             │     │  Private network                         │
│                                     │     │                                          │
│  user-service ◄──── order-service   │     │  user-service ◄──────── order-service    │
│   :8080      http://user-service    │     │   :8080   http://user-service            │
│   [H2]        :8080   [H2]          │     │   [H2]    .railway.internal:8080  [H2]   │
└─────┬─────────────────┬─────────────┘     └─────┬──────────────────────┬─────────────┘
      │ localhost:8081  │ localhost:8082          │ https://...          │ https://...
      ▼                 ▼                         ▼                      ▼
                You (Postman / curl)                       You (Postman / curl)
```

### Microservices Concepts Applied

- ✅ **Independent deployment** - each service has its own Dockerfile, image and Railway service
- ✅ **Decentralized data** - each service owns its own database
- ✅ **Inter-service communication** - synchronous REST with `RestTemplate`
- ✅ **Configuration management** - the same image runs locally and on Railway; only `USER_SERVICE_URL` changes
- ✅ **Fault awareness** - you saw a synchronous call fail with `503` when its dependency was down

---

## Part 8: Advanced Concepts (Awareness Only)

### Service Discovery

You've already used a simple form of service discovery: **DNS**. Docker Compose resolves `user-service`, and Railway resolves `user-service.railway.internal`. Order Service only needs to know the **name**, not an IP address.

In larger systems with many instances that come and go, a **service registry** (e.g. Netflix Eureka, Consul) keeps track of every running instance. Services register themselves, and other services query the registry to find a healthy instance.

> ℹ️ Not required to implement - awareness only. Reference: https://spring.io/guides/gs/service-registration-and-discovery/

---

### API Gateway

**Current limitation:** clients must know a separate URL for every service (one Railway domain each).

An **API Gateway** gives clients a **single entry point** and routes each request to the right service - for example `/api/users/**` → User Service, `/api/orders/**` → Order Service. It's also a natural place for authentication, rate limiting and logging. With a gateway in front, User Service wouldn't need a public domain at all.

> ℹ️ Not required to implement - awareness only. Reference: https://spring.io/projects/spring-cloud-gateway

---

## Part 9: Clean Up

**Local:**

```bash
docker compose down
```

Stops and removes the containers and the network that `up` created. The images stay on your computer, so the next `up` is faster.

**Railway:** to save your trial credit, open the project → **Settings** → **Danger** → **Delete Project** once you've finished (and taken any screenshots for the lesson exercise).

---

## Troubleshooting

### Issue 1: Port already in use

**Error:** `Bind for 0.0.0.0:8081 failed: port is already allocated`

**Solution:** another app is using the host port. Stop it, or change the **host** side of the mapping, e.g. `"9081:8080"`, and use `localhost:9081`.

### Issue 2: Order returns 503 locally

**Cause:** Order Service can't reach User Service.

**Solution:**
1. `docker compose ps` - is `user-service` running and **healthy**?
2. Check `USER_SERVICE_URL` is `http://user-service:8080` - the **service name** and the **container** port, not `localhost:8081`

### Issue 3: Order returns 400 for a user that should exist

**Cause:** H2 is in-memory - the user disappeared when User Service restarted.

**Solution:** create the user again with `POST /api/users`, then retry the order.

### Issue 4: `exec format error` on Railway

**Cause:** the image was built for ARM (Apple Silicon) instead of `amd64`.

**Solution:** rebuild with `--platform linux/amd64`, push again, then click **"Redeploy"** on the Railway service.

### Issue 5: Order returns 503 on Railway

**Cause:** Order Service can't reach User Service over the private network.

**Solution:**
1. Check the User Service's **service name** is exactly `user-service` (Settings)
2. Check order-service's `USER_SERVICE_URL` is `http://user-service.railway.internal:8080` - `http`, not `https`, and port `8080`
3. Check both services are in the **same project** and **environment**
4. Check the variable change was **deployed** (Railway shows pending changes in a banner)

### Issue 6: Railway can't pull the image

**Cause:** the image name or tag is wrong, or the Docker Hub repository is private.

**Solution:** check the exact name on Docker Hub (`YOUR_DOCKERHUB_USERNAME/user-service:v4.16.0`) and make the repository **Public**.

---

## Summary

### What You Did

- **Containerized** two microservices with the same multi-stage Dockerfile
- **Ran** the whole system locally with one `docker-compose.yml` - service names, environment variables and a healthcheck
- **Tested** inter-service calls, including what happens when a dependency is down
- **Published** one image per service to Docker Hub
- **Deployed** both services to Railway and connected them over the private network

### Key Takeaways

- Each microservice is built, versioned and deployed **independently** - one Dockerfile, one image, one Railway service
- Inside a Compose network or a Railway project, services find each other by **name**, not IP address
- The **same image** runs everywhere - only environment variables like `USER_SERVICE_URL` change
- Docker Compose is a **local** tool - hosting platforms like Railway and Render recreate the setup service by service
- Service Discovery and API Gateway are the natural next steps for production systems

---

## Additional Resources

- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Docker Compose - Networking](https://docs.docker.com/compose/how-tos/networking/)
- [Railway - Private Networking](https://docs.railway.com/guides/private-networking)
- [Render - Deploy a Prebuilt Docker Image](https://render.com/docs/deploying-an-image)
- [Spring Cloud Documentation](https://spring.io/projects/spring-cloud)
- "Building Microservices" by Sam Newman

---

END
