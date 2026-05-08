# Docker Networks and Volumes Complete Guide

# Production-Level Deep Explanation

---

# Table of Contents

1. What is Docker Network?
2. Why Docker Networks Are Needed
3. How Containers Communicate
4. Internal Docker DNS
5. Types of Docker Networks
6. Bridge Network Deep Dive
7. Host Network Deep Dive
8. Overlay Network Deep Dive
9. None Network
10. Real Production Network Flow
11. Docker Network Commands
12. What is Docker Volume?
13. Why Docker Volumes Are Needed
14. How Docker Volumes Work Internally
15. Named Volumes
16. Bind Mounts
17. tmpfs Mounts
18. Named Volume vs Bind Mount
19. Real Production Volume Usage
20. Docker Volume Commands
21. Common Interview Questions

---

# 1. What is Docker Network?

Docker network allows containers to communicate with each other.

Containers are isolated by default.

Without Docker network:

* app container cannot talk to DB container
* nginx cannot talk to app
* redis cannot talk to app

Docker network solves this.

---

# Real Example

Suppose we have:

* app container
* postgres container
* redis container
* nginx container

Application needs database connection.

Flow:

```text
App → PostgreSQL
```

But PostgreSQL runs inside another container.

Docker network connects them.

---

# 2. Why Docker Networks Are Needed

Docker networks provide:

* container communication
* isolation
* internal DNS
* service discovery
* security
* traffic control

---

# Without Docker Network

Containers become isolated environments.

App cannot communicate with:

* database
* redis
* nginx

Application fails.

---

# 3. How Containers Communicate

Example:

```yaml
services:
  app:
    networks:
      - backend_network

  postgres:
    networks:
      - backend_network
```

Both containers join same network.

Now app container can talk to postgres container.

---

# Real Internal Communication

Application environment variable:

```env
DB_HOST=postgres
```

Why `postgres`?

Because Docker provides internal DNS.

Docker automatically resolves:

```text
postgres → container IP
```

No need to use actual IP address.

---

# Example Flow

```text
Node.js App
    ↓
DB_HOST=postgres
    ↓
Docker DNS
    ↓
postgres container IP
    ↓
PostgreSQL Database
```

---

# 4. Internal Docker DNS

Docker automatically creates DNS entries for containers.

Suppose container name:

```text
postgres
```

Inside app container:

```bash
ping postgres
```

works automatically.

Docker handles internal name resolution.

This is why Docker Compose service names are important.

---

# 5. Types of Docker Networks

Docker supports multiple network types.

---

# Bridge Network

Most common.

Default Docker Compose network.

Used for:

```text
Containers running on SAME Docker host
```

---

# Host Network

Container directly uses host machine network.

Higher performance.

Less isolation.

Mostly Linux-only.

---

# Overlay Network

Used in:

* Docker Swarm
* Multi-host communication

Containers across multiple servers communicate.

---

# None Network

No networking.

Fully isolated container.

---

# 6. Bridge Network Deep Dive

Example:

```yaml
networks:
  backend_network:
    driver: bridge
```

This creates:

```text
Custom Bridge Network
```

named:

```text
backend_network
```

---

# Why Bridge Network Is Used

Bridge network provides:

* internal communication
* isolation
* internal DNS
* safer architecture

Most production-style Docker Compose setups use bridge networks.

---

# How Bridge Network Works Internally

Docker creates:

* virtual switch
* internal subnet
* routing tables
* internal DNS

Containers connect to this virtual network.

Example:

```text
app ↔ postgres ↔ redis ↔ nginx
```

All communication stays INSIDE Docker.

---

# Why This Is Secure

Suppose PostgreSQL runs on port:

```text
5432
```

With bridge network:

Database stays internal.

Only containers inside network can access it.

External users cannot directly access DB.

---

# 7. Host Network Deep Dive

Example:

```bash
docker run --network host nginx
```

Container shares host network directly.

Advantages:

* better performance
* no NAT overhead

Disadvantages:

* less security
* no isolation
* port conflicts

Mostly used for:

* monitoring tools
* high-performance systems

---

# 8. Overlay Network Deep Dive

Used in:

* Docker Swarm
* Multi-node clusters

Allows containers across MULTIPLE servers to communicate.

Example:

```text
Server 1 container ↔ Server 2 container
```

---

# 9. None Network

Example:

```bash
docker run --network none nginx
```

Container gets NO network access.

Used for:

* security-sensitive workloads
* offline processing

---

# 10. Real Production Network Flow

```text
Internet
   ↓
Nginx Container
   ↓
App Container
   ↓
Postgres Container
```

