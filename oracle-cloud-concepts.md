# Oracle Cloud — Concepts

## VCN (Virtual Cloud Network)

Your own isolated private network inside Oracle's data center. Every resource (VM, database, load balancer) must live inside a VCN. Other tenants cannot see or reach into your VCN.

- **CIDR block** defines the private IP range the VCN owns (e.g. `10.0.0.0/16` = `10.0.0.0` to `10.0.255.255`, 65,536 addresses)
- These are **private IPs only** — not internet-routable, only visible between resources inside the VCN
- One VCN per region is typical for small setups; you can have multiple

```
Oracle Cloud (Mumbai region)
├── Other tenants' VCNs (invisible to you)
└── Your VCN: main-vcn (10.0.0.0/16)
    └── All your subnets, VMs, etc.
```

---

## Subnet

A slice of the VCN's IP range, carved out for a specific purpose.

- **CIDR** is a subset of the VCN's range (e.g. `10.0.0.0/24` = 254 usable IPs out of the VCN's 65k)
- You leave room for more subnets later (e.g. `10.0.1.0/24` for a private subnet)

### Public vs Private Subnet

| Type | Public IP possible? | Internet access | Use case |
|------|-------------------|----------------|----------|
| **Public** | Yes | Direct (inbound + outbound) | Web servers, APIs, anything internet-facing |
| **Private** | No | Outbound only via NAT Gateway (not free) | Databases, internal services |

A VM in a **private subnet** cannot be SSHed into directly — you'd need a bastion host or VPN.

---

## VNIC (Virtual Network Interface Card)

A virtual network adapter attached to a VM. It's how the VM connects to a subnet (and through it, the VCN and internet).

- **Primary VNIC** — created automatically at instance launch, cannot be removed. Lives in the subnet you pick during creation.
- **Secondary VNICs** — optional, added later. Can attach to different subnets (even different VCNs). Use case: multi-homed VMs, network appliances.

Each VNIC gets:
- A **private IP** from its subnet's CIDR (always assigned)
- Optionally a **public IP** (only if the subnet is public)

```
VM: main-server
├── Primary VNIC
│   ├── Private IP: 10.0.0.45 (from public-subnet)
│   └── Public IP:  129.154.52.73 (Oracle assigns)
└── (Secondary VNIC — if you ever add one)
```

---

## Public IP

An internet-routable address (e.g. `129.154.52.73`) mapped to a VNIC's private IP via **1:1 NAT**.

- The VM itself only sees its private IP (`ip addr` shows `10.0.0.45`)
- Oracle transparently routes traffic: internet → public IP → private IP → VM
- **Ephemeral** (default): released when instance stops/terminates, may change on restart
- **Reserved**: persists independently, survives instance restarts, can be moved between VNICs

Free tier includes one public IP per instance.

```
Internet ──> 129.154.52.73 (public) ──NAT──> 10.0.0.45 (private) ──> VM
```

---

## Security List

Firewall rules attached to a **subnet**. Controls what traffic is allowed in (ingress) and out (egress).

- Default security list: allows all egress, allows SSH (port 22) ingress only
- You must add rules for each port you want open (e.g. 8080, 80, 443)
- Rules are **stateful** by default — if you allow inbound on port 8080, the response traffic is automatically allowed out

### Oracle's Two-Firewall Gotcha

Oracle has **two firewalls** that both must allow traffic:

1. **Security List** (cloud console) — subnet-level, managed in the OCI web UI
2. **iptables** (inside the VM) — OS-level, managed via SSH

Opening a port in the Security List but not in iptables (or vice versa) = traffic still blocked.

```
Internet
  │
  ▼
Security List (console): port 8080 allowed? ── No ──> BLOCKED
  │ Yes
  ▼
iptables (inside VM): port 8080 allowed? ── No ──> BLOCKED
  │ Yes
  ▼
Your app on port 8080
```

---

## Internet Gateway + Route Table

A public IP alone is not enough. The VCN needs two more things to actually reach the internet:

### Internet Gateway

The **door** between your VCN and the internet. Without it, no traffic flows in or out — even if your VM has a public IP.

