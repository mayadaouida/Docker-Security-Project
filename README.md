# Docker Security Project

A security-focused multi-container deployment of the VProfile Java application using Docker Compose.

## Project Overview

This project focuses on deploying and hardening a multi-container application environment. It uses Docker Compose to manage the application and its supporting services, with network isolation, container security controls, persistent storage, health checks, and vulnerability scanning.

## Architecture

The environment consists of six services:

| Service | Purpose |
|---|---|
| NGINX | Reverse proxy and application entry point |
| Tomcat | Java application server |
| MariaDB | Relational database |
| Memcached | Application caching |
| RabbitMQ | Message broker |
| Elasticsearch | Search and indexing |

NGINX exposes the application on port `8080` and forwards requests to Tomcat.

## Network Isolation

Two Docker networks are configured:

- **Frontend:** Connects NGINX and Tomcat.
- **Backend:** An internal network used by Tomcat and the supporting services.

NGINX is connected only to the frontend network. Database and supporting services are isolated from direct external access.

## Security Hardening

The following security measures were implemented:

- Read-only root filesystems for NGINX, Tomcat, and Memcached.
- Dropped unnecessary Linux capabilities and added back only required capabilities where needed.
- `no-new-privileges` enabled for containers.
- Resource limits configured using memory and CPU constraints.
- Sensitive credentials stored in an environment file rather than hardcoded in the Compose configuration.
- NGINX configuration mounted as read-only.
- Tomcat custom `server.xml` mounted as read-only.
- Temporary filesystems configured for writable runtime directories.

## Persistent Storage

Named Docker volumes are configured for persistent application data, including:

- MariaDB data
- RabbitMQ data
- Elasticsearch data

This keeps service data separate from the container lifecycle.

## Health Checks

Health checks are configured for all six services to monitor service availability.

The NGINX health endpoint is available at `/health`.

> Note: The RabbitMQ health-check settings were updated in the Compose configuration, but the running RabbitMQ container has not yet been recreated with the updated settings. Its previous container state was unhealthy during testing.

## Vulnerability Scanning

Trivy was used to scan the container images for HIGH and CRITICAL vulnerabilities.

### Scan Results

| Image | HIGH | CRITICAL |
|---|---:|---:|
| Tomcat | 0 | 0 |
| NGINX (after remediation) | 0 | 0 |
| MariaDB | 25 | 1 |
| Memcached | 0 | 0 |
| RabbitMQ | 0 | 0 |
| Elasticsearch 7.10.2 | 247 | 9 |

### Remediation and Findings

**NGINX**

The initial scan identified three HIGH vulnerabilities related to `libxslt`.

A custom NGINX image was built using a Dockerfile that upgrades the affected package:

```dockerfile
FROM nginx:stable-alpine

RUN apk upgrade --no-cache libxslt
```

The custom image was rescanned and reported zero HIGH and CRITICAL vulnerabilities.

**MariaDB**

The scan identified vulnerabilities associated with the inherited `gosu` binary and its Go standard-library dependencies. These findings belong to the upstream image components.

**Elasticsearch**

The pinned Elasticsearch `7.10.2` image contains multiple vulnerabilities in inherited Java dependencies and application libraries. A supported-version upgrade requires compatibility testing and data backup before implementation.

These findings are documented rather than being hidden or treated as resolved.

## Validation

The following checks were performed during the project:

- Docker Compose configuration validation.
- Container status and health-check inspection.
- Application access through the NGINX reverse proxy.
- Network isolation checks.
- Trivy image vulnerability scans.
- Verification of the custom NGINX image after remediation.

The application returned HTTP `200 OK` through the NGINX endpoint during testing.

## Project Structure

```text
Docker-Security-Project/
├── docker-compose.yml
├── .gitignore
├── nginx/
│   ├── Dockerfile
│   └── conf.d/
│       └── default.conf
└── tomcat/
    └── conf/
        └── server.xml
```

Application binaries, database initialization dumps, environment files, and upstream application source files are intentionally excluded from this repository.

## Technologies Used

- Docker
- Docker Compose
- NGINX
- Apache Tomcat
- MariaDB
- Memcached
- RabbitMQ
- Elasticsearch
- Trivy
- Linux

## Notes

This repository focuses on the Docker security configuration and deployment work. It does not claim that every inherited image vulnerability has been remediated. Remaining findings and runtime limitations are documented transparently.
