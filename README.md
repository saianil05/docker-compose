# docker-compose
docker-compose


# Complete Production-Level Docker Compose Guide

# Industry Standard Docker Compose + Dockerfile + Volumes + Networks + Deep Explanation

---

# Table of Contents

1. Introduction
2. Real Production Architecture
3. Project Structure
4. .env File
5. .env.example
6. .gitignore
7. Docker Compose File
8. Dockerfile
9. Nginx Configuration
10. Deep Explanation of Every Docker Compose Line
11. Docker Networks Deep Dive
12. Docker Volumes Deep Dive
13. Bind Mount vs Named Volume
14. Health Checks
15. Restart Policies
16. Docker Commands
17. Real Production Best Practices
18. CI/CD Flow
19. Common Interview Questions

---

# 1. Introduction

This document explains a REAL production-style Docker Compose setup.

We are building:

* Node.js Backend Application
* PostgreSQL Database
* Redis Cache
* Nginx Reverse Proxy

Using:

* Docker Compose
* Dockerfile
* Named Volumes
* Docker Networks
* Environment Variables
* Health Checks
* Non-root Security
* Logging Configuration

This setup is very close to real enterprise environments.

---

# 2. Real Production Architecture

```text
                 INTERNET
                     │
                     │
                 NGINX CONTAINER
                     │
         ┌───────────┴───────────┐
         │                       │
         │                       │
     NODE.JS APP            REDIS CACHE
         │
         │
    POSTGRES DATABASE
```

---

# Why This Architecture?

## Nginx

Used for:

* Reverse Proxy
* SSL Termination
* Load Balancing
* Security
* Rate Limiting
* Static File Serving

---

## Node.js App

Handles:

* Business Logic
* APIs
* Authentication
* Database Communication

---

## Redis

Used for:

* Caching
* Session Storage
* Queueing
* Performance Improvement

---

## PostgreSQL

Used for:

* Persistent Storage
* Relational Database
* Transactions

---

# 3. Project Structure

```text
project/
│
├── docker-compose.yml
├── .env
├── .env.example
├── .gitignore
├── README.md
│
├── app/
│   ├── Dockerfile
│   ├── package.json
│   ├── package-lock.json
│   └── src/
│
├── nginx/
│   └── nginx.conf
│
└── logs/
```

---

# 4. .env File

Location:

```text
project/.env
```

IMPORTANT:

This file contains REAL secrets.

NEVER push this file to GitHub.

```env
# =========================================================
# APPLICATION
# =========================================================

APP_NAME=my-node-app

NODE_ENV=production

APP_PORT=3000

# =========================================================
# DATABASE
# =========================================================

POSTGRES_DB=myappdb

POSTGRES_USER=appuser

POSTGRES_PASSWORD=supersecretpassword

POSTGRES_PORT=5432

# =========================================================
# REDIS
# =========================================================

REDIS_PORT=6379

# =========================================================
# NGINX
# =========================================================

NGINX_PORT=80

# =========================================================
# CONTAINER NAMES
# =========================================================

APP_CONTAINER=production_app

POSTGRES_CONTAINER=production_postgres

REDIS_CONTAINER=production_redis

NGINX_CONTAINER=production_nginx
```

---

# Why Use .env?

Instead of hardcoding:

```yaml
POSTGRES_PASSWORD: mypassword
```

we use:

```yaml
POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
```

Benefits:

* Better security
* Easier CI/CD
* Different values for dev/stage/prod
* Industry standard

---

# 5. .env.example

This file is SAFE to push to GitHub.

```env
POSTGRES_DB=your_db
POSTGRES_USER=your_user
POSTGRES_PASSWORD=your_password
```

When someone clones the repository:

```bash
cp .env.example .env
```

Then they update the values.

---

# 6. .gitignore

```gitignore
# Ignore secrets
.env

# Ignore node modules
node_modules/

# Ignore logs
logs/

# Ignore override files
docker-compose.override.yml
```

---

# 7. docker-compose.yml

