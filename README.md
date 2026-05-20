# LibraryAppDeploy

                    ┌──────────────┐
                    │    Nginx     │
                    │ reverse proxy│
                    └──────┬───────┘
                           │
            ┌──────────────┴──────────────┐
            │                             │
      ┌──────────────┐             ┌──────────────┐
      │  Frontend    │             │   Backend    │
      │  (UI)        │             │   API        │
      └──────────────┘             └──────┬───────┘
                                          │  
                        ┌─────────────────┴─────────────────┐
                        │                                   │
                  ┌──────────────┐                  ┌──────────────┐
                  │ PostgreSQL   │                  │    Redis     │
                  │ (database)   │                  │   (cache)    │
                  └──────────────┘                  └──────────────┘

## Overview:
This project is a full-stack library management application deployed using Docker Compose. It consists of a React frontend, a Spring Boot backend, a PostgreSQL database, and a Redis cache. Nginx is used as a reverse proxy to route requests to the appropriate services. It is a library management system that allows users to manage books, authors, and genres. The application is designed to be scalable and easily deployable using Docker.
## Components:
- PostgreSQL 16-alpine
- Redis 7-alpine
- Nginx alpine (latest)
- Backend ghcr.io/nataliacharz/library-backend:latest
- Frontend ghcr.io/nataliacharz/library-frontend:latest

## Backend:
- Java 21
- Spring Boot 3.2.0
- Spring Data JPA
- Spring Data Redis
- Spring Web
- Spring Actuator
- PostgreSQL Driver
- Spring Security
- JWT
- Lombok

## Frontend:
- React 18.2.0
- Next.js 16.0.7

## Docker Compose:
- Single file `docker-compose.yml` to orchestrate all services
- Services: nginx, frontend, backend, postgres, redis

## Docerfiles:
- Frontend and backend have their own Dockerfiles for building images
- Nginx uses a custom configuration file `nginx.conf` for reverse proxy setup
- PostgreSQL and Redis use official alpine images with environment variables for configuration


## Running the application:
```
- docker-compose up -d
```

## Checking on redis:
```
- docker exec -it redis redis-cli
- keys *
```

## Checking on postgres:
```
- docker exec -it postgres pg_isready -U admin -d library
```

## Checking on backend:
```
- wget -q0- http://localhost:8080/actuator/health
```

## Checking on backend logs:
```
- docker-compose logs -f backend
```