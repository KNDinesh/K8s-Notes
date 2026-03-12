**API Server**

**The Kubernetes API Server works as a secure REST gateway + state manager:**

  1. Receives REST request

  2. Authenticates user

  3. Authorizes permissions

  4. Runs admission controllers

  5. Validates and converts object

  6. Stores state in **etcd**

  7. Notifies watchers

  8. Controllers reconcile cluster state

**Internal Architecture:**

                         Clients (kubectl, apps)
                                  │
                                  ▼
                         Kubernetes API Server
                         ─────────────────────
                         Authentication
                         Authorization
                         Admission Controllers
                         API Validation
                         Version Conversion
                                  │
                                  ▼
                             Storage Layer
                                  │
                                  ▼
                                 etcd
                                  │
                                  ▼
                           Watch / Event Stream
                                  │
               ┌──────────────────┼──────────────────┐
               ▼                  ▼                  ▼
          Scheduler        Controller Manager      Kubelet

**etcd**

The **etcd** datastore is a **strongly consistent distributed key-value store** that acts as the **persistent backend for Kubernetes**.

It ensures that all cluster state is **reliably stored, replicated, and synchronized** across nodes.

**Core Internal Workflow**

  1. The **Kubernetes API Server** sends a read/write request to etcd via **gRPC API**.

  2. The request is forwarded to the **etcd leader node**.

  3. The leader proposes the change through the **Raft consensus algorithm**.

  4. The leader replicates the change to follower nodes.

  5. Once a **majority of nodes acknowledge**, the change is committed.

  6. The committed entry is applied to the **state machine**.

  7. Data is persisted in the storage engine (**BoltDB**).

  8. etcd increments the **global revision number**.

  9. Watchers receive update events for the changed keys.

**Internal Architecture**

                            Client
                              │
                              ▼
                        gRPC API Layer
                              │
                              ▼
                        Request Validation
                              │
                              ▼
                          Raft Module
                ┌─────────────────────────┐
                │  Leader Election        │
                │  Log Replication        │
                │  Commit Mechanism       │
                └─────────────────────────┘
                              │
                              ▼
                       State Machine
                              │
                              ▼
                       Storage Engine
                           (BoltDB)
                              │
                              ▼
                          Disk Files
                              │
                              ▼
                         Watch System
                              │
                              ▼
                       Client Notifications

**Controller Manager**

The **Kubernetes Controller Manager** runs a collection of controllers that continuously monitor resources in Kubernetes and reconcile differences between the desired state and the actual cluster state.

**Core Workflow**

  1. Controllers watch resources through the **Kubernetes API Server** using informers.

  2. Resource changes trigger events added to controller work queues.

  3. Controllers process queue items through reconciliation loops.

  4. The controller compares desired and actual state.

  5. If a difference exists, the controller updates the API server.

  6. The API server persists changes in **etcd**.

  7. Updated state triggers new watch events.

**Internal Architecture**

                             Kubernetes Controller Manager
                         ─────────────────────────────────────
                                 Controller Factory
                                        │
                                        ▼
                              Shared Informer Cache
                                        │
                        ┌───────────────┼───────────────┐
                        ▼               ▼               ▼
                  ReplicaSet Ctrl   Node Ctrl      Job Ctrl
                        │               │               │
                        ▼               ▼               ▼
                      Work Queue      Work Queue      Work Queue
                        │
                        ▼
                   Reconciliation Logic
                        │
                        ▼
                   Update API Server
                        │
                        ▼
                        etcd

**Kubernetes Scheduler**

The **Kubernetes Scheduler** is a control-plane component in **Kubernetes** responsible for assigning **unscheduled Pods to the most appropriate Node** in the cluster.

It continuously watches the **kube-apiserver** for Pods that do not yet have a node assigned and selects a node based on resource availability, constraints, and scheduling policies.

**Core Workflow**

  1. The scheduler watches for Pods without a **nodeName** through the Kubernetes API Server.

  2. Unscheduled Pods are added to the scheduler **internal scheduling queue**.

  3. The scheduler starts the **scheduling cycle** to evaluate cluster nodes.

  4. Nodes that cannot run the Pod are eliminated through filtering.

  5. Remaining nodes are **scored and ranked** using scheduling policies.

  6. The node with the highest score is selected.

  7. The scheduler enters the **binding cycle**.

  8. The scheduler updates the Pod with the selected node through the API server.

  9. The API server stores the updated Pod state in **etcd**.

  10. The **Kubelet** on the selected node receives the Pod and starts the containers.