```yaml
version: "3.9"

services:

  # =====================================================
  # APPLICATION CONTAINER
  # =====================================================

  app:

    # Build image from Dockerfile
    build:
      context: ./app
      dockerfile: Dockerfile

    # Custom container name
    container_name: ${APP_CONTAINER}

    # Restart automatically if container crashes
    restart: unless-stopped

    # Load variables from .env
    env_file:
      - .env

    # Environment variables inside container
    environment:
      NODE_ENV: ${NODE_ENV}

      APP_PORT: ${APP_PORT}

      DB_HOST: postgres

      DB_PORT: ${POSTGRES_PORT}

      DB_NAME: ${POSTGRES_DB}

      DB_USER: ${POSTGRES_USER}

      DB_PASSWORD: ${POSTGRES_PASSWORD}

      REDIS_HOST: redis

      REDIS_PORT: ${REDIS_PORT}

    # Publish ports
    ports:
      - "${APP_PORT}:${APP_PORT}"

    # Start only after DB and Redis
    depends_on:
      postgres:
        condition: service_healthy

      redis:
        condition: service_started

    # Named volume
    volumes:
      - app_logs:/app/logs

    # Docker network
    networks:
      - backend_network

    # Health check
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:${APP_PORT}/health"]

      interval: 30s

      timeout: 10s

      retries: 3

    # Logging configuration
    logging:
      driver: json-file

      options:
        max-size: "10m"

        max-file: "3"

  # =====================================================
  # POSTGRES DATABASE
  # =====================================================

  postgres:

    image: postgres:16-alpine

    container_name: ${POSTGRES_CONTAINER}

    restart: unless-stopped

    env_file:
      - .env

    environment:
      POSTGRES_DB: ${POSTGRES_DB}

      POSTGRES_USER: ${POSTGRES_USER}

      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}

    ports:
      - "${POSTGRES_PORT}:5432"

    # Persistent volume
    volumes:
      - postgres_data:/var/lib/postgresql/data

    networks:
      - backend_network

    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]

      interval: 10s

      timeout: 5s

      retries: 5

  # =====================================================
  # REDIS CACHE
  # =====================================================

  redis:

    image: redis:7-alpine

    container_name: ${REDIS_CONTAINER}

    restart: unless-stopped

    command: redis-server --appendonly yes

    ports:
      - "${REDIS_PORT}:6379"

    volumes:
      - redis_data:/data

    networks:
      - backend_network

  # =====================================================
  # NGINX REVERSE PROXY
  # =====================================================

  nginx:

    image: nginx:stable-alpine

    container_name: ${NGINX_CONTAINER}

    restart: unless-stopped

    ports:
      - "${NGINX_PORT}:80"

    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro

    depends_on:
      - app

    networks:
      - backend_network

# =====================================================
# NETWORKS
# =====================================================

networks:

  backend_network:

    driver: bridge

# =====================================================
# VOLUMES
# =====================================================

volumes:

  postgres_data:

  redis_data:

  app_logs:
```

---

# 8. Dockerfile

Location:

```text
project/app/Dockerfile
```

```dockerfile
# =====================================================
# BASE IMAGE
# =====================================================

FROM node:20-alpine

# =====================================================
# WORKING DIRECTORY
# =====================================================

WORKDIR /app

# =====================================================
# COPY PACKAGE FILES
# =====================================================

COPY package*.json ./

# =====================================================
# INSTALL DEPENDENCIES
# =====================================================

RUN npm ci --only=production

# =====================================================
# COPY APPLICATION CODE
# =====================================================

COPY . .

# =====================================================
# CREATE NON-ROOT USER
# =====================================================

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# =====================================================
# CHANGE OWNERSHIP
# =====================================================

RUN chown -R appuser:appgroup /app

# =====================================================
# SWITCH USER
# =====================================================

USER appuser

# =====================================================
# EXPOSE PORT
# =====================================================

EXPOSE 3000

# =====================================================
# START APPLICATION
# =====================================================

CMD ["npm", "start"]
```

---

# 9. nginx.conf

