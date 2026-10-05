# Top 10 Core Coding & Algorithmic Patterns for Cloud Engineers

In technical interviews for Cloud, DevOps, and Platform Engineering roles, candidates are increasingly tested on software engineering baselines. You do not need to solve academic graph-coloring puzzles, but you must demonstrate mastery over **10 core algorithmic patterns** directly applicable to log parsing, distributed state reconciliation, rate limiting, and infrastructure automation.

---

#### The 10 Core Algorithmic Patterns

```
+----+------------------------------------+---------------------------------------------------+
| #  | ALGORITHMIC PATTERN                | CLOUD & INFRASTRUCTURE ENGINEERING APPLICATION   |
+----+------------------------------------+---------------------------------------------------+
| 01 | Two Pointers & Sliding Window      | Log spike detection, rate limiting, time windows  |
| 02 | Hash Maps & Frequency Dictionaries | Access log analysis, top offending IPs, quotas    |
| 03 | Interval Merging & Scheduling      | EC2 maintenance windows, deployment blackouts     |
| 04 | String Parsing & Regex Tokenization| CIDR validation, log extraction, URL routing      |
| 05 | Trie Data Structures               | Longest Prefix Match (LPM) in network route tables|
| 06 | Directed Acyclic Graphs (DAG / BFS)| Terraform & CloudFormation dependency resolution  |
| 07 | Stacks & Queues                    | JSON/YAML syntax validation, event worker buffers |
| 08 | Binary Search                      | Fast log timestamp searching, bisecting commits   |
| 09 | Token Bucket & Leaky Bucket        | API throttling, rate limiting, proxy buffering    |
| 10 | Least Recently Used (LRU) Caching  | Edge proxy cache eviction, DNS resolver cache     |
+----+------------------------------------+---------------------------------------------------+
```

---

### Pattern 1: Two Pointers & Sliding Window

* **Cloud Scenario:** You receive a real-time stream of HTTP request timestamps. Determine if an application has exceeded a threshold of $M$ requests in any rolling $W$-second window.
* **Core Technique:** Maintain a `left` pointer and a `right` pointer. Expand `right` to ingest events; advance `left` whenever the time delta between `timestamps[right] - timestamps[left] > W`.
* **Complexity:** $O(N)$ time, $O(1)$ extra space.

```python
def check_rate_limit_spike(timestamps: list[int], window_seconds: int, max_allowed: int) -> bool:
    left = 0
    for right in range(len(timestamps)):
        while timestamps[right] - timestamps[left] > window_seconds:
            left += 1
        if (right - left + 1) > max_allowed:
            return True  # Rate limit exceeded!
    return False
```

---

### Pattern 2: Hash Maps & Frequency Dictionaries

* **Cloud Scenario:** Process 500,000 WAF log entries to find the top $K$ malicious IP addresses attempting SQL injection attacks.
* **Core Technique:** Single-pass hash map frequency counter, followed by a Min-Heap of size $K$ to prevent sorting the entire dictionary in memory.
* **Complexity:** $O(N + U \log K)$ where $U$ is unique IPs.

```python
from collections import Counter
import heapq

def top_k_malicious_ips(logs: list[str], k: int) -> list[str]:
    counts = Counter(logs)
    return heapq.nlargest(k, counts.keys(), key=counts.get)
```

---

### Pattern 3: Interval Merging & Scheduling

* **Cloud Scenario:** AWS schedules automated maintenance reboots for 50 EC2 instances. Given a list of maintenance windows `[start_time, end_time]`, merge all overlapping windows to calculate total operational downtime.
* **Core Technique:** Sort intervals by `start_time`. Compare current interval's start against the previous interval's end time.

```python
def merge_maintenance_windows(windows: list[list[int]]) -> list[list[int]]:
    if not windows:
        return []
    windows.sort(key=lambda x: x[0])
    merged = [windows[0]]
    for start, end in windows[1:]:
        if start <= merged[-1][1]:
            merged[-1][1] = max(merged[-1][1], end)
        else:
            merged.append([start, end])
    return merged
```

---

### Pattern 4: String Parsing & CIDR IP Mathematics

