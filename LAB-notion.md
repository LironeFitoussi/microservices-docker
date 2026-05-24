# Lab: Dockerize a Microservices App

You're handed a working microservices app that runs on your **host machine**. Your job: dockerize it from scratch, then evolve the setup through 7 milestones until you reach a production-grade, network-isolated, distroless deployment.

By the end you will have written, from scratch:

- A `Dockerfile` for each service
- Hand-crafted `docker network` plumbing with `docker run`
- A single `docker-compose.yml` that replaces all the manual commands
- A multi-database setup (one Mongo per service)
- An Nginx **API gateway** that load-balances by path
- A **segmented network topology** where the LB is the only bridge between zones
- **Distroless** runtime images with no shell and a non-root user

---

## What you start with

```
.
├── README.md
├── LAB.md
├── frontend/
│   ├── nginx.conf
│   └── public/index.html
├── lb/
│   └── nginx.conf
└── services/
    ├── users-service/
    │   ├── server.js
    │   └── package.json
    └── products-service/
        ├── server.js
        └── package.json
```

No `Dockerfile`, no `docker-compose.yml`, no Docker anything. Just source code.

## What you need installed

| Tool | Version | When you need it |
| --- | --- | --- |
| Docker Desktop (Win/Mac) or `docker` + `docker compose` (Linux) | 20.10+ | All milestones except 0 |
| Node.js | 20+ | Milestone 0 only |
| MongoDB | 7 (running on `localhost:27017`) | Milestone 0 only |
| Shell | PowerShell / bash / zsh | All milestones |

## How to use this lab

- **Each milestone is independent.** You can stop after any one and the app still works.
- **Verify after every milestone** using the `curl` commands at the end of each section.
- **Stuck?** A complete reference implementation lives on the `solution` branch:

```bash
git checkout solution -- <path/to/file>   # grab one file
git diff solution -- <path/to/file>       # compare yours to the solution
```

- **Reset between milestones:**

```bash
docker compose down -v     # stop everything, wipe volumes
docker ps -a               # confirm nothing's left
```

---

# Milestone 0: Run it on the host (no Docker)

**Goal:** Confirm the app works before we add any complexity. If it doesn't run here, no amount of Docker will fix it.

## Steps