- Free, one per VCN is enough
- Must be explicitly created when setting up a VCN manually (the VCN wizard auto-creates it, manual creation does not)

### Route Table

**Directions** telling the VCN where to send traffic. The critical rule:

| Destination | Target |
|-------------|--------|
| `0.0.0.0/0` (anywhere on the internet) | Internet Gateway |

Without this rule, the VCN has the door but no directions to use it.

```
Without Internet Gateway + Route:

Internet ──X──> VM has public IP but no path

With both:

Internet ──> Internet Gateway ──> Route Table ──> Subnet ──> VM
             (door)               (directions)
```

**Common gotcha:** Creating a VCN manually (not via wizard) does NOT auto-create the Internet Gateway or the route rule. You must add both yourself, or SSH and all internet traffic will time out.

---

## Full Architecture (Single Free Tier VM)

```
Internet
    │
    ▼
Public IP: 129.154.52.73
    │
    ▼ (1:1 NAT)
┌──────────────────────────────────┐
│  VCN: main-vcn (10.0.0.0/16)    │
│  ┌────────────────────────────┐  │
│  │ public-subnet (10.0.0.0/24)│  │
│  │  ┌──────────────────────┐  │  │
│  │  │ VM: main-server      │  │  │
│  │  │ Shape: A1.Flex ARM   │  │  │
│  │  │ 2 OCPU, 12 GB RAM   │  │  │
│  │  │ Private: 10.0.0.45   │  │  │
│  │  │ OS: Ubuntu 24.04 arm │  │  │
│  │  │                      │  │  │
│  │  │ ┌──────────────────┐ │  │  │
│  │  │ │ Docker           │ │  │  │
│  │  │ │ ├─ CLIProxyAPI   │ │  │  │
│  │  │ │ ├─ app-2         │ │  │  │
│  │  │ │ └─ app-n         │ │  │  │
│  │  │ └──────────────────┘ │  │  │
│  │  └──────────────────────┘  │  │
│  └────────────────────────────┘  │
└──────────────────────────────────┘
```

---

## Free Tier Limits (as of June 2026)

| Resource | Limit |
|----------|-------|
| ARM (A1.Flex) OCPUs | 2 (was 4 before June 2026) |
| ARM RAM | 12 GB (was 24 GB) |
| x86 Micro VMs | 2 × (1 OCPU + 1 GB each) |
| Boot volume | 200 GB total (across all instances) |
| Public IPs | 1 per instance |
| Outbound data | 10 TB/month |

- "Always Free Eligible" badge means the **shape type** qualifies, not that your specific config is within limits. The UI does NOT warn you.
- Never click "Upgrade to Pay-As-You-Go" on free tier — Oracle physically cannot charge you while on it.
- Set a **$1 budget alert** (Billing → Budgets) as a safety net.

---

## CIDR Notation Cheat Sheet

| CIDR | IPs | Mask | Use |
|------|-----|------|-----|
| `/16` | 65,536 | 255.255.0.0 | VCN level |
| `/24` | 256 (254 usable) | 255.255.255.0 | Subnet level |
| `/32` | 1 | 255.255.255.255 | Single host |
| `0.0.0.0/0` | All IPs | — | "Anywhere" (used in security list rules) |

---

## OCI vs AWS — Naming Comparison

| Concept | Oracle (OCI) | AWS |
|---------|-------------|-----|
| Private network | VCN | VPC |
| Network section | Subnet | Subnet |
| VM | Compute Instance | EC2 Instance |
| Virtual NIC | VNIC | ENI (Elastic Network Interface) |
| Firewall (subnet-level) | Security List | NACL (Network ACL) |
| Firewall (instance-level) | Network Security Group | Security Group |
| Internet access | Internet Gateway | Internet Gateway |
| NAT for private subnets | NAT Gateway | NAT Gateway |
| CPU unit | 1 OCPU = 2 vCPUs | 1 vCPU |
| Storage | Block Volume | EBS |
| Object storage | Object Storage | S3 |
| DNS | VCN DNS | Route 53 (internal) |

### Key Differences That Matter

