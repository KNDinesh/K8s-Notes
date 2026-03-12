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
