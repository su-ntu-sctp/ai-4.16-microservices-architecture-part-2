# [4.16] Microservices Architecture - Part 2 (Practical Demo)

## Lesson Overview

![Microservices Architecture Part 2](./assets/images/infographic-4.16-microservices-practical.png)

## Dependencies

- [Lesson](./lesson.md) / [Assignment](./assignment.md) / [Slide Deck](./slides.md)

> ℹ️ **No Self Studies for this lesson.** Lesson 4.16 is a hands-on implementation session. Students should ensure Docker Desktop, Java 21, Maven, and Postman are installed and working before class.

## Lesson Objectives

By the end of this lesson, students will be able to:

* **Build** two independent Spring Boot microservices from scratch
* **Implement** REST-based inter-service communication using RestTemplate
* **Deploy** and test the complete microservices system using Docker Compose
* **Describe** Service Discovery and API Gateway as next steps beyond this foundation

## Lesson Plan

| Duration | What | How or Why |
|---|---|---|
| 10 min | Warm-up | Recap microservices theory from 4.15 — review the User/Order service diagram and remind students what they're about to build |
| 15 min | Part 1: Project Setup | Create directory structure; generate both Spring Boot projects via start.spring.io; verify folder structure |
| 40 min | Part 2: Build User Service | Code-along — entity, repository, service, controller, `application.properties`, Dockerfile; test locally in Postman before moving on |
| 45 min | Part 3: Build Order Service | Code-along — entity, UserDTO, repository, service (with RestTemplate call), `AppConfig`, controller, `application.properties`, Dockerfile |
| 10 min | Break | — |
| 20 min | Part 4: Docker Compose Configuration | Create `docker-compose.yml`; walk through each section — services, databases, network, volumes, environment variables |
| 25 min | Part 5: Build and Deploy | Build both JARs; run `docker compose up`; verify all containers are running; check logs |
| 30 min | Part 6: Testing | Test all endpoints in Postman — create users, create valid orders, test invalid user error; verify inter-service communication is working |
| 15 min | Part 7 & 8: Recap + Advanced Concepts | Walk through architecture diagram; review what was demonstrated; cover Service Discovery and API Gateway as awareness concepts |
| 10 min | Part 9: Clean Up + Wrap-up | Run `docker compose down`; summarise key takeaways; highlight what comes next |
| **220 min** | **Total** | |