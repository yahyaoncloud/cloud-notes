# SSL/TLS Handshake, Cryptographic PKI & AWS Certificate Manager (ACM)

In modern enterprise architectures, plaintext HTTP is completely forbidden. All traffic in transit must be encrypted. To design, troubleshoot, and defend secure networks, a cloud engineer must understand how Public Key Infrastructure (PKI) validates trust and how TLS handshakes establish encrypted tunnels.

---

#### Explanation & Mentorship Breakdown

##### 1. The Real-World Analogy: The Embassy Seal and the Locked Padlock
Imagine you want to buy secret blueprints from a company in a foreign country you have never visited:
* **The Problem:** Anyone could intercept your letters, or an imposter could pretend to be that company.
* **The Government Passport / Embassy Seal (Public Key Infrastructure & Certificate Authority):** The company doesn't ask you to trust them blindly. They present an official document signed by an internationally recognized government agency (**Certificate Authority like DigiCert or Let's Encrypt**). Because your laptop comes pre-installed with the government's official stamp (**Root CA Trust Store**), you can verify their certificate is genuine.
* **The Asymmetric Padlock:** The company sends you an open brass padlock (**Public Key**). Anyone can see it. You write your secret communication codebook on a napkin, put it inside a strong metal box, click their padlock shut, and mail it back.
* Only the company has the unique physical brass key (**Private Key**) that can open that padlock.
* Once they open the box and read your secret codebook, you both switch to communicating via that shared secret code (**Symmetric Session Key**), which is thousands of times faster than shipping heavy padlocks.

---

##### 2. Plain-English Breakdown: What is Actually Happening?

### Cryptography: Asymmetric vs. Symmetric

```
+------------------------------------+------------------------------------+
| ASYMMETRIC ENCRYPTION (Public Key) | SYMMETRIC ENCRYPTION (Shared Key)  |
+------------------------------------+------------------------------------+
| Uses a Key Pair (Public + Private) | Uses a Single Shared Secret Key    |
| Public encrypts, Private decrypts  | Same key encrypts and decrypts     |
| Computationally heavy (CPU tax)    | Hardware accelerated (ultra-fast)  |
| Algorithms: RSA (2048/4096), ECDSA | Algorithms: AES-256-GCM, ChaCha20  |
| Purpose: Identity & Key Exchange   | Purpose: Bulk Payload Encryption   |
+------------------------------------+------------------------------------+
```

### The TLS 1.2 vs. TLS 1.3 Handshake

TLS 1.3 (RFC 8446) fundamentally overhauled transport security by eliminating vulnerable legacy ciphers (RC4, 3DES, CBC mode) and cutting connection setup latency in half:

```
          TLS 1.2 (2-RTT) Handshake                       TLS 1.3 (1-RTT) Handshake
=============================================   =============================================
CLIENT                            SERVER        CLIENT                            SERVER
  |                                 |             |                                 |
  | -------- ClientHello ---------> |             | --- ClientHello + Key Share --> |
  |                                 |             |                                 |
  | <------- ServerHello ---------- |             | <-- ServerHello + Key Share --- |
  | <------- Certificate ---------- |             | <-- Encrypted Server Cert ----- |
  | <------- ServerKeyExchange ---- |             | <-- Finished ------------------ |
  | <------- ServerHelloDone ------ |             |                                 |
  |                                 |             | === [ENCRYPTED DATA FLOWS] ===> |
  | -------- ClientKeyExchange ---> |
  | -------- ChangeCipherSpec ----> |
  | -------- Finished ------------> |
  |                                 |
  | <------- ChangeCipherSpec ----- |
  | <------- Finished ------------- |
  |                                 |
  | === [ENCRYPTED DATA FLOWS] ===> |
```

#### Why TLS 1.3 is Superior:
1. **Latency reduction (1-RTT vs 2-RTT):** The client guesses the server's elliptic curve and sends its Diffie-Hellman key share in the very first `ClientHello`. The server responds with its key share and immediately sends encrypted application data.
2. **Mandatory Perfect Forward Secrecy (PFS):** Static RSA key exchanges are completely removed. Every session generates temporary (ephemeral) keys. Even if an attacker steals the server's private key 5 years later, past captured traffic cannot be decrypted!
3. **Zero-RTT (0-RTT) Resumption:** Returning visitors can send encrypted data in the first packet using pre-shared keys.

---

