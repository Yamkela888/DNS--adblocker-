# Private DNS Stack — Pi-hole + Unbound

A fully self-hosted, privacy-focused DNS stack running in Docker on macOS.

This project builds a two-phase DNS architecture that blocks ads at the network
level AND eliminates dependency on third-party DNS providers like Cloudflare and
Google. Your DNS queries stay private, validated, and entirely under your control.

---

## The Problem This Project Solves

Every time you visit a website, your device asks a DNS server "what is the IP
address for this domain?" Most people's devices send these queries to their ISP
or a public resolver like Google (8.8.8.8) or Cloudflare (1.1.1.1).

This means:
- **Ad networks load freely** — nothing filters DNS requests for trackers
- **Third parties log your queries** — Cloudflare and Google can see every domain you visit
- **You trust someone else** with your complete browsing history in DNS form
- **DNS responses can be tampered with** — without DNSSEC, answers can be forged

This project solves all four problems.

---

## The Story So Far

### Phase 1 — Ad Blocking with Pi-hole ✅

The first phase introduced Pi-hole as a DNS sinkhole running in Docker.
Pi-hole intercepts DNS requests and blocks domains belonging to ad networks
and trackers before anything loads — across all browsers and apps, with no
extensions required.

**What Phase 1 achieved:**
- Network-level ad blocking
- Pi-hole web dashboard for monitoring
- Works on macOS via Docker

**What Phase 1 did not solve:**
- DNS queries still forwarded to Cloudflare or Google
- Third parties could still see every domain you queried
- No cryptographic validation of DNS responses

### Phase 2 — Private Recursive Resolution with Unbound ✅

The second phase adds Unbound as a local recursive resolver sitting between
Pi-hole and the internet. Instead of forwarding queries to a public DNS
provider, Unbound resolves them itself by starting at the root DNS servers
and walking the DNS tree — no third party involved.

**What Phase 2 adds:**
- Complete elimination of third-party DNS dependency
- DNSSEC validation — tampered DNS responses are cryptographically rejected
- Local DNS cache — repeated queries answered in under 5ms
- Query minimisation — only the minimum information is sent to each DNS server

---

## Architecture

### Before — Phase 1 Only

```
┌─────────────────────────────────────────────────────┐
│                    YOUR MAC                         │
│                                                     │
│   ┌─────────────────────────────────────────────┐  │
│   │           Pi-hole Container                 │  │
│   │           Port 53 (DNS)                     │  │
│   │           Port 8080 (Web UI)                │  │
│   │                                             │  │
│   │   Filters ads and trackers                  │  │
│   │   Forwards clean queries upstream ───────►  │  │
│   └─────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
                          │
                          ▼
        ┌─────────────────────────────────┐
        │  Cloudflare 1.1.1.1 or         │
        │  Google 8.8.8.8                │
        │                                │
        │  ⚠ Sees all your DNS queries   │
        └─────────────────────────────────┘
```

### After — Phase 1 + Phase 2

```
┌─────────────────────────────────────────────────────┐
│                    YOUR MAC                         │
│                                                     │
│   ┌──────────────────┐   ┌──────────────────────┐  │
│   │    Pi-hole       │──►│      Unbound         │  │
│   │   10.0.0.3:53    │   │     10.0.0.2:53      │  │
│   │   (public)       │   │   (internal only)    │  │
│   │                  │   │                      │  │
│   │  Blocks ads and  │   │  Validates DNSSEC    │  │
│   │  trackers        │   │  Caches results      │  │
│   └──────────────────┘   └──────────────────────┘  │
│                                    │                │
└────────────────────────────────────┼────────────────┘
                                     │
                                     ▼
              ┌──────────────────────────────────────┐
              │         ROOT DNS SERVERS             │
              │                                      │
              │   a.root-servers.net (198.41.0.4)   │
              │   b.root-servers.net (199.9.14.201) │
              │   + 11 others                        │
              │                                      │
              │   ✅ No third party sees your queries│
              └──────────────────────────────────────┘
```

---

## What Each Component Does

**Pi-hole** is a DNS sinkhole. It intercepts DNS requests and blocks domains
that belong to ad networks and trackers before anything loads. It works across
all browsers and apps without installing any extensions.

**Unbound** is a validating recursive resolver. Instead of forwarding your
queries to Cloudflare or Google, it resolves them itself by starting at the
root DNS servers and walking the DNS tree. It also validates DNSSEC signatures,
meaning tampered DNS responses are cryptographically rejected.

