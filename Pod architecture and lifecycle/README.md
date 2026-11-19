# Pod Architecture and Lifecycle Lab

In this hands-on Minikube practice lab, create and inspect a multi-container Pod, manage replicas with Deployments, and run a DaemonSet across eligible nodes.

For the repository overview and the AWS/kOps lab, see the [main README](../README.md).

## Learning objectives

- Inspect a Pod and its containers with `kubectl`.
- Compare a standalone Pod with a Deployment-managed Pod.
- Scale a Deployment and observe its ReplicaSet maintain the desired count.
- Observe how a DaemonSet places one Pod on each eligible node.

## Prerequisites

- A Minikube cluster, or another Kubernetes cluster you can use for the exercises.
- `kubectl` configured to access that cluster.
- `minikube` installed for the worker node exercise.
- Run the commands in this guide from this directory.

Check the active cluster before creating resources:

```bash
kubectl config current-context
kubectl get nodes -o wide
```

Create the namespaces used by the manifests:

```bash
kubectl create namespace dev --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace qa --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace prod --dry-run=client -o yaml | kubectl apply -f -
```

## Exercise 1: Create a Pod

Open [`apache-jenkins-pod.yaml`](apache-jenkins-pod.yaml). It defines a Pod with Apache and Jenkins containers in the `dev` namespace. Before applying it, remove the `replicas: 2` line: Pods do not support a `replicas` field; replica counts belong to workload controllers such as Deployments.

Apply the corrected manifest and inspect the Pod:

```bash
kubectl apply -f apache-jenkins-pod.yaml
kubectl wait --for=condition=Ready pod/apache-pod -n dev --timeout=180s
kubectl get pod apache-pod -n dev -o wide
kubectl describe pod apache-pod -n dev
kubectl logs apache-pod -n dev -c apache-dev-container
```

To view Apache locally, start port forwarding:

```bash
kubectl port-forward pod/apache-pod 8080:80 -n dev
```

Keep that command running, then open `http://localhost:8080` or run `curl http://localhost:8080` in another terminal. Stop port forwarding with `Ctrl+C` when finished.

Delete the standalone Pod before continuing:

```bash
kubectl delete pod apache-pod -n dev
```

## Exercise 2: Use a Deployment and scale it

Apply [`apache-jenkins-deployment.yaml`](apache-jenkins-deployment.yaml). It creates a Deployment with two replicas. Each Pod contains Apache and Jenkins containers.

```bash
kubectl apply -f apache-jenkins-deployment.yaml
kubectl rollout status deployment/apache-pod-deployment -n dev
kubectl get deployment,replicaset,pods -n dev
```

Scale the Deployment to three replicas and check the result:

```bash
kubectl scale deployment/apache-pod-deployment --replicas=3 -n dev
kubectl get deployment,replicaset,pods -n dev
```

The Deployment and ReplicaSet work together to maintain the requested number of Pods. The manifest declares two replicas, so reapplying it returns the Deployment to two.

## Exercise 3: Deploy Apache in the `prod` practice namespace

Apply [`apache-production-deployment.yaml`](apache-production-deployment.yaml). It creates five Apache replicas in the `prod` namespace. Despite the namespace and filename, this is a local practice example, not a production-hardened deployment.

```bash
kubectl apply -f apache-production-deployment.yaml
kubectl rollout status deployment/apache-production -n prod
kubectl get deployment,replicaset,pods -n prod -o wide
```

## Exercise 4: Run a DaemonSet

Apply [`fluent-bit-daemonset.yaml`](fluent-bit-daemonset.yaml). It runs Fluent Bit on eligible nodes and mounts the node's `/var/log` directory read-only.

```bash
kubectl apply -f fluent-bit-daemonset.yaml
kubectl rollout status daemonset/fluent-bit-agent -n qa
kubectl get daemonset,pods -n qa -o wide
```

Compare the `NODE` column with the output of `kubectl get nodes`. A DaemonSet creates one Pod on each eligible node; taints or other scheduling constraints can affect placement.

### Add a worker node to Minikube

Keep the DaemonSet running, then add a worker to your Minikube cluster:

```bash
minikube node add
```

Watch the new node join and become ready:

```bash
kubectl get nodes -w
```

When the new node is `Ready`, stop the watch with `Ctrl+C` and watch the DaemonSet Pods:

```bash
kubectl get pods -n qa -o wide -w
```

The DaemonSet should create another Fluent Bit Pod for the new eligible node. Wait for it to reach `Running`, then stop the watch with `Ctrl+C`. Confirm that the DaemonSet has a ready Pod on each eligible node:

```bash
kubectl get daemonset fluent-bit-agent -n qa
kubectl get pods -n qa -o wide
```

In the provided Minikube session, worker `minikube-m02` was initially `NotReady`, became `Ready` after about 50 seconds, and its Fluent Bit Pod moved from `ContainerCreating` to `Running` about 72 seconds after creation. Startup time varies. If a Pod remains in `ContainerCreating`, inspect its details and recent namespace events:

```bash
kubectl describe pod -n qa <pod-name>
kubectl get events -n qa --sort-by=.lastTimestamp
```

## Compare the workload types

| Workload | What it does in this lab |
| --- | --- |
| Pod | Runs the Apache and Jenkins containers together in one Pod. |
| Deployment | Maintains the requested number of replicated Pods and supports scaling. |
| DaemonSet | Runs a Fluent Bit Pod on each eligible node. |

## Cleanup

Remove the workloads when you finish:

```bash
kubectl delete -f fluent-bit-daemonset.yaml
kubectl delete deployment apache-pod-deployment -n dev --ignore-not-found
kubectl delete deployment apache-production -n prod --ignore-not-found
kubectl delete pod apache-pod -n dev --ignore-not-found
```

If you created `dev`, `qa`, and `prod` only for this lab, remove them too. Deleting a namespace removes all resources inside it:

```bash
kubectl delete namespace dev qa prod
```

## Manifest reference

| File | Resource |
| --- | --- |
| [`apache-jenkins-pod.yaml`](apache-jenkins-pod.yaml) | Pod with Apache and Jenkins containers. Remove the Pod-level `replicas` field before applying. |
| [`apache-jenkins-deployment.yaml`](apache-jenkins-deployment.yaml) | Two-replica Deployment with Apache and Jenkins containers. |
| [`apache-production-deployment.yaml`](apache-production-deployment.yaml) | Five-replica Apache practice Deployment in the `prod` namespace. |
| [`fluent-bit-daemonset.yaml`](fluent-bit-daemonset.yaml) | Fluent Bit DaemonSet for eligible nodes. |
