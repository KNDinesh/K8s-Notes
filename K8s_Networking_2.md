Kubernetes networking stops being a “black box” once you visualize **what actually happens step-by-step when a packet travels**. I’ll walk you through the system from the ground up, focusing on:

  1. Why Kubernetes networking exists

  2. What objects are involved

  3. What happens when a Pod is created

  4. Pod → Pod communication

  5. Pod → Service communication

  6. External → Cluster communication

  7. Where CNI, kube-proxy, and NetworkPolicies fit

Think of this as a **mental model of the entire networking pipeline**.

**1. The Core Philosophy of Kubernetes Networking**

*Kubernetes networking was designed around one very important rule:*

  | Every Pod gets its own IP address and behaves like a small machine on the network.

This is very different from Docker’s default model.

**Old Docker model**

Multiple containers share the same machine IP.

*Example:*

    Machine IP: 192.168.1.10
    
    Container A → Port 8080
    Container B → Port 8081
    Container C → Port 8082

*Problems:*

  * Port conflicts

  * Harder scaling

  * Harder service discovery

  * Applications must support configurable ports

**Kubernetes Model**

Each Pod has its own IP.

    Pod A → 10.244.1.5:80
    Pod B → 10.244.1.6:80
    Pod C → 10.244.2.3:80

*Now:*

  * All apps can run on their default ports

  * Pods behave like independent machines

  * Networking becomes simpler for developers

**2. Kubernetes Networking Requirements (Important)**

Kubernetes enforces 4 networking guarantees.

1️⃣ **Every Pod has a unique IP**

Across the entire cluster.

    Pod A → 10.244.1.5
    Pod B → 10.244.2.9

No duplicates.

2️⃣ **Pods can communicate without NAT**

Pod → Pod communication uses real IPs.

*Example:*

    Pod A (10.244.1.5) → Pod B (10.244.2.9)

No translation needed.

3️⃣ **Nodes can talk to Pods**

*Nodes need this for:*

  * health checks

  * logs

  * exec

  * probes

4️⃣ **Pods see their own IP**

*Inside the container:*

    hostname -i

Returns the **same Pod IP**.

**3. What Objects Are Involved in Kubernetes Networking?**

There are several key components.

**Kubernetes Components**

**Component**	                **Role**
Pod	                          Basic network unit
Node	                        Physical/virtual machine
Service	                      Stable access to Pods
kube-proxy	                  Traffic routing
NetworkPolicy	                Security rules

**Networking Components**

**Component**	                **Role**
CNI Plugin	                  Creates Pod networking
IPAM	                        Manages IP addresses
Overlay Network	              Connects nodes
Linux Networking	            veth pairs, bridges

**Popular CNI plugins**

  * Calico

  * Flannel

  * Cilium

  * Weave Net

Each implements Kubernetes networking rules differently.

**4. What Happens When a Pod Is Created (Networking Setup)**

This is where CNI comes in.

Let’s walk through the exact process.

**Step 1 — Pod Scheduled to Node**

The scheduler decides:

    Pod A → Node 2

**Step 2 — kubelet Creates Pod**

*The kubelet on Node 2:*

  * pulls container image

  * creates container

  * sets up networking

But kubelet does not implement networking.

Instead it calls the **CNI plugin**.

**Step 3 — kubelet Calls CNI**

Kubelet executes a CNI command.

*Example:*

    CNI ADD

*This tells the plugin:*

  | Create networking for this Pod.

**Step 4 — CNI Assigns IP**

The plugin asks **IPAM** for an address.

*Example:*

    Node CIDR: 10.244.2.0/24
    
    Available IPs:
    10.244.2.2
    10.244.2.3
    10.244.2.4

*Pod gets:*

    10.244.2.3

**Step 5 — Linux Network Interfaces Created**

Now Linux networking is used.

Two interfaces are created.

**veth pair**

Think of it like a virtual ethernet cable.

    Pod side        Node side
    eth0  <------>  veth123

*One end in:*

  * container namespace

*Other end in:*

  * node namespace

**Step 6 — Connect to Bridge**

The node usually has a Linux bridge.

*Example:*

    cni0

*Network layout:*

    Pod
     |
    eth0
     |
    veth pair
     |
    cni0 bridge
     |
    Node network

**5. Pod to Pod Communication (Same Node)**

*Example:*

    Pod A → 10.244.1.5
    Pod B → 10.244.1.6

Both on **Node 1**.

