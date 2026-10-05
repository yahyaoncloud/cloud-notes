# Cloudflare Workers, Pages, and WAF Security

Serverless edge computing with Google V8 isolates and application security at Cloudflare's edge.

---

## 1. Workers vs. Containers / VMs

| Architecture | Cold Start | Memory Footprint | Runtime Isolation |
| :--- | :--- | :--- | :--- |
| **Traditional VM** | Minutes | 1 - 4 GB OS overhead | Hardware virtualization (Hypervisor) |
| **Container (Docker)**| 1 - 5 seconds | 100 - 500 MB container layers | Linux cgroups & namespaces |
| **Cloudflare Workers** | **0 milliseconds** | **< 5 MB** | **Google V8 Isolates** |

Workers execute JavaScript/TypeScript, WebAssembly, and Python inside V8 isolates across hundreds of worldwide data centers instantly on incoming requests.

---

## 2. Production Worker Implementation (TypeScript)

```typescript
export interface Env {
  AUTH_KV: KVNamespace;
  API_SECRET: string;
}

export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    const url = new URL(request.url);

    // Edge Authentication Check
    const token = request.headers.get("Authorization");
    if (!token) {
      return new Response(JSON.stringify({ error: "Unauthorized" }), {
        status: 401,
        headers: { "Content-Type": "application/json" }
      });
    }

    // Geolocation Routing
    const country = request.cf?.country || "US";
    const cacheKey = new Request(url.toString(), request);
    const cache = caches.default;

    let response = await cache.match(cacheKey);
    if (!response) {
      response = await fetch(request);
      response = new Response(response.body, response);
      response.headers.set("Cache-Control", "s-maxage=3600");
      ctx.waitUntil(cache.put(cacheKey, response.clone()));
    }

    return response;
  }
};
```

---

## 3. Web Application Firewall (WAF) Rule Expressions

Cloudflare WAF uses the Wireshark-like wirefilter expression syntax:

```text
# Block suspicious traffic targeting sensitive admin paths
(http.request.uri.path contains "/admin" or http.request.uri.path contains "/wp-login.php")
and not (ip.src in { 203.0.113.50 198.51.100.0/24 })
and cf.threat_score gt 10

# Rate Limit API endpoints (e.g. max 100 requests per minute per IP)
http.request.uri.path eq "/api/v1/auth/login"
```
