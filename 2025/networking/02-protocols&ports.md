# 🔌 Protocols & Ports for DevOps — Week 1 Networking

---

## Why Protocols Matter

Every time a DevOps engineer writes a Security Group rule, configures a firewall, builds a Docker network, or writes a CI/CD pipeline, they're working with protocols and ports. Knowing which port belongs to which protocol — and why — avoids misconfigurations and security gaps.

---

## Core Protocol Reference Table

| Protocol | Port(s) | Transport | Description | DevOps Relevance |
|----------|---------|-----------|-------------|-----------------|
| **HTTP** | 80 | TCP | Unencrypted web communication | Health checks, internal service traffic, load balancer listeners |
| **HTTPS** | 443 | TCP | TLS-encrypted web communication | All public APIs, webhooks, browser traffic, cloud provider APIs |
| **SSH** | 22 | TCP | Secure remote shell access | Accessing EC2 instances, Ansible connections, Git over SSH |
| **FTP** | 20, 21 | TCP | File transfer (unencrypted) | Legacy systems; replaced by SFTP in modern workflows |
| **SFTP** | 22 | TCP | Secure file transfer over SSH | Transferring deployment artifacts to remote servers |
| **DNS** | 53 | UDP/TCP | Domain name to IP resolution | Every outbound service call; DNS failures cause cascading failures |
| **DHCP** | 67, 68 | UDP | Automatic IP address assignment | EC2 instances receiving IPs from VPC subnet DHCP servers |
| **SMTP** | 25, 587 | TCP | Email transmission | Pipeline alerts, notification services (587 = auth submission) |
| **NTP** | 123 | UDP | Network time synchronization | Required for valid TLS certs, log timestamps, Kubernetes token expiry |
| **MySQL** | 3306 | TCP | MySQL / MariaDB database | App servers connecting to RDS MySQL; restrict to app subnet only |
| **PostgreSQL** | 5432 | TCP | PostgreSQL database | Backend services, data pipelines connecting to RDS Postgres |
| **Redis** | 6379 | TCP | In-memory key-value store | Session caching, queues, feature flags in application stacks |
| **MongoDB** | 27017 | TCP | NoSQL document database | Microservices with document-based data models |
| **Kubernetes API** | 6443 | TCP | Kubernetes control plane API | `kubectl` commands, CI/CD deployments, admission webhooks |
| **Docker Daemon** | 2376 | TCP | Remote Docker API (TLS) | Remote image builds; always use 2376 (TLS) not 2375 (plaintext) |
| **Prometheus** | 9090 | TCP | Metrics collection | Scraping infrastructure and application metrics |
| **Grafana** | 3000 | TCP | Metrics dashboards | Visualizing Prometheus, CloudWatch, and other data sources |
| **Jenkins** | 8080 | TCP | CI/CD server web UI | Pipeline management; keep behind a reverse proxy or VPN |
| **Elasticsearch** | 9200, 9300 | TCP | Search and log indexing | ELK stack log aggregation (9200 = HTTP API, 9300 = node transport) |

---

## Protocol Groups by Function

### Access & Remote Management
| Protocol | Port | Notes |
|----------|------|-------|
| SSH | 22 | Primary method for secure remote access to Linux servers |

### Web & API Traffic
| Protocol | Port | Notes |
|----------|------|-------|
| HTTP | 80 | Redirect to HTTPS or internal health checks only |
| HTTPS | 443 | All production traffic |

### Data Storage
| Protocol | Port | Notes |
|----------|------|-------|
| MySQL | 3306 | Never expose directly to the internet |
| PostgreSQL | 5432 | Access via private subnet or VPN |
| MongoDB | 27017 | Access via private subnet or VPN |
| Redis | 6379 | Access via private subnet or VPN |

### Infrastructure Services
| Protocol | Port | Notes |
|----------|------|-------|
| DNS | 53 | Required for all name resolution; AWS Route 53 uses this |
| NTP | 123 | Time drift causes TLS errors and authentication failures |
| DHCP | 67/68 | Automatic IP assignment in VPC subnets |

### Observability Stack
| Protocol | Port | Notes |
|----------|------|-------|
| Prometheus | 9090 | Accessible within private network or authenticated reverse proxy |
| Grafana | 3000 | Accessible within private network or authenticated reverse proxy |
| Elasticsearch | 9200/9300 | 9200 = HTTP API, 9300 = node-to-node transport |

### CI/CD & Container Tooling
| Protocol | Port | Notes |
|----------|------|-------|
| Jenkins | 8080 | Always place behind a reverse proxy or VPN |
| Docker Daemon | 2376 | Use TLS (2376), never plaintext (2375) |
| Kubernetes API | 6443 | Restrict access to CI/CD runners and admin IPs |

---

## Security Rules of Thumb

- **Never expose database ports** (3306, 5432, 27017, 6379) to `0.0.0.0/0`
- **Always use HTTPS (443)** for public-facing traffic; redirect HTTP (80) → HTTPS
- **Restrict SSH (22)** to specific IPs or a bastion host — never open to the world
- **Use TLS variants** — Docker on 2376 not 2375, SMTPS on 587 not 25
- **NTP (123) must be allowed outbound** — time sync failures break TLS and Kubernetes
