+++
draft = false
date = '2026-03-14T21:22:30+07:00'
title = 'Devstack'
type = 'project'
description = 'A technical look at building a standardized local development infrastructure — managing PostgreSQL, MariaDB, RabbitMQ, Redis, and more through a single Docker Compose file with persistent storage and custom service images.'
image = ''
repository = 'https://github.com/mnabila/devstack'
languages = ['dockerfile']
tools = ['docker', 'docker-compose', 'podman', 'postgresql', 'mariadb', 'rabbitmq', 'redis', 'portainer', 'n8n']
+++

Every developer eventually faces the same problem: "it works on my machine, but I need a database, a message broker, a cache layer, and three other services running before I can even start coding." Setting up local infrastructure should not be the thing that slows you down on day one of a project.

Devstack is a single Docker Compose file that provisions an entire local development infrastructure — databases, message brokers, caching, workflow automation, and container management — with one command. It is the foundation I use across all my projects.

## Problem Background

Working across multiple projects often means dealing with different combinations of infrastructure services. One project needs PostgreSQL and Redis. Another needs MariaDB and RabbitMQ with the delayed message exchange plugin. A third needs all of the above plus a workflow automation tool.

The typical approach is ad-hoc: spin up individual containers, remember the port mappings, manage credentials in your head, and hope nothing conflicts. Over time this leads to a few recurring problems:

- **Port conflicts** between services from different projects
- **Inconsistent credentials** that waste debugging time
- **Lost data** when containers are removed without persistent volumes
- **Missing plugins** or custom configurations that are not captured anywhere reproducible
- **Onboarding friction** when a new team member needs to replicate the same setup

I needed a single, declarative source of truth for all the infrastructure services I use in development — something I could `docker compose up -d` and forget about.

## Solution Overview

Devstack consolidates all commonly used development infrastructure into a single `docker-compose.yml` with sensible defaults, persistent storage, and standardized credentials. It is designed to be started once and left running, serving as the shared backend for any project on the machine.

**Tech stack:** Docker / Podman, Docker Compose, PostgreSQL, MariaDB, RabbitMQ, Redis, Portainer, n8n

**My role:** Sole developer — service selection, configuration, custom image builds, and volume management

## System Architecture

The architecture is deliberately simple: a flat Docker Compose file that defines six services sharing a common bridge network. Each service exposes its standard ports to the host, and persistent data is stored under a local `data/` directory (gitignored) to survive container restarts and rebuilds.

```
.
├── data/                    # Persistent volumes (gitignored)
│   ├── portainer/
│   ├── postgresql/
│   ├── mariadb/
│   ├── rabbitmq/
│   └── n8n/
├── services/
│   └── rabbitmq/
│       └── Dockerfile       # Custom RabbitMQ image with delayed message exchange
├── docker-compose.yml       # The entire stack definition
└── README.md
```

The service definitions follow a consistent pattern: official images where possible, environment variables for configuration, host-mounted volumes for persistence, and explicit port mappings.

| Service    | Image                            | Ports            | Purpose                                          |
|------------|----------------------------------|------------------|--------------------------------------------------|
| Portainer  | `portainer/portainer-ce`         | `9443`, `8000`   | Web-based container management UI                |
| PostgreSQL | `postgres:17`                    | `5432`           | Primary relational database                      |
| MariaDB    | `mariadb:lts`                    | `3306`           | MySQL-compatible relational database             |
| RabbitMQ   | Custom build (see below)         | `5672`, `15672`  | Message broker with delayed message exchange     |
| Redis      | `redis:8.2`                      | `6379`           | In-memory key-value store and cache              |
| n8n        | `n8nio/n8n:latest`               | `5678`           | Visual workflow automation tool                  |

All services are connected via a shared bridge network, allowing inter-service communication by container name — useful when n8n workflows need to query PostgreSQL or when application services within the same Compose context need to reach the broker.

## Key Features

