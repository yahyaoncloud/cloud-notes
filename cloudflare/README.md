# Cloudflare Edge Architecture & Security Notes

Comprehensive documentation for Cloudflare Anycast CDN, Zero Trust, Cloudflare Tunnels, and Edge Compute.

---

## Navigation & Topics

* [01. Anycast DNS & Edge Caching Architecture](./01-dns-and-cdn-architecture.md)
* [02. Zero Trust & SASE Architecture](./02-zero-trust-and-sase.md)
* [03. Cloudflare Tunnels (cloudflared) & Ingress Management](./03-cloudflare-tunnels-cloudflared.md)
* [04. Workers, Edge Computing & Web Application Firewall (WAF)](./04-workers-pages-and-waf.md)

---

## Core Focus Areas
* Zero-trust security and identity-aware proxies eliminating traditional bastion hosts.
* Outbound encrypted tunnels with `cloudflared` for direct homelab and VPC connectivity.
* Global edge performance via Anycast DNS, tiered caching, and Argo Smart Routing.
* High-concurrency V8 isolate edge functions and custom WAF rate limiting.
