# Docker Cheatsheet & Production Debugging Guide

A quick reference for daily Docker operations, Docker Compose, and real-world production troubleshooting.

---

## 1. Container Lifecycle & Inspection

```bash
docker ps                                       # Running containers
docker ps -a                                    # All containers (including stopped)
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"  # Clean custom table

docker run -d --name app -p 8080:80 nginx       # Run detached with port mapping
docker run --rm -it ubuntu:24.04 bash           # Ephemeral interactive container
docker start|stop|restart <container>           # Manage container state
docker rm -f <container>                        # Force-remove running container

docker inspect <container>                      # Full JSON config, mounts, network & state
docker top <container>                          # Processes running inside container
docker diff <container>                         # Filesystem changes since container started
```

---

## 2. Logs, Exec & File Operations

```bash
docker logs <container>                         # All container logs
docker logs -f --tail 100 <container>           # Follow last 100 lines
docker logs --since 30m <container>             # Logs from the last 30 minutes
docker logs -t <container>                      # Include timestamps

docker exec -it <container> /bin/sh             # Interactive shell inside running container
docker exec -u 0 -it <container> /bin/bash      # Exec as root user

docker cp <container>:/path/file ./local-file   # Copy file out of container
docker cp ./local-file <container>:/path/file   # Copy file into container
```

---

## 3. Images & Building

```bash
docker images                                   # List local images
docker pull <image:tag>                         # Pull image from registry
docker push <image:tag>                         # Push image to registry
docker rmi <image>                              # Remove image

docker build -t myapp:1.0 .                     # Build image from Dockerfile
docker build --no-cache -t myapp:1.0 .          # Build without layer cache
docker history --no-trunc <image>               # Inspect layer sizes & build commands
docker tag myapp:1.0 registry.io/myapp:1.0      # Tag image for registry

docker save -o app.tar myapp:1.0                # Export image to tar (air-gapped transfer)
docker load -i app.tar                          # Load image from tar
```

---

## 4. Volumes & Storage

```bash
docker volume ls                                # List named volumes
docker volume create <vol>                      # Create named volume
docker volume inspect <vol>                     # Show host mountpoint (/var/lib/docker/volumes/...)
docker volume rm <vol>                          # Remove unused volume

# Backup a named volume to a tar.gz file
docker run --rm -v <vol>:/data -v $(pwd):/backup alpine tar czf /backup/vol-backup.tar.gz -C /data .
```

---

## 5. Networking

```bash
docker network ls                               # List Docker networks
docker network create <net>                     # Create user-defined bridge (enables DNS by name)
docker network inspect <net>                    # Show attached containers & IP addresses
docker network connect <net> <container>        # Attach running container to a network
docker port <container>                         # Show container port mappings
```

---

## 6. Docker Compose (`docker compose`)

```bash
docker compose up -d                            # Start stack in background
docker compose up -d --build                    # Rebuild images and start
docker compose up -d --force-recreate <svc>     # Recreate a single service
docker compose down                             # Stop and remove containers & networks
docker compose down -v                          # Also remove named volumes (destructive!)

docker compose ps                               # List stack containers
docker compose logs -f --tail=100 <svc>         # Follow logs for a service
docker compose exec <svc> /bin/sh               # Shell into a service container
docker compose config                           # Validate & render merged compose YAML
```

---

## 7. Resource Usage & Cleanup

```bash
docker stats                                    # Live CPU, memory, network & I/O per container
docker stats --no-stream                        # One-shot resource snapshot
docker system df                                # Disk usage summary (images, containers, volumes, cache)
docker system df -v                             # Detailed per-object disk usage

docker container prune                          # Remove stopped containers
docker image prune -a                           # Remove all unused images
docker builder prune                            # Clear BuildKit build cache
docker system prune -af                         # Clean stopped containers, unused images & networks (keeps volumes)
```

---

## 8. Production Debugging Scenarios

### Scenario 1: Container Exits Immediately or Restart-Loops
**Symptoms:** `docker ps` doesn't show it, or `STATUS` shows `Restarting (1) ...` / `Exited (...)`.
```bash
docker ps -a                                    # Check exit code
docker logs --tail 100 <container>              # Check crash output
docker inspect <container> --format '{{json .State}}' | jq .
```
**Inspect a broken image by overriding its entrypoint:**
```bash
docker run --rm -it --entrypoint /bin/sh <image:tag>
```
**Common Exit Codes:**
- `0` — Main process finished (e.g., daemonized in background instead of foreground)
- `1` — Application error / bad config
- `126` / `127` — Permission denied or command/binary not found in image
- `137` — `SIGKILL` (usually **OOMKilled** or `docker kill`)
- `143` — `SIGTERM` (graceful stop)