### The PKI Certificate Chain of Trust

How does your browser know `https://aws.amazon.com` is legitimate?

```
[ Root CA: Starfield Services Root Certificate Authority - G2 ]
   | (Stored permanently in OS / Browser Trust Store)
   v [Digitally Signs Intermediate Certificate]
[ Intermediate CA: Amazon RSA 2048 M02 ]
   | (Issued by AWS PKI team)
   v [Digitally Signs Leaf / End-Entity Certificate]
[ Leaf Server Certificate: *.aws.amazon.com ]
   | (Presented by AWS Application Load Balancer / CloudFront)
```

If any link in this chain has an invalid signature, an expired date, or a mismatched domain name, the browser aborts with `SEC_ERROR_UNKNOWN_ISSUER` or `NET::ERR_CERT_COMMON_NAME_INVALID`.

---

### AWS Certificate Manager (ACM) Deep Dive

ACM provides free, automatically renewed SSL/TLS certificates for AWS services.

#### 1. Validation Methods:
* **DNS Validation (Best Practice):** ACM generates a unique `CNAME` record (e.g., `_a7b8c...example.com` pointing to `_x9y2z...acm-validations.aws.`). You add this record to Route 53 or your DNS provider. As long as this CNAME exists, ACM automatically renews the certificate 60 days before expiration with zero manual intervention.
* **Email Validation (Legacy / Discouraged):** Sends approval emails to `admin@example.com`, `postmaster@...`. Requires a human to click a link annually.

#### 2. The Core ACM Constraint:
> **ACM Private Keys CANNOT Be Exported.**
> You cannot download an ACM certificate's private key to install on an NGINX server or an EC2 instance.
> ACM certificates can **only** be deployed to AWS managed termination endpoints:
> * Elastic Load Balancing (ALB / NLB)
> * Amazon CloudFront distributions
> * AWS API Gateway custom domains
> * AWS App Runner

*If you need certificates directly on an EC2 virtual machine, you must use Let's Encrypt (Certbot) or AWS Private CA.*

---

### Mutual TLS (mTLS) for Zero-Trust

In standard TLS, only the **server** proves its identity to the client. The server doesn't know who the client is at the transport layer.
In **mTLS (Mutual TLS)**:
* The server presents its certificate to the client (Server Authentication).
* The client **must also** present an X.509 certificate to the server (Client Authentication).
* If the client certificate is missing or not signed by the trusted Root CA, the connection is instantly severed before application code ever runs.
* Used heavily in banking APIs, IoT device connections (AWS IoT Core), and Kubernetes Service Meshes (Istio / Linkerd).

---

##### 3. Visual Flow Diagram: mTLS Authentication Handshake

```
CLIENT (Microservice A)                               SERVER (Microservice B / ALB)
      |                                                             |
      | ---------------------- ClientHello -----------------------> |
      |                                                             |
      | <--- ServerHello + Server Certificate + Cert Request ------ |
      |                                                             |
      | [Validates Server Cert against Root CA]                     |
      |                                                             |
      | --- Client Certificate + CertificateVerify (Signature) ---> |
      |                                                             |
      |                                 [Validates Client Cert]     |
      | <----------------------- Finished ------------------------- |
      |                                                             |
      | <============ Mutual Encrypted Secure Channel ============> |
```

---

##### 4. The "Aha!" Moment / Why People Get Confused

* **"Why is my CloudFront certificate throwing an error even though ACM says Issued?"**
  * *The Trap:* ACM is a regional service, **EXCEPT for CloudFront**.
  * Amazon CloudFront is a global service. To attach a custom SSL certificate to a CloudFront distribution, the certificate **MUST be requested in the `us-east-1` (N. Virginia) region**. If you request the ACM certificate in `eu-west-1` or `ap-southeast-1`, it will simply not show up in the CloudFront console dropdown!

---

##### 5. Quick Hands-On Check

Test and inspect the live TLS certificate chain, cipher suite, and protocol version of any cloud endpoint using `openssl s_client`:

```bash
# Connect and print full TLS handshake, certificate chain, and negotiated cipher
openssl s_client -connect aws.amazon.com:443 -servername aws.amazon.com -brief
```

Output highlights:
```text
CONNECTION ESTABLISHED
Protocol version: TLSv1.3
Ciphersuite: TLS_AES_128_GCM_SHA256
Peer certificate: CN = *.aws.amazon.com
Verification: OK
```
