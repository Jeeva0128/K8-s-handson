# Kubernetes Hands-on Practice

This repository contains guided exercises for learning Kubernetes architecture, Pods, workload controllers, and cluster setup with AWS and kOps. Work through the labs in order and use a Kubernetes cluster you are allowed to modify.

## Labs

### Lab 1: Cluster architecture and AWS setup with kOps

Read [Lab 01: Kubernetes Cluster Architecture and AWS Setup](CLuster-Architecure-Kops/README.md). It walks through Kubernetes architecture, preparing an AWS management instance, configuring IAM access, installing the management tools, and creating an S3 state store for kOps.

### Lab 2: Pods and workload controllers

Use the manifests in [Pod architecture and lifecycle](Pod%20architecture%20and%20lifecycle/) to practice creating and inspecting Pods, Deployments, and a DaemonSet, including adding a worker node to Minikube.

### Lab 3: ClusterIP Services

Use [Kubernetes Services](Kubernetes%20Services/README.md) to expose an Nginx Pod inside the cluster with a ClusterIP Service, inspect the Service's endpoints, and send a request from a BusyBox client Pod using Kubernetes DNS.

### Lab 4: NodePort Services

Continue in the [Kubernetes Services lab](Kubernetes%20Services/README.md) to expose an Nginx Deployment with a NodePort Service created from the terminal, then inspect and access the assigned node port.

## Prerequisites

- A Kubernetes cluster that you can use for the exercises.
- `kubectl` installed and configured to connect to that cluster.
- Minikube for the multi-node practice exercise.
- For Lab 1, an AWS account and the tools and access described in its instructions.

Check your active context and cluster connection before you begin:

```bash
kubectl config current-context
kubectl get nodes -o wide
```

## Lab 2 walkthrough

The examples use the `dev`, `qa`, and `prod` namespaces. Create them first:

```bash
kubectl create namespace dev --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace qa --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace prod --dry-run=client -o yaml | kubectl apply -f -
```

### Exercise 1: Create and inspect a Pod

Open [`apache-jenkins-pod.yaml`](Pod%20architecture%20and%20lifecycle/apache-jenkins-pod.yaml). Before applying it, remove the `replicas: 2` line: `replicas` belongs to a workload controller such as a Deployment, not a Pod. The Pod has Apache and Jenkins containers.

Apply the manifest and inspect the Pod:

```bash
kubectl apply -f "Pod architecture and lifecycle/apache-jenkins-pod.yaml"
kubectl wait --for=condition=Ready pod/apache-pod -n dev --timeout=180s
kubectl get pod apache-pod -n dev -o wide
kubectl describe pod apache-pod -n dev
kubectl logs apache-pod -n dev -c apache-dev-container
```

When you are done, remove the standalone Pod before moving on:

```bash
kubectl delete pod apache-pod -n dev
```

### Exercise 2: Run a Deployment and scale it

Apply [`apache-jenkins-deployment.yaml`](Pod%20architecture%20and%20lifecycle/apache-jenkins-deployment.yaml). It creates two replicas, each with Apache and Jenkins containers.

```bash
kubectl apply -f "Pod architecture and lifecycle/apache-jenkins-deployment.yaml"
kubectl rollout status deployment/apache-pod-deployment -n dev
kubectl get deployment,replicaset,pods -n dev
```

Scale the Deployment to three replicas and verify the result:

```bash
kubectl scale deployment/apache-pod-deployment --replicas=3 -n dev
kubectl get deployment,pods -n dev
```

Observe how the Deployment and its ReplicaSet keep the requested number of Pods running. The manifest specifies two replicas, so applying it again restores that desired count.

### Exercise 3: Deploy Apache in the `prod` practice namespace

Apply [`apache-production-deployment.yaml`](Pod%20architecture%20and%20lifecycle/apache-production-deployment.yaml). It creates five Apache replicas in the `prod` namespace. This is a local practice example, not a production-hardened deployment.

```bash
kubectl apply -f "Pod architecture and lifecycle/apache-production-deployment.yaml"
kubectl rollout status deployment/apache-production -n prod
kubectl get deployment,replicaset,pods -n prod -o wide
```

### Exercise 4: Run a DaemonSet

Apply [`fluent-bit-daemonset.yaml`](Pod%20architecture%20and%20lifecycle/fluent-bit-daemonset.yaml). The DaemonSet runs Fluent Bit on eligible nodes and mounts the host's container log directory read-only.

```bash
kubectl apply -f "Pod architecture and lifecycle/fluent-bit-daemonset.yaml"
kubectl rollout status daemonset/fluent-bit-agent -n qa
kubectl get daemonset,pods -n qa -o wide
```

Compare the Pods' `NODE` values with `kubectl get nodes`. A DaemonSet creates a Pod on each eligible node; node taints or other scheduling constraints can affect placement.

On Minikube, add a worker while the DaemonSet is running and watch the node and its Fluent Bit Pod join:

```bash
minikube node add
```

Wait for the new node to reach `Ready`, then stop the watch with `Ctrl+C`:

```bash
kubectl get nodes -w
```

Watch the DaemonSet Pods until the new Fluent Bit Pod is `Running`, then stop the watch:

