# Systems, Concurrency & Infrastructure Algorithms Question Bank

In advanced technical interview rounds, candidates are frequently asked to implement foundational distributed system primitives in code. This question bank focuses on the four most common systems algorithms asked by cloud, platform, and infrastructure teams:
1. The Token Bucket API Rate Limiter
2. The Least Recently Used (LRU) Cache
3. Thread-Safe Worker Pool & Task Pipeline
4. Producer-Consumer Queue with Backpressure

---

## 1. The Token Bucket Rate Limiter

### Real-World Cloud Context
AWS API Gateway, Envoy proxy, and NGINX all use variants of the Token Bucket algorithm to protect backend microservices against denial-of-service spikes and enforce customer API tier throttling (e.g., 100 requests/sec with a burst of 20).

### Python Implementation

```python
import time
import threading

class TokenBucketRateLimiter:
    def __init__(self, capacity: int, refill_rate_per_sec: float):
        """
        :param capacity: Maximum number of tokens the bucket can hold (burst limit).
        :param refill_rate_per_sec: How many tokens are added to the bucket every second.
        """
        self.capacity = capacity
        self.refill_rate = refill_rate_per_sec
        self.tokens = float(capacity)
        self.last_refill_timestamp = time.time()
        self.lock = threading.Lock()

    def _refill(self):
        now = time.time()
        time_elapsed = now - self.last_refill_timestamp
        # Add newly accrued tokens based on time passed
        new_tokens = time_elapsed * self.refill_rate
        self.tokens = min(float(self.capacity), self.tokens + new_tokens)
        self.last_refill_timestamp = now

    def allow_request(self, tokens_required: int = 1) -> bool:
        """Atomically checks if enough tokens are available."""
        with self.lock:
            self._refill()
            if self.tokens >= tokens_required:
                self.tokens -= tokens_required
                return True
            return False

# Usage Demonstration:
if __name__ == '__main__':
    limiter = TokenBucketRateLimiter(capacity=5, refill_rate_per_sec=2.0)
    
    # Burst 5 requests immediately
    for i in range(7):
        allowed = limiter.allow_request(1)
        print(f"Request {i+1}: {'ALLOWED (200 OK)' if allowed else 'THROTTLED (429 Too Many Requests)'}")
```

### Interview Discussion
* **Lazy Refill vs Background Thread:** Notice that we do not have a background thread constantly ticking every second. Instead, we calculate tokens dynamically based on `elapsed_time = now - last_timestamp` only when a request arrives. This avoids wasting CPU cycles when the service is idle.

---

## 2. Least Recently Used (LRU) Cache (LeetCode 146)

### Real-World Cloud Context
Edge proxies (CloudFront, Squid, NGINX) maintain an in-memory cache of static web assets or DNS query responses. When memory is full, the least recently requested asset must be evicted in $O(1)$ time.

### Python Implementation (Doubly Linked List + Hash Map)

```python
class Node:
    def __init__(self, key: str, val: str):
        self.key = key
        self.val = val
        self.prev = None
        self.next = None

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = {}  # key -> Node
        # Dummy head and tail to eliminate null pointer checks
        self.head = Node("HEAD", "HEAD")
        self.tail = Node("TAIL", "TAIL")
        self.head.next = self.tail
        self.tail.prev = self.head

    def _remove(self, node: Node):
        """Remove a node from the linked list."""
        prev_node = node.prev
        next_node = node.next
        prev_node.next = next_node
        next_node.prev = prev_node

    def _add_to_front(self, node: Node):
        """Insert node right after dummy head (most recently used)."""
        node.next = self.head.next
        node.prev = self.head
        self.head.next.prev = node
        self.head.next = node

    def get(self, key: str) -> str:
        if key not in self.cache:
            return "-1"
        node = self.cache[key]
        # Move to front (mark as recently used)
        self._remove(node)
        self._add_to_front(node)
        return node.val

    def put(self, key: str, val: str):
        if key in self.cache:
            self._remove(self.cache[key])
        
        new_node = Node(key, val)
        self._add_to_front(new_node)
        self.cache[key] = new_node

        if len(self.cache) > self.capacity:
            # Evict least recently used (node right before tail)
            lru_node = self.tail.prev
            self._remove(lru_node)
            del self.cache[lru_node.key]

# Time Complexity: O(1) for both get() and put()
# Space Complexity: O(capacity)
```

---

## 3. Producer-Consumer Pipeline with Backpressure

### Real-World Cloud Context
Decoupled asynchronous worker nodes polling an SQS queue or Kafka partition. When downstream database writes slow down, the consumer pipeline must apply backpressure to prevent out-of-memory crashes.

### Python Implementation

```python
import queue
import threading
import time
import random

def producer(q: queue.Queue, total_items: int):
    """Simulates message ingestion from an external webhook or queue."""
    for i in range(total_items):
        item = f"task-payload-{i}"
        # q.put() blocks when queue reaches maxsize (Backpressure!)
        q.put(item)
        print(f"[PRODUCER] Queued: {item} (Buffer size: {q.qsize()})")
        time.sleep(random.uniform(0.01, 0.05))
    
    # Poison pill to signal worker termination
    q.put(None)

def consumer(q: queue.Queue, worker_id: int):
    """Simulates background processor handling work items."""
    while True:
        item = q.get()
        if item is None:
            q.task_done()
            break  # Exit signal received
        
        print(f"  [WORKER-{worker_id}] Processing: {item}")
        time.sleep(random.uniform(0.05, 0.15))  # Simulate DB write or processing
        q.task_done()
        print(f"  [WORKER-{worker_id}] Completed: {item}")

if __name__ == '__main__':
    # Bounded buffer of size 5 forces backpressure on producer
    buffer = queue.Queue(maxsize=5)
    
    t_prod = threading.Thread(target=producer, args=(buffer, 10))
    t_cons = threading.Thread(target=consumer, args=(buffer, 1))

    t_cons.start()
    t_prod.start()

    t_prod.join()
    buffer.join()
    t_cons.join()
    print("All tasks processed successfully.")
```
