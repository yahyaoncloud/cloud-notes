# Cloudflare Tunnels (cloudflared) & Hybrid Cloud Connectivity

Cloudflare Tunnel creates an encrypted, outbound-only connection between your origin infrastructure (servers, Kubernetes clusters, on-prem homelabs) and Cloudflare's edge network without opening any public inbound ports or exposing public IP addresses.

---

## 1. Tunnel Architecture

```text
End User
   │ (HTTPS: late5530.yahyaoncloud.com)
   ▼
Cloudflare Edge Network
   │
   │ (Outbound multiplexed QUIC/HTTP2 connection over port 7844 / 443)
   ▼
Origin Server (Rocky Linux / Ubuntu)
   └── cloudflared daemon (systemd service)
         └── Local forwarding (localhost:22, localhost:8080)
```

### Why Cloudflare Tunnel Eliminates Attack Surfaces
* **Zero Open Inbound Ports**: Router firewalls can block all inbound traffic entirely (`DENY INBOUND *`).
* **DDoS Immune Origin**: Since origin has no public IP DNS records, direct IP targeting is impossible.
* **Automatic Failover**: `cloudflared` can run multiple replicas across redundant nodes with automatic load balancing.

---

## 2. Production `config.yml` Specification

```yaml
tunnel: 4c3d2e1a-0000-4444-9999-abcdef123456
credentials-file: /etc/cloudflared/4c3d2e1a-0000-4444-9999-abcdef123456.json

ingress:
  # Internal Web Application
  - hostname: app.yahyaoncloud.com
    service: http://localhost:8080
    originRequest:
      noTLSVerify: false
      connectTimeout: 10s

  # Secure SSH Terminal Access
  - hostname: late5530.yahyaoncloud.com
    service: ssh://localhost:22

  # Internal Prometheus / Grafana Metric Scraper
  - hostname: metrics.yahyaoncloud.com
    service: http://192.168.1.40:9090

  # Default Catch-all Rule (Required)
  - service: http_status:404
```

---

## 3. Client SSH ProxyCommand Setup

To connect to private servers protected by Cloudflare Access over SSH:

```ssh
# ~/.ssh/config
Host late5530
  HostName late5530.yahyaoncloud.com
  User yahya
  IdentityFile ~/.ssh/late5530
  IdentitiesOnly yes
  ProxyCommand cloudflared access ssh --hostname %h
```

When running `ssh late5530`, `cloudflared` initiates the browser authentication flow (or uses active session token) and connects seamlessly through the edge tunnel.

---

## 4. Useful Service Management Commands

```bash
# Check tunnel status
cloudflared tunnel info my-tunnel

# Run tunnel as a systemd service
sudo cloudflared service install
sudo systemctl enable --now cloudflared

# Route internal private CIDR over WARP
cloudflared tunnel route ip add 10.0.0.0/16 my-tunnel
```