Location:

```text
project/nginx/nginx.conf
```

```nginx
events {}

http {

    upstream backend {

        server app:3000;
    }

    server {

        listen 80;

        location / {

            proxy_pass http://backend;

            proxy_http_version 1.1;

            proxy_set_header Upgrade $http_upgrade;

            proxy_set_header Connection 'upgrade';

            proxy_set_header Host $host;
        }
    }
}
```

---

# 10. Deep Explanation of Important Docker Compose Concepts

---

# version: "3.9"

Defines Docker Compose specification version.

Controls supported features.

---

# services:

Each service = one container.

Example:

* app
* postgres
* redis
* nginx

---

# build:

```yaml
build:
  context: ./app
```

Docker builds image from:

```text
./app
```

folder.

---

# container_name

Custom container name.

Without this:

Docker creates random names.

---

# restart: unless-stopped

If container crashes:

Docker restarts it automatically.

Very common in production.

---

# env_file

```yaml
env_file:
  - .env
```

Loads variables from:

```text
.env
```

---

# environment

Environment variables INSIDE container.

Example in Node.js:

```javascript
process.env.DB_HOST
```

---

# ports

```yaml
ports:
  - "3000:3000"
```

Format:

```text
HOST:CONTAINER
```

Meaning:

Browser → Host Port → Container Port

---

# depends_on

Controls startup order.

Application starts AFTER database.

---

# healthcheck

Verifies application is healthy.

Used heavily in:

* Kubernetes
* ECS
* Docker Swarm
* Load Balancers

---

# logging

Prevents unlimited log growth.

Without this:

Server disk may become full.

---

# 11. Docker Networks Deep Dive

# Why Networks Exist

Containers are isolated.

Without networks:

Containers cannot communicate safely.

---

# Example

```yaml
networks:
  - backend_network
```

All containers join same internal network.

---

# Internal Communication

Application connects to PostgreSQL using:

```text
postgres
```

NOT IP address.

Docker automatically provides DNS.

---

# Example

Inside app container:

```bash
ping postgres
```

works automatically.

Docker resolves:

```text
postgres → container IP
```

---

# Types of Docker Networks

## Bridge Network

```yaml
driver: bridge
```

Default Docker Compose network.

Most commonly used.

Works on same Docker host.

---

## Host Network

Container directly uses host machine network.

Higher performance.

Less isolation.

Mostly Linux-only.

---

## Overlay Network

Used for:

* Docker Swarm
* Multi-host communication

---

## None Network

No networking.

Fully isolated container.

---

# Why Use Custom Networks?

Benefits:

* Better security
* Isolation
* Internal communication
* Service discovery

---

# 12. Docker Volumes Deep Dive

# What Problem Volumes Solve

Containers are temporary.

Deleting container deletes data.

Dangerous for databases.

---

# Example

```yaml
volumes:
  - postgres_data:/var/lib/postgresql/data
```

LEFT SIDE:

```text
postgres_data
```

Docker-managed named volume.

RIGHT SIDE:

```text
/var/lib/postgresql/data
```

Actual PostgreSQL storage path INSIDE container.

---

# Real Flow

PostgreSQL writes data:

```text
inside container
```

Docker stores it in:

```text
Docker Volume
```

Even if container dies:

Data survives.

---

# Do We Need To Create Volumes Manually?

NO.

Docker Compose automatically creates volumes.

When you run:

```bash
docker compose up
```

Docker automatically creates:

* postgres_data
* redis_data
* app_logs

---

# View Volumes

```bash
docker volume ls
```

---

# Inspect Volume

```bash
docker volume inspect project_postgres_data
```

---

# Remove Volumes

WARNING:

Deletes database data.

```bash
docker compose down -v
```

---

# 13. Bind Mount vs Named Volume

# Bind Mount

```yaml
volumes:
  - ./code:/app
```

Host folder mounted directly.

Mostly used in development.

Changes reflect immediately.

---

# Named Volume

```yaml
volumes:
  - postgres_data:/var/lib/postgresql/data
```