**Internal Architecture**

                                 Kubernetes Scheduler
                             ───────────────────────────────
                                   API Server Watch
                                          │
                                          ▼
                                  Scheduling Queue
                        (Active Queue / Backoff / Unschedulable)
                                          │
                                          ▼
                                  Scheduling Cycle
                                          │
                 ┌───────────────┬───────────────┬───────────────┐
                 ▼               ▼               ▼
              PreFilter        Filter          PostFilter
          (validate pod)   (remove nodes)   (handle failures)
                                          │
                                          ▼
                                 Node Candidate List
                                          │
                                          ▼
                                    Scoring Phase
                             (PreScore → Score → Normalize)
                                          │
                                          ▼
                                   Select Best Node
                                          │
                                          ▼
                                   Binding Cycle
                             (PreBind → Bind → PostBind)
                                          │
                                          ▼
                                   Update API Server
                                          │
                                          ▼
                                          etcd
                                          │
                                          ▼
                                  Kubelet Starts Pod

**Cloud Controller Manager**

The **Cloud Controller Manager** is a control-plane component in **Kubernetes** that allows Kubernetes to integrate with **external cloud provider APIs**.

It runs controllers that manage cloud-specific resources such as load balancers, storage volumes, and node lifecycle, without requiring cloud-specific code in the main **kube-controller-manager**.

**Core Workflow**

The Cloud Controller Manager watches the **kube-apiserver** for resources that require cloud provider interactions.

Controllers maintain **cloud resources** to match the desired state of Kubernetes objects.

Changes to cloud-dependent resources trigger events in the controller work queue.

Controllers process each event through a **reconciliation loop**.

The controller compares the desired state (from Kubernetes API objects) with the actual state in the cloud provider.

If discrepancies exist, the controller makes API calls to the cloud provider to **create, update, or delete resources**.

The controller updates the Kubernetes API server with the latest resource status.

Cluster state is persisted in **etcd**.

**Internal Architecture**

                                 Cloud Controller Manager
                                 ─────────────────────────────
                                      Controller Factory
                                             │
                                             ▼
                                   Shared Informer Cache
                                             │
                        ┌─────────────┬─────────────┬─────────────┐
                        ▼             ▼             ▼
                 Node Controller  Route Controller  Service Controller
                        │             │             │
                        ▼             ▼             ▼
                     Work Queue     Work Queue     Work Queue
                        │
                        ▼
                   Reconciliation Logic
                        │
                        ▼
                  Cloud Provider API
                        │
                        ▼
                   Update API Server
                        │
                        ▼
                         etcd

**Kubelet**

The Kubelet is the primary node agent in **Kubernetes** that runs on every worker node. It ensures that containers described in Pod specifications are running and healthy.

The kubelet continuously watches the **kube-apiserver** for Pods scheduled to its node and manages container lifecycle through the container runtime.

**Core Workflow**

  1. Kubelet registers the node with the **kube-apiserver** when it starts.

  2. It continuously watches the API server for Pods assigned to its node.

  3. Pod specifications are received through the kubelet **Pod configuration sources**.

  4. The kubelet adds Pods to its **Pod worker queue**.

  5. Pod workers process Pods through synchronization loops.

  6. Kubelet interacts with the container runtime via the **Container Runtime Interface (CRI)**.

  7. The container runtime pulls container images and creates containers.

  8. Kubelet monitors container health through probes.

  9. Node and Pod status are periodically reported back to the API server.

  10. Cluster state updates are stored in **etcd**.

