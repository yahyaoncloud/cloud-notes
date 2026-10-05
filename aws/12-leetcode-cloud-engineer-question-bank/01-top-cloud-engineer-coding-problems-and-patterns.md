# Top Cloud Engineer LeetCode Coding Problems & Algorithmic Patterns

Unlike pure software development engineering interviews that focus on esoteric dynamic programming or complex tree rotations, **Cloud, DevOps, and Site Reliability Engineer (SRE) coding rounds** focus heavily on:
1. Log stream processing, sliding windows, and rate calculation.
2. String parsing, IP address validation, and CIDR subnet math.
3. Interval merging (deployment windows, resource reservations).
4. Hash maps and frequency counting (top offending IP addresses, API quotas).
5. Graph traversals (DAGs in Terraform, CloudFormation dependency trees).

Below are the highest-frequency algorithmic problems with cloud engineering contexts, optimized Python solutions, and step-by-step walkthroughs.

---

## Pattern 1: Merge Intervals & Non-Overlapping Scheduling

### Problem 1.1: Merge Disjoint Maintenance Windows (LeetCode 56)
* **Cloud Scenario:** You are given an array of scheduled EC2 maintenance reboot intervals. Overlapping windows must be merged so the SRE team can execute patches in unified downtime blocks.
* **Input:** `intervals = [[1, 3], [2, 6], [8, 10], [15, 18]]`
* **Output:** `[[1, 6], [8, 10], [15, 18]]`

#### Python Solution
```python
def merge_intervals(intervals: list[list[int]]) -> list[list[int]]:
    if not intervals:
        return []

    # Step 1: Sort intervals by start time - O(N log N)
    intervals.sort(key=lambda x: x[0])

    merged = [intervals[0]]

    for current in intervals[1:]:
        prev_start, prev_end = merged[-1]
        curr_start, curr_end = current

        # Overlap condition: Current start is <= previous end
        if curr_start <= prev_end:
            # Merge by extending the end time to the maximum
            merged[-1][1] = max(prev_end, curr_end)
        else:
            merged.append(current)

    return merged

# Time Complexity: O(N log N) due to sorting
# Space Complexity: O(N) to store output
```

---

## Pattern 2: CIDR Block Matching & IP Validation (Trie / Bit Manipulation)

### Problem 2.1: Validate IP Address (IPv4 & IPv6) (LeetCode 468)
* **Cloud Scenario:** A security group automation tool receives user inputs for ingress rules. You must validate whether the incoming string is a valid IPv4 address, IPv6 address, or invalid before passing it to AWS APIs.

#### Python Solution
```python
def valid_ip_address(query_ip: str) -> str:
    def is_ipv4(s: str) -> bool:
        parts = s.split('.')
        if len(parts) != 4:
            return False
        for p in parts:
            # Must be digits only, between 0 and 255, no leading zeros unless single "0"
            if not p.isdigit() or not 0 <= int(p) <= 255:
                return False
            if len(p) > 1 and p[0] == '0':
                return False
        return True

    def is_ipv6(s: str) -> bool:
        parts = s.split(':')
        if len(parts) != 8:
            return False
        hexdigits = "0123456789abcdefABCDEF"
        for p in parts:
            # 1 to 4 hexadecimal characters
            if not 1 <= len(p) <= 4 or not all(c in hexdigits for c in p):
                return False
        return True

    if query_ip.count('.') == 3 and is_ipv4(query_ip):
        return "IPv4"
    if query_ip.count(':') == 7 and is_ipv6(query_ip):
        return "IPv6"
    return "Neither"

# Time Complexity: O(N) where N is length of string
# Space Complexity: O(1)
```

### Problem 2.2: Longest Prefix Match for Network Routing (Trie)
* **Cloud Scenario:** You are implementing a simplified VPC Route Table in software. Given a set of CIDR routing rules and a destination IP, find the route with the Longest Prefix Match (most specific subnet mask).

