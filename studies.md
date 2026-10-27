# Self Studies: Microservices Architecture - Part 2

## Overview

Lesson 4.16 is a hands-on session. You'll containerize two microservices, run them together with Docker Compose, push the images to Docker Hub and deploy them to Railway. The Java code is provided - your preparation is about **tools and accounts**, so you can spend class time deploying instead of installing.

**Estimated Prep Time:** 30-45 minutes

---

## Task 1: Check Your Tools and Accounts

Make sure each of these works **before class**:

| Check | How | Expected |
|---|---|---|
| Docker Desktop is running | `docker version` | Shows both **Client** and **Server** |
| Docker Compose is available | `docker compose version` | Shows a version number |
| You can log in to Docker Hub | `docker login` | `Login Succeeded` |
| Railway account works | Log in at [railway.app](https://railway.app) | You see your dashboard (you created it in Lesson 4.10) |
| Postman or curl | Open Postman, or run `curl --version` | Either one works |

> You do **not** need Java or Maven on your computer for this lesson - the multi-stage Dockerfiles build the JARs inside Docker.

---

## Task 2: Refresh the Building Blocks

Today combines three earlier lessons. Skim the parts below so the commands feel familiar.

**Read:**
- Lesson 4.4 - Part 4: multi-stage builds
- Lesson 4.6 - your `docker-compose.yml` (services, `ports`, `environment`, `depends_on`)
- Lesson 4.10 - Part 3: build with `--platform linux/amd64`, push to Docker Hub, deploy to Railway from a Docker image
- Lesson 4.15 - Parts 5 and 6: synchronous communication and environment-variable configuration

**Guiding Questions:**
- Why does a multi-stage build mean you don't need Maven installed locally?
- In a Docker Compose file, what is the difference between the host port and the container port in `"8081:8080"`?
- How does one container reach another container in the same Compose project?
- Why must images for Railway be built for `linux/amd64`?

---

## Task 3: Learn About Railway Private Networking

In class, two services on Railway will talk to each other without going over the public internet.

**Read:** [Railway - Private Networking](https://docs.railway.com/guides/private-networking)

**Guiding Questions:**
- What hostname does a Railway service get on the private network?
- Why should internal calls use `http://` rather than `https://`?
- Which service in today's system needs to be reachable by the other?

---

## Additional Reading Material

- [Docker Compose - Networking](https://docs.docker.com/compose/how-tos/networking/)
- [Docker Compose - Control startup order](https://docs.docker.com/compose/how-tos/startup-order/)
- [Render - Deploy a Prebuilt Docker Image](https://render.com/docs/deploying-an-image) (alternative platform)
