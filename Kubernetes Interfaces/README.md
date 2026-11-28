# Kubernetes interfaces, network policy, pod security, and upgrades

This standalone lab introduces the Container Runtime Interface (CRI),
Container Network Interface (CNI), and Container Storage Interface (CSI), then
works through NetworkPolicy, Pod Security Admission, and a cautious EKS upgrade
runbook. The exercises are independent of the other repository labs.

Run commands from the repository root unless a command says otherwise. Use a
disposable learning cluster that you are authorized to modify. The EKS upgrade
section is a planning and operations exercise: do not run upgrade, drain, or
delete commands against production by following this guide alone.

## Prerequisites

- `kubectl` configured for a disposable cluster.
- A cluster CNI that implements Kubernetes NetworkPolicy. Check the CNI
  provider's documentation; NetworkPolicy objects have no effect if the
  provider does not enforce them.
- For the upgrade walkthrough only: a disposable EKS cluster, AWS CLI access,
  and `eksctl`. EKS upgrades and LoadBalancer-backed resources can incur cost.

Confirm the current context before making changes:

```bash
kubectl config current-context
kubectl cluster-info
kubectl get nodes -o wide
```

## CRI, CNI, and CSI

Kubernetes uses standard interfaces so that runtime, networking, and storage
implementations can evolve independently:

| Interface | Responsibility | Examples |
| --- | --- | --- |
| CRI | Kubelet-to-container-runtime operations, such as creating containers and pulling images. | containerd, CRI-O |
| CNI | Pod network attachment and IP configuration. | Amazon VPC CNI, Calico, Cilium, Flannel |
| CSI | Storage provisioning, attachment, and mounting. | EBS CSI, EFS CSI, GCE PD CSI |

Inspect what the current cluster exposes:

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get csidrivers
kubectl get storageclass
kubectl get pv
kubectl get pvc -A
```

`crictl` and node-level CNI configuration are implementation-specific and
usually require access to a node. Use the runtime or CNI provider's documented
diagnostics rather than assuming a node filesystem path is available.

## Exercise 1: NetworkPolicy

The exercise starts with an unrestricted request from a client Pod to an Nginx
Service, applies a default-deny ingress policy, and then permits HTTP from the
client namespace. The namespace label selector uses Kubernetes' automatically
applied `kubernetes.io/metadata.name` label.

### Create the test workloads

```bash
kubectl apply -f "Kubernetes Interfaces/Network Policy/namespaces.yaml"
kubectl apply -f "Kubernetes Interfaces/Network Policy/server.yaml"
kubectl apply -f "Kubernetes Interfaces/Network Policy/server-service.yaml"
kubectl apply -f "Kubernetes Interfaces/Network Policy/client.yaml"
kubectl rollout status deployment/nginx -n server --timeout=120s
kubectl wait --for=condition=Ready pod/client -n client --timeout=120s
kubectl get pods,service -n server -o wide
```

Test before applying a policy. The command should return the Nginx welcome
page:

```bash
kubectl exec -n client pod/client -- \
  wget -qO- -T 3 http://nginx.server.svc.cluster.local
```

### Deny ingress, then allow the client namespace

```bash
kubectl apply -f "Kubernetes Interfaces/Network Policy/default-deny-ingress.yaml"
kubectl get networkpolicy -n server
```

Repeat the `wget` command. It should time out or fail while the default-deny
policy is active. Now add the narrowly scoped allow rule:

```bash
kubectl apply -f "Kubernetes Interfaces/Network Policy/allow-client-http.yaml"
kubectl get networkpolicy -n server
kubectl exec -n client pod/client -- \
  wget -qO- -T 3 http://nginx.server.svc.cluster.local
