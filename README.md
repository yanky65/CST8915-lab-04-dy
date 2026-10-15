# CST8915 Lab 4 - Introduction to Docker

**Student:** David Yin

**Submission status: Draft.** The application files are prepared. Complete and verify the Azure exercises, add the YouTube link, and review the reflection answers before submitting.

## Demo Video

**TODO:** Replace this line with the YouTube link to my Part 6 demonstration (maximum five minutes).

The recording procedure is in [DEMO.md](DEMO.md).

## Azure VM Configuration

The following settings were selected in the Azure Review + create screen supplied on October 14, 2026. This records the selected configuration; it does not independently confirm deployment or network access.

| Setting | Selected value |
| --- | --- |
| Subscription | Azure for Students |
| Resource group | `rg-cst8915-lab4` |
| VM name | `dy-lab4-docker` |
| Region | North Central US |
| Image | Ubuntu Server 24.04 LTS |
| Size | `Standard_B2als_v2` (portal label: Standard B2als_v2) |
| Architecture | x64 |
| Availability | No infrastructure redundancy required |
| Security | Trusted launch; secure boot and vTPM enabled |
| Integrity monitoring | Disabled |
| Authentication | SSH public key, RSA |
| Username | `azureuser` |
| Key source | Custom public key |

**Size difference:** The assignment specifies `Standard_B2s`; my selected configuration uses `Standard_B2als_v2`. Record the reason for this substitution and any instructor approval before final submission. No approval is claimed here.

**TODO:** Confirm SSH access and that the VM's network security rules allow the intended HTTP access on TCP 80. The supplied screenshot does not show those rules. Keep the private SSH key outside this repository.

## Project Files

```text
docker-lab/
  app.py                   # Greeting, container hostname, and /health
  requirements.txt
  Dockerfile
  .dockerignore
docker-compose-lab/
  app.py                   # Flask visit counter backed by Redis
  requirements.txt
  Dockerfile
  .dockerignore
  docker-compose.yml       # Single web instance on host port 80
  docker-compose.scale.yml # Three web instances with assigned host ports
RUNBOOK.md                 # Transfer, Docker installation, and lab exercises
DEMO.md                    # Part 6 recording outline
```

## Getting Started on the VM

Follow [RUNBOOK.md](RUNBOOK.md) to copy the files from Windows, connect with VS Code Remote SSH, and install Docker Engine using its official APT repository. All commands below run in the Ubuntu VM terminal after that setup.

### Standalone Flask application

```bash
cd ~/CST8915-lab-04-dy/docker-lab
docker build -t my-python-app:v1 .
docker run -d -p 80:5000 --name python-app my-python-app:v1
curl --retry 10 --retry-connrefused --retry-delay 1 http://localhost/
curl http://localhost/health
```

Stop the standalone container before starting Compose, because both use host port 80:

```bash
docker stop python-app
```

### Flask and Redis with Compose

```bash
cd ~/CST8915-lab-04-dy/docker-compose-lab
docker compose config
docker compose up -d --build
docker compose ps
curl --retry 10 --retry-connrefused --retry-delay 1 http://localhost/
curl http://localhost/
curl http://localhost/
docker compose logs --tail=20
```

Open `http://<VM_PUBLIC_IP>/` from my computer and refresh to show the count increasing. Redis is reachable internally as `redis:6379`; its port is not published to the Internet.

### Scaling

Use the alternative file by itself, rather than merging it with the fixed-port configuration:

```bash
docker compose -f docker-compose.scale.yml up -d --scale web=3
docker compose -f docker-compose.scale.yml ps
docker compose -f docker-compose.scale.yml port --index 1 web 5000
docker compose -f docker-compose.scale.yml port --index 2 web 5000
docker compose -f docker-compose.scale.yml port --index 3 web 5000
```

Use the assigned ports with `curl http://localhost:<PORT>/` on the VM. Restore one instance and host port 80 with:

```bash
docker compose up -d --scale web=1
```

### Configuration adjustments

- The Compose files omit the obsolete `version` field and the unused `FLASK_ENV` variable.
- Redis has a `redis-cli ping` health check. The web service waits for Redis to be healthy before starting.
- A separate scaling file avoids three web instances competing for host port 80.
- The runbook explicitly saves Redis data before a persistence demonstration. The named volume retains saved files; an incrementing counter alone does not prove persistence across container recreation.
- The copy-on-write exercise appends a Python comment instead of invalid Python text.

## Reflection Questions - Draft Answers

Review these answers after completing the exercises and add any relevant observations from the actual run.

### 1. What are the main differences between a Docker image and a Docker container?

A Docker image is a reusable, read-only template containing application code, dependencies, and configuration. A container is an instance created from that image, with its own processes, networking, and writable filesystem layer. Multiple containers can use the same image while maintaining separate changes. A container can be running or stopped; stopping it does not remove its writable layer.

### 2. Explain how Docker's layered architecture improves efficiency.

Images consist of reusable filesystem layers, allowing images and containers to share unchanged content instead of storing or downloading duplicate copies. Docker also caches build steps so an unchanged step can be reused. Copying requirements and installing dependencies before copying the application means a change to `app.py` can reuse the dependency installation step. An input change invalidates the affected cache entry and subsequent build steps.

### 3. Why does each container get its own writable layer?

Each container needs an isolated place to store filesystem changes without modifying its underlying image or another container. Writes to an existing image file are represented in the container's writable layer, leaving the shared image content unchanged. This allows independent instances to create different files while sharing the same image layers. Removing a container deletes its writable layer, so data that must outlive it should be stored in a volume or another external store.

### 4. What are the benefits of using Docker Compose over running containers individually?

Docker Compose describes services, ports, dependencies, networks, and volumes in one configuration file. It reduces the number of separate commands needed to reproduce and manage an application, with common commands for startup, shutdown, logs, and scaling. For this application, Compose places Flask and Redis on a shared network and supplies service-name discovery. A dependency with a health check also coordinates startup so the web application starts after Redis is ready.

## Submission Checklist

- [ ] Run and verify the exercises on the Azure VM.
- [ ] Explain the VM size substitution and record any approval actually received.
- [ ] Record Part 6 only, with a video duration of no more than five minutes.
- [ ] Upload to YouTube and verify that the link is accessible to the instructor.
- [ ] Replace the demo TODO above with the working link.
- [ ] Review the four reflection answers and remove draft/TODO text as appropriate.
- [ ] Push the final files to this public repository.
- [ ] Submit `https://github.com/yanky65/CST8915-lab-04-dy` in Brightspace and verify the confirmation.
- [ ] Clean up the lab resources after preserving the recording and verifying submission.

## References

- [Lab 4 assignment](https://github.com/ramymohamed10/26F_CST8915_Lab4)
- [Docker Engine installation on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- [Docker build cache](https://docs.docker.com/build/cache/invalidation/)
- [Container image layers](https://docs.docker.com/get-started/docker-concepts/building-images/understanding-image-layers/)
- [Compose startup order](https://docs.docker.com/compose/how-tos/startup-order/)
- [Redis persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)

The application examples and Dockerfile structure follow the instructor's lab. The README reflection answers are drafts for review; successful Azure execution and a completed demo are not yet claimed.