Database remains INTERNAL.

Only app container accesses database.

This is much safer.

---

# 11. Docker Network Commands

# View Networks

```bash
docker network ls
```

---

# Inspect Network

```bash
docker network inspect network_name
```

---

# Create Network

```bash
docker network create backend_network
```

---

# Remove Network

```bash
docker network rm network_name
```

---

# What is Docker Volume?

Docker volume stores persistent data.

Containers are temporary.

Deleting container deletes container filesystem.

Volumes prevent data loss.

---

# 13. Why Docker Volumes Are Needed

Suppose PostgreSQL stores data INSIDE container.

Then:

```bash
docker rm container_name
```

means:

❌ ALL DATABASE DATA LOST

Very dangerous.

---

# Docker Volume Solves This

Docker stores important data OUTSIDE container.

Even if container dies:

✅ Data survives.

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

Named Docker volume.

RIGHT SIDE:

```text
/var/lib/postgresql/data
```

Actual PostgreSQL storage location INSIDE container.

---

# 14. How Docker Volumes Work Internally

PostgreSQL writes data:

```text
inside container path
```

Docker redirects it to:

```text
Docker-managed volume
```

So:

```text
Container deleted ≠ Data deleted
```

---

# Real Internal Flow

```text
PostgreSQL Container
        ↓
/var/lib/postgresql/data
        ↓
Docker Volume
        ↓
Physical Host Storage
```

---

# Where Docker Stores Volumes

Linux:

```bash
/var/lib/docker/volumes/
```

Windows Docker Desktop:

Inside Docker VM.

---

# 15. Named Volumes

Example:

```yaml
volumes:
  - postgres_data:/var/lib/postgresql/data
```

Docker manages storage automatically.

Advantages:

* production-friendly
* portable
* isolated
* easy backups
* safer permissions

Most production systems use named volumes.

---

# Do We Need To Create Named Volumes Manually?

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

# 16. Bind Mounts

Example:

```yaml
volumes:
  - ./code:/app
```

Host folder directly mounted inside container.

Mostly used in DEVELOPMENT.

---

# Why Bind Mounts Are Used

Suppose developer changes code.

Changes instantly appear inside container.

Very useful for:

* hot reload
* local development
* debugging

---

# Disadvantages of Bind Mounts

* less portable
* host dependent
* permission issues
* weaker isolation

Not preferred for databases in production.

---

# 17. tmpfs Mounts

Temporary in-memory storage.

Example:

```bash
--tmpfs /temp
```

Used for:

* temporary files
* secrets
* caching

Data disappears after container stops.

---

# 18. Named Volume vs Bind Mount

# Named Volume

```yaml
volumes:
  - postgres_data:/var/lib/postgresql/data
```

Advantages:

* Docker managed
* portable
* secure
* production-friendly

---

# Bind Mount

```yaml
volumes:
  - ./code:/app
```

Advantages:

* live code updates
* easy debugging

Mostly used in development.

---

# Production Recommendation

# Use Named Volumes For:

* databases
* Redis persistence
* logs
* uploads

# Use Bind Mounts For:

* local development
* code syncing

---

# 19. Real Production Volume Usage

## PostgreSQL Volume

```yaml
volumes:
  - postgres_data:/var/lib/postgresql/data
```

Stores database permanently.

---

## Redis Volume

```yaml
volumes:
  - redis_data:/data
```

Stores Redis append-only persistence.

---

## Application Logs Volume

```yaml
volumes:
  - app_logs:/app/logs
```

Stores logs permanently.

Useful for:

* debugging
* auditing
* monitoring

---

# 20. Docker Volume Commands

# View Volumes

```bash
docker volume ls
```

---

# Inspect Volume

```bash
docker volume inspect volume_name
```

---

# Remove Volume

```bash
docker volume rm volume_name
```

---

# Remove Containers + Volumes

WARNING:

Deletes database data.

```bash
docker compose down -v
```

---

# 21. Common Interview Questions

1. Why are Docker networks needed?
2. How does Docker internal DNS work?
3. Difference between bridge and overlay network?
4. Difference between host and bridge network?
5. Why use custom networks?
6. Why are Docker volumes needed?
7. Difference between bind mount and named volume?
8. Why are named volumes preferred in production?
9. Where does Docker store volumes?
10. What happens if volume is removed?
11. Why should databases use volumes?
12. Why use internal Docker networking?
13. Why should DB not be publicly exposed?
14. Why are bind mounts mostly used in development?
15. How does Docker networking improve security?
