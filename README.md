# DevOps Technical Assessment - RedLime Solutions

This repository contains the solution for the DevOps Technical Assessment, demonstrating the integration of Web Development, Docker, Nginx, and GitHub Actions CI/CD to deploy two standalone containerized web applications.

## Project Overview

The objective of this project is to serve two lightweight, responsive static websites (Site A and Site B) via an Nginx reverse proxy. Both the reverse proxy and the web servers are fully containerized using Docker, and the orchestration is managed via Docker Compose. Furthermore, the repository is equipped with a GitHub Actions workflow to enable continuous deployment (CI/CD) to a server.

- **Site A (Company Landing Page):** A modern corporate landing page accessible at `/site-a/`.
- **Site B (Portfolio/About Page):** A clean portfolio page accessible at `/site-b/`.

## Folder Structure

```
.
├── .github/
│   └── workflows/
│       └── deploy.yml       # GitHub Actions CI/CD workflow
├── docker-compose.yml       # Docker Compose orchestration file
├── nginx/
│   └── nginx.conf           # Nginx reverse proxy configuration
├── README.md                # Project documentation
├── site-a/                  # Source code for Site A
│   ├── Dockerfile           # Docker configuration for Site A
│   ├── index.html
│   └── style.css
└── site-b/                  # Source code for Site B
    ├── Dockerfile           # Docker configuration for Site B
    ├── index.html
    └── style.css
```

## Server Requirements

To deploy this project to a production or staging Linux server, the following prerequisites must be installed on the host machine:

- **Docker:** Engine version 20.10.0 or higher.
- **Docker Compose:** Version v2.x or higher (usually bundled with modern Docker installations).
- **Git** (for pulling the repository during deployment).
- **SSH access** enabled for GitHub Actions deployments.

## Docker & Docker Compose Setup

### Dockerfiles
Each site relies on a separate `Dockerfile` based on the official lightweight `nginx:alpine` image. The static files (`index.html` and `style.css`) are copied into the default Nginx html directory (`/usr/share/nginx/html`). This makes each site immutable, isolated, and highly performant.

### Docker Compose
`docker-compose.yml` ties the ecosystem together. It spins up three services:
1. `site-a`: Builds the Site A container.
2. `site-b`: Builds the Site B container.
3. `proxy`: The master Nginx reverse proxy that routes outside traffic to the internal containers.

A custom Docker bridge network (`web-network`) is specified so the proxy can use container hostnames (e.g., `http://site-a`) as upstream destinations instead of relying on explicit IP addresses.

## Nginx Configuration

The master Nginx reverse proxy (`nginx/nginx.conf`) is configured to perform path-based routing.
- Requests matching `/site-a/` are proxied to the isolated `site-a` container.
- Requests matching `/site-b/` are proxied to the isolated `site-b` container.
- The root path `/` is configured to automatically redirect to `/site-a/`.

Headers like `Host`, `X-Real-IP`, and `X-Forwarded-For` are preserved and forwarded to the upstream servers for proper logging and routing behavior.

## GitHub Actions CI/CD Workflow

A fully automated CI/CD pipeline is defined in `.github/workflows/deploy.yml`. 
Every time code is pushed to the `main` branch, the workflow:
1. Triggers a GitHub Actions runner (Ubuntu).
2. Connects to the designated Linux deployment server via SSH (using `appleboy/ssh-action`).
3. Performs a `git pull` on the server to grab the latest code.
4. Smoothly rebuilds and restarts the application via `docker compose up -d --build`.
5. Cleans up dangling docker images to prevent space exhaustion over time.

### Required GitHub Secrets
To utilize the CI/CD pipeline on a live server, configure the following **Repository Secrets** in GitHub under *Settings -> Secrets and variables -> Actions*:
- `SERVER_HOST`: The IP address or Hostname of the target server.
- `SERVER_USERNAME`: The SSH user (e.g., `ubuntu` or `root`).
- `SERVER_SSH_KEY`: The private SSH key for the user.

## Deployment Steps (Manual)

If you wish to deploy the project manually on a Linux Server rather than utilizing the GitHub actions pipeline, run:

```bash
# 1. Clone the repository
git clone <your-repository-url> /opt/devops-assessment
cd /opt/devops-assessment

# 2. Spin up the infrastructure
docker compose up -d --build

# 3. Verify
docker ps
```

The application will immediately become available at point `http://<your-server-ip>/`. 

## How to Run the Project Locally

Running the setup locally for development or demonstration purposes is straightforward:

1. Ensure **Docker Desktop** (Mac/Windows) or Docker Engine (Linux) is running on your machine.
2. Open a terminal and clone/navigate to this repository folder.
3. Run:
   ```bash
   docker-compose up -d --build
   ```
4. Open your web browser and navigate to:
   - Site A: `http://localhost/site-a/`
   - Site B: `http://localhost/site-b/`

To stop the environment, run:
```bash
docker-compose down
```

## Assumptions and Design Decisions

- **Routing Methodology:** I opted for **path-based routing** (`/site-a/`, `/site-b/`) rather than subdomains (e.g., `sitea.local`). Path-based routing requires zero local DNS/hostfile tweaking, making it immediately previewable anywhere, including `localhost`.
- **Nginx Base Images:** Used `nginx:alpine` for both the master reverse proxy and the downstream target servers. This minimizes image size and removes known vulnerabilities often found in heavier base images.
- **Port Mapping:** The backend applications (Site A & Site B) do not expose ports directly to the local host machine (no `ports` mapping in docker-compose for these services). This restricts direct access; they can ONLY be accessed securely via the master Nginx reverse proxy.

## Bonus Points Achieved

- **Environment Variables:** Sensitive connection details (host, username, ssh keys) are managed securely via GitHub Actions Secrets rather than hardcoded.
- **Docker Image Optimization:** The `nginx:alpine` base image is used to keep container footprint as minimal as possible, resulting in faster pull/build times and a reduced attack surface.
- **Clean Structure & Organization:** Follows best practices for separating proxy configurations, docker orchestrations, and application code.
