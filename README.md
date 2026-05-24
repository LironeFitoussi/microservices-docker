# Microservices Demo

A minimal but realistic microservices demo: two backend services with **distinct business logic** and **independent databases**, behind an Nginx API gateway, with a static frontend.

## Architecture (6 containers)

```
                          ┌────────────────────────────┐
            Browser ────► │ frontend (nginx :8080)     │
                          └──────────────┬─────────────┘
                                         │  /api/*
                                         ▼
                          ┌────────────────────────────┐
                          │ lb (nginx) — API gateway   │
                          │  /users   → users-service  │
                          │  /products → products-svc  │
                          └──────┬───────────────┬─────┘
                                 ▼               ▼
                       ┌─────────────────┐  ┌──────────────────┐
                       │ users-service   │  │ products-service │
                       │ Express :3000   │  │ Express :3000    │
                       └────────┬────────┘  └────────┬─────────┘
                                ▼                    ▼
                       ┌─────────────────┐  ┌──────────────────┐
                       │ users-db        │  │ products-db      │
                       │ MongoDB :27017  │  │ MongoDB :27017   │
                       └─────────────────┘  └──────────────────┘
```

Each service owns its data — they cannot reach each other's database. That's the core microservices invariant this demo enforces.

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

## Not in scope

No auth, no HTTPS, no tests, no service-to-service calls. Pure architectural demo.
