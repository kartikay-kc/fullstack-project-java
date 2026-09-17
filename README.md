# Customer Management Platform

A full-stack customer management system with JWT-based authentication, role-aware access via Spring Security, S3-backed profile image storage, and a React frontend — containerized for local development and deployable to AWS Elastic Beanstalk.

**Stack:** Spring Boot 3 (Java 17) · React (Vite) · PostgreSQL · Docker · AWS

---

## Overview

The backend exposes a REST API for managing customer records, backed by PostgreSQL with Flyway-versioned migrations. Authentication is stateless JWT, issued on registration and login and validated on every subsequent request. The frontend is a React SPA that consumes the API for auth flows and a live customer dashboard, including protected routing so unauthenticated users can't reach the dashboard.

---

## Key Features

- **JWT authentication, no server-side session** — `Customer` implements Spring Security's `UserDetails` directly (email as username, `ROLE_USER` authority), so the same entity is both the domain model and the security principal. A token is issued on registration and returned via the `Authorization` header on login.
- **Profile image storage via AWS S3** — customers can upload a profile image (`multipart/form-data`) and retrieve it back as raw JPEG bytes; the API stores only a `profileImageId` reference, not the binary, in Postgres. The S3 wrapper is a thin, testable layer over the AWS SDK v2 `putObject` / `getObject` calls.
- **Schema versioned with Flyway** — the database schema is defined as migrations rather than `hibernate.ddl-auto=update`, so the schema history is explicit and reproducible across environments.
- **Integration-tested against a real database** — tests run with Testcontainers, spinning up an actual PostgreSQL container rather than mocking the repository layer, so query behavior (unique constraints, sequence generation) is verified for real.
- **Multi-service local dev environment** — `docker-compose.yml` wires up Postgres, the Spring Boot API, and the React client as three services with proper startup ordering (`depends_on`), so the whole stack comes up with one command.
- **Cloud-ready image build** — the API is packaged with Jib (multi-platform: ARM64/AMD64) rather than a hand-written Dockerfile, and ships with `Dockerrun.aws.json` for AWS Elastic Beanstalk deployment.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | Java 17, Spring Boot 3, Spring Web, Spring Data JPA |
| Auth | Spring Security 6, JJWT (`jjwt-api`/`impl`/`jackson`) |
| Database | PostgreSQL, Flyway migrations |
| File Storage | AWS S3 (AWS SDK v2) |
| Frontend | React, Vite, Chakra UI, Axios |
| Testing | JUnit, Mockito, Testcontainers, Spring Security Test |
| Infra | Docker, Docker Compose, Jib, AWS Elastic Beanstalk |

---

## API Endpoints

### Auth — `/api/v1/auth`
| Method | Endpoint | Description |
|---|---|---|
| POST | `/login` | Authenticate; returns `AuthenticationResponse` and a JWT in the `Authorization` header |

### Customers — `/api/v1/customers`
| Method | Endpoint | Description |
|---|---|---|
| GET | `/` | List all customers |
| GET | `/{customerId}` | Get a single customer |
| POST | `/` | Register a new customer; returns a JWT in the `Authorization` header |
| PUT | `/{customerId}` | Update a customer record |
| DELETE | `/{customerId}` | Delete a customer |
| POST | `/{customerId}/profile-image` | Upload a profile image (multipart) |
| GET | `/{customerId}/profile-image` | Retrieve the profile image as JPEG bytes |

---

## Data Model

**Customer**
| Field | Type | Notes |
|---|---|---|
| `id` | Integer | Primary key, sequence-generated |
| `name` | String | Required |
| `email` | String | Required, unique |
| `age` | Integer | Required |
| `gender` | Enum (`MALE`/`FEMALE`) | Required |
| `password` | String | Required, encoded |
| `profileImageId` | String | Unique, nullable — references the object stored in S3 |

`Customer` doubles as the Spring Security principal by implementing `UserDetails`, so no separate `User`/`Customer` split is needed for authentication.

---

## Architecture

```
┌─────────────┐      REST + JWT       ┌──────────────────┐
│ React (Vite)│  ───────────────────► │  Spring Boot API  │
│ Chakra UI   │ ◄─────────────────── │  (Spring Security) │
└─────────────┘                       └─────────┬─────────┘
                                                  │
                             ┌────────────────────┼───────────────────┐
                             ▼                                        ▼
                     ┌───────────────┐                        ┌─────────────┐
                     │  PostgreSQL   │                        │   AWS S3    │
                     │ (Flyway-      │                        │ (profile    │
                     │  versioned)   │                        │  images)    │
                     └───────────────┘                        └─────────────┘
```

---

## Getting Started

### Run everything with Docker Compose

```bash
git clone https://github.com/kartikay-kc/fullstack-project-java.git
cd fullstack-project-java
docker-compose up
```

This starts:
- PostgreSQL on `localhost:5332`
- API on `localhost:8088`
- React client on `localhost:3000`

### Run the backend locally

```bash
cd backend
./mvnw spring-boot:run
```

Requires a running PostgreSQL instance and the relevant environment variables (DB connection, JWT secret, AWS credentials for S3) configured for your environment.

### Run the frontend locally

```bash
cd frontend/react
npm install
npm run dev
```

---

## Testing

```bash
cd backend
./mvnw test
```

Integration tests spin up a real PostgreSQL instance via Testcontainers, exercising the repository and controller layers against an actual database rather than mocks.

---

## Possible Next Steps

- Add refresh-token support so JWTs don't require re-login on expiry
- Add role-based endpoints (e.g. admin vs. standard user) beyond the current single `ROLE_USER`
- Add pagination/search on the customer list endpoint
- Add a CI workflow to run the Testcontainers suite on every push

---

## Author

**Kartikay Teotia**
[GitHub](https://github.com/kartikay-kc) · [LinkedIn](https://www.linkedin.com/in/kartikay-teotia-11a4b7216)