Docker-managed.

Mostly used in production.

More portable and secure.

---

# 14. Health Checks

Without health checks:

Container may RUN but app may be broken.

Health checks verify application works properly.

---

# Example

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
```

---

# 15. Restart Policies

## restart: no

Never restart.

---

## restart: always

Always restart.

---

## restart: on-failure

Restart only on failure.

---

## restart: unless-stopped

Most common production option.

---

# 16. Docker Commands

# Start Containers

```bash
docker compose up -d
```

---

# View Containers

```bash
docker ps
```

---

# View Logs

```bash
docker compose logs -f
```

---

# Stop Containers

```bash
docker compose down
```

---

# Stop + Remove Volumes

```bash
docker compose down -v
```

---

# View Networks

```bash
docker network ls
```

---

# View Volumes

```bash
docker volume ls
```

---

# Build Images

```bash
docker compose build
```

---

# Rebuild Containers

```bash
docker compose up --build
```

---

# 17. Real Production Best Practices

* Never hardcode secrets
* Use non-root users
* Use health checks
* Use restart policies
* Use named volumes
* Limit logs
* Use internal networks
* Use lightweight images
* Use CI/CD
* Scan images for vulnerabilities
* Use .env.example
* Use reverse proxy

---

# 18. Real CI/CD Flow

```text
Developer
    ↓
GitHub
    ↓
CI/CD Pipeline
    ↓
Docker Image Build
    ↓
Push to Registry
    ↓
Deploy Containers
    ↓
Nginx Routes Traffic
    ↓
Application Uses DB + Redis
```

---

# 19. Common Interview Questions

1. Difference between CMD and ENTRYPOINT?
2. Difference between bind mount and named volume?
3. Why use health checks?
4. Why use Docker networks?
5. Why avoid root user?
6. Why use Alpine images?
7. Difference between expose and ports?
8. Why use reverse proxy?
9. What happens without volumes?
10. How does Docker DNS work?
11. Why use restart policies?
12. Difference between bridge and overlay network?
13. Why use .env files?
14. Why should secrets not be committed?
15. Difference between image and container?
16. Why use Docker Compose?
17. Why use Redis?
18. What is Nginx upstream?
19. Why use health checks in production?
20. Why separate services into containers?

---

# FILE-BY-FILE EXPLANATION

---

# docker-compose.yml

Purpose:

This is the MAIN orchestration file.

Docker Compose reads this file and creates:

* Containers
* Networks
* Volumes
* Port mappings
* Environment variables
* Health checks
* Restart policies

Think of this file as:

```text
Infrastructure Blueprint
```

This file tells Docker:

```text
What to create
How to connect everything
How containers should behave
```

---

# .env

Purpose:

Stores REAL values.

Example:

* passwords
* usernames
* ports
* secrets
* container names

Instead of hardcoding:

```yaml
POSTGRES_PASSWORD: mypassword
```

we use:

```yaml
POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
```

Benefits:

* safer
* reusable
* production standard
* environment-specific

IMPORTANT:

Never push `.env` to GitHub.

---

# .env.example

Purpose:

Template file for developers.

Contains dummy/sample values.

Safe to push to GitHub.

Used when someone clones repository.

Example:

```bash
cp .env.example .env
```

---

# .gitignore

Purpose:

Tells Git which files/folders should NOT be committed.

Example:

```gitignore
.env
node_modules/
logs/
```

Without `.gitignore`:

Secrets may leak to GitHub.

---

# Dockerfile

Purpose:

Defines HOW Docker image should be built.

Think of Dockerfile as:

```text
Recipe for creating Docker Image
```

Docker Compose uses Dockerfile to build app image.

Flow:

```text
Dockerfile
   ↓
Docker Image
   ↓
Docker Container
```

---

# nginx.conf

Purpose:

Configures Nginx reverse proxy.

Nginx sits BEFORE application.

Flow:

```text
User Request
   ↓
Nginx
   ↓
