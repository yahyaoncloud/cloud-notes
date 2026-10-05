# Elastic Load Balancing (ALB, NLB, GWLB) & AWS WAF Integration

Elastic Load Balancing (ELB) automatically distributes incoming application traffic across multiple targets (Amazon EC2 instances, containers, IP addresses, and Lambda functions) in multiple Availability Zones. Choosing between an Application Load Balancer (ALB), Network Load Balancer (NLB), or Gateway Load Balancer (GWLB) and securing it with AWS WAF is a central theme in cloud architecture interviews.

---

#### Explanation & Mentorship Breakdown

##### 1. The Real-World Analogy: Different Concierges for Different Venues
* **Application Load Balancer (ALB - The Multilingual Hotel Concierge):**
  * Reads the envelope of your letter, opens the letter, reads the language and request headers.
  * If the letter says `/api/payments`, they deliver it to the Finance Office. If it says `/images`, they deliver it to the Media Storage team.
  * They can inspect cookies to ensure returning guests always talk to the same receptionist (**Sticky Sessions**), and they check passports (**SSL/TLS Termination**).
* **Network Load Balancer (NLB - The High-Speed Automated Rail Switch):**
  * Doesn't care what is written inside the cargo boxes; doesn't speak HTTP.
  * Only looks at the barcode label: Destination Port 5432 or UDP Port 53.
  * Operates at blistering sub-millisecond speeds, handles millions of requests per second, and provides a fixed, permanent GPS street address (**Static Anycast Public IP per AZ**) that never changes.
* **Gateway Load Balancer (GWLB - The Airport Luggage X-Ray Conveyor Belt):**
  * Transparently routes every suitcase through a bank of specialized 3rd-party inspection scanners (**Fortinet / Palo Alto virtual firewalls**) without modifying the original luggage tags (**GENEVE Encapsulation**), and seamlessly returns clean bags to the passenger terminal.

---

##### 2. Plain-English Breakdown: What is Actually Happening?

### Architectural Comparison: ALB vs. NLB vs. GWLB

```
+--------------------------+--------------------------+--------------------------+
| APPLICATION LOAD BALANCER| NETWORK LOAD BALANCER    | GATEWAY LOAD BALANCER    |
| (ALB)                    | (NLB)                    | (GWLB)                   |
+--------------------------+--------------------------+--------------------------+
| OSI Layer: Layer 7 (App) | OSI Layer: Layer 4 (Xport)| OSI Layer: Layer 3 / 4   |
| Protocols: HTTP, HTTPS,  | Protocols: TCP, UDP, TLS | Protocol: GENEVE         |
| HTTP/2, gRPC             |                          | (UDP Port 6081)          |
| Latency: ~5 - 15 ms      | Latency: < 1 ms (Ultra)  | Latency: Low             |
| Static IP: NO (Uses DNS) | Static IP: YES (1 Elastic| Uses private VPC Endpoint|
|                          | IP per Availability Zone)| Service routing          |
| Client IP Preservation:  | Client IP Preservation:  | Client IP Preservation:  |
| Uses X-Forwarded-For     | Native (Preserves source | Encapsulated in GENEVE   |
| HTTP request header      | IP without proxy header) | packet header            |
| Routing Rules: Path,     | Routing Rules: Port and  | Routing Rules:           |
| Host, Query Param, Header| Protocol match ONLY      | All traffic inspection   |
| WAF Integration: Direct  | WAF Integration: Direct  | WAF Integration: N/A     |
| (AWS WAF Web ACL)        | (ALB or CloudFront best) | (Uses 3rd party firewalls|
+--------------------------+--------------------------+--------------------------+
```

---

### Application Load Balancer Routing Capabilities

An ALB is powered by a high-performance routing engine capable of evaluating complex conditions:

