# AppDynamics Machine Agent - Docker Deployment & Troubleshooting Guide

This guide covers setup, environment configuration, `docker run` usage, and air-gapped / offline deployment procedures for running the AppDynamics Machine Agent with Docker Container Visibility.

---

## 1. Environment Configuration (`.env`)

Create a `.env` file in your working directory containing your controller details and host settings:

```env
# Host Docker Group ID
DOCKER_GID=983

# AppDynamics Controller Settings
APPDYNAMICS_CONTROLLER_HOST_NAME=192.168.1.143
APPDYNAMICS_CONTROLLER_PORT=8090
APPDYNAMICS_CONTROLLER_SSL_ENABLED=false
APPDYNAMICS_AGENT_ACCOUNT_NAME=customer1
APPDYNAMICS_AGENT_ACCOUNT_ACCESS_KEY=d402b3a3-a206-45fb-b316-409f1b7710f7

# Java System Properties required for Container & Server Visibility
MACHINE_AGENT_PROPERTIES=-Dappdynamics.sim.enabled=true -Dappdynamics.docker.enabled=true
```

---

## 2. Docker Compose Deployment

### `docker-compose.yml`
```yaml
services:
  machine-agent:
    image: appdynamics/machine-agent:latest
    container_name: machine-agent
    restart: unless-stopped

    # Dynamically inject the Docker group GID from .env
    group_add:
      - "${DOCKER_GID:?DOCKER_GID is not set in .env}"

    # System mounts for Docker & Host Monitoring
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /etc:/host/etc:ro

    # Pass configuration from .env
    environment:
      - APPDYNAMICS_CONTROLLER_HOST_NAME=${APPDYNAMICS_CONTROLLER_HOST_NAME}
      - APPDYNAMICS_CONTROLLER_PORT=${APPDYNAMICS_CONTROLLER_PORT}
      - APPDYNAMICS_CONTROLLER_SSL_ENABLED=${APPDYNAMICS_CONTROLLER_SSL_ENABLED:-false}
      - APPDYNAMICS_AGENT_ACCOUNT_NAME=${APPDYNAMICS_AGENT_ACCOUNT_NAME}
      - APPDYNAMICS_AGENT_ACCOUNT_ACCESS_KEY=${APPDYNAMICS_AGENT_ACCOUNT_ACCESS_KEY}
      - MACHINE_AGENT_PROPERTIES=${MACHINE_AGENT_PROPERTIES}
```

---

## 3. Deployment using `docker run` CLI

If you prefer using pure CLI commands instead of Docker Compose, use `--env-file` to import the `.env` variables directly:

```bash
docker run -d \
  --name machine-agent \
  --restart unless-stopped \
  --env-file .env \
  --group-add "$(getent group docker | cut -d: -f3)" \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /proc:/host/proc:ro \
  -v /sys:/host/sys:ro \
  -v /etc:/host/etc:ro \
  appdynamics/machine-agent:latest
```

---

## 4. Air-Gapped / Offline Deployment Guide

When deploying to offline servers without internet access:

### Step 1: On an Online Machine (Export Image)
```bash
# Pull the latest image
docker pull appdynamics/machine-agent:latest

# Save the image into a compressed tar archive
docker save appdynamics/machine-agent:latest | gzip > appdynamics-machine-agent.tar.gz
```

### Step 2: Transfer Artifacts to Offline Server
Transfer `appdynamics-machine-agent.tar.gz`, `docker-compose.yml`, and `.env` to the target server via SCP, USB, or internal artifact repository.

### Step 3: On the Offline Machine (Import & Run)
```bash
# 1. Load the Docker image
docker load -i appdynamics-machine-agent.tar.gz

# 2. Verify image loading
docker images | grep appdynamics

# 3. Update host Docker GID in .env
echo "DOCKER_GID=$(getent group docker | cut -d: -f3)" > .env

# 4. Start the container
docker compose up -d
```

---

## 5. Verification & Log Checking

To check if the agent is actively communicating with the controller:

```bash
# View last 50 log entries from the agent log
docker compose exec machine-agent tail -n 50 /opt/appdynamics/logs/machine-agent.log

# Check for any errors
docker compose exec machine-agent grep -i "error" /opt/appdynamics/logs/machine-agent.log
```