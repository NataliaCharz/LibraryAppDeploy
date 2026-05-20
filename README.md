# LibraryAppDeploy

## Components:
- PostgreSQL 16-alpine
- Redis 7-alpine
- Nginx alpine (latest)
- Backend ghcr.io/nataliacharz/library-backend:latest
- Frontend ghcr.io/nataliacharz/library-frontend:latest

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


## Checking on backend health:
```
- wget -q0- http://localhost:8080/actuator/health

```
## Checking on backend logs:
```
- docker-compose logs -f backend
```