```
[ Incoming HTTPS Request to: https://api.example.com/v1/orders?priority=high ]
                                    |
                                    v
                       [ ALB Listener: Port 443 ]
                                    |
     +------------------------------+------------------------------+
     |                              |                              |
     v (Rule 1: Host-based)         v (Rule 2: Path-based)         v (Rule 3: Query Param)
Host = 'api.example.com'     Path = '/v1/orders'            Param 'priority' = 'high'
     |                              |                              |
     +------------------------------+------------------------------+
                                    | (All conditions matched)
                                    v
                    [ Target Group: TG-HighPriorityOrders ]
                                    |
                    +---------------+---------------+
                    |                               |
                    v                               v
          [ ECS Fargate Task A ]          [ ECS Fargate Task B ]
```

#### Advanced ALB Actions:
* **Redirects:** Automatically redirect HTTP Port 80 to HTTPS Port 443 with a 301 Permanent Redirect without touching backend servers.
* **Fixed Responses:** Return a custom JSON 403 Forbidden or 200 OK maintenance page directly from the load balancer.
* **Weighted Target Groups:** Route 90% of traffic to Target Group V1 and 10% to Target Group V2 for seamless **Canary Deployments**.

---

### AWS WAF (Web Application Firewall) Integration

AWS WAF attaches directly to your ALB, CloudFront distribution, or API Gateway. It inspects Layer 7 HTTP payloads to block malicious exploits before they reach compute instances:

```
[ INCOMING TRAFFIC ]
        |
        v
[ AWS WAF Web ACL ]
  |-- Rule 1: AWSManagedRulesCommonRuleSet (OWASP Top 10: SQLi, XSS, Local File Inclusion)
  |-- Rule 2: AWSManagedRulesKnownBadInputsRuleSet (Log4j, Remote Code Execution)
  |-- Rule 3: Rate-Based Rule: Block any client IP exceeding 2,000 requests in 5 minutes!
  `-- Rule 4: Geo-Match Rule: Block or challenge traffic from non-serviceable countries.
        |
   +----+----+
   |         |
[ BLOCK ] [ ALLOW ]
 (403)       |
             v
       [ ALB Target Group ]
```

---

##### 3. Visual Flow Diagram: Client IP Preservation (ALB vs. NLB)

```
SCENARIO A: Application Load Balancer (ALB)
[Client: 203.0.113.10] ===> [ALB: Translates Source IP to 10.100.1.50] ===> [EC2 Instance]
* The EC2 instance sees the incoming TCP connection originating from the ALB's private IP!
* To see the real client, EC2 must inspect the HTTP header:
  X-Forwarded-For: 203.0.113.10
  X-Forwarded-Proto: https

SCENARIO B: Network Load Balancer (NLB) with Target Type = 'instance'
[Client: 203.0.113.10] ===> [NLB] ===> [EC2 Instance sees Source IP = 203.0.113.10!]
* NLB preserves the original client IP directly in the TCP IP header!
* No header parsing required.
```

---

##### 4. The "Aha!" Moment / Why People Get Confused

* **"Why can't I assign a static Elastic IP to an Application Load Balancer?"**
  * *The Trap:* An ALB is designed to dynamically scale its underlying nodes based on traffic volume. Its IP addresses change constantly; you must always point DNS records to its **CNAME / Alias DNS name**, never hardcoded IPs!
  * *The Solution if clients demand static IP whitelisting:* Place an **AWS Global Accelerator** or a **Network Load Balancer (NLB)** in front of the ALB. The NLB provides 2 fixed Elastic IPs and forwards traffic directly to the ALB!

---

##### 5. Quick Hands-On Check

Query the health of all target instances in an ALB Target Group using the AWS CLI:

```bash
# Describe target health and inspect reason codes for any unhealthy instances
aws elbv2 describe-target-health \
    --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/web-tg/0123456789abcdef \
    --query "TargetHealthDescriptions[*].{Target:Target.Id,Port:Target.Port,State:TargetHealth.State,Reason:TargetHealth.Reason}" \
    --output table
```
