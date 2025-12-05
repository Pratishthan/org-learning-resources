# Kubernetes


## Kubernetes Architecture Overview

```txt
                   +--------------------+
                   |   Control Plane    |
                   |   (Master Node)    |
                   +--------------------+
                   |  API Server        |
                   |  Scheduler         |
                   |  Controller Manager|
                   |  etcd (key store)  |
                   +--------------------+
                            |
        -----------------------------------------------
        |                       |                     |
+----------------+   +----------------+       +----------------+
|  Worker Node 1 |   | Worker Node 2  |       |  Worker Node N |
+----------------+   +----------------+       +----------------+
| kubelet        |   | kubelet        |       | kubelet        |
| kube-proxy     |   | kube-proxy     |       | kube-proxy     |
| container runtime| | container runtime|     | container runtime|
| Pods            |  | Pods           |       | Pods            |
+----------------+   +----------------+       +----------------+
```

## Required Components

- cluster - one or more worker machine
- node - one worker machine/virtual machine
- Namespace - folder or workspace for each node
- pods - running applications

## Cluster Info

```bash
kubectl version                         # Version of kubectl and Kubernetes server
kubectl cluster-info                    # Info about cluster endpoints
kubectl get nodes                       # List all nodes
kubectl describe node <node-name>       # Detailed info about a node ( if node is down )
```

## Namespaces Info

```bash
kubectl get ns                         # List namespaces(autobaml/pyora)
kubectl create namespace <name>        # Create new namespace
kubectl delete namespace <name>        # Delete namespace
```

## Pods Info

```bash
kubectl get pods                       # List all pods
kubectl get pods -A                    # List pods in all namespaces
kubectl get pods -n <namespace>        # List pods in a namespace
kubectl describe pod <pod-name>        # Details of a specific pod
kubectl logs <pod-name>                # View logs of a pod
kubectl logs <pod-name> -c <container> # Logs for specific container in pod
kubectl exec -it <pod-name> -- /bin/bash  # SSH into a pod
kubectl delete pod <pod-name>          # Delete a pod
```

**NOTE:**
    If our pods are running within a specific namespace, we'll need to use the `-n` flag with `kubectl` to target that namespace.
    Example:
    To get pods in the `payments` namespace:

    ```bash
    kubectl get pods -n payments
    ```
    Using Shell Alias
    we can create an alias in our .bashrc, .zshrc, or shell configuration file to make this easier:
    ```bash
    alias pay='kubectl -n payments'
    ```
    After reloading our shell (or sourcing the file), you can simply run:
    ```bash
    pay get pods
    ```
    List All Aliases
    To list all defined aliases in your shell:
    ```bash
    alias
    ```

## relation diagram of Kubernetes Deployment, ConfigMap, and Service

```txt

                         +----------------+
                         |  ConfigMap     |
                         | (app settings) |
                         +-------+--------+
                                 |
                                 |
                      mounted / injected into
                                 |
                         +-------v--------+
                         | Deployment     |
                         | (manages Pods) |
                         +-------+--------+
                                 |
                  creates        |     uses
               +-----------------v----------------+
               |             Pod                  |
               |  (includes app containers)       |
               +--------+--------------+----------+
                        |              |
                uses env/config   exposes port
                        |              |
                        +--------------v-------------+
                                       |
                                +------+------------+
                                |   Service         |
                                | (load balancer)   |
                                +-------------------+

```


## Deployments & Services

```bash
kubectl get deployments                 # List deployments
kubectl describe deployment <name>     # Deployment details
kubectl create deployment nginx --image=nginx  # Create a deployment
kubectl scale deployment nginx --replicas=3    # Scale deployment
kubectl delete deployment <name>       # Delete deployment

kubectl get svc                         # List services
kubectl expose deployment nginx --port=80 --type=NodePort  # Expose deployment

kubectl -n <namespace> edit deploy <app name> # to edit deployment script for app
```

## ConfigMaps & Secrets

```bash
kubectl get configmaps                 # List ConfigMaps
kubectl describe configmap <name>      # Show ConfigMap data
kubectl get secrets                    # List secrets
kubectl describe secret <name>         # View secret metadata
kubectl create secret generic my-secret --from-literal=key1=val1
```

## Events & Debugging

```bash
kubectl get events                     # Get recent events
kubectl describe pod <pod-name>        # View pod issues
kubectl top pod                        # Show pod resource usage
kubectl top node                       # Show node resource usage
```

## Resource Exploration

```bash
kubectl api-resources                  # All available resources
kubectl explain pod                    # Show pod definition and fields
```

## Endpoint ( db ip is defined here )

- When you create a Service, Kubernetes automatically creates an associated Endpoint object.
- This Endpoint stores a list of IP addresses (and ports) for the Pods that match the Service’s selector.
- So, the Service routes traffic to the IPs listed in its Endpoints.

```bash
kubectl get endpoints
```

### Service vs Endpoint in Kubernetes

| **Feature**               | **Service**                                   | **Endpoint**                                                |
|---------------------------|-----------------------------------------------|------------------------------------------------------------|
| **What it is**             | An abstraction to expose a set of Pods        | A list of actual Pod IPs behind the Service                |
| **Type**                   | Kubernetes resource (Service)                 | Kubernetes resource (Endpoints)                            |
| **Created by**             | You, as a user (kubectl apply, YAML)          | Automatically by Kubernetes when a Service is created      |
| **Purpose**                | Load balances and routes traffic              | Keeps track of Pod IPs the Service can send traffic to    |
| **Stable?**                | Yes — has a stable virtual IP and DNS name    | No — Pod IPs can change as pods restart/scale              |
| **Visible with**           | `kubectl get svc`                             | `kubectl get endpoints`                                    |
| **Works With**             | ClusterIP, NodePort, LoadBalancer, Headless   | Headless or regular services                               |


### Service Mesh ( pending write up ... )
