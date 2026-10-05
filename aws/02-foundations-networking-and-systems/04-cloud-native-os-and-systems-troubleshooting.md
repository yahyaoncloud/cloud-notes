# Cloud-Native Operating Systems & Systems Performance Troubleshooting

When an alert pages you at 2:00 AM because an API latency has jumped from 20ms to 4,000ms, or an EC2 instance becomes unresponsive, you need a deterministic troubleshooting methodology. This guide deconstructs cloud-native operating systems (like AWS Bottlerocket and AL2023) and provides a rigorous, command-by-command triage framework for CPU, memory, disk, and network bottlenecks.

---

#### Explanation & Mentorship Breakdown

##### 1. The Real-World Analogy: The Emergency Room Triage Doctor
When a patient arrives in the ER, doctors do not guess or start random surgeries. They check four vital signs first: Heart Rate, Blood Pressure, Breathing Rate, and Body Temperature.
In Linux and Cloud Engineering, you do not reboot instances randomly. You check the vital signs of the machine:
1. **CPU:** Is the brain overwhelmed with math, or is it waiting in line?
2. **Memory:** Are applications suffocating and about to be executed by the kernel's Out-Of-Memory (OOM) killer?
3. **Storage (Disk I/O):** Is the disk waiting on physical hardware or IOPS credit exhaustion?
4. **Network:** Are packets being dropped at the interface, blocked by a firewall, or stuck in connection state buffers?

---

##### 2. Plain-English Breakdown: What is Actually Happening?

### Cloud-Native Operating Systems: AL2023 vs. Bottlerocket

In the early days of cloud, engineers launched full-featured general-purpose operating systems (Ubuntu Desktop or bulky RHEL) containing package managers, compilers, and GUI libraries. In modern cloud architecture:

```
+------------------------------------+------------------------------------+
| AMAZON LINUX 2023 (AL2023)         | AWS BOTTLEROCKET                   |
+------------------------------------+------------------------------------+
| General-purpose cloud OS           | Purpose-built container host OS    |
| RPM/DNF package management         | No package manager (rpm/dnf)       |
| Mutable root filesystem (read/write)| Read-only root filesystem (verity) |
| Standard SSH / SSM administration  | Admin container via API / SSM only |
| Uses systemd and SysV init scripts | Minimal init system written in Rust|
| Ideal for: EC2 apps, microservices | Ideal for: Amazon EKS & ECS hosts  |
+------------------------------------+------------------------------------+
```

#### Why Bottlerocket is Revolutionizing Container Security:
* **Immutability:** Attackers cannot install rootkits, download malicious curl binaries to `/tmp`, or alter kernel binaries because `/` is mounted read-only and cryptographically verified using `dm-verity`.
* **Zero Shell:** The host OS does not even have an interactive bash shell or python interpreter. If you need to debug a Bottlerocket node, you spawn an ephemeral, privileged "Admin Container" via SSM.

---

### Systems Triage Framework: The USE Method

Created by systems performance pioneer Brendan Gregg:
* **U**tilization: Percentage of time that the resource was busy (e.g., 90% CPU busy).
* **S**aturation: The degree to which extra work is queued waiting for the resource (e.g., Load Average > number of CPU cores).
* **E**rrors: Count of error events (e.g., dropped packets on `eth0`).

---

### Triage Category 1: Compute & CPU Bottlenecks

#### Symptoms:
* High Load Average reported by `uptime` or `top`.
* CloudWatch CPUUtilization at 99%.

#### Step-by-Step Investigation:
1. Check the Load Average:
   ```bash
   uptime
   # 19:30:01 up 45 days, 4 users, load average: 8.50, 4.20, 1.10
   ```
   * *The Aha Rule:* On a 4-core machine, a Load Average of 4.0 means 100% capacity. A Load Average of 8.5 means 4 processes are running and 4.5 processes are stalled waiting for CPU or uninterruptible disk I/O!
2. Inspect Process States in `top`:
   * Look at `%Cpu(s)`:
     * `us` (User): Application code (e.g., bad regex, infinite loop).
     * `sy` (System): Kernel overhead (high context switching or system calls).
     * `wa` (iowait): CPU is idle because it is waiting for disk/EBS or network I/O.
     * `st` (Steal): **The Cloud Hypervisor Steal Time!** If `st` is high (> 5%), the physical AWS host is oversubscribed, or you are running a burstable `t3/t4g` instance that has exhausted its CPU credits.
3. Identify top CPU consuming processes:
   ```bash
   ps aux --sort=-%cpu | head -n 10
   ```

---

### Triage Category 2: Memory Starvation & The OOM Killer