*Traffic flow:*

    Pod A
      |
    eth0
      |
    veth pair
      |
    cni0 bridge
      |
    veth pair
      |
    eth0
      |
    Pod B

Just like **two machines on a switch**.

**6. Pod to Pod Communication (Different Nodes)**

*Example:*

    Pod A (Node1) → 10.244.1.5
    Pod B (Node2) → 10.244.2.9

Now traffic must cross nodes.

Two models exist.

**Overlay Networking (Most Common)**

* Used by plugins like:*

  * Flannel

  * Weave Net

Pods have virtual IPs.

Traffic is encapsulated.

**Packet Journey**

*Pod A sends:*

    src: 10.244.1.5
    dst: 10.244.2.9

Node 1 wraps packet inside another packet.

    outer src: Node1 IP
    outer dst: Node2 IP

Like an **envelope**.

*Node2 receives it:*

  * unwraps packet

  * forwards to Pod B

**Direct Routing (More Advanced)**

*Used by:*

  * Calico

  * Cilium

*Instead of encapsulation:*

Nodes update routing tables.

*Example:*

    Route:
    10.244.2.0/24 → Node2

So Node1 sends traffic directly.

*Advantages:*

  * Faster

  * Less overhead

**7. The Problem With Pod IPs**

Pods are **ephemeral**.

*If a Pod restarts:*

    Old IP → 10.244.1.5
    New IP → 10.244.1.9

Clients break.

**8. Services Solve This Problem**

Kubernetes **Service** provides a stable address.

*Example:*

    Service IP: 10.96.0.10

*Behind it:*

    Pod A
    Pod B
    Pod C

*Traffic flow:*

    Client → Service → Pod

**9. kube-proxy: The Traffic Director**

Each node runs kube-proxy.

It programs Linux networking rules.

*Methods:*

  * iptables

  * IPVS

  * eBPF (modern)

*Example rule:*

    10.96.0.10:80 → PodA
    10.96.0.10:80 → PodB
    10.96.0.10:80 → PodC

Traffic is load balanced.

**10. Types of Services**

**ClusterIP (default)**

Internal only.

    Pod → Service → Pod

**NodePort**

Opens a port on every node.

*Example:*

    NodeIP:30080

External users can access.

**LoadBalancer**

Used in cloud providers.

*Example:*

  * Amazon Web Services

  * Microsoft Azure

  * Google Cloud Platform

*Cloud creates:*

    External IP → NodePort → Pods

**11. Ingress (Layer 7 Routing)**

*Ingress handles:*

  * HTTP

  * HTTPS

  * domain routing

*Example:*

    api.example.com → API service
    shop.example.com → Shop service

*Ingress controllers:*

  * NGINX Ingress Controller

  * Traefik

**12. Network Policies (Security)**

*Default Kubernetes:*

    ALL PODS CAN TALK

Very open.

NetworkPolicies restrict traffic.

*Example:*

*Allow:*

    frontend → backend

*Deny:*

    frontend → database

Policies are label based.

*Example:*

    allow pods with label:
    app=frontend

**13. The Complete Traffic Flow (External User → Pod)**

Let’s trace a real request.

*User opens:*

    https://api.example.com

*Flow:*

    User
     ↓
    Cloud LoadBalancer
     ↓
    NodePort
     ↓
    Ingress Controller
     ↓
    Service
     ↓
    kube-proxy
     ↓
    Pod
    
**14. Full Mental Model**

*Think of Kubernetes networking layers like this:*

    Internet
       |
    LoadBalancer
       |
    Ingress
       |
    Service
       |
    kube-proxy
       |
    CNI Network
       |
    Pod

**15. The Linux Networking Pieces Underneath**

*Under the hood Kubernetes uses:*

  * network namespaces

  * veth pairs

  * bridges

  * iptables

  * routing tables

  * VXLAN / tunnels

*Which means Kubernetes networking is really:*

  | Linux networking + automation.

**16. The Modern Evolution: eBPF**

Newer networking stacks use **eBPF**.

*Example:*

  * Cilium

*Benefits:*

  * faster routing

  * advanced observability

  * better security

  * no kube-proxy needed

**Final Intuition**

*Imagine a Kubernetes cluster as:*

    Cluster = Virtual Data Center
    Nodes   = Servers
    Pods    = Micro machines
    CNI     = Network wiring
    Service = Virtual load balancer
    Ingress = API gateway
    Policies = Firewall
