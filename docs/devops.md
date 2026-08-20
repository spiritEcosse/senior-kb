## 11. DevOps

### Docker

**Image vs Container:**
- Image — immutable filesystem snapshot + metadata (layers)
- Container — running instance of an image; isolated via Linux namespaces + cgroups

---

### Container vs Host system

A container is **not a VM** — it shares the host OS kernel. Isolation comes from two Linux kernel features:

**Namespaces** — what the container *can see*:

| Namespace | Isolates |
|---|---|
| `pid` | Process tree — container sees only its own processes; PID 1 inside = your app |
| `net` | Network interfaces, routing tables, ports — container gets its own `eth0` |
| `mnt` | Filesystem mount points — container has its own `/` |
| `uts` | Hostname and domain name |
| `ipc` | Shared memory, semaphores |
| `user` | UID/GID mapping — root inside container ≠ root on host (with user namespaces) |

**cgroups** — what the container *can use*:

```
cpu    → limit CPU share or number of cores
memory → limit RAM; OOM-killer fires if exceeded
blkio  → limit disk I/O bandwidth
pids   → limit number of processes (prevent fork bombs)
```

**Practical differences:**

```
                    Host            Container
─────────────────────────────────────────────
Kernel              shared          shared (same kernel!)
Filesystem          /               isolated (own rootfs from image)
Process namespace   all PIDs        only container's PIDs
Network             eth0, lo        veth pair → bridge → host NAT
Users               real UIDs       mapped UIDs (root in container ≠ root on host by default)
Resources           unlimited       bounded by cgroup limits
Startup time        boot (minutes)  milliseconds (no kernel boot)
```

