# Microservices Lab — Starter

This is the **starter** for a hands-on lab on dockerizing a microservices app.

You're given a working application — two backend services, an Nginx config for a load balancer, and a static frontend — but **no Docker files**. Your task is to dockerize it step by step.

➡️ **[Open LAB.md](./LAB.md) and start at Milestone 0.**

The complete reference implementation lives on the `solution` branch:

```sh
git checkout solution                          # browse the full solution
git checkout solution -- path/to/file          # grab one file
git diff solution -- path/to/file              # compare yours to the solution
```

## The app

Two independent services, each meant to own its own MongoDB:

- **`users-service`** (`services/users-service/`) — manages users (name, email)
- **`products-service`** (`services/products-service/`) — manages products with atomic stock decrement (`POST /products/:id/buy`)

Plus the pieces you'll wire into Docker through the lab:

- **`lb/nginx.conf`** — API gateway config: routes `/users/*` to the users service, `/products/*` to the products service
- **`frontend/`** — static HTML/CSS/JS app that calls `/api/users` and `/api/products`. Designed to be served by Nginx, with `/api/*` reverse-proxied to the LB

## Project layout

```
.
├── LAB.md                       ← start here
├── frontend/
│   ├── nginx.conf
│   └── public/index.html
├── lb/
│   └── nginx.conf
└── services/
    ├── users-service/{server.js, package.json}
    └── products-service/{server.js, package.json}
```
