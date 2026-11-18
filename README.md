# Kubernetes Hands-on Practice

This repository contains guided exercises for learning Kubernetes architecture, Pods, workload controllers, and cluster setup with AWS and kOps. Work through the labs in order and use a Kubernetes cluster you are allowed to modify.

## Labs

### Lab 1: Cluster architecture and AWS setup with kOps

Read [Lab 01: Kubernetes Cluster Architecture and AWS Setup](CLuster-Architecure-Kops/README.md). It walks through Kubernetes architecture, preparing an AWS management instance, configuring IAM access, installing the management tools, and creating an S3 state store for kOps.

### Lab 2: Pods and workload controllers

Use the manifests in [Pod architecture and lifecycle](Pod%20architecture%20and%20lifecycle/) to practice creating and inspecting a Pod, a Deployment, and a DaemonSet.

## Prerequisites

- A Kubernetes cluster that you can use for the exercises.
- `kubectl` installed and configured to connect to that cluster.
- For Lab 1, an AWS account and the tools and access described in its instructions.

Check your active context and cluster connection before you begin:

```bash
kubectl config current-context
kubectl get nodes -o wide
```

## Lab 2 walkthrough

The examples use the `dev` and `qa` namespaces. Create them first:

```bash
kubectl create namespace dev --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace qa --dry-run=client -o yaml | kubectl apply -f -
```

### Exercise 1: Create and inspect a Pod

Open [`apache-pod.yaml`](Pod%20architecture%20and%20lifecycle/apache-pod.yaml). Before applying it, remove the `replicas: 2` line: `replicas` belongs to a workload controller such as a Deployment, not a Pod. The Pod has Apache and Jenkins containers.

Apply the manifest and inspect the Pod:

```bash
kubectl apply -f "Pod architecture and lifecycle/apache-pod.yaml"
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

Apply [`replica-deploy.yaml`](Pod%20architecture%20and%20lifecycle/replica-deploy.yaml). It creates two replicas, each with Apache and Jenkins containers.

```bash
kubectl apply -f "Pod architecture and lifecycle/replica-deploy.yaml"
kubectl rollout status deployment/apache-pod-deployment -n dev
kubectl get deployment,replicaset,pods -n dev
```

Scale the Deployment to three replicas and verify the result:

```bash
kubectl scale deployment/apache-pod-deployment --replicas=3 -n dev
kubectl get deployment,pods -n dev
```

Observe how the Deployment and its ReplicaSet keep the requested number of Pods running. The manifest specifies two replicas, so applying it again restores that desired count.

### Exercise 3: Run a DaemonSet

Apply [`daemon-deploy`](Pod%20architecture%20and%20lifecycle/daemon-deploy). The DaemonSet runs Fluent Bit on eligible nodes and mounts the host's container log directory read-only.

```bash
kubectl apply -f "Pod architecture and lifecycle/daemon-deploy"
kubectl rollout status daemonset/fluent-bit-agent -n qa
kubectl get daemonset,pods -n qa -o wide
```

Compare the Pods' `NODE` values with `kubectl get nodes`. A DaemonSet creates a Pod on each eligible node; node taints or other scheduling constraints can affect placement.

## Cleanup

Remove the workloads when you finish:

```bash
kubectl delete -f "Pod architecture and lifecycle/daemon-deploy"
kubectl delete deployment apache-pod-deployment -n dev --ignore-not-found
kubectl delete pod apache-pod -n dev --ignore-not-found
```

If the `dev` and `qa` namespaces were created only for these exercises, remove them too:

```bash
kubectl delete namespace dev qa
```

## Manifest reference

| File | Exercise |
| --- | --- |
| [`apache-pod.yaml`](Pod%20architecture%20and%20lifecycle/apache-pod.yaml) | A Pod with Apache and Jenkins containers; remove the invalid Pod-level `replicas` field before applying. |
| [`replica-deploy.yaml`](Pod%20architecture%20and%20lifecycle/replica-deploy.yaml) | A Deployment with two replicas. |
| [`daemon-deploy`](Pod%20architecture%20and%20lifecycle/daemon-deploy) | A Fluent Bit DaemonSet. |
