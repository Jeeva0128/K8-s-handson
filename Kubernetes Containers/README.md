# Kubernetes Containers

This lab introduces the container types and patterns used with Kubernetes
Pods. The examples are standalone Pods in the `default` namespace. Run them
only in a cluster you are authorized to modify.

## Container and Pod patterns

| Pattern | Where it is defined | Purpose |
| --- | --- | --- |
| Application container | `spec.containers` in a Pod | Runs the workload for the lifetime of the Pod. A Pod can contain one or more application containers. |
| Init container | `spec.initContainers` in a Pod | Runs to completion before application containers start. Init containers run in order. |
| Sidecar container | Usually another entry in `spec.containers` | Runs alongside the main application container to provide a supporting function, such as log forwarding. |
| Static Pod | A Pod manifest watched by the kubelet on a particular node | Runs without a workload controller. It is not a special container type; the kubelet manages the Pod from a local manifest. |
| Ephemeral container | Added to an existing Pod through the `ephemeralcontainers` API | Provides temporary troubleshooting access, for example when the application image has no shell or debugging tools. |

Static Pods are configured on the node in the kubelet's configured static Pod
directory. The kubelet may create a mirror Pod in the API server, but that
mirror is not a way to manage the static Pod. The location of the static Pod
directory depends on the cluster setup; do not copy a manifest onto a node
unless you administer that node.

Ephemeral containers are not declared when creating a Pod and cannot be
removed or restarted after they have been added. They share the Pod's
namespaces and can target another container's process namespace when the
runtime supports it. They do not automatically share the target container's
filesystem.

## Repository examples

| Manifest | Purpose |
| --- | --- |
| [`initcont.yaml`](./initcont.yaml) | An init container completes its short initialization step before a BusyBox application container runs. |
| [`sidecarcont.yaml`](./sidecarcont.yaml) | An application container writes to a shared `emptyDir` volume while a sidecar tails the log. |
| [`ephemeral-container.yaml`](./ephemeral-container.yaml) | A target Pod for the ephemeral-container debugging exercise. The file creates the Pod; use `kubectl debug` to add the ephemeral container. |

## Exercise 1: init container

Apply the init-container example and wait for the Pod to become ready:

```bash
kubectl apply -f "Kubernetes Containers/initcont.yaml"
kubectl wait --for=condition=Ready pod/init-demo --timeout=120s
kubectl get pod init-demo
kubectl logs init-demo -c init-myservice
kubectl logs init-demo -c app
```

The init container runs before `app` starts. It exits after its initialization
step; the application container continues running.

## Exercise 2: sidecar container

Apply the sidecar example and inspect both containers:

```bash
kubectl apply -f "Kubernetes Containers/sidecarcont.yaml"
kubectl wait --for=condition=Ready pod/sidecar-demo --timeout=120s
kubectl get pod sidecar-demo
kubectl logs sidecar-demo -c app
kubectl logs sidecar-demo -c log-watcher
```

Both containers mount the same `emptyDir` volume. The app writes log lines
there and the sidecar follows the shared file.

## Exercise 3: ephemeral container

Create the target Pod from the example manifest:

```bash
kubectl apply -f "Kubernetes Containers/ephemeral-container.yaml"
kubectl wait --for=condition=Ready pod/ephemeral-debug-demo --timeout=120s
```

Add a temporary BusyBox debugger targeting the `app` container:

```bash
kubectl debug -it pod/ephemeral-debug-demo \
  --image=busybox:1.36 \
  --target=app \
  --container=debugger \
  -- sh
```

From the debugger shell, inspect processes (if process namespace targeting is
supported by the container runtime), then type `exit`:

```sh
ps
```

The ephemeral container remains recorded on the Pod after exiting. Inspect it
with:

```bash
kubectl get pod ephemeral-debug-demo \
  -o jsonpath='{.spec.ephemeralContainers[*].name}{"\n"}'
kubectl describe pod ephemeral-debug-demo
```

The ephemeral container is added through the Pod's `ephemeralcontainers`
subresource. Do not apply a Pod YAML that sets `spec.ephemeralContainers`:
Kubernetes does not allow ephemeral containers to be specified at Pod creation.
The checked-in YAML deliberately defines only the target Pod.

## Cleanup

Delete only the Pods created for this lab:

```bash
kubectl delete pod init-demo sidecar-demo ephemeral-debug-demo \
  --ignore-not-found
```