```

The request should succeed again. NetworkPolicies are additive: the allow
policy permits matching traffic while the default-deny policy continues to
block other ingress traffic to selected Pods.

### Cleanup

Delete only the test namespaces if they were created for this exercise and
contain no other work:

```bash
kubectl delete -f "Kubernetes Interfaces/Network Policy/namespaces.yaml"
```

## Exercise 2: Pod Security Admission

Pod Security Admission (PSA) enforces Pod Security Standards at namespace
scope. The `restricted` profile requires settings such as non-root execution,
no privilege escalation, dropping Linux capabilities, and a seccomp profile.

Create a namespace that enforces and warns about the restricted profile:

```bash
kubectl apply -f "Kubernetes Interfaces/Pod Security Admission/restricted-namespace.yaml"
kubectl get namespace restricted-lab --show-labels
```

Try the intentionally non-compliant manifest. The API server should reject
the Pod and report one or more restricted-profile violations:

```bash
kubectl apply -f "Kubernetes Interfaces/Pod Security Admission/violating-pod.yaml"
```

Apply the compliant Pod and verify that it is admitted and becomes Ready:

```bash
kubectl apply -f "Kubernetes Interfaces/Pod Security Admission/compliant-pod.yaml"
kubectl wait --for=condition=Ready pod/compliant -n restricted-lab --timeout=120s
kubectl get pod compliant -n restricted-lab
```

The manifests use `latest` for the PSA profile version so that the API server
evaluates its current restricted standard. For a version-pinned policy, set the
label values to a Kubernetes minor version supported by the cluster.

Cleanup:

```bash
kubectl delete namespace restricted-lab
```

## Exercise 3: EKS cluster-upgrade runbook

Kubernetes control planes are upgraded one minor version at a time. EKS
availability, supported versions, add-on compatibility, and `eksctl` command
syntax can change; consult the current AWS EKS documentation and release notes
before choosing a target version.

### Preflight and backup

1. Confirm the AWS account, region, cluster name, current Kubernetes version,
   node groups, and current `kubectl` context.
2. Read the Kubernetes and EKS release notes. Check deprecated APIs and
   validate manifests and add-on compatibility.
3. Test the procedure in a staging cluster and confirm an application-level
   recovery plan.
4. Back up cluster state with an approved backup solution and verify that
   recovery works. `kubectl get all` is not a complete cluster backup.
   Kubernetes Secret data is sensitive: do not save plaintext Secret exports
   in this repository or an unprotected location.
5. Check PodDisruptionBudgets and available capacity before any node
   replacement or drain.

Useful read-only checks:

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pdb -A
kubectl get pods -A
kubectl get events -A --sort-by=.lastTimestamp
eksctl get cluster --name <CLUSTER_NAME> --region <AWS_REGION>
eksctl get addons --cluster <CLUSTER_NAME> --region <AWS_REGION>
```

The sample workload and PDB below illustrate how a PDB protects replicas from
voluntary disruptions. Create them only in a disposable cluster:

```bash
kubectl apply -f "Kubernetes Interfaces/Cluster Upgrade/nginx-workload-and-pdb.yaml"
kubectl rollout status deployment/nginx -n upgrade-lab --timeout=120s
kubectl get deployment,pods,pdb -n upgrade-lab
```

### Upgrade sequence

Use your organization's change process and the current EKS documentation for
the exact commands and supported target version. A typical managed-cluster
sequence is:

1. Upgrade the EKS control plane by one supported minor version.
2. Update EKS add-ons to versions compatible with that control-plane version.
   Review configuration conflicts before applying add-on changes.
3. Roll out compatible worker nodes, preferably in a new managed node group
   with sufficient capacity for workload rescheduling.
4. Verify application health and system add-ons on the new nodes.
5. Cordon and drain old nodes one at a time, checking PDB status and workload
   availability before each drain.
6. Remove the old node group only after workloads and services have been
   verified on the replacement capacity.

The following are command shapes, not a complete upgrade script. Replace
placeholders only after verifying the cluster, region, version, node group,
and node name. They can disrupt workloads or incur cost:

```bash
# Example only: upgrade one minor version after preflight and approval.
eksctl upgrade cluster \
  --name <CLUSTER_NAME> \
  --region <AWS_REGION> \
  --version <TARGET_VERSION> \
  --approve

# Example only: create replacement capacity after confirming node and AMI
# compatibility, subnet capacity, and the expected AWS cost.
eksctl create nodegroup \
  --cluster <CLUSTER_NAME> \
  --name <NEW_NODEGROUP_NAME> \
  --region <AWS_REGION> \
  --version <TARGET_VERSION> \
  --node-type <NODE_TYPE> \
  --nodes <NODE_COUNT>

# Example only: choose a compatible add-on version from the EKS documentation.
eksctl update addon \
  --name <ADDON_NAME> \
  --cluster <CLUSTER_NAME> \
  --region <AWS_REGION> \
  --version <ADDON_VERSION> \
  --approve

# Example only: cordon and drain one old node after checking PDBs and capacity.
kubectl cordon <OLD_NODE_NAME>
kubectl drain <OLD_NODE_NAME> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=60

# Re-enable scheduling only if the node is intentionally being kept.
kubectl uncordon <NODE_NAME>
```

Do not use `--force` or bypass a blocked drain without understanding which
workload or disruption budget is preventing eviction. A PDB does not replace
replica health checks, spare capacity, backups, or an application recovery
plan.

### Post-upgrade verification

```bash
kubectl version -o yaml
kubectl get nodes -o wide
kubectl get pods -A
kubectl get csidrivers
kubectl get storageclass
kubectl get events -A --sort-by=.lastTimestamp
eksctl get addons --cluster <CLUSTER_NAME> --region <AWS_REGION>
```

Verify application behavior, storage mounts, networking, and monitoring before
declaring the upgrade complete. Remove the example workload only if it was
created for this exercise:

```bash
kubectl delete -f "Kubernetes Interfaces/Cluster Upgrade/nginx-workload-and-pdb.yaml"
```