**Internal Architecture**

                                     Kubelet
                             ──────────────────────────
                                  Node Registration
                                         │
                                         ▼
                                  API Server Watch
                                         │
                                         ▼
                                 Pod Configuration
                       (API Server / File / HTTP Sources)
                                         │
                                         ▼
                                   Pod Workers
                                (Sync Pod Loop)
                                         │
                    ┌────────────────────┼────────────────────┐
                    ▼                    ▼                    ▼
               Volume Manager       Network Manager      Secret Manager
                    │                    │                    │
                    ▼                    ▼                    ▼
                                 Container Runtime
                           (via Container Runtime Interface)
                                         │
                                         ▼
                                 Container Lifecycle
                           (Create / Start / Stop Containers)
                                         │
                                         ▼
                                 Health Monitoring
                            (Liveness / Readiness Probes)
                                         │
                                         ▼
                                  Status Manager
                                         │
                                         ▼
                                 Update API Server
                                         │
                                         ▼
                                        etcd

**Container Runtime Interface**

The **Container Runtime** is the component in **Kubernetes** responsible for **running containers on a node**. It pulls container images, creates containers, starts and stops them, and manages their lifecycle.

The runtime interacts with the **Kubelet** through the **Container Runtime Interface (CRI)**, allowing Kubernetes to support multiple container runtimes such as **containerd** and **CRI-O**.

**Core Workflow**

  1. The **Kubelet** receives Pod specifications from the **kube-apiserver**.

  2. Kubelet sends container lifecycle requests to the runtime through the **Container Runtime Interface**.

  3. The runtime pulls required container images from container registries.

  4. Images are unpacked and stored locally.

  5. The runtime creates a container using the image and configuration.

  6. The container is started using the low-level runtime.

  7. Networking and storage are attached to the container.

  8. The runtime monitors container status.

  9. Container status is returned to **Kubelet**, which updates the API server.

  10. Cluster state updates are stored in **etcd**.

**Internal Architecture**

                                   Container Runtime
                               ──────────────────────────
                                   CRI gRPC Server
                                        │
                                        ▼
                               Container Runtime Engine
                              (containerd / CRI-O runtime)
                                        │
                                        ▼
                                  Image Management
                             (Pull / Store / Remove Images)
                                        │
                                        ▼
                                 Container Management
                             (Create / Start / Stop Pods)
                                        │
                                        ▼
                                   Runtime Shim
                               (Runtime Isolation Layer)
                                        │
                                        ▼
                               Low-Level OCI Runtime
                                   (runc execution)
                                        │
                                        ▼
                                 Linux Kernel Features
                         (Namespaces / cgroups / filesystem)
                                        │
                                        ▼
                                   Running Containers
                                        │
                                        ▼
                                    Status Feedback
                                        │
                                        ▼
                                      Kubelet

**Kube-Proxy**

kube-proxy is a network component in **Kubernetes** that runs on every node and manages **network communication for Services**.

It maintains network rules that allow Pods to communicate with other Pods and Services inside the cluster by implementing **Service abstraction and load balancing**.

Kube-proxy watches the **kube-apiserver** for changes to Services and Endpoints and updates node-level network rules accordingly.

**Core Workflow**

  1. Kube-proxy starts on each node and connects to the **kube-apiserver**.

  2. It watches for **Service and Endpoint updates** through the API server.

  3. Service and endpoint changes trigger events.

  4. Events are processed through kube-proxy's internal synchronization loop.

  5. Kube-proxy generates network rules based on the Service configuration.

  6. Rules are programmed into the node’s networking layer (such as **iptables**, **IPVS**, or **nftables**).

  7. Incoming traffic to a Service IP is redirected to one of the backend Pods.

  8. Load balancing is performed across available Pod endpoints.

  9. Network rules are periodically synchronized with the cluster state.

**Internal Architecture**

                                         kube-proxy
                                 ──────────────────────────
                                      API Server Watch
                                 (Services / Endpoints)
                                             │
                                             ▼
                                      Informer Cache
                                             │
                                             ▼
                                    Event Processing
                                             │
                                             ▼
                                      Sync Loop
                               (Service & Endpoint Sync)
                                             │
                        ┌────────────────────┼────────────────────┐
                        ▼                    ▼                    ▼
                    Service Map         Endpoint Map         Node Info
                        │                    │                    │
                        ▼                    ▼                    ▼
                                 Network Rule Generation
                                             │
                                             ▼
                                     Proxy Mode Engine
                              (iptables / IPVS / nftables)
                                             │
                                             ▼
                                      Kernel Network Stack
                                             │
                                             ▼
                                   Traffic Routing Rules
                                             │
                                             ▼
                                     Pod Load Balancing