1. **Security model** — Both are default-deny. But Oracle has **two firewalls** (Security List in console + iptables inside the VM). AWS Security Groups don't require OS-level firewall config.
2. **CPU naming** — Oracle's 2 OCPUs = 4 vCPUs in AWS terms. Don't compare numbers directly.
3. **Subnet access is explicit** — In Oracle you pick "Public" or "Private" at creation time. In AWS, a subnet becomes "public" implicitly by attaching a route to an Internet Gateway.

---

## OCPU vs vCPU

Oracle uses "OCPU" to mean one **physical core with hyperthreading** — each OCPU = 2 vCPUs.

| OCPUs | vCPUs (threads) | Equivalent in AWS |
|-------|-----------------|-------------------|
| 1 OCPU | 2 vCPUs | 2 vCPUs |
| 2 OCPUs | 4 vCPUs | 4 vCPUs |

`nproc` inside the VM shows `4` for a 2-OCPU instance. Oracle's numbers sound smaller but represent more compute than AWS's vCPU count.

---

## Availability Domain vs Fault Domain

- **Availability Domain (AD):** A separate physical data center within a region. Mumbai has only 1 AD. Some regions have 3.
- **Fault Domain (FD):** A grouping of hardware within an AD. Each AD has 3 fault domains. If one FD's hardware fails, the others are unaffected.

```
Region: ap-mumbai-1
└── AD-1 (only one in Mumbai)
    ├── FD-1 (rack group 1)
    ├── FD-2 (rack group 2)
    └── FD-3 (rack group 3)
```

For a single VM it doesn't matter much. It matters when you run multiple VMs and want them on separate hardware for redundancy.

---

## Capacity and Instance Lifecycle

- **"Out of capacity"** means Oracle has no free physical ARM hardware in that AD. The instance simply fails to create — nothing is charged, no resources consumed.
- Once created, the VM is **yours permanently** (on free tier). No CPU throttling, no burst credits. You can peg 100% CPU 24/7.
- Oracle won't kill a running instance. The only risk: **~90 days of account inactivity** → Oracle may stop (not delete) the instance.
- ARM capacity fluctuates — best creation times are **2-6am IST** (lowest demand). Auto-retry scripts exist on GitHub if manual retries fail.

---

## SSH (Secure Shell)

A remote terminal — you type commands on your Mac, they execute on the server. Everything is encrypted in transit.

### Key Pair System

SSH uses **asymmetric encryption** — two mathematically linked keys:

```
Public key  (lock)  → lives on the server (~/.ssh/authorized_keys)
Private key (key)   → lives on your Mac (~/.ssh/oracle_key)
```

Anyone can have the lock. Only you have the key. The private key **never leaves your machine**.

### Connection Flow

```
ssh -i ~/.ssh/oracle_key ubuntu@140.238.229.137
     │                   │       │
     │                   │       └── server IP
     │                   └── username on the server
     └── identity file (your private key)
```

What happens step by step:

```
1. TCP Connection
   Your Mac ──port 22──> Server

2. Server Proves Identity
   Server sends host key fingerprint
   First time → "Are you sure?" (saved to ~/.ssh/known_hosts)
   After that → auto-verified

3. You Prove Identity (without sending the private key)
   Server: random challenge ──> your Mac
   Your Mac: signs with private key ──> sends signature back
   Server: verifies with public key ──> match? You're in.

4. Encrypted Session
   Your Mac ◄══ encrypted tunnel ══► Server bash shell
```

### Why It's Secure

| Attack | Protected? | How |
|--------|-----------|-----|
| Intercepts traffic | Yes | Encrypted |
| Steals public key | Doesn't matter | Public key can't unlock anything |
| Impersonates server | Yes | Host key fingerprint check |
| Brute-force login | Yes | Key-based auth is practically uncrackable |

Only real risk: someone steals your **private key file**.

### Key Permissions

SSH refuses to use a private key if other users can read it.

```
chmod 600 ~/.ssh/oracle_key
```

`600` breakdown:
- **6** (owner) = read + write
- **0** (group) = no access
- **0** (others) = no access

