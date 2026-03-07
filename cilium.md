**1. What is eBPF?**

  eBPF (Extended Berkeley Packet Filter) is a technology in the Linux kernel that allows programs to run inside the kernel safely.

Normally:

  * The kernel runs in kernel space

    => Kernel space is like the engine room of a ship. Only trusted engineers can enter because mistakes there can stop the entire ship.

  * Applications run in user space

    => User space is like passengers on the ship. They cannot directly operate the engine; they must request services from the crew.

        Application (User Space)
            |
            | system call (read, write, open)
            v
        Kernel (Kernel Space)
            |
            v
        Hardware (disk, network, CPU)
    
=> User programs cannot normally run inside the kernel because a crash would crash the entire OS.

=> eBPF changes this by allowing small, verified programs to run in the kernel safely.

**Why is this powerful?**

Because the kernel is where:

  * networking

  * file systems

  * processes

  * security

  * scheduling

are handled.

So with eBPF you can observe and control system behavior at the deepest level.

Simple Analogy

=> Imagine the kernel is an airport control tower.

=> Normally you can only watch airplanes from outside.

=> With eBPF you can install a small monitoring robot inside the control tower that:

  * observes flights

  * logs events

  * blocks suspicious activity

  * routes traffic

But the robot is strictly verified before entering.

**2. Why is eBPF safe?**

Before an eBPF program runs, the eBPF Verifier checks it.

**The verifier ensures the program:*

**1. Cannot run forever**

No infinite loops.

Example (not allowed):

    while(true) {
    }

This would hang the kernel.

**2. Cannot access invalid memory**

The program cannot:

  * read random memory

  * corrupt kernel memory

**3. Cannot crash the kernel**

**The verifier ensures:*

  * safe memory access

  * bounded execution

  * predictable behavior

Only verified programs are allowed into the kernel.

**3. What is Cilium?**

Cilium is a cloud-native networking, security, and observability platform for Kubernetes that uses eBPF.

**It replaces traditional networking tools like:*

  * iptables

  * kube-proxy

  * some service mesh proxies

with high-performance kernel-level networking using eBPF.

It is widely used in Kubernetes clusters.

**4. Is “native kernel speed” true?**

Yes — mostly true.

**Traditional networking:*

    Pod → iptables rules → kernel routing → destination

**Cilium:*

    Pod → eBPF program in kernel → destination

**Because eBPF runs inside the kernel, it avoids:*

  * multiple rule traversals

  * context switching

  * large iptables rule chains

**So networking is:*

  * faster

  * lower latency

  * lower CPU usage

That’s what people mean by “native kernel speed.”

**5. How Cilium Works (Step-by-Step)**

**Step 1 — Kubernetes creates a Pod*

The Kubelet runs on every Kubernetes node.

**When a Pod is scheduled:*

    Scheduler → Node
               → Kubelet

Kubelet creates the container.

But the container needs networking.

**Step 2 — Kubelet calls the CNI plugin*

Kubernetes networking uses Container Network Interface (CNI) plugins.

**Examples:*

  * Flannel

  * Calico

  * Cilium

**Kubelet asks Cilium:*

"Give networking to this Pod."

**Cilium then:*

  * creates virtual network interface

  * assigns IP

  * connects Pod to cluster network

**Step 3 — Cilium assigns an Identity*

Cilium assigns every workload a security identity.

**Example:*

    frontend pod → ID 1001
    backend pod → ID 1002
    database pod → ID 1003

Policies can then reference identities instead of IP addresses.

This solves a big Kubernetes problem because pod IPs constantly change.

**Step 4 — Cilium attaches an eBPF program*

  * Cilium attaches an eBPF program to the Pod's network namespace.

  * This program runs in the kernel network stack.

**It can observe events like:**

**Example events:*

    Pod A → opened TCP socket
    Pod A → sending packet to Pod B
    Pod A → sending traffic outside cluster
    Pod A → DNS query

This enables observability.

**Why Observability becomes powerful**

**Traditional networking tools show:*

    IP → IP

**Cilium can show:*

    Pod → Pod
    Process → Destination
    Service → Service

Because it observes traffic at kernel level.

**6. Network Policy Enforcement**

Cilium can enforce security policies using eBPF.

**Example policy:*

    frontend → backend : allowed
    frontend → database : denied

Every packet triggers eBPF evaluation.

Packet vs Session (important concept)

**Traditional firewall:*

    Evaluate connection session

**Cilium:*

    Evaluate each packet

**Example:*

  * HTTP request contains multiple packets.

  * Cilium can inspect each packet in real time.

  * This allows fine-grained control.

**7. Socket Layer Load Balancing**

**Traditional Kubernetes networking:*

    Pod → Node → iptables → Pod

