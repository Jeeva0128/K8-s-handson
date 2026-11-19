# Kubernetes Services Labs

Practice exposing an Nginx Pod to other Pods with a `ClusterIP` Service, then exposing an Nginx Deployment with a `NodePort` Service created from the terminal. The labs use the manifests in [`ClusterIP/`](ClusterIP/) and [`Nodeport/`](Nodeport/).

For the repository overview and the Pod/workload exercises, see the [main README](../README.md).

## Learning objectives

- Inspect Pod labels and understand how a Service selector finds its backends.
- Inspect the ClusterIP Service and its EndpointSlice.
- Send an HTTP request from another Pod using the Service's DNS name.
- Create a NodePort Service with `kubectl expose` and inspect its assigned port.

## Prerequisites

- A Kubernetes cluster you can use for the exercise.
- `kubectl` configured to access that cluster.
- `minikube` if using the Minikube access command in the NodePort exercise.
- Run the commands below from the repository root, or adjust the manifest paths if you start elsewhere.

Check which cluster you are connected to and create the namespace if it does not exist:

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl create namespace dev --dry-run=client -o yaml | kubectl apply -f -
```

## Exercise: deploy and inspect the Service

Apply the Nginx server Pod and its ClusterIP Service:

```bash
kubectl apply -f "Kubernetes Services/ClusterIP/nginx-server.yaml"
kubectl apply -f "Kubernetes Services/ClusterIP/nginx-service.yaml"
kubectl wait --for=condition=Ready pod/nginx-server -n dev --timeout=120s
```

Inspect the Pod, Service, and endpoints:

```bash
kubectl get pod nginx-server -n dev --show-labels -o wide
kubectl get service nginx-service -n dev -o wide
kubectl describe service nginx-service -n dev
kubectl get endpointslice -n dev -l kubernetes.io/service-name=nginx-service
```

The Service selects Pods labeled `app: nginx-server` and forwards TCP port `80` to the selected Pod's port `80`. The EndpointSlice should show the address of the Nginx Pod. If there are no endpoints, compare the Pod labels with the Service selector and check that the Pod is ready.

## Exercise: connect from a client Pod

Create the BusyBox client Pod and wait for it to become ready:

```bash
kubectl apply -f "Kubernetes Services/ClusterIP/client-pod.yaml"
kubectl wait --for=condition=Ready pod/client-pod -n dev --timeout=120s
kubectl get pods -n dev -o wide
```

Run a request from the client Pod using the Service's short DNS name:

```bash
kubectl exec -n dev client-pod -- wget -qO- http://nginx-service
```

You should see the Nginx welcome page HTML. Because both Pods are in `dev`, `nginx-service` resolves to the Service in that namespace. You can also try the fully qualified Service name:

```bash
kubectl exec -n dev client-pod -- wget -qO- http://nginx-service.dev.svc.cluster.local
```

Check the Service's cluster-internal IP and compare it with the server Pod address:

```bash
kubectl get service nginx-service -n dev -o wide
kubectl get pod nginx-server -n dev -o wide
```

The ClusterIP is a stable in-cluster address for reaching the Service. Clients should use the Service name instead of relying on a Pod IP, which can change when a Pod is replaced.

## Exercise: expose a Deployment with NodePort

This exercise deploys two Nginx replicas in the `qa` namespace. Create the namespace if needed, then apply the Deployment manifest:

```bash
kubectl create namespace qa --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -f "Kubernetes Services/Nodeport/nginx-qa-deploy.yaml"
kubectl rollout status deployment/nginx-qa -n qa --timeout=120s
kubectl get deployment,pods -n qa -o wide
```

Create a NodePort Service from the terminal. The Service selects the Deployment's `app: nginx-qa` Pods and exposes port `80`:

```bash
kubectl expose deployment nginx-qa --type=NodePort --port=80 --target-port=80 -n qa
```

Inspect the Service, allocated node port, endpoints, and node addresses:

```bash
kubectl get service nginx-qa -n qa -o wide
kubectl describe service nginx-qa -n qa
kubectl get endpointslice -n qa -l kubernetes.io/service-name=nginx-qa
kubectl get nodes -o wide
kubectl get service nginx-qa -n qa -o jsonpath='{.spec.ports[0].nodePort}{"\n"}'
```

The last command prints the dynamically assigned NodePort. From a machine that can reach a cluster node, request `http://<node-ip>:<node-port>` using an address from `kubectl get nodes -o wide` and the port printed above. With Minikube, you can ask it for a reachable URL and request that URL:

```bash
minikube service nginx-qa -n qa --url
```

Copy the URL printed by Minikube and run `curl` with that URL from another terminal. NodePort makes the Service available on a port on each node. Whether that node address is reachable from your workstation depends on the cluster's network and firewall configuration.

## Troubleshooting command practice

If the client request fails, inspect the resources and recent events:

```bash
kubectl describe pod nginx-server -n dev
kubectl describe pod client-pod -n dev
kubectl describe service nginx-service -n dev
kubectl get endpointslice -n dev -l kubernetes.io/service-name=nginx-service
kubectl get events -n dev --sort-by=.lastTimestamp
```

For the NodePort exercise, inspect the Deployment, Service, endpoints, and events in `qa`:

```bash
kubectl describe deployment nginx-qa -n qa
kubectl describe service nginx-qa -n qa
kubectl get endpointslice -n qa -l kubernetes.io/service-name=nginx-qa
kubectl get events -n qa --sort-by=.lastTimestamp
```

## Cleanup

Delete only the resources from this exercise:

```bash
kubectl delete -f "Kubernetes Services/ClusterIP/client-pod.yaml" --ignore-not-found
kubectl delete -f "Kubernetes Services/ClusterIP/nginx-service.yaml" --ignore-not-found
kubectl delete -f "Kubernetes Services/ClusterIP/nginx-server.yaml" --ignore-not-found
```

For the NodePort exercise, remove the Service created by `kubectl expose` and then the Deployment:

```bash
kubectl delete service nginx-qa -n qa --ignore-not-found
kubectl delete -f "Kubernetes Services/Nodeport/nginx-qa-deploy.yaml" --ignore-not-found
```

If you created `dev` and `qa` only for these Services exercises and are done with any other work in them, remove the namespaces after deleting the lab resources. Namespace deletion also removes every other resource in those namespaces:

```bash
kubectl delete namespace dev qa
```

## Manifest reference

| File | Resource |
| --- | --- |
| [`nginx-server.yaml`](ClusterIP/nginx-server.yaml) | Nginx server Pod labeled `app: nginx-server`. |
| [`nginx-service.yaml`](ClusterIP/nginx-service.yaml) | ClusterIP Service selecting the Nginx Pod on port 80. |
| [`client-pod.yaml`](ClusterIP/client-pod.yaml) | BusyBox client Pod for in-cluster DNS and HTTP request practice. |
| [`nginx-qa-deploy.yaml`](Nodeport/nginx-qa-deploy.yaml) | Two-replica Nginx Deployment used to practice creating a NodePort Service with `kubectl expose`. |
