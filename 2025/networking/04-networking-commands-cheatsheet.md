# 🖥️ Networking Commands Cheat Sheet — Week 1 Networking

---

## Quick Reference

| Command | Primary Use | Key Flag |
|---------|-------------|----------|
| `ping` | Check host reachability | `-c 4` (limit packets) |
| `traceroute` | Find where packets are dropping | `-I` (use ICMP) |
| `netstat` / `ss` | See open ports and active connections | `-tulnp` |
| `curl` | Make HTTP requests, test APIs | `-I` (headers), `-v` (verbose) |
| `dig` | Detailed DNS record queries | `+short`, `+trace`, `-x` |
| `nslookup` | Cross-platform DNS lookup | `-type=MX/NS/TXT` |

---

## `ping` — Test Reachability

**Purpose:** Sends ICMP echo requests to a host to check whether it is reachable and measure round-trip time.

```bash
# Basic ping
ping google.com

# Limit to 4 packets (Linux/macOS)
ping -c 4 google.com

# Ping a specific IP
ping 8.8.8.8

# Ping with interval (every 0.5s)
ping -i 0.5 -c 10 google.com
```

**What to look for:**
- `0% packet loss` → host is reachable
- High latency (>200ms) → potential network congestion
- `Request timeout` → host is unreachable or blocking ICMP

**DevOps use:** Verify that an EC2 instance is reachable from a bastion host, or confirm that two containers in a Docker network can communicate.

---

## `traceroute` / `tracert` — Trace the Packet Route

**Purpose:** Shows the path packets take from your machine to a destination, listing each intermediate hop (router).

```bash
# Linux / macOS
traceroute google.com

# Windows
tracert google.com

# Use ICMP instead of UDP (Linux)
traceroute -I google.com

# Limit hops (useful for large networks)
traceroute -m 15 8.8.8.8
```

**What to look for:**
- Each line is one hop (router) along the path
- `* * *` means that hop is not responding (may be filtered, not necessarily broken)
- High latency at a specific hop indicates a bottleneck at that point

**DevOps use:** Diagnose where a connection is failing between your application server and an external API, or between services in different VPCs.

---

## `netstat` / `ss` — Network Statistics

**Purpose:** Displays active connections, listening ports, routing tables, and network interface statistics.

> **Note:** On modern Linux systems, `ss` is the preferred replacement for `netstat`.

```bash
# Show all listening ports
netstat -tuln
ss -tuln          # modern equivalent

# Show all active TCP connections
netstat -tan
ss -tan

# Show which process is using which port
sudo netstat -tulnp
sudo ss -tulnp

# Show network interface statistics
netstat -i

# Show the routing table
netstat -r
ip route show     # modern equivalent
```

**Common flags:**

| Flag | Meaning |
|------|---------|
| `-t` | TCP connections |
| `-u` | UDP connections |
| `-l` | Listening sockets only |
| `-n` | Show IPs instead of resolving hostnames |
| `-p` | Show the process using the socket |

**DevOps use:** Confirm that a web server is listening on port 80/443, identify a port conflict during container startup, or check what processes are holding open connections.

---

## `curl` — Make HTTP Requests

**Purpose:** Transfers data to or from a server using various protocols. Most commonly used to interact with HTTP/HTTPS APIs from the command line.

```bash
# Basic GET request
curl https://example.com

# Show only the HTTP status code
curl -o /dev/null -s -w "%{http_code}\n" https://example.com

# Follow redirects
curl -L http://example.com

# Include response headers
curl -I https://example.com

# POST request with JSON body
curl -X POST https://api.example.com/users \
  -H "Content-Type: application/json" \
  -d '{"name": "devops", "role": "engineer"}'

# POST with Authorization header
curl -X GET https://api.example.com/data \
  -H "Authorization: Bearer <your-token>"

# Download a file
curl -O https://example.com/file.tar.gz

# Test with a timeout (useful in scripts)
curl --max-time 5 https://example.com

# Send to a specific IP (bypass DNS)
curl --resolve example.com:443:93.184.216.34 https://example.com

# Verbose output (shows full request/response)
curl -v https://example.com
```

**DevOps use:** Test API endpoints in CI pipelines, check health check routes after deployments, interact with cloud provider APIs (AWS, GitHub), and validate SSL certificate responses.

---

## `dig` — DNS Lookup

**Purpose:** Queries DNS servers and returns detailed information about DNS records. More informative than `nslookup`.

```bash
# Basic A record lookup
dig example.com

# Short output (just the answer)
dig example.com +short

# Look up a specific record type
dig example.com MX        # Mail servers
dig example.com NS        # Name servers
dig example.com TXT       # TXT records (SPF, DKIM, etc.)
dig example.com AAAA      # IPv6 address
dig example.com CNAME     # Canonical name

# Query a specific DNS server
dig @8.8.8.8 example.com

# Reverse DNS lookup (IP to hostname)
dig -x 93.184.216.34

# Trace the full DNS resolution path
dig example.com +trace
```

**DevOps use:** Verify DNS propagation after a Route 53 record change, confirm a CNAME resolves to the right endpoint, debug why an application can't resolve a service name, and validate MX records for email delivery.

---

## `nslookup` — DNS Lookup (Cross-Platform)

**Purpose:** Queries DNS servers to resolve domain names. Available on Linux, macOS, and Windows — useful when `dig` is not installed.

```bash
# Basic lookup
nslookup example.com

# Query a specific DNS server
nslookup example.com 8.8.8.8

# Look up a specific record type
nslookup -type=MX example.com
nslookup -type=NS example.com
nslookup -type=TXT example.com

# Reverse lookup
nslookup 93.184.216.34

# Interactive mode
nslookup
> set type=MX
> example.com
> exit
```

**DevOps use:** Quick DNS verification on Windows systems, checking DNS records from within a container that doesn't have `dig` installed.