* **Cloud Scenario:** Validate whether a user-provided string is a valid IPv4 CIDR block (e.g. `10.100.0.0/16`) and calculate the total number of usable host addresses.

```python
def parse_and_validate_cidr(cidr: str) -> tuple[bool, int]:
    parts = cidr.split('/')
    if len(parts) != 2:
        return False, 0
    ip_str, prefix_str = parts[0], parts[1]
    if not prefix_str.isdigit() or not 0 <= int(prefix_str) <= 32:
        return False, 0
    prefix = int(prefix_str)
    
    octets = ip_str.split('.')
    if len(octets) != 4:
        return False, 0
    for octet in octets:
        if not octet.isdigit() or not 0 <= int(octet) <= 255:
            return False, 0
        if len(octet) > 1 and octet[0] == '0':
            return False, 0
            
    # Calculate AWS usable IPs: 2^(32 - prefix) - 5 AWS reserved
    total_ips = 2 ** (32 - prefix)
    usable_aws_ips = max(0, total_ips - 5) if prefix <= 28 else 0
    return True, usable_aws_ips
```

---

### Pattern 5: Trie for Longest Prefix Match (LPM)

* **Cloud Scenario:** Implement an in-memory routing table engine. When a packet arrives destined for `10.100.1.50`, query an arbitrary number of CIDR routing rules (`10.0.0.0/8`, `10.100.0.0/16`, `10.100.1.0/24`) and return the route matching the longest prefix.
* **Core Technique:** Binary Radix Tree / Trie where each node represents a bit (`0` or `1`). Lookup traverses bits until the longest matching prefix node is reached.

---

### Pattern 6: Directed Acyclic Graph (DAG) & Topological Sort

* **Cloud Scenario:** Terraform resource dependency graph resolution. Given a set of cloud resources and their dependencies, determine a valid parallelized creation order, or detect circular dependency deadlocks.
* **Core Technique:** Kahn's Algorithm using an in-degree counter and a queue. If nodes processed $< N$, a circular cycle exists!

```python
from collections import deque, defaultdict

def resolve_terraform_dag(resources: int, deps: list[list[int]]) -> list[int]:
    adj = defaultdict(list)
    in_degree = [0] * resources
    for child, parent in deps:
        adj[parent].append(child)
        in_degree[child] += 1
    
    queue = deque([i for i in range(resources) if in_degree[i] == 0])
    build_order = []
    while queue:
        node = queue.popleft()
        build_order.append(node)
        for child in adj[node]:
            in_degree[child] -= 1
            if in_degree[child] == 0:
                queue.append(child)
    if len(build_order) != resources:
        raise ValueError("Circular dependency detected in IaC DAG!")
    return build_order
```

---

### Pattern 7: Stacks for Syntax & Bracket Validation

* **Cloud Scenario:** In a CI/CD pre-commit hook, validate that a custom JSON, YAML, or IAM policy template has balanced brackets, braces, and quotation blocks before uploading to AWS.
* **Core Technique:** Push opening delimiters to a stack; pop and match against closing delimiters.

---

### Pattern 8: Binary Search on Monotonic Data

* **Cloud Scenario:** In an SRE post-mortem investigation, you need to find the exact commit that introduced an error out of 10,000 deployments, or find the line in a 50 GB sorted log file corresponding to timestamp `2026-10-03T14:00:00Z`.
* **Core Technique:** Bisect search range by half every iteration ($O(\log N)$ instead of $O(N)$ linear scan).

---

### Pattern 9: Token Bucket Rate Limiting

* **Cloud Scenario:** Guard a microservice API against sudden surges. Allow bursts up to $C$ requests, but refill capacity steadily at $R$ tokens per second.
* **Core Technique:** Lazy evaluation: compute new tokens based on `now - last_refill_timestamp` without running background timer threads.

---

### Pattern 10: Least Recently Used (LRU) Eviction

* **Cloud Scenario:** Manage an in-memory cache of API session tokens or DNS resolutions. When memory reaches capacity, evict the least recently accessed item in $O(1)$ constant time.
* **Core Technique:** Hash Map for $O(1)$ key lookup + Doubly Linked List for $O(1)$ node removal and head insertion.