Node.js App
```

Responsibilities:

* reverse proxy
* SSL termination
* load balancing
* request forwarding
* security

---

# package.json

Purpose:

Defines:

* Node.js dependencies
* scripts
* metadata

Example:

```json
{
  "scripts": {
    "start": "node server.js"
  }
}
```

Docker uses this during:

```dockerfile
RUN npm ci
```

---

# package-lock.json

Purpose:

Locks exact dependency versions.

Ensures:

```text
Same versions everywhere
```

Very important in production.

---

# src/

Contains actual application code.

Example:

* APIs
* controllers
* services
* business logic

---

# NETWORK SECTION DEEP EXPLANATION

---

# Why Docker Networks Are Used

Suppose we have:

* app container
* postgres container
* redis container
* nginx container

All containers are separate isolated environments.

Without Docker network:

* app cannot talk to postgres
* nginx cannot talk to app
* redis cannot talk to app

So Docker network is used for:

* container communication
* internal DNS
* isolation
* security
* service discovery

---

# Real Example

Node.js application needs database connection.

Normally:

```text
Application → PostgreSQL
```

But PostgreSQL is running inside ANOTHER container.

Docker network connects them.

---

# How App Connects To Database

In application:

```env
DB_HOST=postgres
```

Why `postgres`?

Because Docker network automatically creates internal DNS.

Docker resolves:

```text
postgres → container IP
```

Automatically.

So application never needs actual IP address.

---

# Why Custom Network Is Better

Without custom network:

Containers may use default network.

But custom network provides:

* better isolation
* better security
* cleaner architecture
* easier debugging
* service grouping

Industry standard approach.

---

# Example

```yaml
networks:
  backend_network:
    driver: bridge
```

This creates:

```text
Custom Network Named:
backend_network
```

All services attached to this network can communicate.

---

# Why driver: bridge Is Used

```yaml
driver: bridge
```

Bridge is MOST COMMON Docker network type.

Used when:

* containers run on SAME machine
* containers need internal communication
* isolated container networking is needed

This is default production-style Compose networking.

---

# How Bridge Network Works

Docker creates:

* virtual internal switch
* internal subnet
* internal DNS

Containers connect to this internal virtual network.

Example:

```text
app ↔ postgres ↔ redis ↔ nginx
```

All communication happens INSIDE Docker network.

Not exposed publicly.

---

# Why This Is Secure

Suppose postgres container has port 5432.

Without Docker network:

Database may need public IP.

With Docker network:

Only internal containers can access database.

Much safer.

---

# Real Production Flow

```text
Internet
   ↓
Nginx
   ↓
App Container
   ↓
Postgres Container
```

Postgres stays INTERNAL.

Users cannot directly access database.

---

# TYPES OF DOCKER NETWORKS

---

# What Is Docker Network?

Docker network allows containers to communicate.

Without network:

Containers are isolated.

---

# Example From Compose

```yaml
networks:
  backend_network:
    driver: bridge
```

This creates:

```text
Custom Docker Network
```

named:

```text
backend_network
```

---

# Why Network Is Needed

Application container needs to talk to:

* PostgreSQL
* Redis
* Nginx

Without network:

containers cannot communicate properly.

---

# Internal DNS

Docker automatically creates internal DNS.

So app connects to DB using:

```text
postgres
```

instead of:

```text
192.168.x.x
```

Docker resolves:

```text
postgres → container IP
```

automatically.

---

# Real Flow

App container:

```text
DB_HOST=postgres
```

Docker network resolves:

```text
postgres
```

into actual container IP.

---

# driver: bridge

```yaml
driver: bridge
```

Most common Docker network type.

Used for:

```text
Single Docker Host Communication
```

Containers communicate securely internally.

---

# Types of Docker Networks

## Bridge Network

Default.

Most common.

Single-host communication.

---

## Host Network

Container uses host machine network directly.

Higher performance.

Less isolation.

Mostly Linux-only.

---

## Overlay Network

Used in:

* Docker Swarm
* Multi-host communication

---

## None Network

No networking.

Fully isolated.

---

# View Networks

```bash
docker network ls
```

---

# Inspect Network

```bash
docker network inspect project_backend_network
```

---

# VOLUME SECTION DEEP EXPLANATION

---

# Why Docker Volumes Are Used

Containers are TEMPORARY.

Suppose PostgreSQL stores data INSIDE container.

Then:

```bash
docker compose down
```

or deleting container means:

❌ ALL DATABASE DATA LOST

This is dangerous.

---

# What Docker Volume Does

Docker volume stores data OUTSIDE container.

Even if container dies:

✅ Data survives.

This is why volumes are VERY important in production.

---

# Real Example

```yaml
volumes:
  - postgres_data:/var/lib/postgresql/data
