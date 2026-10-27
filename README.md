# [4.16] Microservices Architecture - Part 2 (Practical)

## Lesson Overview

![Microservices Architecture Part 2](./assets/images/infographic-4.16-microservices-practical.png)

## Dependencies

- [Self Studies](./studies.md) / [Lesson](./lesson.md) / [Assignment](./assignment.md) / [Slide Deck](./slides.md)

> ℹ️ **Hands-on DevOps session.** The two Spring Boot services are provided in the [`microservices-demo`](https://github.com/aie-devops/microservices-demo) starter project (tag `v4.16.0`). Students write the Dockerfiles and Docker Compose file, then publish and deploy the images. Students need Docker Desktop, a Docker Hub account and a Railway account before class - see [studies.md](./studies.md). Java and Maven are **not** needed locally.

## Lesson Objectives

By the end of this lesson, students will be able to:

* **Containerize** two Spring Boot microservices with multi-stage Dockerfiles
* **Run and test** a multi-service system locally with Docker Compose, using service names, environment variables and healthchecks
* **Publish** each service's image to Docker Hub
* **Deploy** the services to Railway from Docker Hub and connect them over Railway's private network
* **Describe** Service Discovery and API Gateway as next steps beyond this foundation

## Lesson Plan

| Duration | What | How or Why |
|---|---|---|
| 10 min | Warm-up | Recap 4.15 - the User/Order service diagram, synchronous REST calls and environment-based configuration. Today students are the DevOps engineers shipping code the developers wrote |
| 15 min | Part 1: Get the Project | Download the `microservices-demo` `v4.16.0` zip; tour the two services, `application.properties` (`SERVER_PORT`, `USER_SERVICE_URL`) and the REST call in `OrderService`; why H2 (in-memory, one per service) |
| 20 min | Part 2: Write the Dockerfiles | Same multi-stage Dockerfile and `.dockerignore` for both services (reuses Lesson 4.4); both listen on 8080 inside their containers |
| 25 min | Part 3: Docker Compose | Write `docker-compose.yml`: two services, host ports 8081/8082, `USER_SERVICE_URL=http://user-service:8080`, healthcheck + `depends_on: condition: service_healthy`; `docker compose up --detach --build`, `ps`, `logs` |
| 25 min | Part 4: Test Locally + Activity | Create users and orders (201 / 400) in Postman or curl; Activity - stop user-service, observe 503, restart, discuss sync vs async |
| 10 min | Break | - |
| 20 min | Part 5: Publish to Docker Hub | `docker build --platform linux/amd64` + `docker push` for `user-service:v4.16.0` and `order-service:v4.16.0`; instructor images as fallback |
| 35 min | Part 6: Deploy to Railway | One project, two services from Docker Hub images; service names; `USER_SERVICE_URL=http://user-service.railway.internal:8080`; generate domains; test live; Compose vs Railway mapping; Render as an alternative (no private network on free plan, spin-down) |
| 10 min | Part 7 & 8: Recap + Advanced Concepts | Architecture recap local vs Railway; DNS as simple service discovery; service registries and API Gateway (awareness) |
| 10 min | Part 9: Clean Up + Wrap-up | `docker compose down`; delete the Railway project to save credit; key takeaways |
| **180 min** | **Total** | |
