# Assignment (Optional)

## Brief

Build a working microservices system with two independent services that communicate via REST APIs and deploy using Docker Compose.

1. **Build User Service and Order Service Microservices**
   - Create two separate Spring Boot projects:
     - **User Service** (port 8081)
     - **Order Service** (port 8082)
   - For User Service, create:
     - User entity with id, firstName, lastName, email fields
     - UserRepository extending JpaRepository
     - UserService with methods: getAllUsers(), getUserById(), createUser()
     - UserController with REST endpoints:
       - GET /api/users (get all users)
       - GET /api/users/{id} (get user by ID)
       - POST /api/users (create new user)
   - For Order Service, create:
     - Order entity with id, userId, productName, quantity, totalPrice, status, orderDate fields
     - UserDTO class to receive user data from User Service
     - OrderRepository extending JpaRepository
     - OrderService that uses RestTemplate to call User Service for validation
     - OrderController with REST endpoints:
       - GET /api/orders (get all orders)
       - GET /api/orders/{id} (get order by ID)
       - GET /api/orders/user/{userId} (get orders by user)
       - POST /api/orders (create order - validates user exists first)
     - AppConfig class with RestTemplate bean
   - Configure application.properties for both services with:
     - Server port (8081 for User, 8082 for Order)
     - Database connection (PostgreSQL)
     - For Order Service: add user.service.url property
   - Create Dockerfile for both services
   - Build JAR files: `mvn clean package -DskipTests`

2. **Configure and Deploy with Docker Compose**
   - Create a docker-compose.yml file that defines:
     - user-service (maps to port 8081)
     - order-service (maps to port 8082)
     - user-db (PostgreSQL on port 5433)
     - order-db (PostgreSQL on port 5434)
   - Configure environment variables for each service:
     - Database URLs, usernames, passwords
     - For order-service: USER_SERVICE_URL=http://user-service:8080
   - Set up proper service dependencies (depends_on)
   - Create named volumes for database persistence
   - Create a custom network for service communication
   - Start all services: `docker compose up -d --build`
   - Verify all containers are running: `docker compose ps`
   - Check logs to ensure services started successfully

3. **Test Inter-Service Communication**
   - Create at least 3 users via User Service API:
```bash
     curl -X POST http://localhost:8081/api/users \
       -H "Content-Type: application/json" \
       -d '{"firstName":"John","lastName":"Doe","email":"john@example.com"}'
```
   - Verify users were created: `curl http://localhost:8081/api/users`
   - Create orders for existing users via Order Service:
```bash
     curl -X POST http://localhost:8082/api/orders \
       -H "Content-Type: application/json" \
       -d '{"userId":1,"productName":"Laptop","quantity":1,"totalPrice":1299.99}'
```
   - Verify Order Service successfully called User Service by checking:
     - Orders were created successfully
     - Service logs show the HTTP call
   - Test error handling by creating an order for non-existent user (userId: 999)
   - Verify order creation fails with appropriate error
   - Get all orders: `curl http://localhost:8082/api/orders`
   - Get orders for specific user: `curl http://localhost:8082/api/orders/user/1`
   - Take screenshots showing:
     - All containers running
     - Successful user creation
     - Successful order creation
     - Failed order creation for invalid user
   - Write a brief explanation (5-6 sentences) describing:
     - How the two services communicate
     - Why each service has its own database
     - What happens when Order Service creates an order
     - The benefits of this microservices architecture
     - One challenge you encountered and how you solved it

## Submission (Optional)

- Submit the URL of the GitHub Repository that contains your work to NTU black board.
- Should you reference the work of your classmate(s) or online resources, give them credit by adding either the name of your classmate or URL.

## References
- Java: https://docs.oracle.com/javase/
- Spring Boot: https://docs.spring.io/spring-boot/docs/current/reference/html/
- PostgreSQL: https://www.postgresql.org/docs/
- OWASP: https://cheatsheetseries.owasp.org/