1. Start MongoDB locally on port `27017`. (Skip if it's already running.)
2. Install dependencies for both services:

```bash
cd services/users-service && npm install && cd ../..
cd services/products-service && npm install && cd ../..
```

3. Open **two terminals**, one per service.

Terminal 1 (users-service on port 3000):

```bash
cd services/users-service && npm start
```

Terminal 2 (products-service on port 3001 — both default to 3000, so override):

```bash
# PowerShell
$env:PORT=3001; npm start

# bash / zsh
PORT=3001 npm start
```

> `server.js` reads `process.env.PORT || 3000`. Outside of Docker this override matters; inside Docker you give each container its own network namespace and they can both use `:3000`.

## Verify

```bash
curl -X POST http://localhost:3000/users \
  -H 'content-type: application/json' \
  -d '{"name":"Ada","email":"a@b.c"}'

curl http://localhost:3000/users

curl -X POST http://localhost:3001/products \
  -H 'content-type: application/json' \
  -d '{"name":"Widget","price":9.99,"stock":3}'
```

Each response should include `"service":"users-service"` or `"service":"products-service"`.

**Stop both services (Ctrl-C) before moving on.**

---

# Milestone 1: One Dockerfile, one container

**Goal:** Package `users-service` into a Docker image and run it. Mongo and products-service stay on the host for now.

## Steps

1. Create `services/users-service/Dockerfile`. Use a Node 20 base image. The image should:
    - Set `/app` as the working directory
    - Copy `package*.json` and run `npm install --omit=dev` **before** copying source (for layer caching)
    - Copy `server.js`
    - Expose port 3000
    - Run the server
2. Create `services/users-service/.dockerignore` so `node_modules` isn't copied into the build context:

```
node_modules
npm-debug.log
```

3. Build the image:

```bash
docker build -t users-service ./services/users-service
```

4. Run it. The container needs to reach Mongo on your host.

Windows / Mac (Docker Desktop):

```bash
docker run --rm -p 3000:3000 \
  -e MONGO_URI=mongodb://host.docker.internal:27017 \
  users-service
```

Linux:

```bash
docker run --rm -p 3000:3000 --network host \
  -e MONGO_URI=mongodb://localhost:27017 \
  users-service
```

## Verify

```bash
curl http://localhost:3000/users
```

Should return `{"service":"users-service","count":0,"users":[]}` (after a fresh Mongo).

## Things to notice

- Without `.dockerignore`, your local `node_modules` gets baked into the image — slow and wrong (host vs container architecture mismatch).
- Without the `package*.json`-before-source ordering, every code change re-runs `npm install`.
- `host.docker.internal` only exists on Docker Desktop. Linux uses `--network host` or the docker bridge gateway IP.

## Solution reference

```bash
git diff solution -- services/users-service/Dockerfile
```

---

# Milestone 2: Two services, one Docker network

**Goal:** Run both services in containers, on a shared user-defined network so they can talk to each other by name. Mongo still on host.

## Steps

1. Write the matching `Dockerfile` and `.dockerignore` for `services/products-service` — identical to users-service except the directory.
2. Build both images:

```bash
docker build -t users-service ./services/users-service
docker build -t products-service ./services/products-service
```

3. Create a user-defined bridge network. **This is the key concept:** containers on the default bridge can't resolve each other by name; on a user-defined network, they can.

```bash
docker network create app-net
```

4. Run both containers, attached to `app-net`, with explicit `--name`:

```bash
docker run -d --rm --name users-service --network app-net -p 3000:3000 \
  -e MONGO_URI=mongodb://host.docker.internal:27017 users-service

docker run -d --rm --name products-service --network app-net -p 3001:3000 \
  -e MONGO_URI=mongodb://host.docker.internal:27017 products-service
```

## Verify

```bash
docker exec users-service env
curl http://localhost:3000/users
curl http://localhost:3001/products
```

Cross-container DNS — from inside `users-service`, can we reach the other service by name?

```bash
docker exec users-service node -e "require('http').get('http://products-service:3000/products', r => r.pipe(process.stdout))"
```

The last command proves containers on the same user-defined network resolve each other by `--name`.

## Things to notice

- Try the same command on the **default** bridge network (omit `--network app-net`). It fails: default-bridge containers can only talk via IP, not name.
- Both services bind container port 3000, but you publish them on different host ports (`-p 3000:3000` and `-p 3001:3000`). Inside the network they're both `:3000` — that's normal.

## Cleanup before next milestone

```bash
docker rm -f users-service products-service
docker network rm app-net
```

---

# Milestone 3: Replace `docker run` with Compose

**Goal:** Capture everything from Milestone 2 in a single `docker-compose.yml` so `docker compose up` replaces multiple `docker run` commands.

## Steps

1. Create `docker-compose.yml` at the repo root with two services: `users-service` and `products-service`. Each:
    - `build:` its own subdirectory
    - Sets `MONGO_URI` via `environment:`
    - Joins a shared network called `app`
    - Publishes its port (3000 and 3001)
2. Define the `app` network at the bottom of the file with `driver: bridge`.
3. Bring it up:

```bash
docker compose up --build
```

## Verify

```bash
docker compose ps
curl http://localhost:3000/users
curl http://localhost:3001/products
docker compose logs users-service
```

The logs should show: `[users-service] connected to mongo`.

## Things to notice

- Compose creates the network automatically (`<project>_app`) and attaches containers to it.
- Service names from the YAML become container hostnames — `products-service` can reach `users-service` by name with no extra config.
- `docker compose down` cleans up containers and network in one shot.

## Solution reference

```bash
git checkout solution -- docker-compose.yml
```

> The solution version has more than just these two services — peek at the network section only for now.

---

# Milestone 4: Mongo in containers, one DB per service

**Goal:** Stop depending on host-installed Mongo. Each service gets its own MongoDB container, with persistent storage.

## Steps

1. Add two more services to `docker-compose.yml`: `users-db` and `products-db`. Both use the `mongo:7` image.
2. Give each Mongo a **named volume** so data survives `docker compose down`:

```yaml
volumes:
  - users-db-data:/data/db
```

Declare the volumes at the bottom of the file.

3. Update each backend service's `MONGO_URI` to point at its DB container by name:
    - `users-service` → `mongodb://users-db:27017`
    - `products-service` → `mongodb://products-db:27017`
4. Add `depends_on:` so each service waits for its DB to start.

> **Important:** `depends_on` only waits for container **start**, not for Mongo to be **ready** to accept connections. The provided `server.js` already retries Mongo for ~20 seconds at startup, so this is fine — but be aware of it.

## Verify

```bash
docker compose up -d --build
docker compose ps

curl -X POST http://localhost:3000/users \
  -H 'content-type: application/json' \
  -d '{"name":"Ada","email":"a@b.c"}'

docker compose down
docker compose up -d
curl http://localhost:3000/users
```

Ada should still be there after the restart.

## Things to notice

- Each service can resolve its own DB by hostname (`users-db`, `products-db`) but not the other service's DB — well, actually it *can* right now, because they're all on one shared network. You'll fix that in Milestone 6.
- `docker compose down -v` (with `-v`) wipes volumes too. Use that to reset to a clean state.

## Solution reference

```bash
git diff solution -- docker-compose.yml
```

---

# Milestone 5: Nginx as an API gateway + the frontend

**Goal:** Add the Nginx load balancer (the LB) that routes `/users/*` and `/products/*` to the right service, and serve the frontend through its own Nginx that proxies `/api/*` to the LB. The browser then talks to **one origin** (no CORS issues).

## Steps

1. **Read** `lb/nginx.conf` — it's already written for you. It uses `upstream` blocks to route paths to service names. You'll just dockerize it.
2. Add an `lb` service to `docker-compose.yml`:

```yaml
lb:
  image: nginx:alpine
  volumes:
    - ./lb/nginx.conf:/etc/nginx/nginx.conf:ro
  depends_on:
    - users-service
    - products-service
```

No port published — the LB is internal. The frontend will reach it by name.

3. **Read** `frontend/nginx.conf` and `frontend/public/index.html`. The frontend HTML fetches `/api/users` and `/api/products`. The nginx config strips `/api/` and forwards to the LB.
4. Create `frontend/Dockerfile`:

```docker
FROM nginx:alpine
COPY nginx.conf /etc/nginx/nginx.conf
COPY public/ /usr/share/nginx/html/
EXPOSE 80
```

5. Add the `frontend` service to compose. **This** is the one that publishes a port to the host:

```yaml
frontend:
  build: ./frontend
  ports:
    - "8080:80"
  depends_on:
    - lb
```

6. Now **remove** the `ports:` mappings from `users-service` and `products-service` — nothing on the host needs them directly anymore.

## Verify

```bash
docker compose up -d --build
docker compose ps
```

Open `http://localhost:8080` in a browser. You should see the polished UI. Add a user, add a product, try the buy button.

From the command line:

```bash
curl http://localhost:8080/api/users
curl http://localhost:8080/api/products
```

## Things to notice

- The LB has no host port published — you can't reach it from outside Docker. The browser hits the **frontend**, which proxies to the LB internally.
- Watch the request path: browser → frontend nginx (strips `/api/`) → lb nginx (routes by `/users` or `/products`) → service → mongo. Four network hops.
- If you try `curl http://localhost:8080/users` (without `/api/`), it returns the HTML page — because frontend nginx only proxies the `/api/` path.

## Solution reference

```bash
git diff solution -- docker-compose.yml frontend/Dockerfile
```

---

# Milestone 6: Network segmentation

**Goal:** Right now all 6 containers share one network. That means `users-service` can reach `products-db` directly, and the frontend could (if it knew the hostname) bypass the LB entirely. Real microservices enforce isolation at the network layer. You'll split the single `app` network into **four**.

## The target topology

| Network | Members | Notes |
| --- | --- | --- |
| `public-net` | frontend, lb | The only network with a host port published |
| `edge-net` | lb, users-service, products-service | `internal: true` — gateway-to-services |
| `users-net` | users-service, users-db | `internal: true` — private DB segment |
| `products-net` | products-service, products-db | `internal: true` — private DB segment |

The **lb** is the only container on multiple networks (`edge-net` + `public-net`). The services bridge `edge-net` (to receive traffic from lb) and their own DB net. The DBs touch only their service's net.

## Steps

1. Replace the single `app` network at the bottom of `docker-compose.yml` with four:

```yaml
networks:
  users-net:
    driver: bridge
    internal: true
  products-net:
    driver: bridge
    internal: true
  edge-net:
    driver: bridge
    internal: true
  public-net:
    driver: bridge
```

> `internal: true` means containers on this network **cannot reach the public internet**. Only `public-net` needs outbound — that's where the frontend lives.

2. Update each service's `networks:` list to match the topology above.

## Verify

```bash
docker compose down -v
docker compose up -d --build
```

Smoke test the API as before — everything should still work.

Then prove the isolation. **users-service cannot see products-db** (different network — DNS doesn't even resolve):

```bash
docker compose exec users-service node -e "require('dns').lookup('products-db', (e,a) => console.log(e||a))"
# Expect: ENOTFOUND
```

**frontend cannot see users-service directly:**

```bash
docker compose exec frontend sh -c "nc -zvw2 users-service 3000"
# Expect: bad address
```

**But lb can see both services** (it's on edge-net):

```bash
docker compose exec lb sh -c "nc -zvw2 users-service 3000 && nc -zvw2 products-service 3000"
# Expect: open ... open
```

This is real isolation: Docker's DNS resolver only resolves names of containers on **shared** networks. Stronger than a firewall — the name literally doesn't exist.

## Things to notice

- `internal: true` blocks outbound internet. If you later add a webhook from `products-service` to an external API, you'll have to drop `internal` from `edge-net` or add an egress network.
- The build-time `npm install` still works because builds use the Docker daemon's default network, not your compose networks.

## Solution reference

```bash
git diff solution -- docker-compose.yml
```

---

# Milestone 7: Distroless runtime images

**Goal:** Replace the `node:20-alpine` runtime with **Google's distroless image**: no shell, no package manager, runs as non-root. The build still uses alpine — but the final image you ship has only the Node binary, minimal libs, and your code.

## Why

| Benefit | What it means |
| --- | --- |
| Smaller attack surface | No `sh`, `apk`, `curl`, `wget` for an attacker to leverage if they break out of your app |
| Forced security baseline | The `:nonroot` tag runs as UID 65532, not root |
| CVE reduction | Fewer binaries means fewer CVEs to track |

## Steps

Convert `services/users-service/Dockerfile` to a multi-stage build:

```docker
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm install --omit=dev
COPY server.js ./

FROM gcr.io/distroless/nodejs20-debian12:nonroot
WORKDIR /app
COPY --from=build --chown=nonroot:nonroot /app /app
USER nonroot
EXPOSE 3000
CMD ["server.js"]
```

> **Key change:** `CMD ["server.js"]` not `CMD ["node", "server.js"]` — the distroless nodejs image has `node` as its `ENTRYPOINT`, so it appends whatever you put in `CMD`.

Do the same for `products-service`.

## Verify

```bash
docker compose up -d --build
curl http://localhost:8080/api/users
```

Confirm the distroless promise — there's no shell:

```bash
docker compose exec users-service sh
# Expect: OCI runtime exec failed: ... "sh": executable file not found
```

Confirm the non-root user:

```bash
docker inspect $(docker compose ps -q users-service) --format '{{.Config.User}}'
# Expect: nonroot
```

## Things to notice

- You can no longer `docker exec ... sh` into the container — that's the point. Debug via logs, or temporarily swap the image for an alpine variant.
- The image is ~190 MB (mostly distroless base + node_modules). For comparison, the alpine-only version was ~150 MB. Distroless trades some size for a much smaller attack surface — the right tradeoff for runtime images.

## Solution reference

```bash
git diff solution -- services/users-service/Dockerfile services/products-service/Dockerfile
```

---

# Final state

After Milestone 7 your tree should look like:

```
.
├── docker-compose.yml
├── frontend/
│   ├── Dockerfile
│   ├── nginx.conf
│   └── public/index.html
├── lb/
│   └── nginx.conf
└── services/
    ├── users-service/
    │   ├── Dockerfile
    │   ├── .dockerignore
    │   ├── server.js
    │   └── package.json
    └── products-service/
        ├── Dockerfile
        ├── .dockerignore
        ├── server.js
        └── package.json
```

**6 containers, 4 networks, 2 named volumes, distroless backends.**

To compare against the reference:

```bash
git diff solution           # see everything that differs
git diff solution --stat    # just the file list
```

---

# Where to go next

- **CI:** build images in GitHub Actions, push to a registry, tag by git SHA
- **Healthchecks:** add `HEALTHCHECK` to backends so `depends_on: condition: service_healthy` becomes possible — and drop the retry loop in `server.js`
- **Secrets:** store `MONGO_URI` credentials via Docker secrets instead of environment variables
- **Multi-replica:** scale `users-service` with `docker compose up --scale users-service=3` and watch nginx round-robin
- **Observability:** add an ELK or Loki sidecar to aggregate logs across containers
