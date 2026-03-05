**I have a deployment.yaml which contains a replicaset whose desired state is 4 pods. If I am using Kubectl apply what is the workflow, how the call happens to each component?**

When you run:

    kubectl apply -f deployment.yaml

and your Deployment specifies 4 replicas, here’s the exact workflow inside Kubernetes and how each component is involved.

🔁 High-Level Flow

    kubectl → kube-apiserver → etcd
                             ↓
                      Deployment Controller
                             ↓
                      ReplicaSet Controller
                             ↓
                         Scheduler
                             ↓
                          kubelet
                             ↓
                      Container Runtime

Step-by-Step Breakdown
1️⃣ kubectl (Client Side)

  1. kubectl reads deployment.yaml

  2. Sends a REST API request to the kube-apiserver

  3. Uses PATCH (for apply) with declarative configuration

  4. Authentication & TLS happen here

No pods are created yet.

2️⃣ kube-apiserver

Component: Kubernetes API Server

The API server:

  1. Validates the YAML

  2. Runs admission controllers (Authentication → Authorization → Admission → Persist to etcd)

  3. Stores the object in etcd

  4. Returns success to kubectl

At this point:
✔ Deployment object exists
❌ No Pods yet

3️⃣ etcd

Component: etcd

  1. Stores the Deployment object

  2. Acts as the source of truth

4️⃣ Deployment Controller

**Component: Kubernetes Deployment Controller**

Running inside:

    kube-controller-manager

What happens:

  1. Watches for new Deployments

  2. Sees desired replicas = 4

  3. Creates a ReplicaSet

  4. Sets owner reference

Now:

✔ ReplicaSet created

❌ Pods not yet created

5️⃣ ReplicaSet Controller

**Component: Kubernetes ReplicaSet Controller**

  1. Watches ReplicaSets

  2. Sees desired replicas = 4

  3. Creates 4 Pod objects

Now:

✔ 4 Pod objects exist in etcd

❌ Pods not scheduled yet

6️⃣ Scheduler

Component: kube-scheduler

  1. Watches for unscheduled Pods

  2. Chooses best nodes

  3. Updates Pod .spec.nodeName

Now:

✔ Pods assigned to nodes

❌ Containers not running yet

7️⃣ kubelet (On Each Node)

**Component: kubelet**

On each selected node:

  1. Watches for Pods assigned to it

  2. Pulls container images

  3. Calls container runtime

  4. Reports status back to API server

8️⃣ Container Runtime

**Example: containerd**

  1. Creates containers

  2. Starts containers

  3. Manages lifecycle

Now:

✔ 4 Pods running

To expose the application to the outside world, we need to have an object called "service" and the type can be a NodePort (if Kubernetes specific) and LoadBalancer (Cloud Provider).
