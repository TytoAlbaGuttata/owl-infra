# Owl-infra: Personal self-hosted tools

Welcome to the infrastructure repository for my personal self-hosted cloud environment. This repository contains the Infrastructure as Code (IaC) required to deploy and manage a secure, containerized set of services using Docker Compose.

## Software Stack

This infrastructure relies on a modern, open-source stack designed for security, privacy, and performance:

* **Network and ingress:**
  *  [Cloudflared](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) (Secure outbound tunnel, zero open ports)
  *  [Traefik v3](https://traefik.io/traefik/) (Dynamic Reverse Proxy)
  *  [AdGuard Home](https://adguard.com/en/adguard-home/overview.html) (Network-wide DNS ad and tracker blocker)

* **Identity and access management:**
   * [Authentik](https://goauthentik.io/) (Centralized SSO, SAML, and OIDC provider)

* **Services:**
  * [Nextcloud AIO](https://github.com/nextcloud/all-in-one) (Personal cloud, file sync, and collaboration). *Note: Hosted on a dedicated Intel Mac, with traffic dynamically routed via Traefik*.
  * [Vaultwarden](https://github.com/dani-garcia/vaultwarden) (Self-hosted password manager)
  * [Vikunja](https://vikunja.io/) (Task management and to-do application)
  * [Actual Budget](https://actualbudget.org/) (Local-first personal finance system)
  * [Navidrome](https://www.navidrome.org/) (Music server and streamer)
  * [Audiobookshelf](https://www.audiobookshelf.org/) (Audiobook and podcast server)
  * [SonarQube](https://www.sonarsource.com/products/sonarqube/) (Continuous Code Quality and Security Analysis)
  * [GitHub Stats](https://github.com/stats-organization/github-stats-extended) (GitHub statistics dashboard)

* **Monitoring and observability:**
  * [Prometheus](https://prometheus.io/) (Time-series database for metrics)
  * [Grafana](https://grafana.com/) (Visualization and dashboards)
  * [Node-Exporter](https://github.com/prometheus/node_exporter) (Hardware and OS metrics)
  * [cAdvisor](https://github.com/google/cadvisor) (Container resource usage)
  * [Grafana Loki](https://grafana.com/oss/loki/) (Log aggregation system)
  * [Grafana Alloy](https://grafana.com/oss/alloy/) (Log collector and pipeline)

## Hardware

This infrastructure runs across two distinct physical nodes to balance lightweight services and heavier storage/compute workloads:

* **Node 1 (Core Infra and services):**
  * **Host:** Raspberry Pi 5
  * **Storage:** NVMe SSD (good for database performance and Elasticsearch stability required by SonarQube)
  * **OS:** Linux (Raspberry Pi OS lite) with modified kernel parameters (`vm.max_map_count=524288`) for Elasticsearch.

* **Node 2 (Heavy workloads and storage):**
  * **Host:** Mac Intel (Late 2019)
  * **OS:** Ubuntu Server 26.04 LTS
  * **Role:** Dedicated host for heavier workloads like Nextcloud, dynamically routed via the primary node's reverse proxy.


## Repository structure and secrets management

To maintain absolute security and prevent secret leakage, this repository follows a strict `env` variable philosophy. **No passwords or API keys are hardcoded in this repository.**

```mermaid
treeView-beta
owl-infra/
├── docker-compose.yml                ## Cloudflare tunnel & Traefik
├── .env.example
├── adguard/                          ## DNS and ad-blocking
│   └── docker-compose.yml
├── audiobookshelf/                   ## Audiobook server
│   └── docker-compose.yml
├── authentik/                        ## SSO
│   ├── .env.example
│   └── docker-compose.yml
├── budget/                           ## Actual Budget
│   └── docker-compose.yml
├── dynamic/                          ## Traefik dynamic configurations
│   └── nextcloud-router.yml
├── github-stats/                     ## GitHub statistics
│   ├── .env.example
│   ├── Dockerfile
│   └── docker-compose.yml
├── monitoring/                       ## Observability stack
│   ├── .env.example
│   ├── alloy-config.alloy
│   ├── docker-compose.yml
│   ├── loki-config.yaml
│   └── prometheus.yml
├── navidrome/                        ## Music server
│   └── docker-compose.yml
├── sonarqube/                        ## Code analysis
│   ├── .env.example
│   └── docker-compose.yml
├── vaultwarden/                      ## Password manager
│   └── docker-compose.yml
└── vikunja/                          ## Task management
    ├── .env.example
    └── docker-compose.yml
```

> **Note:** A lot of directories contain a `.env.example` file. You must duplicate this file, rename it to `.env`, and fill in your secure passwords before deploying.

## Deployment guide

If you are restoring this infrastructure from scratch, follow these steps:

1. **Clone the repository:**
 ```bash
 git clone git@github.com:TytoAlbaGuttata/owl-infra.git
 cd owl-infra
 ```

2. **Prepare the network:** Create the external Docker network that allows the proxy to talk to the containers:

```bash
docker network create proxy-net
```

3. **Configure Secrets:**
Go into the root folder and each service folder, copy the example variables, and fill them with your secure data:
```bash
cp .env.example .env
cp authentik/.env.example authentik/.env
# Do the same with the other services: budget, github-stats, monitoring, sonarqube, vikunja.
```
*Edit each `.env` file with nano/vim to inject your production secrets.*
4. **Prepare persistent volumes:**
For security reasons, containers are heavily restricted. Loki runs as a non-root user (UID 10001). You must create its data directory and set the correct permissions before starting the monitoring stack to avoid crash loops:
```bash
mkdir -p monitoring/loki_data
sudo chown -R 10001:10001 monitoring/loki_data
```
*(Note: Most other services are mapped to user `1000:1000` which matches the default Pi user. Ensure your local storage directories have appropriate ownership).*
5. **Spin up the services:**
Launch the services folder by folder, respecting the dependency chain (Core Infra -> Auth -> Services -> Monitoring):
```bash
# 1. Core ingress and tunnel
docker compose up -d

# 2. Identity provider
cd authentik && docker compose up -d && cd ..

# 3. Development and Analytics
cd sonarqube && docker compose up -d && cd ..
cd github-stats && docker compose up -d && cd ..

# 4. Productivity and Privacy
cd vikunja && docker compose up -d && cd ..
cd budget && docker compose up -d && cd ..
cd vaultwarden && docker compose up -d && cd ..

# 5. Media and network services
cd navidrome && docker compose up -d && cd ..
cd audiobookshelf && docker compose up -d && cd ..
cd adguard && docker compose up -d && cd ..

# 6. Observability
cd monitoring && docker compose up -d && cd ..
```

> Note: Nextcloud is not started here as it is hosted on a dedicated Mac Intel node. Traefik will automatically start routing traffic to it using the configuration in `dynamic/nextcloud-router.yml`.


## Post-deployment configuration

This repository strictly handles the **container deployment**. It does not automatically configure the internal logic of the applications. Once the containers are running, you must manually link the services together using their respective administrative interfaces:

* **Authentik:** You will need to create Providers and Applications to enable SSO for all your services.
  * *See Docs:* [Authentik Application Integration](https://docs.goauthentik.io/docs/providers/)

### Identity and access integrations (OIDC / SAML)
These services require an Application and a Provider configured in Authentik, matching the credentials stored in their respective `.env` files:
* **Nextcloud:** Must be connected to Authentik via OpenID Connect (OIDC).
  * *See Docs:* [Nextcloud OIDC integration](https://docs.goauthentik.io/integrations/services/nextcloud/)
* **SonarQube:** Must be connected to Authentik via SAML, and the base URL must be updated in the settings.
  * *See Docs:* [SonarQube SAML Authentication](https://docs.sonarsource.com/sonarqube-server/latest/instance-administration/authentication/saml/overview/)
* **Grafana:** Connect to Authentik via OAuth. *(Note: Environment variables for Generic OAuth are already pre-configured in the `docker-compose.yml`)*.
* **Vikunja & Audiobookshelf:** Both are natively configured to expect an OIDC provider. Ensure the Client ID and Client Secret in Authentik match your environment variables.

### Header-based authentication (ForwardAuth)
* **Navidrome:** This service is placed behind Traefik's ForwardAuth middleware. It does not have native OIDC but relies on the `X-authentik-username` HTTP header for seamless login. You must configure a **Proxy Provider** in Authentik to inject this header.

### Internal logic and security
* **Grafana Data Sources:** Once logged in, manually add **Prometheus** (`http://prometheus:9090`) and **Loki** (`http://loki:3100`) as internal data sources.
* **Vaultwarden:** For security reasons, public signups are explicitly disabled (`SIGNUPS_ALLOWED=false`). You will need to manage users via the admin backend or via command line.


## Security posture

This infrastructure is built with strict security principles at both the network and container levels:

*   **Network Security (Zero Trust):** **There are absolutely no incoming ports open on the local home router.** All external traffic is securely routed through a Cloudflare Tunnel directly to Traefik.
*   **Identity-Aware Access:** Critical applications are placed behind Authentik's Forward Auth middleware or natively integrated via OIDC/SAML. Unauthenticated users cannot even reach the login pages of the underlying services.
*   **Container Hardening:** Following the principle of least privilege, services are heavily restricted to reduce the attack surface:
    *   **Non-root execution:** Containers run as restricted users (e.g., `1000:1000` or `10001:10001`).
    *   **Immutable filesystems:** Root filesystems are set to read-only (`read_only: true`), with strictly scoped `tmpfs` mounts for temporary application data.
    *   **No privilege escalation:** Explicitly denied via `security_opt: - no-new-privileges:true`.
    *   **Capability dropping:** Unnecessary kernel capabilities are dropped (e.g., `cap_drop: - ALL`).

## Disclaimer

This repository is public for educational purposes, to share architectural ideas, and to serve as my personal backup. Because it is tailored to my specific home network (including mixed ARM/AMD64 hardware) and domain names, **I do not accept Pull Requests (PRs)**. However, feel free to fork this project and adapt it to build your own infrastructure.
