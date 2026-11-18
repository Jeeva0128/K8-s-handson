# Pod Architecture and Lifecycle Lab

In this hands-on lab, create and inspect a multi-container Pod, manage replicas with a Deployment, and run a DaemonSet across eligible nodes.

For the repository overview and the AWS/kOps lab, see the [main README](../README.md).

## Learning objectives

- Inspect a Pod and its containers with `kubectl`.
- Compare a standalone Pod with a Deployment-managed Pod.
- Scale a Deployment and observe its ReplicaSet maintain the desired count.
- Observe how a DaemonSet places one Pod on each eligible node.

## Prerequisites

- A Kubernetes cluster you can use for the exercises.
- `kubectl` configured to access that cluster.
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
```

## Exercise 1: Create a Pod

Open [`apache-pod.yaml`](apache-pod.yaml). It defines a Pod with Apache and Jenkins containers in the `dev` namespace. Before applying it, remove the `replicas: 2` line: Pods do not support a `replicas` field; replica counts belong to workload controllers such as Deployments.

Apply the corrected manifest and inspect the Pod:

```bash
kubectl apply -f apache-pod.yaml
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

Apply [`replica-deploy.yaml`](replica-deploy.yaml). It creates a Deployment with two replicas. Each Pod contains Apache and Jenkins containers.

```bash
kubectl apply -f replica-deploy.yaml
kubectl rollout status deployment/apache-pod-deployment -n dev
kubectl get deployment,replicaset,pods -n dev
```

Scale the Deployment to three replicas and check the result:

```bash
kubectl scale deployment/apache-pod-deployment --replicas=3 -n dev
kubectl get deployment,replicaset,pods -n dev
```

The Deployment and ReplicaSet work together to maintain the requested number of Pods. The manifest declares two replicas, so reapplying it returns the Deployment to two.

## Exercise 3: Run a DaemonSet

Apply [`daemon-deploy`](daemon-deploy). It runs Fluent Bit on eligible nodes and mounts the node's `/var/log` directory read-only.

```bash
kubectl apply -f daemon-deploy
kubectl rollout status daemonset/fluent-bit-agent -n qa
kubectl get daemonset,pods -n qa -o wide
```

Compare the `NODE` column with the output of `kubectl get nodes`. A DaemonSet creates one Pod on each eligible node; taints or other scheduling constraints can affect placement.

## Compare the workload types

| Workload | What it does in this lab |
| --- | --- |
| Pod | Runs the Apache and Jenkins containers together in one Pod. |
| Deployment | Maintains the requested number of replicated Pods and supports scaling. |
| DaemonSet | Runs a Fluent Bit Pod on each eligible node. |

## Cleanup

Remove the workloads when you finish:

```bash
kubectl delete -f daemon-deploy
kubectl delete deployment apache-pod-deployment -n dev --ignore-not-found
kubectl delete pod apache-pod -n dev --ignore-not-found
```

If you created `dev` and `qa` only for this lab, remove them too. Deleting a namespace removes all resources inside it:

```bash
kubectl delete namespace dev qa
```
