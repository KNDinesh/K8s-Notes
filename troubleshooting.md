🧠 🔥 **THE MASTER DEBUGGING MENTAL MODEL**

_When something breaks, never guess. Follow this strict order:_

    1. Is the object CREATED?
    2. Is it SCHEDULED?
    3. Is it RUNNING?
    4. Is it STABLE?
    5. Is it REACHABLE?

👉 Every Kubernetes issue fits into one of these layers.

🔍 **UNIVERSAL DEBUG TOOLKIT (Memorize This)**

_These are your “weapons”:_

    kubectl get <resource>
    kubectl describe <resource>
    kubectl logs <pod>
    kubectl exec -it <pod> -- sh
    kubectl get events --sort-by=.metadata.creationTimestamp
    kubectl top pod/node

👉 If you master just `describe` + `events`, you solve 70% of issues.

🧩 **1. POD TROUBLESHOOTING (Deep Dive)**

🚨 **Scenario 1: Pod stuck in `Pending`**

**Root causes:**

* No available nodes

* NodeSelector / Affinity mismatch

* Taints blocking scheduling

* Resource shortage

🔍 **Debug**

    kubectl describe pod <pod>

👉 _Look for:_

    Events:
      FailedScheduling
  
💥 _Example error:_

    0/3 nodes are available: 3 node(s) didn't match node selector

✅ **Fix**

_Check labels:_

    kubectl get nodes --show-labels

_Fix selector or label node:_

    kubectl label node <node> disktype=ssd

🚨 **Scenario 2: Pod in `CrashLoopBackOff`**

**Root causes:**

  * App crash

  * Wrong command

  * Missing env/config

🔍 **Debug**

    kubectl logs <pod>
    kubectl describe pod <pod>

👉 _Look for:_

  * Exit code

  * Restart count

💥 **Example:**

    Error: cannot connect to DB

✅ **Fix**

  * Fix env vars / config

  * Check secrets/configmaps

🚨 **Scenario 3: Pod Running but not Ready**

_Root cause:_

  * Readiness probe failing

        kubectl describe pod <pod>

👉 _Look for:_

        Readiness probe failed

🔁 **2. REPLICASET TROUBLESHOOTING**

🚨 **Scenario 1: Desired ≠ Current Pods**

    kubectl get rs

🔍 **Debug**

    kubectl describe rs <rs>

_Root causes:_

  * Pod template invalid

  * Scheduler issues

🚨 **Scenario 2: ReplicaSet not managing Pods**

_Root cause:_

  * Label mismatch

        kubectl get pods --show-labels
        kubectl get rs -o yaml

👉 Selector must match Pod labels EXACTLY

🚀 **3. DEPLOYMENT TROUBLESHOOTING**

🚨 **Scenario 1: Rollout stuck**

    kubectl rollout status deployment <name>

🔍 **Debug**

    kubectl describe deployment <name>

👉 _Look for:_

  * ProgressDeadlineExceeded

🚨 **Scenario 2: New version not working**

    kubectl rollout history deployment <name>

  _Rollback:_

    kubectl rollout undo deployment <name>

🚨 **Scenario 3: Pods not updating**

_Cause:_

  * Selector immutable

  * Wrong labels

🌐 **4. SERVICE TROUBLESHOOTING (CRITICAL)**

🚨 **Scenario 1: Service not reachable**

🔍 _Debug chain:_

    kubectl get svc
    kubectl get endpoints

👉 If **endpoints = EMPTY → PROBLEM**

_Root cause:_

  * Label mismatch

        kubectl get pods --show-labels
        kubectl describe svc <svc>

🚨 **Scenario 2: NodePort not accessible**

_Check:_

    kubectl get svc
    kubectl get nodes -o wide

_Test:_

    curl <node-ip>:<nodeport>

🚨 **Scenario 3: Pod can’t reach Service**

_Exec into pod:_

    kubectl exec -it <pod> -- sh

_Test DNS:_

    nslookup <service-name>
    
🏷️ **5. NAMESPACE ISSUES**

🚨 **Scenario: “Resource not found”**

  * Classic mistake.

        kubectl get pods -A

👉 Resource exists in different namespace

_Fix:_

    kubectl config set-context --current --namespace=<ns>

📊 **6. RESOURCE QUOTA / LIMIT RANGE**

🚨 **Scenario: Pod not created**

    kubectl describe pod

_Error:_

    exceeded quota

_Debug:_

    kubectl describe quota
    kubectl describe limitrange

_Fix:_

  * Reduce resource requests

  * Increase quota

🧭 **7. NODE SELECTOR / AFFINITY**

🚨 **Scenario: Pod stuck Pending**

    kubectl describe pod

_Error:_

    node(s) didn't match node selector

_Debug:_
  
    kubectl get nodes --show-labels
    
_Fix:_

  * Correct label mismatch

🚫 **8. TAINTS & TOLERATIONS**

🚨 **Scenario: Pod not scheduling**

    kubectl describe pod

_Error:_

    node(s) had taint {key=value:NoSchedule}

_Debug:_
    
    kubectl describe node <node>

_Fix options:_

**Option 1: Add toleration**

    tolerations:
    - key: "key"
      operator: "Equal"
      value: "value"
      effect: "NoSchedule"
  
**Option 2: Remove taint**

    kubectl taint nodes <node> key=value:NoSchedule-

🔥 **REAL ADMIN DEBUG FLOW (VERY IMPORTANT)**

_When production breaks:_

🧠 **Step 1: What is broken?**

  * App not reachable?

  * Pod restarting?

  * Deployment stuck?

🧠 **Step 2: Follow hierarchy**

    Deployment
      ↓
    ReplicaSet
      ↓
    Pods
      ↓
    Node

🧠 **Step 3: Check events FIRST**

    kubectl get events --sort-by=.metadata.creationTimestamp

👉 This tells you the story of failure.

🧠 **Step 4: Narrow down**

    Symptom	        Focus
    Pending	        Scheduler
    CrashLoop	      App
    No traffic	    Service
    Not found	      Namespace

💣 **ADVANCED REAL-WORLD SCENARIOS**

🔥 **Scenario: Everything looks fine but app not working**

_Check inside pod:_

    kubectl exec -it <pod> -- sh

_Test:_

    curl localhost:port

👉 If fails → app issue, not Kubernetes

🔥 **Scenario: Service works internally but not externally**

  * ClusterIP works

  * NodePort fails

👉 _Likely:_

  * Firewall issue

  * Cloud security group

🔥 **Scenario: Intermittent failures**

_Check:_

    kubectl top pod

👉 _Could be:_

  * CPU throttling

  * OOMKilled

🧠 **FINAL MENTAL MODEL (Burn this in memory)**

_When debugging:_

    1. kubectl get → What is status?
    2. kubectl describe → WHY?
    3. kubectl logs → App issue?
    4. kubectl exec → Verify inside container
    5. kubectl get events → Timeline

🎯 **HOW TO MASTER (SERIOUS PRACTICE PLAN)**

_Do this intentionally:_

🧪 **Break your cluster:**

1. Wrong labels → Service fails

2. Wrong image → CrashLoopBackOff

3. Add taints → Pods Pending

4. Set quota → Pod rejected

🔁 **Then fix WITHOUT googling**

That’s how real K8s admins think.
