# Multi-Container Fibonacci Calculator

> Built as part of **Stephen Grider's Docker and Kubernetes: The Complete Guide** course on Udemy.

A deliberately over-engineered Fibonacci calculator — the point is never the app itself, but everything around it: multi-container architecture, Docker orchestration, a full CI/CD pipeline, and a production deployment on AWS.

---

## What the App Does

A user submits a Fibonacci index through a React UI. The Express API stores that index in PostgreSQL, then publishes it to a Redis channel. A background Worker picks it up, computes the Fibonacci value recursively, and stores the result back in Redis. The UI polls the API to display both the seen indexes and the calculated values.

Simple problem. Complex infrastructure. That's the point.

---

## Architecture

```
Browser
  └── Nginx (reverse proxy, port 80 / 3050 in dev)
        ├── /          → React Client (port 3000)
        └── /api/*     → Express Server (port 5000)
                            ├── PostgreSQL  (stores submitted indexes)
                            └── Redis       (pub/sub + result cache)
                                    └── Worker (computes fib, writes result)
```

Six containers, each with a single responsibility, communicating over a Docker-managed network.

---

## Services

| Service | Tech | Role |
|---|---|---|
| `nginx` | Nginx | Reverse proxy — routes `/` to client, `/api` to server |
| `client` | React + Nginx | Frontend UI |
| `server` (api) | Node.js + Express | REST API, DB writes, Redis publish |
| `worker` | Node.js | Subscribes to Redis, computes Fibonacci, stores result |
| `postgres` | PostgreSQL | Persistent storage of submitted indexes |
| `redis` | Redis | In-memory store + pub/sub broker |

---

## Key Learning Areas

### 1. Docker Fundamentals
- Writing `Dockerfile` and `Dockerfile.dev` separately — dev images use `nodemon` for hot-reload, production images run the final built artifact
- Multi-stage builds in the client: `node:16-alpine` builds the React app, then `nginx` serves the static output — keeping the final image lean
- The `COPY . .` pattern and why `package.json` is copied first (layer caching — only re-runs `npm install` when dependencies change)

### 2. Multi-Container Orchestration with Docker Compose
- `docker-compose-dev.yml` wires all six services together for local development
- Named services (`postgres`, `redis`, `api`, `client`) act as hostnames on the internal Docker network — no hardcoded IPs
- Volume mounts (`./server:/app`) enable live code reload without rebuilding the image
- `/app/node_modules` anonymous volume trick — prevents the host mount from overwriting the container's installed modules
- `depends_on` to control startup order (nginx waits for api and client)
- `WDS_SOCKET_PORT=0` environment variable to fix WebSocket issues with Create React App behind a proxy

### 3. Nginx as a Reverse Proxy
- Two Nginx instances: one at the top level routing between client and API, one inside the client container serving the built React files
- `upstream` blocks in `default.conf` define named backend targets
- `rewrite /api/(.*) /$1 break` strips the `/api` prefix before forwarding to Express
- WebSocket proxy config (`proxy_http_version 1.1`, `Upgrade`, `Connection` headers) for CRA's hot-reload websocket

### 4. Environment Variable Management
- `keys.js` files in `server` and `worker` act as a single source of truth, reading from `process.env`
- Dev values set directly in `docker-compose-dev.yml`; production values injected via Elastic Beanstalk environment configuration
- No secrets hardcoded anywhere — credentials flow in through CI/CD environment variables (`$DOCKER_PASSWORD`, `$AWS_ACCESS_KEY`, etc.)

### 5. CI/CD with Travis CI
The `.travis.yml` pipeline has four distinct phases:

```
1. before_install  → Build dev image for testing
2. script          → Run test suite (CI=true for non-interactive mode)
3. after_success   → Build all 4 production images, login to Docker Hub, push images
4. deploy          → Trigger Elastic Beanstalk deployment via Travis provider
```

- `CI=true` flag makes the React test runner exit after one run instead of watching
- `echo "$DOCKER_PASSWORD" | docker login --password-stdin` — avoids password in shell history
- Travis environment variables (`$AWS_ACCESS_KEY`, `$AWS_SECRET_KEY`, `$DOCKER_USERNAME`, `$DOCKER_PASSWORD`) store credentials securely

### 6. Production Deployment on AWS

**Elastic Beanstalk (Multi-Docker)**
- EB reads `docker-compose.yml` to pull images from Docker Hub and run the containers
- `mem_limit` set per container to stay within free-tier instance limits
- `hostname` values in `docker-compose.yml` match the upstream names Nginx expects in `default.conf`

**RDS (PostgreSQL)**
- Managed PostgreSQL in production — no DB container needed
- SSL conditionally enabled in `server/index.js`: `rejectUnauthorized: false` when `NODE_ENV === 'production'` to handle RDS certificates

**ElastiCache (Redis)**
- Managed Redis in production — worker and server connect using `REDIS_HOST` and `REDIS_PORT` environment variables injected at runtime

### 7. The "Why Over-Engineer It?" Lesson
The Fibonacci calculator is intentionally simple so the complexity of the infrastructure stands out. The core lesson: in real production systems, you separate concerns (compute, storage, cache, serving) into independent, replaceable units. Each container can be scaled, updated, or swapped independently.

---

## Local Development

```bash
docker-compose -f docker-compose-dev.yml up --build
```

App runs at `http://localhost:3050`

---

## Project Structure

```
/
├── client/         # React app + its own Nginx config for production
├── server/         # Express API
├── worker/         # Fibonacci background processor
├── nginx/          # Top-level reverse proxy
├── docker-compose.yml       # Production (used by Elastic Beanstalk)
├── docker-compose-dev.yml   # Local development
├── .travis.yml              # CI/CD pipeline
└── AWS_CHEAT_SHEET.md       # Step-by-step AWS setup reference
```

---

## Tech Stack

**App:** React, Node.js, Express, PostgreSQL, Redis  
**Containers:** Docker, Docker Compose, Nginx  
**CI/CD:** Travis CI, Docker Hub  
**Cloud:** AWS Elastic Beanstalk, RDS, ElastiCache

---

## Course Reference

[Docker and Kubernetes: The Complete Guide — Stephen Grider (Udemy)](https://www.udemy.com/course/docker-and-kubernetes-the-complete-guide/)