**Docker** runs both services in isolated containers on your Mac, connected
by a private internal network that is never exposed externally.

---

## Privacy and Security Gains

| | Phase 1 Only | Phase 1 + Phase 2 |
|---|---|---|
| Ad blocking | ✅ Yes | ✅ Yes |
| Third party sees queries | ⚠️ Yes (Cloudflare/Google) | ✅ No |
| DNSSEC validation | ⚠️ Depends on upstream | ✅ Locally validated |
| Query logging by upstream | ⚠️ Possible | ✅ Eliminated |
| DNS resolver you trust | Third party | Yourself |
| Root of trust | Upstream provider | Root DNS servers |
| Cache speed | ⚠️ Remote cache | ✅ Local, under 5ms |

---

## Before and After

Without Pi-hole, ads load freely on every website:

![Website with ads](screenshots/ads-before.png)

With Pi-hole running and DNS pointed to `127.0.0.1`, ads are blocked:

![Website without ads](screenshots/ads-after.png)

Pi-hole dashboard showing active blocking:

![Pi-hole dashboard active](screenshots/dashboard-active.png)

---

## Prerequisites

- macOS (Apple Silicon M1/M2/M3/M4 or Intel)
- [Docker Desktop for Mac](https://www.docker.com/products/docker-desktop) installed and running

---

## Project Structure

```
dns-stack/
├── docker-compose.yml        # Pi-hole + Unbound container configuration
├── etc-pihole/               # Pi-hole config (auto-created on first run)
├── etc-dnsmasq.d/            # DNS config (auto-created on first run)
└── etc-unbound/
    └── unbound.conf          # Unbound recursive resolver configuration
```

---

## Setup Instructions

### Step 1 — Install Docker Desktop

Download and install [Docker Desktop for Mac](https://www.docker.com/products/docker-desktop).
Choose **Apple Silicon** if you have an M1/M2/M3/M4 chip, or **Intel** if your
Mac is older.

Open Docker Desktop and wait for the whale icon in your menu bar to stop
animating — that means Docker is ready.

Verify Docker is working:

```bash
docker --version
docker compose version
```

### Step 2 — Create the Project Folder

```bash
mkdir ~/dns-stack
cd ~/dns-stack
mkdir etc-unbound
```

### Step 3 — Create the Unbound Configuration File

Create a file at `etc-unbound/unbound.conf` with the following content:

```yaml
server:
    verbosity: 0
    port: 53
    interface: 0.0.0.0
    access-control: 0.0.0.0/0 allow

    do-ip4: yes
    do-ip6: yes
    do-tcp: yes
    do-udp: yes

    num-threads: 1
    msg-cache-size: 50m
    rrset-cache-size: 100m
    cache-min-ttl: 3600
    cache-max-ttl: 86400
    prefetch: yes
    prefetch-key: yes

    hide-identity: yes
    hide-version: yes
    harden-glue: yes
    harden-dnssec-stripped: yes
    harden-algo-downgrade: yes
    use-caps-for-id: yes
    qname-minimisation: yes

    auto-trust-anchor-file: "/var/lib/unbound/root.key"
```

**What each section does:**

| Section | Purpose |
|---|---|
| Basic settings | Port and interface configuration |
| Protocol | Support IPv4, IPv6, TCP and UDP |
| Performance | Local cache reduces repeated lookups |
| Privacy | Hides server identity, minimises query exposure |
| DNSSEC | Cryptographically validates every DNS response |

### Step 4 — Create the Docker Compose File

Create `docker-compose.yml` with the following content:

```yaml
services:

  unbound:
    container_name: unbound
    image: klutchell/unbound:latest
    networks:
      dns-net:
        ipv4_address: 10.0.0.2
    restart: unless-stopped

  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "8080:80/tcp"
    environment:
      TZ: 'Africa/Johannesburg'
      WEBPASSWORD: 'your_password_here'
      FTLCONF_dns_listeningMode: 'all'
      FTLCONF_dns_upstreams: '10.0.0.2#53'
    volumes:
      - './etc-pihole:/etc/pihole'
      - './etc-dnsmasq.d:/etc/dnsmasq.d'
    networks:
      dns-net:
        ipv4_address: 10.0.0.3
    depends_on:
      - unbound
    restart: unless-stopped

networks:
  dns-net:
    driver: bridge
    ipam:
      config:
        - subnet: 10.0.0.0/24
```

> **Important:** Replace `your_password_here` with a strong password.

**Key configuration decisions explained:**

`FTLCONF_dns_upstreams: '10.0.0.2#53'` — Points Pi-hole at Unbound's fixed
internal Docker IP address instead of a public DNS provider. This is the single
line that eliminates Cloudflare and Google from your DNS chain entirely.

`dns-net` with fixed IPs — A private Docker bridge network so Pi-hole and
Unbound can communicate internally. Unbound is never reachable from outside
Docker.

`FTLCONF_dns_listeningMode: 'all'` — Required on macOS. Docker routes traffic
through an internal network and without this setting Pi-hole rejects those
queries with "ignoring query from non-local network".

`klutchell/unbound:latest` — This image has native Apple Silicon (arm64) support
and is actively maintained. Important: do not substitute `mvance/unbound` which
is amd64 only and will fail on Apple Silicon Macs.

### Step 5 — Start the Stack

```bash
docker compose up -d
```

Docker will pull both images and start the containers. Unbound starts first,
then Pi-hole starts and immediately points to it.

Verify both containers are running:

```bash
docker ps
```

You should see both `pihole` and `unbound` with status `Up`.

### Step 6 — Access the Pi-hole Dashboard

Open your browser and go to:

```
http://localhost:8080/admin
```

Log in with the password you set in `docker-compose.yml`.

Navigate to **Settings → DNS** and confirm the upstream DNS server shows
`10.0.0.2#53`. This confirms Pi-hole is forwarding to Unbound instead of
a public DNS provider.

### Step 7 — Point Your Mac's DNS to Pi-hole

1. Open **System Settings → Network → Wi-Fi → Details → DNS**
2. Click **+** and add `127.0.0.1`
3. Remove any other DNS servers listed
4. Click **OK** then **Apply**

Before — DNS pointing to router:

![DNS settings before](screenshots/dns-before.png)

After — DNS pointing to Pi-hole:

![DNS settings after](screenshots/dns-after.png)

Your Mac now sends all DNS queries through Pi-hole → Unbound → Root Servers.

---

## Verification

Run these tests to confirm the full stack is working correctly.

### Test 1 — Full chain resolution

```bash
dig @127.0.0.1 google.com
```

Expected output:
```
;; ANSWER SECTION:
google.com. 300 IN A <ip address>

;; SERVER: 127.0.0.1#53
;; Query time: <5 msec (cached) or <1000 msec (first query)
```

### Test 2 — Unbound resolving directly

```bash
docker exec pihole dig @10.0.0.2 google.com
```

A valid IP address in the ANSWER SECTION confirms Unbound is resolving
recursively from root servers.

### Test 3 — DNSSEC validation working

```bash
docker exec pihole dig @10.0.0.2 sigok.verteiltesysteme.net +dnssec
```

Look for `flags: qr rd ra ad` in the response header. The `ad` flag means
**Authenticated Data** — Unbound cryptographically verified the DNS signatures.
You will also see `RRSIG` records in the answer, which are the actual DNSSEC
signatures.

### Test 4 — DNSSEC correctly rejecting bad signatures

```bash
docker exec pihole dig @10.0.0.2 sigfail.verteiltesysteme.net
```

Expected result: no answer returned. `sigfail.verteiltesysteme.net` is a
deliberately broken DNSSEC domain used to test rejection. Unbound refusing
to serve a result confirms your DNS cannot be tampered with.

### Test 5 — Cache performance

Run the google.com query twice. The second query should return in under 5ms,
confirming Unbound's local cache is working correctly.

---

## How DNS Resolution Works (Step by Step)

When you type `github.com` into your browser, here is exactly what happens
with this stack:

```
1. Your Mac asks Pi-hole: "What is the IP for github.com?"
   └── Pi-hole checks its blocklist — github.com is not blocked

2. Pi-hole forwards the query to Unbound at 10.0.0.2

3. Unbound checks its local cache — not cached yet, so it resolves:
   a. Asks a root server: "Who handles .com?"
   b. Root server replies: "Ask the .com nameservers"
   c. Asks a .com nameserver: "Who handles github.com?"
   d. .com nameserver replies: "Ask GitHub's nameservers"
   e. Asks GitHub's nameserver: "What is the IP for github.com?"
   f. GitHub's nameserver replies: "140.82.121.4"

4. Unbound validates the DNSSEC signature on the response

5. Unbound caches the result for next time

6. Unbound returns the answer to Pi-hole

7. Pi-hole returns the answer to your Mac

8. Your browser connects to 140.82.121.4
```

This entire process takes under 1 second on a cold cache, and under 5ms
on subsequent queries.

---

## Keeping the Stack Running

For Pi-hole and Unbound to work, you need:

- Docker Desktop open and running (set it to launch on startup in Docker Desktop settings)
- Both containers running (`restart: unless-stopped` in the compose file handles automatic restarts)
- Your Mac's DNS still set to `127.0.0.1`

> **Important:** macOS resets DNS settings when you switch networks or wake
> from sleep. If ads reappear or browsing slows down, check
> System Settings → Network → Wi-Fi → Details → DNS and re-add `127.0.0.1`.

If your internet stops working entirely, temporarily set DNS to `8.8.8.8`
while you troubleshoot, then switch back to `127.0.0.1` once fixed.

---

## Troubleshooting

### Ads reappearing / Pi-hole not blocking

Check your Mac's DNS settings:

```bash
scutil --dns | grep nameserver
```

If you see anything other than `127.0.0.1`, your Mac is bypassing Pi-hole.
Go to System Settings → Network → Wi-Fi → Details → DNS and re-add `127.0.0.1`.

### Containers not running

```bash
docker ps
```

If either container is missing, restart the stack:

```bash
cd ~/dns-stack
docker compose up -d
```

### DNS queries timing out

Test each layer of the stack separately to isolate the problem:

```bash
# Test Pi-hole layer
dig @127.0.0.1 google.com

# Test Unbound layer from inside Docker
docker exec pihole dig @10.0.0.2 google.com
```

If Pi-hole responds but Unbound times out, check Unbound logs:

```bash
docker logs unbound
```

### Apple Silicon image compatibility error

If you see this error when starting the stack:

```
The requested image's platform (linux/amd64) does not match
the detected host platform (linux/arm64/v8)
```

You are using an image that does not support Apple Silicon. Make sure your
`docker-compose.yml` uses `klutchell/unbound:latest` which has native arm64
support. Do not use `mvance/unbound` which is amd64 only.

### Port 53 conflict on macOS

If port 53 is already in use:

```bash
sudo lsof -i :53
```

Identify the conflicting process and stop it before starting the stack.

---

## Resetting Your Pi-hole Password

```bash
docker exec -it pihole pihole setpassword yournewpassword
```

---

## Updating Blocklists

Pi-hole uses a blocklist called Gravity. Update it manually:

```bash
docker exec pihole pihole -g
```

---

## Windows Setup

The `docker-compose.yml` is the same for Windows. Follow these additional steps.

### Step 1 — Enable WSL2

Docker Desktop on Windows requires WSL2. Open PowerShell as Administrator:

```powershell
wsl --install
```

Restart when prompted, then install Docker Desktop with the WSL2 backend enabled.

### Step 2 — Fix the Port 53 conflict

Windows has a DNS Client service that occupies port 53 by default:

```powershell
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Dnscache" -Name "Start" -Value 4
Stop-Service -Name "Dnscache" -Force
```

### Step 3 — Point Windows DNS to Pi-hole

1. Open **Control Panel → Network → Network Connections**
2. Right-click your active adapter → **Properties**
3. Select **Internet Protocol Version 4 (TCP/IPv4) → Properties**
4. Set **Preferred DNS server** to `127.0.0.1`

---

## Extending This Setup

This setup filters ads and resolves DNS privately for your Mac only. To
extend it to your entire home network:

- Set the DNS on your **router** to point to the machine running this stack
- For reliable whole-home coverage, run this stack on an always-on device
  like a **Raspberry Pi** rather than a laptop that sleeps and changes networks

The router admin page (Huawei 4G CPE in this case) is accessible at
`192.168.8.1` — DNS settings are typically found under
**Advanced → Router → DHCP**:

![Router DHCP settings](screenshots/router-dhcp.png)

---

## Resources

- [Pi-hole Official Website](https://pi-hole.net)
- [Pi-hole Docker GitHub](https://github.com/pi-hole/docker-pi-hole)
- [Unbound by NLnet Labs](https://unbound.net)
- [klutchell/unbound Docker Image](https://github.com/klutchell/unbound-docker)
- [Docker Desktop for Mac](https://www.docker.com/products/docker-desktop)
- [DNSSEC explained — ICANN](https://www.icann.org/resources/pages/dnssec-what-is-it-why-important-2019-03-05-en)
- [Root Servers](https://www.iana.org/domains/root/servers)
