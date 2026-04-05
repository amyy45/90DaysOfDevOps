# Week 5: Docker Basics & Advanced Challenge — Solution

---

## Task 1: Introduction and Conceptual Understanding

### What is Docker?

Docker is an open-source platform that allows developers to package applications and their dependencies into lightweight, portable units called **containers**. These containers run consistently across any environment — a developer's laptop, a staging server, or a cloud production cluster — eliminating the classic "it works on my machine" problem.

In modern DevOps, Docker is foundational. It enables faster CI/CD pipelines (build once, run anywhere), simplifies dependency management, and makes scaling microservices straightforward.

### Virtualization vs. Containerization

| | Virtualization | Containerization |
|---|---|---|
| **Unit** | Virtual Machine (VM) | Container |
| **OS** | Each VM has its own full OS | Containers share the host OS kernel |
| **Startup time** | Minutes | Seconds |
| **Size** | GBs (full OS + app) | MBs (app + dependencies only) |
| **Isolation** | Strong (hypervisor-level) | Process-level (namespaces + cgroups) |
| **Resource usage** | High | Low |
| **Portability** | Limited | Excellent |

### Why Containerization for Microservices and CI/CD?

Microservices architectures decompose applications into many small, independently deployable services. Running each service in a VM would be wasteful — containers are orders of magnitude lighter. A container starts in seconds versus minutes for a VM, making them ideal for:

- **CI/CD pipelines** — spin up a clean build environment, run tests, and tear it down in seconds.
- **Microservices** — each service gets its own isolated container with its own dependencies, without conflict.
- **Horizontal scaling** — container orchestrators like Kubernetes can spin up dozens of container replicas in moments.

---

## Task 2: Create a Dockerfile for a Sample Project

### Sample Application

A simple Python Flask app that serves "Hello, Docker!" on port 80.

**`app.py`**
```python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello, Docker!"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=80)
```

**`requirements.txt`**
```
flask==3.0.0
```

### Dockerfile

```dockerfile
# Use the official Python slim image as the base
FROM python:3.12-slim

# Set the working directory inside the container
WORKDIR /app

# Copy dependency list first (leverages Docker layer caching)
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy the application source code
COPY app.py .

# Expose port 80 so the container can receive traffic
EXPOSE 80

# Define the default command to run the app
CMD ["python", "app.py"]
```

### Build the Image

```bash
docker build -t <your-username>/sample-app:latest .
```

### Run and Verify

```bash
# Run the container in detached mode, mapping host port 8080 to container port 80
docker run -d -p 8080:80 <your-username>/sample-app:latest

# Verify the container is running
docker ps

# Check logs
docker logs <container_id>

# Test via curl
curl http://localhost:8080
# Expected output: Hello, Docker!
```

---

## Task 3: Docker Terminologies and Components

### Key Terminologies

| Term | Description |
|---|---|
| **Image** | A read-only template containing the OS, runtime, app code, and dependencies. Built from a Dockerfile. Think of it as a blueprint. |
| **Container** | A running instance of an image. Isolated, lightweight, and ephemeral by default. |
| **Dockerfile** | A plain-text script of instructions that Docker uses to build an image layer by layer. |
| **Volume** | A persistent storage mechanism that lives outside the container's filesystem. Data survives container restarts and deletions. |
| **Network** | A virtual network that allows containers to communicate with each other and the outside world. |
| **Registry** | A storage and distribution system for Docker images (e.g., Docker Hub, AWS ECR, GitHub Container Registry). |
| **Docker Compose** | A tool for defining and running multi-container applications using a single `docker-compose.yml` file. |
| **Layer** | Each instruction in a Dockerfile creates a read-only layer. Layers are cached and shared between images for efficiency. |
| **Tag** | A label attached to an image to identify a specific version (e.g., `v1.0`, `latest`). |
| **Build context** | The set of files sent to the Docker daemon during a `docker build` command (usually the current directory). |

### Docker Components