- **Single-command startup** — `docker compose up -d` provisions the entire infrastructure stack. Individual services can be started selectively with `docker compose up -d postgresql redis`
- **Persistent storage** — all service data is mounted to the local `data/` directory, so nothing is lost when containers are stopped, removed, or rebuilt. The directory is gitignored to keep the repository clean
- **Custom RabbitMQ image** — a purpose-built Dockerfile extends the official `rabbitmq:4.2-management` image to include the [delayed message exchange plugin](https://github.com/rabbitmq/rabbitmq-delayed-message-exchange), which is not bundled by default but is commonly needed for scheduling and retry patterns
- **Standardized credentials** — every service uses simple, consistent default credentials documented in the README, eliminating the "what was the password again?" problem during development
- **Podman compatibility** — the Portainer configuration mounts the Podman socket (`/run/user/1000/podman/podman.sock`), making the stack compatible with rootless Podman as a Docker alternative
- **Timezone consistency** — services that support it (MariaDB, n8n) are configured with `Asia/Jakarta` timezone to match the development locale, preventing timestamp confusion during debugging

## Technical Challenges and Solutions

**RabbitMQ delayed message exchange.** The official RabbitMQ Docker image does not include the delayed message exchange plugin, which is essential for scheduling delayed retries, deferred task processing, and time-based routing patterns. The solution was a custom Dockerfile that downloads the plugin from the official GitHub releases and enables it at build time. This ensures the plugin is always available without manual post-setup steps:

```dockerfile
FROM rabbitmq:4.2-management

RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

RUN mkdir -p /opt/rabbitmq/plugins && \
  curl -L -o /opt/rabbitmq/plugins/rabbitmq_delayed_message_exchange.ez \
  https://github.com/rabbitmq/rabbitmq-delayed-message-exchange/releases/download/v4.2.0/rabbitmq_delayed_message_exchange-4.2.0.ez

RUN rabbitmq-plugins enable rabbitmq_delayed_message_exchange
```

**Data persistence across container lifecycle.** By default, Docker containers lose all data when removed. Bind-mounting each service's data directory to the host ensures that database contents, broker state, and workflow definitions survive `docker compose down` and `docker compose up` cycles. The trade-off is that the `data/` directory can grow large over time, but for local development this is acceptable and the gitignore ensures it never gets committed.

**Podman rootless compatibility.** Running Portainer with Podman requires mounting the user-level Podman socket instead of the Docker daemon socket. The `security_opt: label=disable` directive disables SELinux label confinement, which is necessary for Podman socket access in rootless mode. This is a development-only trade-off — in production, you would configure this more tightly.

**Service isolation vs. convenience.** A design decision was whether to put each service in its own Compose file for maximum isolation or consolidate everything into a single file for convenience. I chose the single-file approach because in practice, the overhead of managing multiple Compose files, remembering which one to start, and coordinating networks between them outweighed the isolation benefits for a local development context.

## Lessons Learned

**Declarative infrastructure eliminates drift.** When your development infrastructure is defined in a single file, there is exactly one place to look when something is wrong. No more "I think I changed that setting last week but I'm not sure where." The Compose file is the source of truth, and `docker compose up -d` always produces the same result.

**Custom images are worth the effort.** Building a custom RabbitMQ image added maybe 15 minutes of work upfront but saved hours of repeated manual plugin installation across machines and project setups. Any service that needs a non-default configuration should be captured in a Dockerfile, not in post-setup documentation.

**Standardize your defaults early.** Choosing consistent, simple credentials across all services seems trivial, but it eliminates an entire category of "connection refused" and "authentication failed" debugging sessions. When every service uses known defaults, the mental overhead drops to zero.

**Keep development infrastructure separate from project infrastructure.** Devstack runs independently from any specific project. Multiple projects can share the same PostgreSQL or Redis instance (using different databases), or a project can define its own Compose file that connects to Devstack's network. This separation prevents infrastructure concerns from cluttering project repositories.

## Conclusion

Devstack is not a complex project — and that is the point. It is a deliberate investment in reducing the friction between "I want to start working" and "I am actually working." One file, one command, and the entire infrastructure layer is ready.

```bash
# Start everything
docker compose up -d

# Start only what you need
docker compose up -d postgresql redis

# Check status
docker compose ps

# View logs
docker compose logs -f [service]

# Tear it down (data persists)
docker compose down
```

| Service    | Username   | Password   |
|------------|------------|------------|
| PostgreSQL | `postgres` | `postgres` |
| MariaDB    | `root`     | `toor`     |
| RabbitMQ   | `admin`    | `admin`    |
| n8n        | `admin`    | `admin`    |

The full source is available on [GitHub](https://github.com/mnabila/devstack). Fork it, swap out services for the ones you actually use, and make it your own.
