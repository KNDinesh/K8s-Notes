**Kind**

  * Follow K8s documentation for installation and cluster creation (Create Control Plane with two worker nodes using `--config` flag)

  * Always use kind as a prefix along with **<cluster-name** in `--context` while checking the cluster-info

  * Suppose if we have created a second cluster and switched by using `--context` then to view the nodes of first cluster, we have to switch

  * To view in which cluster we are currently in, use `kubectl config get-contexts`

  * To switch/use the desired cluster, use `kubectl config use-context <cluster-name`

  * Try `kubectl get nodes` to see the number of available nodes in the current context
