# Microservices Demo

A minimal but realistic microservices demo: two backend services with **distinct business logic** and **independent databases**, behind an Nginx API gateway, with a static frontend.

## Architecture (6 containers, 4 segmented networks)

```
                       ┌─────────────────────────────────┐
            Browser ──►│ frontend (nginx :8080)          │
                       └──────────────┬──────────────────┘
                                      │   public-net
                       ┌──────────────▼──────────────────┐
                       │ lb (nginx) — API gateway        │
                       │   /users/*    → users-service   │
                       │   /products/* → products-service│
                       └──┬────────────────────────────┬─┘
                          │       edge-net             │
              ┌───────────▼──────────┐    ┌────────────▼─────────┐
              │ users-service        │    │ products-service     │
              └───────────┬──────────┘    └────────────┬─────────┘
                          │ users-net (internal)       │ products-net (internal)
              ┌───────────▼──────────┐    ┌────────────▼─────────┐
              │ users-db (mongo)     │    │ products-db (mongo)  │
              └──────────────────────┘    └──────────────────────┘
```

### Why 4 networks?

| Network       | Members                                    | Purpose                                                    |
|---------------|--------------------------------------------|------------------------------------------------------------|
| `public-net`  | `frontend`, `lb`                           | The only network reachable from the host (8080 → frontend) |
| `edge-net`    | `lb`, `users-service`, `products-service`  | Where the API gateway reaches the services. `internal`     |
| `users-net`   | `users-service`, `users-db`                | Private DB segment for users. `internal`                   |
| `products-net`| `products-service`, `products-db`          | Private DB segment for products. `internal`                |

The `lb` is the **only** container that bridges multiple segments. Direct consequences:

- The two **services cannot reach each other's database** — Docker DNS only resolves names of containers on shared networks, so `users-service` literally can't see `products-db`.
- The **frontend cannot bypass the LB** to call services directly.
- **DBs have no internet access** (their networks are `internal: true`).
- The two services **cannot talk to each other** directly — any cross-service call must go through the LB.

That's the microservices isolation story enforced at the network layer, not just by convention.

## Services

### users-service
- `GET    /users`         — list all users
- `POST   /users`         — `{ name, email }`
- `DELETE /users/:id`

### products-service
- `GET    /products`              — list all products
- `POST   /products`              — `{ name, price, stock? }`
- `POST   /products/:id/buy`      — atomic stock decrement (returns 409 if out of stock)
- `DELETE /products/:id`

The gateway exposes them as `/api/users/*` and `/api/products/*` to the frontend (same origin, no CORS).

## Run

```sh
docker compose up --build
```

Open <http://localhost:8080>.

- Add users on the left, add products on the right.
- "Buy" decrements stock atomically — try buying until you hit out-of-stock.
- The bottom log shows every API call with the service that handled it.

## Try it from the command line

```sh
# users
curl -X POST http://localhost:8080/api/users \
  -H 'content-type: application/json' \
  -d '{"name":"Ada","email":"ada@example.com"}'
curl http://localhost:8080/api/users

# products
curl -X POST http://localhost:8080/api/products \
  -H 'content-type: application/json' \
  -d '{"name":"Widget","price":9.99,"stock":3}'
curl http://localhost:8080/api/products
```

## Demonstrate service independence

Stop one service while the app is running:

```sh
docker compose stop products-service
```

The users panel keeps working; the products panel fails with a 502. Bring it back:

```sh
docker compose start products-service
```

## Tear down

```sh
docker compose down              # keep db volumes
docker compose down -v           # also wipe mongo data
```

## Layout

```
.
├── docker-compose.yml
├── lb/
│   └── nginx.conf                 # API gateway: path-based routing
├── services/
│   ├── users-service/             # Express + MongoDB driver
│   │   ├── server.js
│   │   ├── package.json
│   │   └── Dockerfile
│   └── products-service/          # Express + MongoDB driver
│       ├── server.js
│       ├── package.json
│       └── Dockerfile
└── frontend/
    ├── public/index.html
    ├── nginx.conf                 # serves static, proxies /api → lb
    └── Dockerfile
```

## Container images

Backend services use a two-stage build:

- **build stage** — `node:20-alpine` runs `npm install`
- **runtime stage** — `gcr.io/distroless/nodejs20-debian12:nonroot` ships only the Node binary, minimal libs, and the app — **no shell, no package manager**, running as the `nonroot` user (UID 65532).

You can verify there's no shell:

```sh
docker compose exec users-service sh   # → "executable file not found in $PATH"
```

The nginx containers (lb, frontend) stay on `nginx:alpine` since they're not running our code and alpine is already minimal.

## Not in scope

No auth, no HTTPS, no tests, no service-to-service calls. Pure architectural demo.
