# How the Internet Works: The Packet Journey, DNS Resolution & Web Browsing

"What happens when you type `https://www.example.com` into your browser and press Enter?" is the single most famous interview question across networking, cloud, and systems engineering. This guide breaks down every nanosecond of that journey from client cache to AWS load balancer.

---

#### Explanation & Mentorship Breakdown

##### 1. The Real-World Analogy: Ordering a Custom Book via Postal Mail
Imagine wanting to read a specific book from a boutique publisher across the ocean:
1. You look in your personal notebook (**Browser & OS Cache**) to see if you already know their physical street address.
2. If not, you ask your local post office clerk (**Local Recursive DNS Resolver**).
3. The clerk doesn't know the exact street, so they check the international postal registry (**Root DNS Server**), which directs them to the Country Department (**TLD Server for `.com`**).
4. The Country Department gives them the address of the Publisher's Private Post Office (**Authoritative DNS Server / Route 53**), which gives the exact GPS street coordinates (**IP Address: 93.184.216.34**).
5. Before sending an order, you send a courier to ring the doorbell and establish a formal greeting: "Hello, are you there?" -> "Yes, ready to receive!" -> "Great, let's talk" (**TCP 3-Way Handshake**).
6. Because spies might read your letters, you and the publisher agree on a secret disposable codebook through a locked glass case (**TLS Handshake & Symmetric Session Key**).
7. Finally, you send the encrypted order form: "Send me the front page" (**HTTP GET Request**).

---

##### 2. Plain-English Breakdown: What is Actually Happening?

### Step-by-Step Chronological Journey

```
+----------------------------------------------------------------------------------------------------+
|                               THE 7-PHASE PACKET LIFECYCLE                                         |
+----------------------------------------------------------------------------------------------------+
| 1. BROWSER & OS RESOLUTION   | Checks Browser DNS Cache -> OS DNS Cache -> /etc/hosts file.        |
| 2. RECURSIVE DNS RESOLUTION  | Recursive Resolver -> Root (.) -> TLD (.com) -> Route 53 (Auth NS)  |
| 3. LOCAL ARP & L2 FRAMING    | OS resolves Local Gateway MAC via Address Resolution Protocol (ARP).|
| 4. TCP 3-WAY HANDSHAKE       | Client and Server exchange SYN -> SYN-ACK -> ACK on Port 443.       |
| 5. TLS 1.3 SECURE HANDSHAKE  | Asymmetric Key Exchange (ECDHE) creates ephemeral Symmetric Session |
| 6. HTTP REQUEST & EDGE FETCH | Client sends HTTP/2 GET. Hits CloudFront CDN or ALB Origin.         |
| 7. DOM RENDERING             | Browser parses HTML, downloads CSS/JS, executes V8 JavaScript engine|
+----------------------------------------------------------------------------------------------------+
```

#### Step 1: The Local Client Cache Check
* When you hit Enter, the browser checks its internal DNS cache (Chrome: `chrome://net-internals/#dns`).
* If not cached, the browser queries the OS stub resolver (`getaddrinfo` in Linux C library).
* The OS checks `/etc/hosts` (or `C:\Windows\System32\drivers\etc\hosts`) and the local OS resolver cache (`systemd-resolved` or `dnsmasq`).

#### Step 2: The Hierarchical DNS Resolution
If the local cache misses, a UDP packet is sent to the configured Recursive Resolver (e.g., your home router, corporate DNS, or public resolvers like `8.8.8.8` / `1.1.1.1`):
1. **Root Nameserver (`.`):** There are 13 logical root server addresses globally (A through M, replicated via Anycast BGP). The Root server responds: *"I don't know `www.example.com`, but here are the authoritative nameservers for `.com`."*
2. **Top-Level Domain (TLD) Nameserver (`.com`):** Run by registry operators like Verisign. The TLD server responds: *"I don't know the IP, but here are the authoritative nameservers for `example.com` (e.g., `ns-123.awsdns.com`)."*
3. **Authoritative Nameserver (Route 53):** Holds the actual DNS Hosted Zone. It looks up the `A` record (IPv4) or `AAAA` record (IPv6) and returns: `www.example.com IN A 93.184.216.34` with a Time-To-Live (TTL).
4. The recursive resolver caches this record and returns the IP address to your computer.

#### Step 3: Layer 2 Framing & ARP Resolution
* Your computer knows the destination IP (`93.184.216.34`), but it cannot send an Ethernet frame directly across the internet. It must send the frame to its **Default Gateway** (your local router).
* If the Default Gateway's MAC address is missing from the local ARP cache (`ip neigh`), your host broadcasts an ARP Request: *"Who has IP 192.168.1.1? Tell 192.168.1.50."*
* The router responds with its physical MAC address. The packet is encapsulated into an Ethernet frame and transmitted out the physical NIC/Wi-Fi antenna.