#### Python Solution
```python
class TrieNode:
    def __init__(self):
        self.children = {}  # '0' or '1'
        self.target_gateway = None

class RouteTableTrie:
    def __init__(self):
        self.root = TrieNode()

    def _ip_to_bits(self, ip_str: str) -> str:
        octets = [int(x) for x in ip_str.split('.')]
        return ''.join(f"{octet:08b}" for octet in octets)

    def insert_route(self, cidr: str, gateway: str):
        ip_str, prefix_len = cidr.split('/')
        prefix_len = int(prefix_len)
        bits = self._ip_to_bits(ip_str)[:prefix_len]

        curr = self.root
        for bit in bits:
            if bit not in curr.children:
                curr.children[bit] = TrieNode()
            curr = curr.children[bit]
        curr.target_gateway = gateway

    def lookup_route(self, dest_ip: str) -> str:
        bits = self._ip_to_bits(dest_ip)
        curr = self.root
        best_gateway = "DEFAULT_DROP"

        for bit in bits:
            if curr.target_gateway is not None:
                best_gateway = curr.target_gateway
            if bit not in curr.children:
                break
            curr = curr.children[bit]

        if curr and curr.target_gateway is not None:
            best_gateway = curr.target_gateway

        return best_gateway

# Usage Example:
# rt = RouteTableTrie()
# rt.insert_route("10.0.0.0/8", "tgw-attachment")
# rt.insert_route("10.100.0.0/16", "local-vpc")
# rt.insert_route("10.100.1.0/24", "subnet-nat-gw")
# print(rt.lookup_route("10.100.1.50")) # Returns "subnet-nat-gw" (longest prefix match /24)
```

---

## Pattern 3: Sliding Window & Rate Calculation

### Problem 3.1: Log Spike Detection over Moving Time Window
* **Cloud Scenario:** In an SRE telemetry service, you receive timestamps of 5xx HTTP errors in seconds. Determine if there is any 60-second window that experienced more than `K` errors.
* **Input:** `timestamps = [10, 15, 25, 45, 65, 70, 72, 75, 80, 120]`, `window_size = 60`, `threshold = 5`

#### Python Solution
```python
def has_error_spike(timestamps: list[int], window_size: int, threshold: int) -> bool:
    left = 0

    # Expand the right pointer across the stream
    for right in range(len(timestamps)):
        # Shrink window from the left if time delta exceeds window_size
        while timestamps[right] - timestamps[left] > window_size:
            left += 1

        # Number of errors within current [left, right] window is (right - left + 1)
        if (right - left + 1) >= threshold:
            return True

    return False

# Time Complexity: O(N) - each index visited at most twice
# Space Complexity: O(1)
```

---

## Pattern 4: Top K Elements & Frequency Analysis

### Problem 4.1: Top K Offending IP Addresses from Access Logs (LeetCode 347)
* **Cloud Scenario:** WAF or CloudFront emits millions of request records. Find the top $K$ most frequent IP addresses to automatically push them to an AWS WAF IP Block Set.

#### Python Solution
```python
from collections import Counter
import heapq

def top_k_frequent_ips(ip_list: list[str], k: int) -> list[str]:
    # Step 1: Count frequencies - O(N)
    counts = Counter(ip_list)

    # Step 2: Use a Min-Heap of size K - O(U log K) where U is unique IPs
    # Storing (frequency, ip)
    min_heap = []

    for ip, freq in counts.items():
        heapq.heappush(min_heap, (freq, ip))
        if len(min_heap) > k:
            heapq.heappop(min_heap)

    # Extract results
    return [ip for freq, ip in min_heap]

# Time Complexity: O(N + U log K)
# Space Complexity: O(U)
```

---

## Pattern 5: Dependency Graph Traversal (DAG / Topological Sort)

### Problem 5.1: Infrastructure Resource Provisioning Order (LeetCode 210)
* **Cloud Scenario:** Terraform or CloudFormation compiles a Directed Acyclic Graph (DAG) of resources where some resources depend on others (e.g., Subnet depends on VPC; EC2 depends on Subnet). Determine a valid provisioning order, or detect circular dependency cycles.

#### Python Solution
```python
from collections import deque, defaultdict

def build_provisioning_order(num_resources: int, dependencies: list[list[int]]) -> list[int]:
    """
    dependencies[i] = [resource_a, resource_b] means resource_a depends on resource_b.
    resource_b must be provisioned before resource_a.
    """
    adj = defaultdict(list)
    in_degree = [0] * num_resources

    for res_a, res_b in dependencies:
        adj[res_b].append(res_a)
        in_degree[res_a] += 1

    # Queue all resources with in_degree == 0 (no prerequisites)
    queue = deque([i for i in range(num_resources) if in_degree[i] == 0])
    provision_order = []

    while queue:
        curr = queue.popleft()
        provision_order.append(curr)

        for neighbor in adj[curr]:
            in_degree[neighbor] -= 1
            if in_degree[neighbor] == 0:
                queue.append(neighbor)

    # If order length doesn't match total resources, a circular cycle exists!
    if len(provision_order) != num_resources:
        raise ValueError("Circular dependency detected! Cannot resolve build order.")

    return provision_order

# Time Complexity: O(V + E)
# Space Complexity: O(V + E)
```
