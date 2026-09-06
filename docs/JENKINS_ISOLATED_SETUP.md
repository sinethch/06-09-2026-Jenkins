# Isolated Jenkins Setup

This repository includes a separate Jenkins controller for the shared server. It does not modify or connect to the existing Jenkins instance on port `8080`.

## Isolation guarantees

- The controller is started from `docker-compose.jenkins.yml`, separate from the application Compose files.
- The Compose project name creates a separate container, network, and named volume.
- Jenkins data is stored in its own `jenkins_home` volume.
- The UI uses host port `8081` by default and container port `8080`.
- The controller is limited to one CPU and 2 GB memory.
- Jenkins agent port `50000` is not published.
- The Docker socket is not mounted, so Jenkins jobs cannot directly control the host Docker daemon.

Docker resources are still shared by all users on the server. These limits reduce the risk of interference, but they cannot replace administrator approval or a separate VM when strict isolation is required.

## Server setup

Run these commands as a user allowed to use Docker:

```bash
mkdir -p ~/jenkins-isolated
cd ~/jenkins-isolated
git clone https://github.com/sinethch/06-09-2026-Jenkins.git repository
cd repository
docker compose -p sineth-jenkins -f docker-compose.jenkins.yml config
docker compose -p sineth-jenkins -f docker-compose.jenkins.yml pull
docker compose -p sineth-jenkins -f docker-compose.jenkins.yml up -d
```

The `-p sineth-jenkins` option is important. Always use it when inspecting, starting, stopping, or updating this Jenkins instance.

Check that it is running:

```bash
docker compose -p sineth-jenkins -f docker-compose.jenkins.yml ps
curl -I http://127.0.0.1:8081/login
```

Open the new UI at:

```text
http://167.172.77.230:8081
```

If port `8081` is already in use, select another unused port without editing the Compose file:

```bash
JENKINS_PORT=8082 docker compose -p sineth-jenkins -f docker-compose.jenkins.yml up -d
```

The chosen port must also be allowed by the server firewall. For UFW, an administrator can run:

```bash
sudo ufw allow 8081/tcp
```

## Initial administrator password

Retrieve the password once the container is running:

```bash
docker compose -p sineth-jenkins -f docker-compose.jenkins.yml exec jenkins \
  cat /var/jenkins_home/secrets/initialAdminPassword
```

Complete the setup wizard, create an administrator account, and set the Jenkins URL to the final URL, for example `http://167.172.77.230:8081/`.

## Safe operations

```bash
# View logs
docker compose -p sineth-jenkins -f docker-compose.jenkins.yml logs -f jenkins

# Stop only this Jenkins instance
docker compose -p sineth-jenkins -f docker-compose.jenkins.yml stop

# Start only this Jenkins instance
docker compose -p sineth-jenkins -f docker-compose.jenkins.yml start

# Update the Jenkins image and recreate only this instance
docker compose -p sineth-jenkins -f docker-compose.jenkins.yml pull
docker compose -p sineth-jenkins -f docker-compose.jenkins.yml up -d
```

Do not run `docker compose down` from the application directory and do not reuse the existing Jenkins volume. Before upgrades, back up the Jenkins volume:

```bash
docker run --rm \
  -v sineth-jenkins_jenkins_home:/data:ro \
  -v "$PWD:/backup" \
  alpine:3.20 tar czf /backup/jenkins-home-backup.tgz -C /data .
```

## Build-job restrictions

Keep the Jenkins controller at one executor initially. Do not mount `/var/run/docker.sock`; that would give jobs broad control over the shared server. If builds need Docker, use a dedicated remote agent or an approved rootless builder with its own resource limits.

Ask the server administrator to confirm the available disk, memory, firewall rule, and Docker permissions before starting the container.