#### Symptoms:
* Applications randomly crash or exit with Code 137 (`SIGKILL`).
* Free memory approaches zero.

#### Step-by-Step Investigation:
1. Check available RAM:
   ```bash
   free -m
   ```
   * *Crucial Distinction:* In Linux, "Free" memory is almost always low because Linux uses spare RAM for caching (`buff/cache`). Look at the **`available`** column. That represents memory that can be reclaimed immediately without swapping.
2. Check if the Linux Kernel Out-Of-Memory Killer terminated processes:
   ```bash
   dmesg -T | grep -i -E "oom|out of memory|killed process"
   ```
   * Output example: `Out of memory: Kill process 14201 (java) score 850 or sacrifice child`.
   * *Resolution:* Increase container memory limits, increase EC2 instance size, or configure a swapfile/tune `vm.swappiness`.

---

### Triage Category 3: Disk I/O & Inode Exhaustion

#### Symptoms:
* Disk write fails with `"No space left on device"`, but `df -h` shows only 40% used.
* Extreme response latency with high `%wa` in CPU metrics.

#### Step-by-Step Investigation:
1. Check Inode Exhaustion:
   ```bash
   df -i /
   ```
   * Every file on Linux requires an inode (metadata record). If a microservice creates millions of 1-byte session files or temporary logs, inodes hit 100% even if 50 GB of disk remains free!
2. Check Disk Latency & IOPS Saturation:
   ```bash
   iostat -xz 1 5
   ```
   * Watch `%util` (percentage of time the disk is servicing requests) and `r_await` / `w_await` (read/write latency in milliseconds). If `await` exceeds 15-20ms, your EBS volume has exhausted its IOPS or throughput burst baseline.

---

### Triage Category 4: Network & Socket Bottlenecks

#### Symptoms:
* Web requests time out with `504 Gateway Timeout` or `Connection refused`.
* Socket allocation errors: `Cannot assign requested address`.

#### Step-by-Step Investigation:
1. Check socket states using `ss`:
   ```bash
   ss -s
   # Summary: Total: 45000, TCP: 44000 (estab 1200, closed 42000, time-wait 41000)
   ```
   * *The TIME_WAIT Storm:* If thousands of sockets sit in `TIME_WAIT`, your client application is making outbound HTTP calls without connection pooling (closing TCP connections after every single request), exhausting the ephemeral port range (`1024-65535`).
2. Verify local listening ports:
   ```bash
   ss -tlpn
   ```
3. Capture live packets on the wire:
   ```bash
   # Sniff HTTP traffic on eth0 to verify if packets arrive at the host
   tcpdump -nn -i eth0 port 80 -c 10
   ```

---

##### 3. Visual Flow Diagram: Comprehensive Systems Triage Decision Tree

```
                                [ INCIDENT DETECTED ]
                                          |
                        +-----------------+-----------------+
                        | Run: uptime; top; free -m; df -h  |
                        +-----------------+-----------------+
                                          |
        +------------------+--------------+---------------+------------------+
        |                  |                              |                  |
        v                  v                              v                  v
  [ High CPU ]      [ Low Memory ]                 [ High %wa ]      [ Network Stalls ]
        |                  |                              |                  |
+-------+-------+  +-------+-------+              +-------+-------+  +-------+-------+
| Check 'st'    |  | Check dmesg   |              | Run: iostat -x|  | Run: ss -s    |
| Steal > 5%:   |  | Look for OOM: |              | await > 20ms: |  | High TIME_WAIT|
| T-instance    |  | JVM/Python    |              | EBS IOPS      |  | Connection    |
| CPU credits   |  | leak killed   |              | exhausted     |  | pool missing  |
| exhausted!    |  | by kernel     |              |               |  |               |
+---------------+  +---------------+              +---------------+  +---------------+
```

---

##### 4. The "Aha!" Moment / Why People Get Confused

* **The T3 CPU Credit Cliff:** A company runs its backend on `t3.medium` instances. During a traffic surge, everything runs fast for 20 minutes, then suddenly crawls to a halt with 100% CPU utilization.
  * *Why:* T-series instances are burstable. They consume CPU credits when running above baseline. Once credits hit zero, the AWS hypervisor **throttles the CPU clock cycle to 20% baseline capacity**, causing catastrophic queuing.
  * *Fix:* Enable `Unlimited` CPU credit mode on the ASG, or upgrade to dedicated compute instances (`c6i` / `c6g`).

---

##### 5. Quick Hands-On Check

Run a 1-line systems health snapshot summarizing CPU, memory, run queues, and I/O wait in real-time:

```bash
# Print system metrics every 1 second for 5 iterations
vmstat 1 5
```
