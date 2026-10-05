# TaskFlow API

A production-style task management REST API built with **Java 21 + Spring Boot 3**, backed by **PostgreSQL**, secured with **Spring Security + JWT**, tested with **JUnit 5 + Mockito**, and shipped with **Docker** and **GitHub Actions CI**.

## Features

- JWT-based registration/login with BCrypt password hashing (Spring Security, stateless sessions)
- Full CRUD for tasks with per-user data isolation (users only see their own tasks)
- Status workflow enforced as a small state machine (`TODO -> IN_PROGRESS -> DONE`) with `409 Conflict` on invalid transitions
- Pagination, sorting and filtering (`GET /api/v1/tasks?status=TODO&page=0&size=20&sort=dueDate,asc`)
- Aggregated summary endpoint using a JPQL `GROUP BY` projection
- Request validation (Jakarta Bean Validation) with structured field-level error responses
- Centralised exception handling via `@RestControllerAdvice` and a consistent `ApiError` envelope
- Optimistic locking (`@Version`) on tasks, DB indexes on hot columns
- SLF4J logging, Spring Boot Actuator health endpoint
- 13 automated tests: Mockito unit tests for the service layer, `@SpringBootTest` + `MockMvc` integration tests against an in-memory H2 (PostgreSQL mode)
- Multi-stage Dockerfile (non-root runtime user, healthcheck), `docker-compose` with Postgres, CI pipeline that builds, tests and builds the image

## Tech stack

| Layer | Tech |
|---|---|
| Language | Java 21 (records, streams, lambdas, switch expressions) |
| Framework | Spring Boot 3.3, Spring Web, Spring Data JPA, Spring Security, Validation, Actuator |
| Database | PostgreSQL 16 (H2 for tests) |
| Auth | JWT (jjwt 0.12), BCrypt |
| Testing | JUnit 5, Mockito, AssertJ, MockMvc |
| Build / Ops | Maven, Docker, docker-compose, GitHub Actions |

## API

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/api/v1/auth/register` | – | Create account, returns JWT |
| POST | `/api/v1/auth/login` | – | Login, returns JWT |
| POST | `/api/v1/tasks` | Bearer | Create task |
| GET | `/api/v1/tasks` | Bearer | List tasks (paginated, `?status=`) |
| GET | `/api/v1/tasks/{id}` | Bearer | Get one task |
| PUT | `/api/v1/tasks/{id}` | Bearer | Replace task |
| PATCH | `/api/v1/tasks/{id}/status` | Bearer | Change status (validated transition) |
| DELETE | `/api/v1/tasks/{id}` | Bearer | Delete task |
| GET | `/api/v1/tasks/summary` | Bearer | Counts by status |
| GET | `/actuator/health` | – | Health check |

Error responses always look like:

```json
{
  "timestamp": "2026-10-05T10:15:30Z",
  "status": 400,
  "error": "Bad Request",
  "message": "Validation failed",
  "path": "/api/v1/tasks",
  "fieldErrors": { "title": "must not be blank" }
}
```

## Run it

**Option A – Docker (recommended)**

```bash
docker compose up --build
```

**Option B – local Postgres**

```bash
docker run -d --name pg -e POSTGRES_DB=taskflow -e POSTGRES_USER=taskflow -e POSTGRES_PASSWORD=taskflow -p 5432:5432 postgres:16-alpine
mvn spring-boot:run
```

Then open `requests.http` (VS Code REST Client / IntelliJ) or use curl:

```bash
curl -X POST localhost:8080/api/v1/auth/register -H 'Content-Type: application/json' \
  -d '{"name":"Tej","email":"tej@example.com","password":"Password123"}'
```

## Run tests

```bash
mvn test
```

## Project structure

```
src/main/java/com/tejaswini/taskflow
├── config/        SecurityConfig (filter chain, password encoder)
├── controller/    REST endpoints (thin, delegate to services)
├── dto/           Request/response records with validation annotations
├── entity/        JPA entities (User, Task)
├── exception/     Custom exceptions + GlobalExceptionHandler + ApiError
├── repository/    Spring Data JPA repositories (derived queries + JPQL projection)
├── security/      JwtService, JwtAuthFilter, AuthenticatedUser principal
└── service/       Business logic, transactions, state machine
```

## Design notes

- **Why records for DTOs?** Immutable, concise, and keep the API contract separate from JPA entities.
- **Why a state machine for status?** Prevents nonsensical transitions and gives a clear place to grow (e.g. audit events on `DONE`).
- **Why `findByIdAndOwnerId`?** Enforces ownership at the query level so a user can never read another user's task, even by guessing IDs.
- **Why H2 in PostgreSQL mode for tests?** Fast, zero-setup integration tests in CI; the JPQL used is portable.

## Roadmap

- [ ] Publish task events to Kafka on status change
- [ ] Redis cache for the summary endpoint
- [ ] Flyway migrations instead of `ddl-auto`
- [ ] OpenAPI / Swagger UI