**iptables must:*

  * evaluate thousands of rules

  * traverse rule chains

This slows down large clusters.

**With Cilium:*

    Pod → eBPF → Pod

Load balancing happens at the socket layer inside kernel.

**This avoids:*

  * iptables

  * extra hops

**Result:*

  * faster communication

  * lower CPU usage

8. Kube-Proxy Replacement

Normally Kubernetes uses kube-proxy.

**kube-proxy implements services like:*

  * ClusterIP

  * NodePort

**It does this using:*

  * iptables

  * or IP Virtual Server (IPVS)

Problem:

**In large clusters:*

    Thousands of services
    → thousands of rules
    → constant updates

This causes slow convergence.

**What Cilium does instead**

Cilium implements service routing using eBPF maps.

**Advantages:**

**Lower CPU*

No massive iptables rule chains.

**Zero convergence time*

Updating a map entry is instant.

**Atomic updates*

Changes happen instantly without rebuilding rule chains.

**9. Additional Networking Features in Cilium**

**Cilium can also handle:**

**Load Balancing*

Internal and external traffic balancing.

**BGP Support**

**Border Gateway Protocol*

Used by routers to advertise networks.

**Example:*

    Kubernetes Service IP
    → advertised to data center routers

This allows external systems to reach services directly.

**IPv6 support**

Modern networking support.

**Egress Gateway**

Control outbound traffic from pods.

**Example:*

    Pod → Internet

You may want traffic to appear from one specific IP.

Cilium can route it through an egress node.

**10. Cilium as a Service Mesh**

**Traditional service meshes:*

  * Istio

  * Linkerd

These use sidecar proxies.

**Example:*

    Pod
     ├ app container
     └ proxy container

Every request goes through the proxy.

**Problems:*

  * extra CPU

  * extra memory

  * extra latency

**Cilium Service Mesh (Sidecar-less)**

Cilium moves the logic into eBPF in the kernel.

**So instead of:*

    Pod → proxy → network

**You get:*

    Pod → kernel eBPF → network

**Result:*

  * faster

  * simpler

  * less overhead

**11. Encryption**

In regulated environments (finance, healthcare), traffic must be encrypted.

Cilium supports two types.

**1. Node-to-Node encryption**

**Using:*

  * WireGuard

  * IPsec

Traffic between nodes is encrypted.

**Example:*

    Node A → Node B

All pod traffic between them is encrypted automatically.

**2. Mutual Authentication**

This is similar to mTLS.

**mTLS normally does:*

    Authentication + Encryption

Cilium can separate these two.

**Benefits:*

  * works for more protocols

  * can meet FIPS compliance

**12. Advanced Load Balancing**

Cilium supports the Kubernetes Gateway API.

**Example:**

**You want:*

    90% traffic → v1
    10% traffic → v2

**This enables:*

  * Canary deployments

  * A/B testing

**13. Layer 7 (Application Layer) Policies**

**Networking layers:*

    L3 → IP
    L4 → TCP/UDP
    L7 → Application (HTTP, gRPC)

Cilium can enforce L7 policies.

**Example:**

**Allow:*

    GET /products

**Block:*

    DELETE /products

So even though traffic is allowed, specific API actions can be blocked.

**14. Observability (One of Cilium's biggest strengths)**

Cilium's observability tool is Hubble.

**It can show:*

    Process → Pod → Service → Destination

**Example log:*

    Process: python
    Pod: payment-service
    Destination: database
    Protocol: TCP
    Result: Allowed

**Traditional service meshes only see:*

    Pod → Pod

**But Cilium sees:*

    Process → Pod → Network

because it observes inside the kernel.

**Real Example (Putting Everything Together)**

**Suppose:*

    frontend pod
    backend pod
    database pod

**User request flow:*

    User
     ↓
    Frontend Pod
     ↓
    Backend Pod
     ↓
    Database

**Cilium does:**

1️⃣ Assign identity to each pod
2️⃣ Attach eBPF programs
3️⃣ Monitor traffic
4️⃣ Enforce security policy
5️⃣ Load balance requests
6️⃣ Provide observability logs

**Example event log:*

    frontend → backend
    HTTP GET /products
    Result: Allowed
    Latency: 12ms

**Final Intuition**

**Think of Cilium as:*

A kernel-powered networking brain for Kubernetes.

**Traditional approach:*

    iptables + kube-proxy + service mesh + monitoring

**Cilium approach:*

    eBPF in kernel
    → networking
    → security
    → observability
    → load balancing

All handled efficiently in the kernel.

✅ **One sentence summary**

Cilium uses eBPF inside the Linux kernel to provide fast networking, security policies, observability, and service mesh capabilities for Kubernetes clusters.
