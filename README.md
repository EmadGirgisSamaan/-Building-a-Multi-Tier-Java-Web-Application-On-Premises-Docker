# vprofile-docker: Multi-Tier Java Web Application with Docker Compose

A containerized version of the **vprofile** multi-tier Java web application, run with Docker Compose.
Five services (Nginx, Tomcat, MariaDB, Memcached, RabbitMQ) start with a single command, with frontend/backend
network isolation, health checks, and persistent database storage.

> The application source code is the open-source [vprofile project](https://github.com/hkhcoder/vprofile-project)
> by **hkhcoder**. It is cloned and built at image build time. This repository contains only the infrastructure:
> the Compose file, Dockerfiles, and Nginx configuration.

## Architecture

```
                      frontend network              backend network
                   ┌──────────────────────┐   ┌─────────────────────────────┐
 Browser ──:80──►  │  Nginx (reverse proxy) │   │                             │
                   └──────────┬───────────┘   │   ┌──────────────────────┐  │
                              │               │   │ MariaDB      :3306   │  │
                              ▼               │   ├──────────────────────┤  │
                   ┌──────────────────────┐   │   │ Memcached    :11211  │  │
                   │ Tomcat 9 + Java 11   │───┼──►├──────────────────────┤  │
                   │ (App, port 8080)     │   │   │ RabbitMQ     :5672   │  │
                   └──────────────────────┘   │   └──────────────────────┘  │
                  (attached to both networks) └─────────────────────────────┘
```

- **Nginx** is the only service published to the host (port 80).
- **Tomcat** is attached to both networks, so it is reachable from Nginx and can reach the backend services.
- **MariaDB, Memcached, RabbitMQ** live on the backend network only, so Nginx cannot reach them directly.

## Services

| Service   | Image                                   | Port (inside Docker) | Role                                   |
|-----------|-----------------------------------------|----------------------|----------------------------------------|
| nginx     | custom, based on `nginx:stable`         | 80 (published)       | Reverse proxy, single entry point      |
| app       | custom, multi-stage (Maven, then `tomcat:9.0-jdk11-temurin`) | 8080 | Spring MVC application |
| db        | `mariadb:10.11`                         | 3306                 | Database with a persistent volume      |
| memcached | `memcached:alpine`                      | 11211                | Caching layer                          |
| rabbitmq  | `rabbitmq:3-management`                 | 5672 (AMQP), 15672 (management UI) | Message broker           |

The backend services have network aliases (`db01`, `mc01`, `rmq01`) so the application finds them by the hostnames
it expects, without changing its code.

## Key Features

- Health checks for MariaDB and RabbitMQ, so the application starts only when its dependencies are ready
- Separate frontend/backend Docker networks
- Persistent volume for the database
- Multi-stage Dockerfile: Maven builds the WAR, and only the result is copied into the Tomcat image
- Secrets kept out of the Compose file, loaded from a `.env` file
- Database schema and sample data loaded automatically on first start

## Project Structure

```
vprofile-docker/
├── docker-compose.yml
├── .env.example          # copy to .env and adjust
├── .gitignore            # ignores .env
├── app/
│   └── Dockerfile        # Maven build, then Tomcat runtime
├── nginx/
│   ├── Dockerfile
│   └── nginx.conf        # reverse proxy to the app
└── db/
    └── db_backup.sql     # initial data (see Quick Start)
```

## Prerequisites

- Docker Engine and the Docker Compose plugin (`docker --version`, `docker compose version`)
- Internet access for the first build (pulls images and Maven dependencies)

## Quick Start

```bash
# 1. Clone the repository
git clone https://github.com/EmadGirgisSamaan/-Building-a-Multi-Tier-Java-Web-Application-On-Premises-Docker.git vprofile-docker
cd vprofile-docker

# 2. Create your environment file
cp .env.example .env

# 3. Download the initial database data (skip if db/db_backup.sql already exists)
mkdir -p db
curl -L -o db/db_backup.sql \
  https://raw.githubusercontent.com/hkhcoder/vprofile-project/main/src/main/resources/db_backup.sql

# 4. Build and start everything
docker compose up -d --build
```

The first build takes several minutes because Maven downloads the project's dependencies. Later builds use the cache.

Then open:

- On the same machine: `http://localhost`
- From another machine: `http://<server-ip>`

## Useful Commands

| Command | Purpose |
|---------|---------|
| `docker compose ps` | Service status (`db` and `rabbitmq` should become `healthy`) |
| `docker compose logs -f app` | Follow the application logs |
| `docker compose up -d --build app` | Rebuild and restart only the app |
| `docker compose restart app` | Restart a service |
| `docker compose exec app bash` | Open a shell inside the app container |
| `docker compose down` | Stop and remove containers (data is kept) |
| `docker compose down -v` | Also delete the database volume |

The SQL file in `db/` is loaded only when the database volume is empty. To reload it, run `docker compose down -v` first.

## Troubleshooting

| Problem | Likely cause and fix |
|---------|----------------------|
| `variable is not set` warnings | `.env` is missing. Run `cp .env.example .env` |
| `build context ... not found` | The `app/` or `nginx/` folder is missing or empty |
| `curl http://localhost` prints nothing | The app redirects `/` to `/login`. Use `curl -L http://localhost` or open it in a browser |
| App cannot connect to the database | Run `docker compose ps` and wait until `db` is `healthy`. Check that `DB_USER` and `DB_PASSWORD` in `.env` match the application's configuration |
| `502 Bad Gateway` | The app is still starting or crashed. Check `docker compose logs app` |
| Port 80 already in use | Change `"80:80"` to `"8081:80"` in `docker-compose.yml` |

## Default Credentials (lab use only)

| What | Value |
|------|-------|
| Application admin user | `admin_vp` / `admin123` |
| Regular user | Register through the `/registration` page |
| Database / RabbitMQ | Values from your `.env` (defaults in `.env.example`) |

These are the demo values of the vprofile project. **Never reuse them anywhere real.**

## Docker Compose vs Vagrant

I first built this same stack with Vagrant (5 VMs provisioned with shell scripts), then rebuilt it with Docker Compose.

| | Vagrant (5 VMs) | Docker Compose (5 containers) |
|---|---|---|
| Isolation | Full virtual machines | Containers sharing the host kernel |
| Resource usage | Higher: each VM runs a full OS | Lower |
| Setup | Shell scripts install each service | Ready-made images plus two small Dockerfiles |
| Networking | Private network with fixed IPs, hostmanager | Docker networks with DNS by service name |
| Portability | Needs VirtualBox and Vagrant | Needs Docker only |

## Learning Journey

- **Part 1:** Vagrant multi-tier setup: 5 VMs with shell provisioning
- **Part 2:** Docker Compose (this repository): the containerized version
- **Part 3:** Ansible automation (planned)

## Credits

- Application: [hkhcoder/vprofile-project](https://github.com/hkhcoder/vprofile-project)
- Infrastructure (Compose file, Dockerfiles, Nginx configuration): Emad Girgis Samaan

## License

The infrastructure files in this repository are released under the MIT License (see `LICENSE`).
The vprofile application keeps its own license from its original author.

## Author

**Emad Girgis Samaan**: IT Manager with 25+ years in infrastructure and networks, moving into DevOps.
Open to remote DevOps roles (EU/NL).

Feedback and suggestions are welcome. Feel free to open an issue or a pull request.
