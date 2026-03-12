**The Kubernetes API Server works as a secure REST gateway + state manager:**

  1. Receives REST request

    The client (kubectl) sends an HTTPS request to the API server.

            POST /api/v1/namespaces/default/pods

    Key details:

    * JSON/YAML request body

    * Auth credentials (token, cert, etc.)

    * TLS encrypted

  2. Authenticates user

  3. Authorizes permissions

  4. Runs admission controllers

  5. Validates and converts object

  6. Stores state in etcd

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