Without this: `WARNING: UNPROTECTED PRIVATE KEY FILE` → connection refused.

### The `-i` Flag

Without `-i`, SSH tries default key names (`~/.ssh/id_rsa`, `~/.ssh/id_ed25519`). Since yours is named `oracle_key`, you must specify it. To avoid typing `-i` every time, add to `~/.ssh/config`:

```
Host oracle
    HostName 140.238.229.137
    User ubuntu
    IdentityFile ~/.ssh/oracle_key
```

Then just: `ssh oracle`

---

## Firewall

A **gatekeeper** that decides which network traffic is allowed in or out of your server. Every incoming connection is checked against a list of rules — allowed or blocked.

```
Without firewall:
  Anyone can reach any port → hackers scan and exploit open services

With firewall:
  Port 22 (SSH)     → allowed
  Port 80/443 (web) → allowed
  Port 3306 (DB)    → blocked, no outside access
  Everything else   → blocked
```

Oracle VMs have **two firewalls stacked** — traffic must pass both:
- **Security List** — Oracle's cloud firewall, in front of the VM in the network
- **iptables** — Linux's built-in firewall, inside the VM itself

Like a security guard at the building entrance AND one at your office door.

---

## Ingress and Egress Rules

**Ingress** = incoming traffic (internet → your server).
**Egress** = outgoing traffic (your server → internet).

In Oracle's Security List, **ingress rules** define what incoming traffic is allowed:

| Field | Meaning | Example |
|-------|---------|---------|
| **Source CIDR** | Who can send traffic | `0.0.0.0/0` = everyone, `49.36.120.5/32` = one IP |
| **Protocol** | TCP, UDP, or ICMP | TCP for web/SSH |
| **Destination Port** | Which port on your server | 80 (HTTP), 443 (HTTPS), 22 (SSH) |

A rule reads as: "Allow traffic FROM [source] TO [port] using [protocol]."

By default, only port 22 (SSH) has an ingress rule. You must add rules for each port you want to open.

**Egress** is fully open by default (your server can reach anything on the internet).

## iptables (VM-level Firewall)

Oracle's Ubuntu VM has iptables **pre-configured to block everything except SSH**. Rules are checked top-to-bottom, first match wins:

```
Rule 1: Allow established connections
Rule 2: Allow ICMP (ping)
Rule 3: Allow SSH (port 22)
Rule 4: Allow SSH (port 22) (duplicate, Oracle default)
Rule 5: Allow DHCP
Rule 6: REJECT everything else ← anything not matched above is blocked
```

To open a new port (e.g. 80), insert a rule **before** the REJECT:

```
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 80 -j ACCEPT
```

| Flag | Meaning |
|------|---------|
| `-I INPUT 6` | Insert at position 6 (before REJECT) |
| `-m state --state NEW` | Only new connections |
| `-p tcp` | TCP protocol |
| `--dport 80` | Port 80 |
| `-j ACCEPT` | Allow it |

### Persisting iptables Rules

```
sudo netfilter-persistent save
```

Saves rules to disk so they **survive a reboot**. Without this, rules disappear on restart. The `netfilter-persistent` package reads saved rules on boot and re-applies them.

### Two Firewalls — Both Must Allow Traffic

Oracle has two independent firewalls. Traffic must pass **both**:

```
Internet
  │
  ▼
Security List (OCI console/OCI CLI)  ← cloud-level, managed in web UI
  │
  ▼
iptables (inside VM, managed via SSH) ← OS-level
  │
  ▼
Your app
```

Opening a port in Security List but not iptables (or vice versa) = still blocked.

### Standard Web Ports

| Port | Protocol | Why both |
|------|----------|---------|
| 80 | HTTP | Exists only to redirect to 443 (HTTPS) |
| 443 | HTTPS | Where actual encrypted traffic goes |

Both are needed because: user types `http://myserver.com` (port 80) → Caddy redirects to `https://myserver.com` (port 443). Without port 80, the redirect fails.

---

## Reverse Proxy

A server that sits in front of your apps, receives all incoming traffic, and routes it to the right service based on the URL path. Without it, you'd expose each service on a separate port and remember which is which.