---

### Scenario 2: Container `OOMKilled` (Exit Code 137)
**Symptoms:** Container restarts randomly under load with no application error in logs.
```bash
docker inspect <container> --format 'OOMKilled={{.State.OOMKilled}} ExitCode={{.State.ExitCode}}'
dmesg -T | grep -i -E "oom|killed process"      # Confirm kernel OOM killer event
docker stats --no-stream <container>            # Check live memory vs limit
```
**Fix:** Increase `--memory` / `deploy.resources.limits.memory`, or tune runtime memory caps (JVM `-Xmx`, Node `--max-old-space-size`, Gunicorn/worker count).

---

### Scenario 3: Disk Full Caused by Docker
**Symptoms:** Host hits 100% disk (`No space left on device`), containers fail to start or write.
```bash
docker system df -v                             # See what is eating space (images vs build cache vs volumes)

# Classic production gotcha: unbounded JSON container logs!
sudo du -sh /var/lib/docker/containers/*/*-json.log | sort -rh | head -n 10

# Safely truncate a runaway log file without restarting the container:
sudo truncate -s 0 /var/lib/docker/containers/<id>/<id>-json.log

# Reclaim unused images & build cache:
docker builder prune -af
docker image prune -af
```
> **Prevention:** Configure log rotation globally in `/etc/docker/daemon.json`:
> ```json
> {
>   "log-driver": "json-file",
>   "log-opts": { "max-size": "50m", "max-file": "3" }
> }
> ```

---

### Scenario 4: Container Networking / DNS Failure
**Symptoms:** `Connection refused`, `Could not resolve host`, or timeouts between containers.
```bash
# 1. Confirm both containers are on the SAME user-defined network
#    (Note: the default `bridge` network does NOT provide automatic container-name DNS!)
docker inspect <container> --format '{{json .NetworkSettings.Networks}}' | jq .

# 2. Attach a netshoot debug container to the target container's exact network namespace
docker run --rm -it --network container:<target-container> nicolaka/netshoot

# Inside netshoot, inspect ports, DNS, and connectivity:
ss -tulpn
dig <other-service-name>
curl -v http://<other-service>:<port>
```
> **Tip:** If a container is unreachable from the host, check if the app bound to `127.0.0.1` instead of `0.0.0.0` inside the container.

---

### Scenario 5: High CPU / Memory Usage by a Container
**Symptoms:** Host sluggish or specific service timing out.
```bash
docker stats --no-stream                        # Identify the offending container
docker top <container> aux                      # See exact processes inside the container
docker inspect <container> --format '{{json .HostConfig.NanoCpus}} {{json .HostConfig.Memory}}'
```

---

### Scenario 6: Port Already in Use (`Bind for 0.0.0.0:<port> failed`)
**Symptoms:** `docker run` or `docker compose up` fails with `address already in use`.
```bash
sudo ss -tulpn | grep :<port>                   # Find which host process or docker-proxy holds the port
docker ps --format "table {{.Names}}\t{{.Ports}}" | grep <port>  # Check if another container uses it
```

---

### Scenario 7: Debugging Distroless / Minimal Containers (No `sh`, `curl`, `ping`)
**Symptoms:** `docker exec -it <container> sh` fails with `executable file not found in $PATH`.
```bash
# Share both network and PID namespaces with a full toolbox container
docker run --rm -it \
  --network container:<target-container> \
  --pid container:<target-container> \
  nicolaka/netshoot
```

---

### Scenario 8: Permission Denied on Mounted Volumes
**Symptoms:** App fails to read/write inside a bind mount (`/app/data: Permission denied`).
```bash
docker exec <container> id                      # Check UID/GID the container runs as
ls -ln /host/mounted/path                       # Compare with host directory UID/GID
# Fix: align ownership on host (e.g. `sudo chown -R 1000:1000 /host/mounted/path`) or pass `--user $(id -u):$(id -g)`
```

---

### Scenario 9: Docker Daemon Unresponsive or Won't Start
**Symptoms:** `Cannot connect to the Docker daemon at unix:///var/run/docker.sock`.
```bash
sudo systemctl status docker                    # Check Docker daemon status
sudo systemctl status containerd                # Check underlying containerd runtime
sudo journalctl -u docker -n 100 --no-pager     # Check daemon startup/crash errors
sudo dockerd --validate                         # Validate /etc/docker/daemon.json syntax
```