#### Step 4: Transport Layer - The TCP 3-Way Handshake
To establish a reliable, stateful stream on port 443:
1. **SYN:** Client sends a TCP packet with the `SYN` (Synchronize) control flag set and an initial Sequence Number $ISN_c$.
2. **SYN-ACK:** Server allocates a Transmission Control Block (TCB) buffer and responds with `SYN` and `ACK` flags set, acknowledging $ISN_c + 1$ and providing its own sequence number $ISN_s$.
3. **ACK:** Client responds with `ACK` acknowledging $ISN_s + 1$. The connection state transitions to `ESTABLISHED`.

```
CLIENT (Browser)                                         SERVER (AWS ALB)
      |                                                        |
      | ----------------- [1] TCP SYN (Seq=X) ---------------> |
      |                                                        |
      | <---------------- [2] TCP SYN-ACK (Seq=Y, Ack=X+1) --- |
      |                                                        |
      | ----------------- [3] TCP ACK (Ack=Y+1) -------------> | (Connection Established)
      |                                                        |
```

#### Step 5: Cryptographic Security - The TLS 1.3 Handshake
In modern TLS 1.3, encryption happens in a single round trip (1-RTT):
1. **ClientHello:** Client sends supported cipher suites and its public key share using Elliptic Curve Diffie-Hellman Ephemeral (ECDHE).
2. **ServerHello + EncryptedExtensions + Certificate + Finished:** Server chooses cipher suite, responds with its ECDHE public key share, presents its X.509 SSL certificate, and sends a cryptographic signature.
3. Both sides combine the exchanged public keys with their private keys to mathematically compute the **identical symmetric session key** (AES-256-GCM or ChaCha20-Poly1305). All subsequent communication is encrypted.

#### Step 6: Application Layer - HTTP Request & Edge Routing
* The browser sends an encrypted `HTTP/2 GET /` request.
* If routed via **Amazon CloudFront**, edge servers inspect the cache. If cached, it returns the payload in 5 milliseconds.
* If a cache miss occurs, CloudFront routes the request over AWS's private optical fiber backbone to the origin **Application Load Balancer (ALB)**, which terminates TLS and distributes the request to backend **ECS Tasks** or **EC2 instances**.

---

##### 3. Visual Flow Diagram: The Complete Packet Traversal

```
[USER LAPTOP]
    |
    v (1) Check Local / Browser DNS Cache -> MISS
    |
    v (2) UDP Port 53 Query to Recursive Resolver (8.8.8.8)
    |     |--> Root NS (.) -> TLD NS (.com) -> Authoritative NS (Route 53)
    |     `-- Returns: 93.184.216.34 (TTL 300s)
    |
    v (3) Check ARP Table for Default Gateway MAC -> Wrap Frame
    |
    +====> [Local Router] ===> [ISP Backbone] ===> [AWS Edge Border Router]
                                                              |
                                                              v
                                              [AWS CloudFront / ALB]
                                                              | (4) TCP 3-Way Handshake
                                                              | (5) TLS 1.3 Handshake
                                                              | (6) HTTP/2 GET Request
                                                              v
                                              [Private App Tier / EC2]
```

---

##### 4. The "Aha!" Moment / Why People Get Confused

* **"Does my computer query the Root DNS server directly?"**
  * *Confusion:* Many junior engineers answer: "My browser asks the Root server."
  * *Correction:* Your computer is a **stub resolver**. It only asks your designated **Recursive Resolver** (like your ISP DNS or `1.1.1.1`). It is the recursive resolver that undertakes the multi-step journey of querying Root, TLD, and Authoritative servers on your behalf.
* **Why do we use Asymmetric encryption to establish Symmetric encryption?**
  * Asymmetric encryption (RSA / ECC) is mathematically expensive and CPU-intensive. Symmetric encryption (AES) is hardware-accelerated by modern CPU instruction sets (AES-NI) and thousands of times faster. We use asymmetric cryptography for 5 milliseconds only to safely exchange a temporary symmetric key!

---

##### 5. Quick Hands-On Check

Trace the exact hierarchical DNS query chain from Root servers down to Route 53 using `dig +trace`:

```bash
# Trace full recursive delegation path for a domain
dig +trace www.amazon.com
```

Inspect the local system's active TCP connections and ARP cache:

```bash
# View active TCP sockets on standard web ports
ss -tuna '( dport = :443 or sport = :443 )'

# View local default gateway MAC resolution in ARP cache
ip neigh show
```