```
Without reverse proxy:
  http://140.238.229.137:3000  → ai-toolbox
  http://140.238.229.137:5678  → n8n
  http://140.238.229.137:8317  → CLIProxyAPI
  (3 ports to open, 3 URLs to remember)

With reverse proxy (Caddy on port 80):
  http://140.238.229.137/api/      → ai-toolbox (:3000)
  http://140.238.229.137/workflow/  → n8n (:5678)
  (1 port, clean paths, internal services hidden)
```

Benefits:
- **One port** exposed to the internet (80/443) instead of many
- **Path-based routing** — clean URLs instead of port numbers
- **HTTPS** — Caddy auto-generates TLS certificates when you add a domain
- **Security** — internal services (CLIProxyAPI :8317) stay unexposed

### Caddy

A web server / reverse proxy. Simpler than nginx — config is a few lines. Auto-HTTPS with domains.

```
# Caddyfile — routing config
:80 {
    handle /api/* {
        uri strip_prefix /api
        reverse_proxy localhost:3000
    }

    handle /workflow/* {
        uri strip_prefix /workflow
        reverse_proxy localhost:5678
    }
}
```

| Directive | Meaning |
|-----------|---------|
| `:80` | Listen on port 80 |
| `handle /api/*` | Match URLs starting with /api/ |
| `uri strip_prefix /api` | Remove /api from the path before forwarding (so /api/health → /health) |
| `reverse_proxy localhost:3000` | Forward the request to ai-toolbox |

### Full traffic flow

```
User's browser: http://140.238.229.137/api/health
      │
      ▼
Oracle Security List: port 80 allowed? → Yes
      │
      ▼
iptables: port 80 allowed? → Yes
      │
      ▼
Caddy (:80): path is /api/health → strip /api → forward to localhost:3000/health
      │
      ▼
ai-toolbox (:3000): responds with {"status":"ok"}
      │
      ▼ (back through Caddy)
User sees: {"status":"ok"}
```

### Opening port 80 — step by step

**1. Oracle Security List (OCI Console):**
- Networking → Virtual Cloud Networks → your VCN → Subnets → click subnet → Security Lists
- Add two Ingress Rules:
  - Source CIDR: `0.0.0.0/0`, Protocol: TCP, Destination Port: `80` (HTTP)
  - Source CIDR: `0.0.0.0/0`, Protocol: TCP, Destination Port: `443` (HTTPS)

**2. iptables (on the VM via SSH):**

```bash
sudo iptables -I INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 443 -j ACCEPT
sudo netfilter-persistent save
```

| Part | Meaning |
|------|---------|
| `sudo` | Run as root |
| `iptables` | Linux firewall command |
| `-I INPUT` | Insert rule at the top of the INPUT chain (incoming traffic) |
| `-p tcp` | Match TCP protocol |
| `--dport 80` | Match traffic to port 80 (HTTP) |
| `--dport 443` | Match traffic to port 443 (HTTPS) |
| `-j ACCEPT` | Allow this traffic |
| `netfilter-persistent save` | Save to disk so rules survive reboot |

Open both ports even if you only use 80 today — when you add a domain later, Caddy auto-enables HTTPS on 443 without any firewall changes.

Both steps are required — Security List is the cloud firewall, iptables is the OS firewall. Missing either = traffic blocked.

---

## Save as Stack

When creating an instance, "Save as Stack" exports your configuration as a **Terraform template** in Oracle's Resource Manager. Useful for:
- Retrying instance creation without re-filling the form (especially during capacity shortages)
- Reproducing the same setup later
- Found under **Developer Services → Resource Manager → Stacks**

---

## Billing on Free Tier

- **Authorization holds** (~110 INR × 4) at signup are temporary card verification charges. Auto-reverse in 3-7 business days.
- **No invoices** exist on free tier — Billing → Payment History will show nothing.
- Oracle **physically cannot charge** you on the free tier. The only danger is clicking "Upgrade to Pay-As-You-Go."
- Set a **$1 budget alert** (Billing → Budgets) as a safety net.
