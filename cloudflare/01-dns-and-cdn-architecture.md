# Cloudflare DNS, Edge Caching & Anycast Network Architecture

Cloudflare operates an Anycast network across 300+ cities globally, terminating TLS, inspecting traffic, and serving cached assets within milliseconds of end-users.

---

## 1. Global Anycast DNS & Proxy Architecture

```text
End User Client
      │ (DNS Lookup for api.yahyaoncloud.com)
      ▼
Cloudflare Anycast PoP (Closest Geolocation)
      │
      ├── [Grey Cloud: DNS Only] ──── Direct IP Resolution ────> Origin Server IP
      │
      └── [Orange Cloud: Proxied]
            ├── DDoS Mitigation & L3/L4 Flood Defense
            ├── WAF & Bot Analysis
            ├── Edge TLS Termination (Full Strict SSL)
            ├── Edge Cache Lookup (HIT -> Returns directly)
            └── (MISS) ──── Argo Smart Routing ────────────> Origin Server (via Tunnel/mTLS)
```

### DNS Records: Proxied vs. DNS-Only
* **Orange Cloud (Proxied)**: Masked origin IP. Traffic routes through Cloudflare reverse proxy. Enables WAF, caching, DDoS protection, and SSL termination.
* **Grey Cloud (DNS Only)**: Direct DNS resolution. Required for non-HTTP services that do not run behind Cloudflare Spectrum or Cloudflare Tunnel (e.g. standard mail MX, custom VPN endpoints).

---

## 2. SSL/TLS Encryption Modes

| Mode | Visitor to Cloudflare | Cloudflare to Origin | Security Rating | Use Case |
| :--- | :--- | :--- | :--- | :--- |
| **Off** | HTTP | HTTP | Insecure (Never use) | Legacy testing |
| **Flexible** | HTTPS | HTTP (Unencrypted) | Vulnerable to MITM | Only if origin cannot run TLS |
| **Full** | HTTPS | HTTPS (Self-signed allowed) | Medium | Origin has self-signed cert |
| **Full (Strict)** | HTTPS | HTTPS (Valid CA / CF Origin CA) | **Production Standard** | Enterprise production setups |

---

## 3. Edge Cache Rules & Cache Control

Cache is controlled via HTTP headers or Cloudflare Cache Rules:

```http
Cache-Control: public, max-age=86400, s-maxage=604800, stale-while-revalidate=3600
```
* `max-age`: Browser cache duration.
* `s-maxage`: Cloudflare edge cache duration.
* `stale-while-revalidate`: Serve cached content to user while asynchronously fetching fresh content from origin in background.
