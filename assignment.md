# Assignment (Optional)

## Brief

Package, run and deploy a two-service microservices system - the `microservices-demo` User Service and Order Service - using Docker, Docker Compose, Docker Hub and Railway. The Java code is provided; this exercise is about the DevOps work around it.

1. **Containerize and Run Locally**
   - Download the starter project (tag `v4.16.0`):
```bash
     curl -L -O https://github.com/aie-devops/microservices-demo/archive/refs/tags/v4.16.0.zip
     unzip v4.16.0.zip
```
   - Create your own GitHub repository and add the project to it
   - Add a multi-stage `Dockerfile` and a `.dockerignore` to both `user-service/` and `order-service/`
   - Add a `docker-compose.yml` in the project root that:
     - Builds both services from their folders
     - Maps user-service to host port 8081 and order-service to host port 8082
     - Sets `USER_SERVICE_URL` for order-service so it can reach user-service by its service name
     - Adds a healthcheck to user-service (`/actuator/health`) and starts order-service only when user-service is healthy
   - Start everything with `docker compose up --detach --build` and confirm both containers are running with `docker compose ps`
   - Create at least 3 users and at least 3 orders for existing users
   - Create an order for a user that doesn't exist (userId `999`) and confirm it fails with `400 Bad Request`
   - Stop only user-service, try to create an order, and record the status code and message you get

2. **Publish and Deploy to Railway**
   - Build both images for `linux/amd64` and push them to Docker Hub as `YOUR_DOCKERHUB_USERNAME/user-service:v4.16.0` and `YOUR_DOCKERHUB_USERNAME/order-service:v4.16.0`
   - In **one** Railway project, deploy both images as two services named `user-service` and `order-service`
   - Set `USER_SERVICE_URL` on order-service so it calls user-service over Railway's **private network**
   - Generate public domains and test the live system: create users, create a valid order (`201`) and an invalid order (`400`)
   - **Challenge:** remove user-service's public domain. Does creating an order still work? Why? What does this tell you about which services need to be public?

3. **Document Your Work**
   - Write a `README.md` in your repository that includes:
     - Your live Railway URLs for both services
     - Screenshots showing:
       - `docker compose ps` with both containers running (user-service healthy)
       - A successful order (`201`) and a failed order for user `999` (`400`)
       - The `503` response when user-service was stopped
       - Your Railway project with both services deployed
     - A table comparing how you configured the system in Docker Compose vs Railway (images, ports, environment variables, how order-service finds user-service)
   - Write a brief explanation (5-6 sentences) describing:
     - How the two services communicate, locally and on Railway
     - Why each service has its own database
     - What happened when user-service was stopped, and how an asynchronous design would change that
     - One challenge you encountered and how you solved it

## Stretch Goal (Optional)

Automate the publishing step: add a CI pipeline (GitHub Actions or CircleCI - refer to Lessons 4.7 and 4.9) that builds **both** images and pushes them to Docker Hub on every push to `main`.

## Submission (Optional)

- Submit the URL of the GitHub Repository that contains your work to NTU black board.
- Should you reference the work of your classmate(s) or online resources, give them credit by adding either the name of your classmate or URL.
- Delete your Railway project afterwards to save your trial credit.

## References
- Docker Compose: https://docs.docker.com/compose/
- Docker Hub: https://docs.docker.com/docker-hub/
- Railway Private Networking: https://docs.railway.com/guides/private-networking
- Spring Boot: https://docs.spring.io/spring-boot/index.html
