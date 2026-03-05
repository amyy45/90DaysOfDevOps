# ☁️ AWS EC2 & Security Groups — Step-by-Step Guide

---

## What Is a Security Group?

An AWS **Security Group** is a virtual firewall attached to an EC2 instance (or other AWS resource). It controls what traffic is allowed in (**inbound rules**) and out (**outbound rules**) at the instance level.

**Key characteristics:**
- **Stateful** — if you allow inbound traffic on port 80, the response is automatically allowed outbound without a separate rule
- **Allow-only** — you can only create allow rules; there is no explicit deny
- **Default behavior** — all inbound traffic is denied by default; all outbound traffic is allowed by default
- **Multiple SGs** — one instance can have up to 5 Security Groups attached

---

## Step 1: Launch an EC2 Instance

1. Sign in to the [AWS Management Console](https://console.aws.amazon.com)
2. Navigate to **EC2** → click **Launch Instance**
3. Configure the instance:
   - **Name:** `devops-week1-server`
   - **AMI:** Ubuntu 24.04 LTS (Free Tier eligible)
   - **Instance type:** `t2.micro` (Free Tier)
   - **Key pair:** Create a new key pair → name it `devops-key` → download the `.pem` file
4. Under **Network settings** → click **Edit**
   - VPC: default VPC
   - Subnet: any public subnet
   - Auto-assign Public IP: **Enable**
   - Firewall: select **Create security group**
5. Leave storage at default (8 GB gp3)
6. Click **Launch Instance**

---

## Step 2: Create a Security Group

**Option A — During Instance Launch (recommended for beginners)**

Done in Step 1 under "Network settings → Create security group."

**Option B — Standalone Creation**

1. In the EC2 console, go to **Network & Security → Security Groups**
2. Click **Create security group**
3. Fill in:
   - **Name:** `devops-web-sg`
   - **Description:** `Security group for web and SSH access`
   - **VPC:** Select your default VPC

---

## Step 3: Configure Inbound Rules

In the Security Group creation screen, add these rules:

| Rule | Type | Protocol | Port | Source | Purpose |
|------|------|----------|------|--------|---------|
| 1 | SSH | TCP | 22 | My IP (`x.x.x.x/32`) | Remote terminal access — restrict to your IP only |
| 2 | HTTP | TCP | 80 | Anywhere (`0.0.0.0/0`) | Web server accessible to the public |
| 3 | HTTPS | TCP | 443 | Anywhere (`0.0.0.0/0`) | Encrypted web traffic |

> ⚠️ **Security note:** For SSH (port 22), always use **My IP** as the source instead of `0.0.0.0/0`. Leaving SSH open to the entire internet exposes your instance to brute-force attempts.

---

## Step 4: Configure Outbound Rules

The default outbound rule allows **all traffic outbound** — acceptable for most use cases.

```
All traffic | All | All | 0.0.0.0/0
```

If you need to restrict outbound traffic (e.g., compliance requirements):
- Allow only HTTPS (443) outbound for security patching
- Allow DNS (53) outbound so the instance can resolve names
- Block all other outbound traffic

---

## Step 5: Attach Security Group to Your Instance

If you created the Security Group separately after launching the instance:

1. Go to **EC2 → Instances** → select your instance
2. Click **Actions → Security → Change security groups**
3. Search for and select `devops-web-sg`
4. Click **Save**

---

## Step 6: Verify Connectivity

```bash
# Connect via SSH (Linux/macOS)
chmod 400 devops-key.pem
ssh -i devops-key.pem ubuntu@<your-instance-public-ip>

# Quick web server test (run on the EC2 instance)
sudo apt update && sudo apt install -y nginx
sudo systemctl start nginx

# From your local machine — verify HTTP is accessible
curl http://<your-instance-public-ip>
```

If `curl` returns an HTML response, your Security Group is configured correctly. ✅

---

## Three-Tier Architecture Example

```
┌─────────────────────────────────────────────┐
│  Web Server SG                              │
│  Inbound:  80, 443 from 0.0.0.0/0          │
│  Inbound:  22  from My IP only             │
│  Outbound: All                              │
├─────────────────────────────────────────────┤
│  App Server SG                              │
│  Inbound:  8080 from Web Server SG only    │
│  Inbound:  22   from My IP only            │
│  Outbound: All                              │
├─────────────────────────────────────────────┤
│  Database SG                                │
│  Inbound:  3306 from App Server SG only    │
│  Outbound: All                              │
└─────────────────────────────────────────────┘
```

---

## Best Practices

| Practice | Reason |
|----------|--------|
| Restrict SSH to your IP (`/32`) | Prevents brute-force attempts from the public internet |
| Use separate Security Groups per service tier | Limits blast radius if one is misconfigured |
| Never open `0.0.0.0/0` to a database port | DB ports should only be accessible from the application subnet |
| Audit rules regularly | Remove rules that are no longer needed |
| Use descriptive names and descriptions | Makes auditing easier across a team |
| Prefer private subnets for databases | Security Groups are one layer; subnet isolation is another |
