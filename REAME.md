# OT-MICROSERVICES — Complete Application Report
 

---

## Table of Contents

1. [What is OT-MICROSERVICES?](#what-is-ot-microservices)
2. [System Architecture](#system-architecture)
3. [Services Overview](#services-overview)
4. [Tech Stack Summary](#tech-stack-summary)
5. [Databases & Infrastructure](#databases--infrastructure)
6. [API Endpoints Reference](#api-endpoints-reference)
7. [Local Setup on Ubuntu/Debian](#local-setup-on-ubuntudebian)
8. [Build & Run Commands](#build--run-commands)
9. [Testing Strategy](#testing-strategy)
10. [Docker & Container Info](#docker--container-info)

---

## What is OT-MICROSERVICES?

OT-MICROSERVICES is a **polyglot microservices-based Employee Management System** built by Opstree for hands-on learning of DevOps and Platform Engineering concepts. It is intentionally written in **multiple programming languages** — Go, Python, Java, and ReactJS — to simulate a real-world microservices environment where different teams own different services.

The system manages:
- Employee records and personal information
- Attendance tracking
- Salary processing
- Email notifications to employees
- A unified React frontend for end users

---

## System Architecture

```
                        ┌─────────────────────┐
                        │      Frontend        │
                        │  ReactJS · npm       │
                        │  Port: 3000          │
                        └──────┬───┬───┬───────┘
                               │   │   │
              ┌────────────────┘   │   └────────────────┐
              ▼                    ▼                     ▼
   ┌─────────────────┐  ┌──────────────────┐  ┌──────────────────┐
   │  employee-api   │  │  attendance-api  │  │   salary-api     │
   │  Go · Gin       │  │  Python · Flask  │  │  Java · Spring   │
   │  Port: 8080     │  │  Port: 8080      │  │  Boot · Port:8080│
   └────────┬────────┘  └────────┬─────────┘  └────────┬─────────┘
            │                    │                      │
     ┌──────┴──────┐      ┌──────┴──────┐       ┌──────┴──────┐
     │  ScyllaDB   │      │ PostgreSQL  │        │  ScyllaDB   │
     │  + Redis    │      │  + Redis    │        │  + Redis    │
     └─────────────┘      └──────┬──────┘        └─────────────┘
                                 │
                    ┌────────────▼──────────────┐
                    │    notification-worker     │
                    │    Python · Scheduled      │
                    │    Email notifications     │
                    └───────────────────────────┘
```

---

## Services Overview

### 1. employee-api
| Property | Details |
|----------|---------|
| Language | Go |
| Framework | Gin (HTTP router) |
| Primary DB | ScyllaDB (NoSQL) |
| Cache | Redis |
| Docs | Swagger at `/swagger/index.html` |
| Metrics | Prometheus at `/metrics` |
| Build tool | Makefile + go build |
| Container image | quay.io/opstree/employee-api |

**Folder structure:**
```
employee-api/
├── api/           # API handler functions
├── client/        # DB & Redis client setup
├── config/        # App configuration loader
├── docs/          # Auto-generated Swagger docs
├── middleware/     # HTTP middleware (logging, auth)
├── migration/     # ScyllaDB CQL migration files
├── model/         # Data models/structs
├── routes/        # Route registration
├── static/        # Static assets (logo, arch diagram)
├── main.go        # Entry point
├── config.yaml    # App configuration
├── migration.json # ScyllaDB connection for migrate tool
├── Makefile
├── Dockerfile
├── go.mod
└── go.sum
```

---

### 2. attendance-api
| Property | Details |
|----------|---------|
| Language | Python |
| Framework | Flask + Gunicorn |
| Primary DB | PostgreSQL |
| Cache | Redis |
| Migrations | Liquibase (`liquibase.properties`) |
| Package manager | Poetry (`pyproject.toml`) |
| Docs | Swagger at `/apidocs` |
| Metrics | Prometheus at `/metrics` |
| Container image | quay.io/opstree/attendance-api |

**Folder structure:**
```
attendance-api/
├── client/        # Redis client
├── migration/     # Liquibase SQL migration files
├── models/        # SQLAlchemy ORM models
├── router/        # Flask route definitions
├── scripts/       # Helper shell scripts
├── static/        # Logo and architecture images
├── utils/         # Utility functions
├── app.py         # Flask app entry point
├── config.yaml    # App configuration
├── liquibase.properties  # DB connection for migrations
├── log.conf       # Gunicorn logging config
├── pyproject.toml # Poetry dependencies
├── poetry.lock
├── pytest.ini
├── Makefile
└── Dockerfile
```

---

### 3. salary-api
| Property | Details |
|----------|---------|
| Language | Java |
| Framework | Spring Boot (Tomcat embedded) |
| Primary DB | ScyllaDB (via Cassandra driver) |
| Cache | Redis |
| Migrations | golang-migrate tool (`migration.json`) |
| Build tool | Maven (`pom.xml`) |
| Docs | Swagger at `/salary-documentation` |
| Metrics | Spring Actuator at `/actuator/prometheus` |
| Container image | quay.io/opstree/salary-api:v0.1.0 |

**Folder structure:**
```
salary-api/
├── src/
│   ├── main/
│   │   ├── java/com/opstree/microservice/salary/
│   │   │   ├── controller/   # @RestController HTTP handlers
│   │   │   ├── service/      # @Service business logic
│   │   │   ├── repository/   # @Repository ScyllaDB queries
│   │   │   └── model/        # @Table entity classes
│   │   └── resources/
│   │       └── application.yml  # Spring Boot config
│   └── test/java/com/opstree/microservice/salary/
├── migration/     # .cql files for ScyllaDB schema
├── static/        # Logo and architecture images
├── .mvn/wrapper/  # Maven wrapper JARs
├── pom.xml        # Maven dependencies
├── migration.json # ScyllaDB DSN for golang-migrate
├── Makefile
├── Dockerfile     # Multi-stage: maven builder + alpine runtime
├── mvnw / mvnw.cmd
└── LICENSE
```

**Dockerfile (multi-stage):**
```dockerfile
# Stage 1: Build
FROM maven:3.6.3-openjdk-17 as builder
WORKDIR /workspace
COPY pom.xml .
COPY src/ src/
RUN mvn clean package -DskipTests

# Stage 2: Runtime
FROM alpine:3.18.0
RUN apk update && apk add openjdk17
COPY --from=builder /workspace/target/salary-0.1.0-RELEASE.jar /app/salary.jar
EXPOSE 8080
ENTRYPOINT ["/usr/bin/java", "-jar", "/app/salary.jar"]
```

---

### 4. frontend
| Property | Details |
|----------|---------|
| Language | JavaScript |
| Framework | ReactJS |
| Dependencies | employee-api, attendance-api, salary-api |
| Package manager | npm |
| Build tool | Makefile |
| Container image | quay.io/opstree/frontend |

**Folder structure:**
```
frontend/
├── public/        # Static HTML shell
├── src/           # React components & pages
├── static/        # Logo and architecture images
├── package.json   # npm dependencies
├── Makefile
└── Dockerfile
```

---

### 5. notification-worker
| Property | Details |
|----------|---------|
| Language | Python |
| Type | Scheduled background worker |
| Function | Sends email notifications to employees |
| Trigger | Attendance/salary events |

---

## Tech Stack Summary

| Layer | Technology | Used By |
|-------|------------|---------|
| Web Framework | Gin (Go) | employee-api |
| Web Framework | Flask + Gunicorn | attendance-api |
| Web Framework | Spring Boot + Tomcat | salary-api |
| Frontend | ReactJS | frontend |
| Primary DB (NoSQL) | ScyllaDB (CQL) | employee-api, salary-api |
| Primary DB (SQL) | PostgreSQL | attendance-api |
| Cache | Redis | all 3 APIs |
| DB Migrations | golang-migrate | employee-api, salary-api |
| DB Migrations | Liquibase | attendance-api |
| API Docs | Swagger / OpenAPI | all 3 APIs |
| Metrics | Prometheus | all 3 APIs |
| Observability | OpenTelemetry | salary-api |
| Containerization | Docker (multi-stage) | all services |
| Image Registry | quay.io/opstree | all services |
| Package Manager | go mod | employee-api |
| Package Manager | Poetry | attendance-api |
| Package Manager | Maven | salary-api |
| Package Manager | npm | frontend |

---

## Databases & Infrastructure

### ScyllaDB
- Cassandra-compatible NoSQL database
- Used by: **employee-api** and **salary-api**
- Port: `9042` (CQL native)
- Keyspace: `employee_db`
- Connection DSN: `cassandra://host:9042/employee_db?username=scylladb&password=password`

### PostgreSQL
- Relational SQL database
- Used by: **attendance-api** only
- Migrations managed by **Liquibase**

### Redis
- In-memory key-value store used as a **cache layer**
- Used by: **all three APIs**
- Caches GET response payloads for fast repeated reads

---

## API Endpoints Reference

### employee-api (Go · Gin)
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/metrics` | Prometheus metrics |
| GET | `/swagger/index.html` | Swagger UI |
| GET | `/api/v1/employee/health` | Shallow health check |
| GET | `/api/v1/employee/health/detail` | Detailed health (DB + Redis) |
| POST | `/api/v1/employee/create` | Create employee record |
| GET | `/api/v1/employee/search` | Search by query param |
| GET | `/api/v1/employee/search/all` | Fetch all employees |
| GET | `/api/v1/employee/search/location` | Count by location |
| GET | `/api/v1/employee/search/designation` | Count by designation |

### attendance-api (Python · Flask)
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/metrics` | Prometheus metrics |
| GET | `/apidocs` | Swagger UI |
| GET | `/api/v1/attendance/health` | Shallow health check |
| GET | `/api/v1/attendance/health/detail` | Detailed health (PostgreSQL + Redis) |
| POST | `/api/v1/attendance/create` | Create attendance record |
| GET | `/api/v1/attendance/search` | Search by query param |
| GET | `/api/v1/attendance/search/all` | Fetch all records |

### salary-api (Java · Spring Boot)
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/actuator/prometheus` | Prometheus metrics |
| GET | `/actuator/health` | Health check |
| GET | `/salary-documentation` | Swagger UI |
| POST | `/api/v1/salary/create/record` | Create salary record |
| GET | `/api/v1/salary/search` | Search by query param |
| GET | `/api/v1/salary/search/all` | Fetch all records |

---

## Local Setup on Ubuntu/Debian

### Prerequisites

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Git
sudo apt install -y git curl wget jq unzip
```

### Step 1 — Install Docker

```bash
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker
docker --version  # verify
```

### Step 2 — Install Go (for employee-api)

```bash
wget https://go.dev/dl/go1.21.0.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.21.0.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc
go version  # verify
```

### Step 3 — Install Python + Poetry (for attendance-api)

```bash
sudo apt install -y python3 python3-pip python3-venv
pip3 install poetry
poetry --version  # verify
```

### Step 4 — Install Java + Maven (for salary-api)

```bash
sudo apt install -y openjdk-17-jdk maven
java -version   # verify
mvn -version    # verify
```

### Step 5 — Install Node.js + npm (for frontend)

```bash
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs
node --version  # verify
npm --version   # verify
```

### Step 6 — Install ScyllaDB (via Docker)

```bash
docker run --name scylladb \
  -p 9042:9042 \
  -d scylladb/scylla:5.2

# Wait ~30 seconds, then verify
docker exec -it scylladb cqlsh -e "DESCRIBE KEYSPACES"
```

### Step 7 — Install PostgreSQL

```bash
sudo apt install -y postgresql postgresql-contrib
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Create user and database for attendance-api
sudo -u postgres psql -c "CREATE USER attendance WITH PASSWORD 'password';"
sudo -u postgres psql -c "CREATE DATABASE attendance_db OWNER attendance;"
```

### Step 8 — Install Redis

```bash
sudo apt install -y redis-server
sudo systemctl start redis
sudo systemctl enable redis
redis-cli ping  # should return PONG
```

### Step 9 — Install golang-migrate (for DB migrations)

```bash
curl -L https://github.com/golang-migrate/migrate/releases/download/v4.16.2/migrate.linux-amd64.tar.gz | tar xvz
sudo mv migrate /usr/local/bin/
migrate --version  # verify
```

### Step 10 — Install Liquibase (for attendance-api migrations)

```bash
wget https://github.com/liquibase/liquibase/releases/download/v4.24.0/liquibase-4.24.0.tar.gz
sudo mkdir /opt/liquibase
sudo tar -xzf liquibase-4.24.0.tar.gz -C /opt/liquibase
echo 'export PATH=$PATH:/opt/liquibase' >> ~/.bashrc
source ~/.bashrc
liquibase --version  # verify
```

### Step 11 — Clone & Set Up Repositories

```bash
mkdir ot-microservices && cd ot-microservices

git clone https://github.com/OT-MICROSERVICES/employee-api
git clone https://github.com/OT-MICROSERVICES/attendance-api
git clone https://github.com/OT-MICROSERVICES/salary-api
git clone https://github.com/OT-MICROSERVICES/frontend
git clone https://github.com/OT-MICROSERVICES/notification-worker
```

---

## Build & Run Commands

### employee-api (Go)

```bash
cd employee-api

# Edit config.yaml with your ScyllaDB and Redis host/port

# Run DB migrations
make run-migrations

# Build binary
make build

# Run the app
export GIN_MODE=release
./employee-api

# Or build and run Docker image
make docker-build
docker run -p 8080:8080 quay.io/opstree/employee-api
```

### attendance-api (Python)

```bash
cd attendance-api

# Install dependencies
make build          # installs via poetry

# Run DB migrations
make run-migrations # uses Liquibase

# Run with Gunicorn
gunicorn app:app --log-config log.conf -b 0.0.0.0:8080

# Code quality check
make fmt            # runs pylint

# Run tests
python3 -m pytest
python3 -m pytest --cov=.   # with coverage
```

### salary-api (Java)

```bash
cd salary-api

# Build JAR
make build          # runs mvn clean package

# Run DB migrations
make run-migrations # uses golang-migrate with cassandra:// DSN

# Run the JAR
java -jar target/salary-0.1.0-RELEASE.jar

# Run tests + coverage
make test           # mvn test (JUnit + Jacoco)

# Code quality
make fmt            # mvn checkstyle:checkstyle

# Docker
make docker-build
make docker-push
```

### frontend (ReactJS)

```bash
cd frontend

# Install npm dependencies + build
make build          # npm install + npm run build

# Docker
make docker-build
make docker-run
```

---

## Testing Strategy

| Service | Test Framework | Coverage Tool | Quality Tool |
|---------|---------------|---------------|-------------|
| employee-api | go test | cover.out (HTML) | — |
| attendance-api | pytest | pytest-cov | pylint |
| salary-api | JUnit | Jacoco | Checkstyle |
| frontend | — (React test integration) | — | — |

### employee-api tests
```bash
go test $(go list ./... | grep -v docs | grep -v model | grep -v main.go) -coverprofile cover.out
go tool cover -html=cover.out  # HTML report
```
Covered packages: `api`, `client`, `config`, `middleware`, `routes`

### attendance-api tests
```bash
python3 -m pytest
python3 -m pytest --cov=.
```
Covered modules: `router`, `client`, `models`, `utils`

### salary-api tests
```bash
mvn test                      # JUnit + Jacoco
mvn checkstyle:checkstyle     # code quality report
```
Test path: `src/test/java/com/opstree/microservice/salary/`

---

## Docker & Container Info

All services follow the same pattern:
- Each has a `Dockerfile` and `Makefile` with `docker-build` and `docker-push` targets
- Images are pushed to `quay.io/opstree/`
- All images expose port `8080`

| Service | Base Image (build) | Base Image (runtime) |
|---------|-------------------|---------------------|
| employee-api | golang:alpine | alpine |
| attendance-api | python:slim | python:slim |
| salary-api | maven:3.6.3-openjdk-17 | alpine:3.18 + openjdk17 |
| frontend | node | nginx / node |

### Quick health check after startup

```bash
# employee-api
curl http://localhost:8080/api/v1/employee/health

# attendance-api
curl http://localhost:8080/api/v1/attendance/health

# salary-api
curl http://localhost:8080/actuator/health
```

---

## Contact

**Opstree Opensource** — opensource@opstree.com  
**Website** — https://opstree.com  
**Twitter** — @opstreevdevops