```
┌──────────────────────────────────────────────────┐
│                  Docker CLI                      │
│  (docker build / run / push / pull / ps ...)     │
└────────────────────┬─────────────────────────────┘
                     │ REST API
┌────────────────────▼─────────────────────────────┐
│               Docker Daemon (dockerd)            │
│  Manages images, containers, networks, volumes   │
└──────┬────────────────────────┬──────────────────┘
       │                        │
┌──────▼──────┐        ┌────────▼────────┐
│  containerd │        │   Docker Hub /  │
│  (runtime)  │        │   Registry      │
└─────────────┘        └─────────────────┘
```

| Component | Role |
|---|---|
| **Docker CLI** | The command-line tool users interact with (`docker build`, `docker run`, etc.) |
| **Docker Daemon (`dockerd`)** | The background service that does the actual work — building images, starting containers, managing networks and volumes |
| **containerd** | The low-level container runtime that the daemon delegates to for starting/stopping containers |
| **Docker Hub** | The default public registry for storing and sharing images |
| **Docker Desktop** | GUI application for Mac/Windows that bundles the daemon, CLI, and Compose |

---

## Task 4: Multi-Stage Docker Build

### The Problem with Single-Stage Builds

In a single-stage build, the final image contains everything used during the build process — compilers, build tools, intermediate files — none of which are needed at runtime. This bloats the image unnecessarily.

### Multi-Stage Dockerfile

```dockerfile
# ─── Stage 1: Build ────────────────────────────────────────────────
# Use full Python image for dependency installation (has build tools)
FROM python:3.12 AS builder

WORKDIR /app

COPY requirements.txt .

# Install dependencies into an isolated prefix directory
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt


# ─── Stage 2: Runtime ──────────────────────────────────────────────
# Use the minimal slim image — no unnecessary tools
FROM python:3.12-slim AS runtime

WORKDIR /app

# Copy only the installed packages from the build stage
COPY --from=builder /install /usr/local

# Copy only the app source code
COPY app.py .

EXPOSE 80

CMD ["python", "app.py"]
```

### Comparing Image Sizes

```bash
# Build single-stage image
docker build -f Dockerfile.single -t sample-app:single .

# Build multi-stage image
docker build -f Dockerfile.multi -t sample-app:multi .

# Compare sizes
docker images | grep sample-app
```

**Example output:**
```
REPOSITORY    TAG       IMAGE ID       SIZE
sample-app    single    a1b2c3d4e5f6   1.02GB
sample-app    multi     f9e8d7c6b5a4   128MB
```

### Benefits of Multi-Stage Builds

- **Smaller image size** — only runtime artifacts are included; build tools are discarded.
- **Reduced attack surface** — fewer packages in the final image means fewer potential vulnerabilities.
- **Faster deployments** — smaller images pull faster in CI/CD pipelines and on Kubernetes nodes.
- **Separation of concerns** — build logic is cleanly separated from runtime configuration.

---

## Task 5: Managing Images with Docker Hub

```bash
# Tag the image with a versioned tag
docker tag <your-username>/sample-app:latest <your-username>/sample-app:v1.0

# Log in to Docker Hub
docker login

# Push the image
docker push <your-username>/sample-app:v1.0

# Verify by pulling the image
docker pull <your-username>/sample-app:v1.0
```

### Tagging Best Practices

- Always use **explicit version tags** (`v1.0`, `v1.2.3`) in production — never rely solely on `latest`.
- `latest` is just a convention; it is not automatically the newest image unless you tag it that way on every push.
- Use **semantic versioning** (MAJOR.MINOR.PATCH) for clarity on what changed between versions.

---

## Task 6: Persisting Data with Docker Volumes

```bash
# Create a named volume
docker volume create my_volume

# Run the container with the volume mounted at /app/data
docker run -d -v my_volume:/app/data <your-username>/sample-app:v1.0

# Inspect the volume
docker volume inspect my_volume

# List all volumes
docker volume ls
```

### Types of Docker Storage

| Type | Description | Use Case |
|---|---|---|
| **Named Volume** | Managed by Docker, stored in Docker's data directory | Databases, persistent app data |
| **Bind Mount** | Maps a host directory directly into the container | Local dev, sharing config files |
| **tmpfs Mount** | Stored in host memory only, not persisted | Sensitive or temporary data |

### Why Volumes Matter

Containers are **ephemeral by design** — when a container is deleted, its writable layer is gone with it. Volumes solve this by storing data outside the container lifecycle:

- A MySQL container stores its database files in a volume — even if the container is upgraded or recreated, the data survives.
- In CI/CD, volumes can cache build artifacts between pipeline runs, speeding up builds.
- In production, volumes enable zero-downtime deployments because the new container attaches to the same data volume as the old one.

---

## Task 7: Docker Networking

```bash
# Create a custom bridge network
docker network create my_network

# Run the sample app on the custom network
docker run -d --name sample-app --network my_network <your-username>/sample-app:v1.0

# Run a MySQL database on the same network
docker run -d \
  --name my-db \
  --network my_network \
  -e MYSQL_ROOT_PASSWORD=root \
  mysql:latest

# Verify both containers are on the network
docker network inspect my_network

# From sample-app container, ping the database by name
docker exec -it sample-app ping my-db
```

### Docker Network Types

| Type | Description | Use Case |
|---|---|---|
| **bridge** (default) | Isolated virtual network on a single host | Single-host multi-container apps |
| **host** | Container shares the host's network stack | High-performance, low-latency needs |
| **none** | No network access | Maximum isolation |
| **overlay** | Spans multiple Docker hosts | Docker Swarm, multi-host networking |

### Why Custom Networks?

On the default bridge network, containers can only communicate by IP address. On a **custom bridge network**, Docker provides automatic **DNS resolution by container name** — meaning `sample-app` can connect to `my-db` simply by using `my-db` as the hostname. This is exactly how Docker Compose networking works under the hood.

---

## Task 8: Docker Compose

### `docker-compose.yml`

```yaml
version: "3.9"

services:

  # ── Web Application ──────────────────────────────────────────────
  web:
    image: <your-username>/sample-app:v1.0
    container_name: sample-app
    ports:
      - "8080:80"           # Map host port 8080 to container port 80
    networks:
      - app-network
    volumes:
      - app-data:/app/data  # Persist app data
    depends_on:
      - db                  # Start db before web
    environment:
      - DB_HOST=db
      - DB_PORT=3306

  # ── MySQL Database ────────────────────────────────────────────────
  db:
    image: mysql:8.0
    container_name: my-db
    networks:
      - app-network
    volumes:
      - db-data:/var/lib/mysql  # Persist database files
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: appdb
      MYSQL_USER: appuser
      MYSQL_PASSWORD: apppass
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

# ── Networks ──────────────────────────────────────────────────────
networks:
  app-network:
    driver: bridge

# ── Volumes ───────────────────────────────────────────────────────
volumes:
  app-data:
  db-data:
```

### Deploy and Tear Down

```bash
# Start all services in detached mode
docker-compose up -d

# View running services
docker-compose ps

# View logs for a specific service
docker-compose logs web

# Stop and remove containers and networks (volumes preserved)
docker-compose down

# Stop and remove containers, networks, AND volumes
docker-compose down -v
```

### Service Configuration Explained

| Key | Purpose |
|---|---|
| `services` | Defines each container in the stack |
| `image` | Docker image to use for the service |
| `ports` | Maps `host:container` ports |
| `networks` | Attaches the service to named networks |
| `volumes` | Mounts named volumes or bind mounts |
| `depends_on` | Controls startup order (not health readiness — use `healthcheck` for that) |
| `environment` | Sets environment variables inside the container |
| `healthcheck` | Defines how Docker verifies the container is ready |

---

## Task 9: Analyzing Images with Docker Scout

```bash
# Quick summary of image security posture
docker scout quickview <your-username>/sample-app:v1.0

# Detailed CVE report
docker scout cves <your-username>/sample-app:v1.0

# Save the report to a file
docker scout cves <your-username>/sample-app:v1.0 > scout_report.txt
```

### Understanding the Report

Docker Scout scans each layer of your image and cross-references its packages against known vulnerability databases (NVD, OSV, etc.).

**Key fields in the output:**

| Field | Description |
|---|---|
| **CVE ID** | Unique identifier for the vulnerability (e.g., `CVE-2024-12345`) |
| **Severity** | Critical / High / Medium / Low / Unspecified |
| **Package** | The library or OS package containing the vulnerability |
| **Fixed in** | The version that patches the vulnerability |
| **Layer** | Which Dockerfile instruction introduced the vulnerable package |

