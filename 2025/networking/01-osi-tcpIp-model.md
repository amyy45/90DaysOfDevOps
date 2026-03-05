# 🌐 OSI & TCP/IP Models — Week 1 Networking
---

## Overview

When two computers communicate, they follow a set of rules. The **OSI model** (Open Systems Interconnection) defines those rules across 7 conceptual layers. The **TCP/IP model** is the 4-layer practical framework the internet actually uses today. Both models help engineers understand *where* in a communication stack something is happening — essential for debugging, system design, and security.

---

## OSI Model — 7 Layers

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 7 │ APPLICATION   │ User-facing protocols (HTTP, DNS) │
│  Layer 6 │ PRESENTATION  │ Encryption, encoding (TLS, JPEG)  │
│  Layer 5 │ SESSION       │ Session management (SSH sessions) │
│  Layer 4 │ TRANSPORT     │ Reliable delivery (TCP, UDP)      │
│  Layer 3 │ NETWORK       │ Routing & IP addressing (IP)      │
│  Layer 2 │ DATA LINK     │ MAC addressing (Ethernet)         │
│  Layer 1 │ PHYSICAL      │ Raw bits (cables, Wi-Fi)          │
└─────────────────────────────────────────────────────────────┘
```

| # | Layer | Core Responsibility | Real-World DevOps Example |
|---|-------|---------------------|--------------------------|
| 7 | **Application** | Protocols that applications use to communicate | A CI/CD pipeline hitting `POST https://api.github.com/repos/.../deployments` via HTTP |
| 6 | **Presentation** | Data formatting, encryption, compression | TLS 1.3 encrypting the payload before your HTTPS request leaves the server |
| 5 | **Session** | Opening, maintaining, and closing communication sessions | An SSH session to an EC2 instance staying alive while you run deployment scripts |
| 4 | **Transport** | Reliable (TCP) or fast (UDP) end-to-end delivery | TCP guaranteeing that all chunks of a Docker image pull arrive in order and intact |
| 3 | **Network** | Logical addressing and routing between networks | AWS routing packets from your VPC through an Internet Gateway to reach S3 |
| 2 | **Data Link** | Frame delivery between two nodes on the same network | Ethernet sending frames between your EC2 instance and its VPC router |
| 1 | **Physical** | Transmission of raw bits over a physical medium | The fiber cable between AWS data centers carrying your data |

---

## TCP/IP Model — 4 Layers

The TCP/IP model maps the same concepts into 4 layers and reflects how protocols are actually implemented.

```
┌──────────────────────────────────────────────────────────────────┐
│  Layer 4 │ APPLICATION    │ HTTP, HTTPS, SSH, DNS, FTP, SMTP     │
│  Layer 3 │ TRANSPORT      │ TCP, UDP                             │
│  Layer 2 │ INTERNET       │ IP, ICMP, ARP                        │
│  Layer 1 │ NETWORK ACCESS │ Ethernet, Wi-Fi (physical + framing) │
└──────────────────────────────────────────────────────────────────┘
```

| # | Layer | Protocols | Real-World DevOps Example |
|---|-------|-----------|--------------------------|
| 4 | **Application** | HTTP, HTTPS, SSH, DNS, FTP | Running `curl https://myapp.com/health` to check if a service is up |
| 3 | **Transport** | TCP, UDP | TCP completing a three-way handshake before Ansible connects to a remote host |
| 2 | **Internet** | IP, ICMP | IP routing a packet from a Jenkins server in one subnet to an RDS instance in another |
| 1 | **Network Access** | Ethernet, Wi-Fi | Ethernet frame delivery within an AWS Availability Zone |

---

## OSI vs TCP/IP — Mapping

```
OSI Model                   TCP/IP Model
──────────────────          ──────────────────────────────────────
7. Application    ──┐
6. Presentation   ──┼──────► Application   (HTTP, SSH, DNS, SMTP)
5. Session        ──┘
4. Transport      ──────────► Transport     (TCP, UDP)
3. Network        ──────────► Internet      (IP, ICMP)
2. Data Link      ──┐
1. Physical       ──┴────────► Network Access (Ethernet, Wi-Fi)
```

---

## Why It Matters for DevOps

When a service fails to connect, layered thinking tells you *where* to look:

| Error Message | Layer | Where to Debug |
|---------------|-------|----------------|
| `connection refused` | Layer 4 | Port not open or process not listening |
| `no route to host` | Layer 3 | Routing or Security Group misconfiguration |
| DNS resolution failure | Layer 7 | DNS config, Route 53, or `/etc/resolv.conf` |
| TLS handshake error | Layer 6 | Certificate mismatch or expired cert |
| SSH timeout | Layer 5 | Session config or firewall dropping the connection |

This mental model saves significant debugging time in production incidents.