**What a container can NOT do by default:**
- See host processes (`ps aux` shows only container's own)
- Bind to host ports without `-p` mapping
- Access host filesystem without a volume mount
- Modify kernel parameters (no `sysctl` write)
- Load kernel modules

**What leaks through (shared kernel surface):**
- Kernel vulnerabilities affect all containers on the host
- `--privileged` flag removes almost all isolation — avoid in production
- `hostNetwork: true` in K8s gives the container direct host network access

```dockerfile
# Security best practice — run as non-root
RUN addgroup --system app && adduser --system --ingroup app app
USER app          # don't run as root inside the container

# Read-only filesystem — prevent runtime writes
docker run --read-only --tmpfs /tmp myapp
```

**Container vs VM:**

| | Container | VM |
|---|---|---|
| Kernel | Shared with host | Own kernel |
| Startup | Milliseconds | Seconds–minutes |
| Size | MBs | GBs |
| Isolation | Namespace/cgroup | Full hardware virtualisation |
| Security boundary | Weaker (shared kernel) | Stronger |
| Use case | App packaging, microservices | Full OS isolation, untrusted workloads |

```dockerfile
# Multi-stage build — small final image without build-time deps
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user -r requirements.txt

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY . .
ENV PATH=/root/.local/bin:$PATH

# Health check
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Key Dockerfile instructions:**
`FROM`, `RUN`, `COPY`, `ADD`, `ENV`, `EXPOSE`, `WORKDIR`, `CMD`, `ENTRYPOINT`, `ARG`, `USER`

**CMD vs ENTRYPOINT:**
- `ENTRYPOINT` — the executable (not overridden by `docker run` args)
- `CMD` — default args to ENTRYPOINT, or the command if no ENTRYPOINT

```bash
docker build -t myapp:1.0 .
docker run -p 8000:8000 --env-file .env myapp:1.0
docker exec -it <container_id> bash
docker logs -f <container_id>
docker system prune -af    # clean up all unused images/containers
```

---

### Docker Compose

```yaml
# docker-compose.yml
version: '3.9'

services:
  web:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/mydb
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - .:/app                 # mount source for dev hot-reload
    restart: unless-stopped

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

```bash
docker compose up -d          # start in background
docker compose logs -f web    # follow logs
docker compose exec web bash  # shell into running container
docker compose down -v        # stop + remove volumes
```

---

### Docker Networking: how containers see each other

**Do two containers need to be on the same network to talk to each other?** Yes. Containers on different Docker networks are isolated from each other by default — same as two machines on different physical networks with no route between them.

**Do they see each other "by ports"?** No — `-p`/`ports:` only publishes a port to the **host** (outside world → container). Container-to-container traffic never goes through the published host port at all; it goes directly over the internal Docker network to the container's port. That's why in the Compose example above, `db` (Postgres) has **no** `ports:` entry, yet `web` still reaches it at `db:5432` — only `redis` publishes `6379:6379`, and that's solely so something on the *host* (e.g. `redis-cli` run locally) can reach it, not because `web` needs it.

```
Host machine                  Docker network "myproject_default"
─────────────                 ───────────────────────────────────
localhost:8000  ──published──→  container "web":8000
                                  │
                                  │  internal traffic — no host port involved
                                  ▼
                                 container "db":5432   (no -p needed)
```

**What actually makes containers reachable by name:**
- Docker Compose creates one **user-defined bridge network** per project automatically and attaches every service in the file to it — so `web` can resolve the hostname `db` via Docker's embedded DNS, entirely inside that network.
- Plain `docker run` containers with no `--network` flag land on the default `bridge` network instead — they *can* reach each other, but only by IP, because the legacy default bridge has no DNS-based name resolution. Fix: create a user-defined network yourself.

```bash
docker network create mynet
docker run --network mynet --name db postgres:16
docker run --network mynet --name web myapp
# from inside 'web': `ping db` resolves — DNS works because it's a user-defined network
```

- `docker network ls` / `docker network inspect mynet` — list networks / see which containers are attached.
- A container can be on multiple networks (`docker network connect othernet web`) — useful for a reverse proxy that needs to reach containers in separate projects.

---

### Kubernetes

**Core concepts:**

```
Node        — a machine (VM or bare metal) in the cluster
Pod         — smallest deployable unit; one or more containers sharing network/storage
Deployment  — manages a set of identical Pods; handles rollout and rollback
Service     — stable network endpoint for a set of Pods (load balances between them)
Ingress     — HTTP/HTTPS routing rules into the cluster
ConfigMap   — non-secret configuration (env vars, config files)
Secret      — sensitive data (base64-encoded; use external secret managers in prod)
Namespace   — virtual cluster within a cluster; isolates resources
```

**Typical Deployment + Service:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp:1.0
          ports:
            - containerPort: 8000
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 5
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: myapp-secrets
                  key: database-url
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8000
  type: ClusterIP
```

**Key kubectl commands:**

```bash
kubectl get pods -n production
kubectl describe pod myapp-abc123
kubectl logs -f myapp-abc123
kubectl exec -it myapp-abc123 -- bash
kubectl apply -f deployment.yaml
kubectl rollout status deployment/myapp
kubectl rollout undo deployment/myapp      # rollback
kubectl scale deployment myapp --replicas=5
kubectl top pods                           # resource usage (needs metrics-server)
```

**liveness vs readiness probe:**
- `livenessProbe` — is the container alive? Fail → restart the container
- `readinessProbe` — is the container ready to serve traffic? Fail → remove from Service endpoints (but don't restart)

---

### CI/CD concepts

**Pipeline stages:**

```
commit → lint → test → build image → push to registry → deploy to staging → smoke test → deploy to prod
```

**Zero-downtime deployment strategies:**

| Strategy | How | Risk |
|---|---|---|
| Rolling update | Replace pods one by one (K8s default) | Brief mixed versions |
| Blue/Green | Run two identical envs, switch traffic | Double resource cost |
| Canary | Route N% traffic to new version | Complex routing |

**12-factor app principles (most relevant):**
- Config in env vars (not in code)
- Stateless processes (store state in DB/Redis, not locally)
- Explicitly declare dependencies (`requirements.txt`, `pyproject.toml`)
- Logs to stdout (let the platform collect them)
- Dev/prod parity — same services, same versions locally and in prod

---

### Observability: Logs, Metrics, Traces

**The three pillars:**
- **Logs** — what happened (structured JSON logs → ELK, Loki)
- **Metrics** — how much / how fast (Prometheus + Grafana)
- **Traces** — where time was spent across services (Jaeger, Zipkin, OpenTelemetry)

```python
# Structured logging (better than print/unstructured)
import structlog

log = structlog.get_logger()
log.info("order.created", order_id=42, user_id=7, amount=99.99)
# → {"event": "order.created", "order_id": 42, "user_id": 7, "amount": 99.99, "timestamp": ...}
```

```python
# Prometheus metrics
from prometheus_client import Counter, Histogram

REQUEST_COUNT = Counter('http_requests_total', 'Total HTTP requests', ['method', 'endpoint'])
REQUEST_LATENCY = Histogram('http_request_duration_seconds', 'Request latency', ['endpoint'])

@app.middleware("http")
async def metrics_middleware(request, call_next):
    with REQUEST_LATENCY.labels(endpoint=request.url.path).time():
        response = await call_next(request)
    REQUEST_COUNT.labels(method=request.method, endpoint=request.url.path).inc()
    return response
```

---

---

### Docker: CMD vs RUN vs ENTRYPOINT

```dockerfile
# RUN — executes during IMAGE BUILD; result is committed as a new layer
RUN apt-get update && apt-get install -y curl
RUN pip install -r requirements.txt

# CMD — default command when CONTAINER STARTS; overridable at runtime
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]

