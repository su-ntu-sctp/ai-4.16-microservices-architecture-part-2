# Assessment / Quiz

## Overview
- **Lesson:** Microservices Architecture - Part 2 (Practical)
- **Format:** 10 questions (mix MCQ / True–False)
- **Time:** ~10–15 minutes
- **Scoring:** 1 point each (unless stated)

## Questions

### Q1 (True/False)

Q1: In the lesson, each microservice is built and published as its own Docker image?

A - True

B - False

---

Q2: Why don't you need Maven installed on your computer to build the images in this lesson?

A - Docker Hub builds the JAR for you

B - The multi-stage Dockerfile runs Maven inside the build stage

C - Railway compiles the Java code

D - Spring Boot does not need to be compiled

---

Q3: Inside the Docker Compose network, Order Service reaches User Service at `http://user-service:8080` - using the service name instead of an IP address?

A - True

B - False

---

Q4: In `ports: "8081:8080"`, which number is the port on your computer (the host)?

A - 8080

B - 8081

C - Both

D - Neither - Docker picks a random port

---

Q5: What does `depends_on: user-service: condition: service_healthy` do?

A - Restarts order-service whenever user-service crashes

B - Starts order-service only after user-service passes its healthcheck

C - Copies the user-service database into order-service

D - Builds both images at the same time

---

Q6: You stop only user-service and try to create an order. What does Order Service return?

A - 201 Created

B - 400 Bad Request

C - 503 Service Unavailable

D - 200 OK

---

Q7: Why are the images built with `--platform linux/amd64` before pushing them for Railway?

A - It makes the image smaller

B - Railway runs on amd64 servers, and an ARM image built on Apple Silicon would fail with `exec format error`

C - Docker Hub only accepts amd64 images

D - It is required for multi-stage builds

---

Q8: Railway runs your `docker-compose.yml` automatically when you deploy from Docker Hub?

A - True

B - False

---

Q9: On Railway, which value of `USER_SERVICE_URL` lets Order Service call User Service over the private network?

A - `http://localhost:8081`

B - `http://user-service:8080`

C - `http://user-service.railway.internal:8080`

D - `https://hub.docker.com/user-service`

---

Q10: An API Gateway provides a single entry point for clients and routes each request to the right microservice?

A - True

B - False

## Notes
- Keep each question atomic and aligned to the lesson objectives.
- Put the answer immediately after each question.