```bash
kubectl get pods -n qa -o wide -w
```

The new node may take time to become `Ready`, and the DaemonSet Pod may remain in `ContainerCreating` briefly. See the [directory lab guide](Pod%20architecture%20and%20lifecycle/README.md) for the observed example and troubleshooting commands.

## Lab 3 walkthrough: expose Nginx with a ClusterIP Service

This exercise uses the `dev` namespace and the manifests in [`Kubernetes Services/ClusterIP`](Kubernetes%20Services/ClusterIP/). From the repository root, create the namespace if needed and apply the Nginx Pod and Service:

```bash
kubectl create namespace dev --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -f "Kubernetes Services/ClusterIP/nginx-server.yaml"
kubectl apply -f "Kubernetes Services/ClusterIP/nginx-service.yaml"
kubectl wait --for=condition=Ready pod/nginx-server -n dev --timeout=120s
```

Inspect the Service and confirm that its selector finds the Nginx Pod:

```bash
kubectl get pod nginx-server -n dev --show-labels
kubectl get service nginx-service -n dev -o wide
kubectl describe service nginx-service -n dev
kubectl get endpointslice -n dev -l kubernetes.io/service-name=nginx-service
```

Start the BusyBox client Pod, then make an HTTP request to the Service by its DNS name from inside the cluster:

```bash
kubectl apply -f "Kubernetes Services/ClusterIP/client-pod.yaml"
kubectl wait --for=condition=Ready pod/client-pod -n dev --timeout=120s
kubectl exec -n dev client-pod -- wget -qO- http://nginx-service
```

The Nginx welcome page in the response confirms that the client reached the Service and that Kubernetes DNS resolved `nginx-service` in the same namespace. For the fully qualified name, try `http://nginx-service.dev.svc.cluster.local`. See the [Services lab guide](Kubernetes%20Services/README.md) for more command practice and cleanup steps.

## Lab 4 walkthrough: expose a Deployment with NodePort

The NodePort practice manifest creates two Nginx replicas in the `qa` namespace. From the repository root, create the namespace if needed, deploy the replicas, and expose them with a Service from the terminal:

```bash
kubectl create namespace qa --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -f "Kubernetes Services/Nodeport/nginx-qa-deploy.yaml"
kubectl rollout status deployment/nginx-qa -n qa --timeout=120s
kubectl get deployment,pods -n qa -o wide
kubectl expose deployment nginx-qa --type=NodePort --port=80 --target-port=80 -n qa
```

Inspect the Service and its backend endpoints, then get the allocated NodePort and node addresses:

```bash
kubectl get service nginx-qa -n qa -o wide
kubectl describe service nginx-qa -n qa
kubectl get endpointslice -n qa -l kubernetes.io/service-name=nginx-qa
kubectl get nodes -o wide
kubectl get service nginx-qa -n qa -o jsonpath='{.spec.ports[0].nodePort}{"\n"}'
```

If a node's IP is reachable from your workstation, use that IP and the NodePort printed above in `curl "http://NODE_IP:NODE_PORT"` (replace the placeholders). With Minikube, `minikube service nginx-qa -n qa --url` can provide a URL to test with `curl`. Access from outside the cluster depends on the cluster's network and firewall configuration. See the [Services lab guide](Kubernetes%20Services/README.md#exercise-expose-a-deployment-with-nodeport) for details.

## Cleanup

Remove the workloads when you finish:

```bash
kubectl delete -f "Pod architecture and lifecycle/fluent-bit-daemonset.yaml"
kubectl delete deployment apache-pod-deployment -n dev --ignore-not-found
kubectl delete deployment apache-production -n prod --ignore-not-found
kubectl delete pod apache-pod -n dev --ignore-not-found
```

If the `dev`, `qa`, and `prod` namespaces were created only for these exercises, remove them too:

```bash
kubectl delete namespace dev qa prod
```

For the ClusterIP and NodePort exercises, remove their resources with the commands in the [Services lab cleanup guide](Kubernetes%20Services/README.md). If either namespace is shared with another exercise, delete only the Services lab resources.

## Manifest reference

| File | Exercise |
| --- | --- |
| [`apache-jenkins-pod.yaml`](Pod%20architecture%20and%20lifecycle/apache-jenkins-pod.yaml) | A Pod with Apache and Jenkins containers; remove the invalid Pod-level `replicas` field before applying. |
| [`apache-jenkins-deployment.yaml`](Pod%20architecture%20and%20lifecycle/apache-jenkins-deployment.yaml) | A Deployment with two replicas, each with Apache and Jenkins containers. |
| [`apache-production-deployment.yaml`](Pod%20architecture%20and%20lifecycle/apache-production-deployment.yaml) | A five-replica Apache practice Deployment in the `prod` namespace. |
| [`fluent-bit-daemonset.yaml`](Pod%20architecture%20and%20lifecycle/fluent-bit-daemonset.yaml) | A Fluent Bit DaemonSet for eligible nodes. |

See the [Services lab manifest reference](Kubernetes%20Services/README.md#manifest-reference) for the ClusterIP server, Service, client Pod, and NodePort Deployment used in Labs 3 and 4.