```

LEFT SIDE:

```text
postgres_data
```

Docker-managed named volume.

RIGHT SIDE:

```text
/var/lib/postgresql/data
```

Actual PostgreSQL storage location INSIDE container.

---

# Real Internal Flow

PostgreSQL writes data:

```text
inside container path
/var/lib/postgresql/data
```

Docker redirects it to:

```text
Docker Volume
postgres_data
```

So even if container is deleted:

volume still exists.

---

# Why Volumes Are Needed In Production

Without volume:

* DB data lost
* logs lost
* uploads lost
* Redis cache lost

Production systems MUST persist important data.

---

# Why app_logs Volume Is Used

```yaml
volumes:
  - app_logs:/app/logs
```

Application logs survive container restart.

Useful for:

* debugging
* monitoring
* auditing

---

# Why redis_data Volume Is Used

```yaml
volumes:
  - redis_data:/data
```

Redis is normally memory-based.

But:

```yaml
command: redis-server --appendonly yes
```

enables persistence.

Volume stores Redis persistent data.

---

# Do We Need To Create Volumes Before?

NO.

Docker Compose automatically creates them.

When you run:

```bash
docker compose up
```

Docker creates:

* postgres_data
* redis_data
* app_logs

automatically.

---

# Why Named Volumes Are Preferred In Production

Benefits:

* managed by Docker
* portable
* safer
* isolated
* better permissions
* easier backups

---

# TYPES OF DOCKER STORAGE

---

# What Is Docker Volume?

Volume stores persistent data.

Containers are temporary.

Without volumes:

Deleting container = deleting data.

Very dangerous for databases.

---

# Example From Compose

```yaml
volumes:
  - postgres_data:/var/lib/postgresql/data
```

LEFT SIDE:

```text
postgres_data
```

Docker-managed named volume.

RIGHT SIDE:

```text
/var/lib/postgresql/data
```

Actual PostgreSQL storage path INSIDE container.

---

# Real Flow

PostgreSQL writes data:

```text
inside container
```

Docker redirects it into:

```text
Docker Volume
```

Even if container is deleted:

Data survives.

---

# Do We Need To Create Volumes Before?

NO.

Docker Compose automatically creates them.

When you run:

```bash
docker compose up
```

Docker automatically creates:

* postgres_data
* redis_data
* app_logs

---

# Where Docker Stores Volumes

Linux:

```bash
/var/lib/docker/volumes/
```

Windows Docker Desktop:

Inside Docker VM.

---

# View Volumes

```bash
docker volume ls
```

---

# Inspect Volume

```bash
docker volume inspect project_postgres_data
```

---

# Remove Volumes

WARNING:

Deletes database data.

```bash
docker compose down -v
```

---

# Bind Mount vs Named Volume

## Bind Mount

```yaml
volumes:
  - ./code:/app
```

Host folder directly mounted.

Mostly used in development.

Changes reflect immediately.

---

## Named Volume

```yaml
volumes:
  - postgres_data:/var/lib/postgresql/data
```

Docker-managed.

Mostly used in production.

More portable and secure.

---

# Final Important Note

Docker Compose is heavily used in:

* Local development
* QA
* Staging
* Small production deployments

Large enterprises often move to:

* Kubernetes
* AWS ECS
* Nomad

BUT:

Docker Compose knowledge is foundational and extremely important for DevOps and backend interviews.