# ENTRYPOINT — fixed executable; CMD becomes its default arguments
ENTRYPOINT ["python", "-m"]
CMD ["myapp"]
# → docker run myimage           runs: python -m myapp
# → docker run myimage othercmd  runs: python -m othercmd
```

| Instruction | When | Overridable |
|---|---|---|
| `RUN` | Build time — creates image layer | N/A (it's baked in) |
| `CMD` | Container start — default command | Yes: `docker run image <cmd>` |
| `ENTRYPOINT` | Container start — fixed executable | Only with `--entrypoint` flag |

**Typical pattern:**

```dockerfile
ENTRYPOINT ["uvicorn"]          # always run uvicorn
CMD ["main:app", "--port", "8000"]   # default args, overridable
```

```bash
docker run myapp                            # uvicorn main:app --port 8000
docker run myapp main:app --port 9000       # override port
docker run --entrypoint bash myapp          # override entrypoint entirely
```

---

### Docker Volumes

A **volume** is persistent storage that lives outside the container's writable layer. When a container is removed, its writable layer is destroyed — volumes survive.

**Three types:**

```bash
# 1. Named volume — managed by Docker, stored in /var/lib/docker/volumes/
docker volume create mydata
docker run -v mydata:/app/data myimage

# 2. Bind mount — maps a host path into the container
docker run -v /home/ihor/project:/app myimage
# Common in dev: host source code → container, so changes reflect immediately

# 3. tmpfs mount — in-memory only, not persisted
docker run --tmpfs /tmp myimage
```

**In docker-compose:**

```yaml
services:
  db:
    image: postgres:16
    volumes:
      - postgres_data:/var/lib/postgresql/data   # named volume — data survives restart

  web:
    build: .
    volumes:
      - .:/app          # bind mount — dev hot-reload
      - /app/node_modules  # anonymous volume — don't overwrite container's node_modules

volumes:
  postgres_data:   # declares the named volume
```

**Why volumes matter:**
- Database data (`/var/lib/postgresql/data`) must survive container restarts → named volume
- Dev workflow: edit code on host, see changes in container immediately → bind mount
- Secrets/config files injected at runtime → bind mount
- Without a volume, all container writes are lost on `docker rm`

---
