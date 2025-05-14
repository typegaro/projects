# Managing a VPS with Docker, GitHub CI/CD, Private Registry & Watchtower

This article describes how I’ve built an automated deployment pipeline on my personal VPS using Docker, GitHub Actions, a private image registry, and Watchtower for periodic updates.

The system supports HTTPS through Traefik and delivers code to production within minutes of pushing to GitHub.

---

## Architecture Overview

```mermaid
graph TD
  Dev[Developer Push] --> GitHub[GitHub Repository]
  GitHub --> CI[GitHub Actions CI/CD]
  CI --> DockerHub[Private Docker Registry]
  DockerHub --> Watchtower[Watchtower on VPS]
  Watchtower --> VPS[VPS: Docker Compose]
  VPS --> Traefik[Traefik Reverse Proxy]
  Traefik --> App[Containerized Web App]
  User[User] -->|HTTPS| Traefik

  linkStyle default stroke:#98be65,stroke-width:2px;
  classDef yellowBox fill:#282c34,stroke:#da8548,stroke-width:2px,color:#bbc2cf;
  class Dev,GitHub,CI,DockerHub,Watchtower,VPS,Traefik,App,User yellowBox;
````

---

## Key Components

* Docker Compose to orchestrate services
* GitHub Actions for CI/CD
* Docker Hub private registry for image storage
* Watchtower to automatically update containers
* Traefik for reverse proxy and TLS certificate management
* Firewall (UFW or iptables) for controlled port access

---

## Workflow

### 1. Code Push to GitHub

Pushing to the `main` branch triggers an automated GitHub Actions workflow.

### 2. CI/CD Pipeline

The CI/CD process:

* Authenticates with Docker Hub via repository secrets
* Builds the application image
* Tags it as `latest`
* Pushes it to a private Docker Hub repository

### 3. Watchtower: Auto-Update Every 5 Minutes

Watchtower runs as a container on the VPS and checks every 300 seconds for updates to images of containers labeled for auto-updating.

When a new image is detected:

* The image is pulled
* The container is restarted with the updated version
* Other services are left untouched

---

## HTTPS with Traefik

Traefik handles HTTPS termination and acts as a dynamic reverse proxy:

* Binds to ports 80 and 443
* Automatically issues and renews Let's Encrypt TLS certificates
* Redirects all HTTP traffic to HTTPS
* Uses Docker labels to discover and route traffic to containers

This eliminates the need to manage certificates or Nginx/Apache manually.

---

## Firewall & Docker Networking

Docker containers typically bypass UFW rules by inserting iptables rules directly. This can lead to a false sense of security when using UFW alone.

There are three main approaches to secure Docker traffic:

### 1. Use `iptables` Directly

Manually configure `iptables` rules instead of relying on UFW to handle container traffic. This provides the most granular control but requires low-level management.

### 2. Use `ufw-docker`

Install the [ufw-docker](https://github.com/chaifeng/ufw-docker) helper script to patch UFW’s behavior, allowing UFW to block unwanted container traffic correctly.

### 3. Rely on Traefik as a Proxy (Safe for Exposed Services)

If you only expose ports **80** and **443** via Traefik, and other containers have `exposedByDefault=false`, you're safe from external access:

* Traefik handles all incoming traffic securely
* Other services remain isolated in Docker’s internal network
* Only Traefik binds to the host’s public ports

For many setups, relying on Traefik and keeping the firewall open **only for ports 80/443** is secure and sufficient.

## Why This Setup Works

* No manual deployments — GitHub CI/CD and Watchtower handle everything
* Security-focused — only essential ports are exposed; Traefik handles TLS
* Easy extensibility — just label new containers, and Traefik serves them
* Low maintenance — containerized, declarative, and reproducible setup

---

## Future Improvements

* Use GitHub Container Registry or a self-hosted registry
* Add uptime monitoring and deploy notifications via webhooks
* Implement health-check-based rollback for broken image pushes

---

## Conclusion

This Docker-based deployment architecture has simplified my VPS management and made it feel like a lightweight cloud provider. With GitHub Actions, private registries, and Watchtower, I can deploy and update production environments automatically and securely.

It's an efficient approach for developers hosting small to medium projects who want control without complexity.

