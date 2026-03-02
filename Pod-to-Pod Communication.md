1️⃣ First Principle: Every Pod Gets Its Own IP

In Kubernetes, every Pod gets its own IP address.

  * Not the node IP.

  * Not NAT.

  * Not port mapping.

A pod is treated like a tiny VM on the network.

This is possible because of the **CNI (Container Network Interface)** plugin.

**Examples:**

  * Calico

  * Flannel

  * Cilium

These plugins create a **flat cluster network**.

**Meaning:**

  * Pod A can directly talk to Pod B using its IP.

  * No NAT required.

  * No port forwarding.

That’s the Kubernetes networking model.

2️⃣ What Actually Happens When a Pod Is Created?

**When a pod starts:**

1. The kubelet asks the CNI plugin to attach networking.

2. The CNI plugin:

  * Creates a **veth pair** (virtual ethernet cable).

  * One end goes inside the pod’s network namespace.

  * The other end attaches to a bridge on the node.

3. The pod is assigned:

  * An IP from the cluster CIDR.

  * A route to reach other pod networks.

So technically:

  * Pods are isolated using Linux network namespaces.

  * But connected using virtual interfaces.

This is Linux networking, not magic.

3️⃣ **Pod-to-Pod Communication (Same Node)**

**If two pods are on the same node:**

    Pod A → virtual bridge → Pod B

Traffic never leaves the node.

**It goes through:**

  * veth

  * Linux bridge (or eBPF if using Cilium)

  * Target pod interface

Fast. Simple.

4️⃣ **Pod-to-Pod Communication (Different Nodes)**

Now the important part.

**If Pod A is on Node 1 and Pod B is on Node 2:**

The CNI plugin handles routing between nodes.

**How?**

Depends on the CNI:

**Overlay Model (Flannel style)**

  * Encapsulates traffic in VXLAN

  * Wraps packet inside another packet

  * Sends over node network

**Routing Model (Calico style)**

  * Uses BGP routing

  * No encapsulation (if configured that way)

  * Nodes know routes to other pod CIDRs

**eBPF Model (Cilium)**

  * Uses eBPF for fast routing

  * Avoids iptables overhead

The key idea:

👉 The node network knows how to reach the other node’s pod CIDR.

That’s it.

5️⃣ **What About Services?**

Now here’s where people get confused.

Pods are ephemeral. IPs change.

So we don’t usually call pods directly.

We use a Service.

**When you create a Service:**

  * It gets a ClusterIP.

  * kube-proxy installs iptables rules.

  * Traffic to the Service IP gets load-balanced to backend pods.

**Internally:**

  * DNS resolves service name to ClusterIP.

  * iptables (or IPVS) rewrites destination to one of the pod IPs.

No external load balancer required for internal traffic.

6️⃣ **DNS in the Picture**

**Kubernetes runs:**

  * CoreDNS

**When Pod A calls:**

    http://backend-service

**Steps:**

  1. DNS resolves service name.

  2. Returns ClusterIP.

  3. kube-proxy forwards traffic to one backend pod.

DNS is critical for service discovery.

7️⃣ **What Can Block Pod-to-Pod Traffic?**

This is where production breaks.

By default:

  * All pods can talk to all pods.

Unless:

**1. Network Policies exist**

These are enforced by CNI.

If you use Calico/Cilium, policies are enforced at dataplane level.

**2. Cloud Security Groups block node traffic**

In AWS:

  * If node security groups don’t allow node-to-node traffic,

  * Pods across nodes won’t talk.

**3. Misconfigured CIDR overlap**

If pod CIDR overlaps with VPC CIDR:

You will break routing.

8️⃣ Important Things Most People Miss

🔴 **Pod IPs are NOT stable**

They change if pod restarts.

🔴 **Services don’t proxy traffic themselves**

kube-proxy manipulates iptables.

🔴 **There is no Docker bridge in modern Kubernetes**

CRI is used. Container runtime doesn’t handle cluster networking.

🔴 **CNI choice affects performance heavily**

Overlay adds overhead.

eBPF reduces it.

9️⃣ **Mental Model (Simple but Accurate)**

Think like this:

  * Each pod = mini server with its own IP

  * CNI = network engineer wiring everything

  * Service = stable front door

  * kube-proxy = traffic cop

  * CoreDNS = phonebook

That’s the whole system.