### Example Findings and Remediation

| Severity | Common Cause | Fix |
|---|---|---|
| **Critical** | Outdated base OS packages | Use `apt-get upgrade` in Dockerfile or switch to a newer base image |
| **High** | Pinned old library version | Update `requirements.txt` / `package.json` to the latest patched version |
| **Medium** | Unused packages in image | Switch to `slim` or `alpine` base; use multi-stage builds to drop build tools |
| **Low** | Informational CVEs with no exploit path | Monitor; no immediate action required |

### Alternative: Trivy (if Docker Scout unavailable)

```bash
# Install Trivy (macOS)
brew install aquasecurity/trivy/trivy

# Scan the image
trivy image <your-username>/sample-app:v1.0
```

### Impact on Strategy

- **Prefer minimal base images** (`alpine`, `distroless`, `slim`) to reduce the number of installed packages and therefore the attack surface.
- **Update base images regularly** — pin to a minor version (e.g., `python:3.12-slim`) rather than a major one (e.g., `python:3`) to receive security patches without unexpected breaking changes.
- **Integrate Scout into CI/CD** — fail the pipeline on Critical/High CVEs using `docker scout cves --exit-code 1`.

---

## Task 10: Reflection on Docker's Impact

### Benefits

- **Consistency** — "Build once, run anywhere" eliminates environment drift between dev, staging, and production.
- **Speed** — Containers start in seconds; CI/CD pipelines using Docker are dramatically faster than VM-based equivalents.
- **Isolation** — Each service runs in its own container with its own dependencies, eliminating version conflicts (the classic "dependency hell").
- **Scalability** — Container orchestrators (Kubernetes, Docker Swarm) can scale containers up and down automatically based on load.
- **Developer experience** — `docker-compose up` gives any developer a fully working local environment in one command, regardless of their OS.

### Challenges

- **Security** — Containers share the host kernel; a kernel vulnerability can affect all containers. Running containers as non-root and scanning images regularly is essential.
- **Storage** — Containers are ephemeral. Forgetting to use volumes for stateful applications leads to data loss. Volume management adds operational complexity.
- **Networking complexity** — Multi-container networking, especially across hosts (overlay networks, service meshes), requires deep understanding to get right.
- **Image bloat** — Without multi-stage builds and careful base image selection, images can grow very large, slowing down deployments and increasing vulnerability exposure.
- **Learning curve** — Concepts like layer caching, networking modes, and Compose dependencies take time to master fully.

### Conclusion

Docker has fundamentally changed how software is built and shipped. It bridges the gap between development and operations, making the DevOps philosophy practical at scale. Understanding Docker deeply — from Dockerfile optimization to vulnerability scanning — is one of the most valuable skills in modern software engineering.

---

## Full Command Reference

```bash
# ── Build & Run ───────────────────────────────────────────────────
docker build -t <user>/sample-app:latest .
docker run -d -p 8080:80 <user>/sample-app:latest
docker ps
docker logs <container_id>
docker exec -it <container_id> /bin/sh

# ── Images ────────────────────────────────────────────────────────
docker images
docker tag <user>/sample-app:latest <user>/sample-app:v1.0
docker rmi <image_id>

# ── Docker Hub ────────────────────────────────────────────────────
docker login
docker push <user>/sample-app:v1.0
docker pull <user>/sample-app:v1.0

# ── Volumes ───────────────────────────────────────────────────────
docker volume create my_volume
docker volume ls
docker volume inspect my_volume
docker run -d -v my_volume:/app/data <user>/sample-app:v1.0

# ── Networking ────────────────────────────────────────────────────
docker network create my_network
docker network ls
docker network inspect my_network
docker run -d --name sample-app --network my_network <user>/sample-app:v1.0

# ── Docker Compose ────────────────────────────────────────────────
docker-compose up -d
docker-compose ps
docker-compose logs <service>
docker-compose down
docker-compose down -v

# ── Docker Scout ──────────────────────────────────────────────────
docker scout quickview <user>/sample-app:v1.0
docker scout cves <user>/sample-app:v1.0
docker scout cves <user>/sample-app:v1.0 > scout_report.txt
```

---
