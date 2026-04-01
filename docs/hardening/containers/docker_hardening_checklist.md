---
title: Docker hardening checklist
layout: home
parent: Containers
nav_order: 1
tags: [hardening, linux, checklist, docker, dockercompose ]
---
# Container Hardening Checklist (Docker)
---

### 1. Use secure images

> Use oficial images (Docker Hub)

> Prefers minimal images (alpine, distroless)

> Remove unnecessary tools(curl, bash etc. in production)

> Scan vulnerabilites(Trivy, Grype)


### 2. Users and privileges

> Do not run containers as root

> Define non-privileged user


```bash
USER appuser
```
#### Note
> Avoid `--privileged` 

### 3. Management of secrets

Do not hardcode credentials in `docker-compose.yml` use secrets
