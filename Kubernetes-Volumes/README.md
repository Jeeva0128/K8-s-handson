# Kubernetes Storage & Volumes on Amazon EKS

Hands-on exercises for ephemeral Pod storage, dynamically provisioned Amazon
EBS volumes, and statically managed EBS volumes on Amazon EKS. The labs focus
on the lifecycle of the data as Pods are replaced, as well as the scheduling
and cleanup decisions that matter when storage is backed by an Availability
Zone-scoped cloud resource.

## Lab environment

| Component | Details |
| --- | --- |
| Kubernetes | EKS `v1.37.1-eks-9adbfea` |
| AWS Region | `ap-south-2` |
| Worker nodes | `ip-172-31-0-26.ap-south-2.compute.internal` and `ip-172-31-20-220.ap-south-2.compute.internal` |
| Worker Availability Zone | `ap-south-2c` |
| Client | Amazon Linux 2023 EC2 jump box with AWS CLI and `kubectl` |
| EBS integration | `aws-ebs-csi-driver` EKS add-on with the required IAM permissions |

These manifests use the default namespace. Confirm your current context before
applying them, and use a learning cluster you are authorized to modify.

```bash
kubectl config current-context
kubectl cluster-info
kubectl get nodes -o wide
```

The EBS CSI driver must be installed and healthy before running Labs 2 and 3.
Check the CSI pods and existing attachment objects:

```bash
kubectl get pods -n kube-system -o wide
kubectl get volumeattachments
```

## Numbered lab structure

This tree shows the requested logical grouping and teaching order; it does not
represent the manifests' current on-disk directories. The current manifest
paths are listed below the tree.

```text
Kubernetes-Volumes/
├── 01-emptydir/
│   └── emptyDir.yaml
├── 02-ebs-dynamic/
│   └── ebs-dynamic.yaml
└── 03-ebs-static/
    └── ebs-static.yaml
```

Current manifest paths:

```text
Kubernetes-Volumes/emptyDir.yaml
Kubernetes-Volumes/Persistent Volumes/ebs-dynamic.yaml
Kubernetes-Volumes/Persistent Volumes/ebs-static.yaml
```

Commands below use the current manifest paths, including quoting the directory
name that contains a space.

## Storage concepts at a glance

| Example | Backing storage and lifetime | Access mode | Binding behavior | Reclaim policy |
| --- | --- | --- | --- | --- |
| `emptyDir` | Node-local ephemeral directory; removed when the Pod is removed from the node | Shared by containers in that Pod; not a PVC access mode | Not applicable | Not applicable |
| Dynamic EBS (`gp3`) | EBS volume created for the PVC; survives Pod replacement while the claim remains | `ReadWriteOnce` | `WaitForFirstConsumer` delays provisioning/binding until a consuming Pod can be scheduled | `Delete` |
| Static EBS (`gp3`) | Existing EBS volume `vol-0f6adf447d0e48658` | `ReadWriteOnce` | Pre-created PV matched to a PVC using `manual-ebs`; no dynamic StorageClass provisioning | `Retain` |

`ReadWriteOnce` allows a volume to be mounted read-write by a single node at a
time. It does not mean only one Pod can ever use the volume: multiple Pods on
the same node may be able to mount it, subject to the workload and CSI driver.
EBS volumes are Availability Zone scoped, so the consuming node must be in the
volume's AZ.

## Lab 1: Ephemeral storage with `emptyDir`

### What this demonstrates

The `emptydir-sidecar-demo` Pod has an `app-writer` container that appends a
timestamped entry to `/shared-logs/app.log`, and a `log-reader-sidecar`
container that tails the same file. Kubernetes mounts one `emptyDir` volume
into both containers, allowing them to share files without a network service
or persistent storage.

```yaml
volumes:
  - name: shared-storage
    emptyDir: {}
# Both containers mount shared-storage at /shared-logs.
```

Manifest: [`emptyDir.yaml`](./emptyDir.yaml)

### Deploy and verify

```bash
kubectl apply -f Kubernetes-Volumes/emptyDir.yaml
kubectl wait --for=condition=Ready pod/emptydir-sidecar-demo --timeout=120s
kubectl logs emptydir-sidecar-demo -c log-reader-sidecar
```

The sidecar log should show the entries written by `app-writer`. The writer
appends every five seconds, so run the log command again if the Pod has only
just started.

### Clean up

```bash
kubectl delete pod emptydir-sidecar-demo
```

The `emptyDir` data is temporary: deleting the Pod removes the volume and its
contents.

## Lab 2: Dynamic Amazon EBS provisioning

### What this demonstrates

The `ebs-dynamic-sc` StorageClass asks the EBS CSI driver to provision `gp3`
storage. The 4Gi `ebs-dynamic-pvc` claim requests `ReadWriteOnce`. Its consumer,
`ebs-dynamic-pod`, mounts the claim at `/data`.

`WaitForFirstConsumer` is useful for topology-aware storage: provisioning is
delayed until Kubernetes knows which node will consume the claim, allowing the
EBS volume to be created in a compatible Availability Zone.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-dynamic-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
parameters:
  type: gp3
```

Manifest: [`ebs-dynamic.yaml`](<./Persistent Volumes/ebs-dynamic.yaml>)

### Deploy and verify

```bash
kubectl apply -f "Kubernetes-Volumes/Persistent Volumes/ebs-dynamic.yaml"
kubectl wait --for=jsonpath='{.status.phase}'=Bound pvc/ebs-dynamic-pvc --timeout=180s
kubectl wait --for=condition=Ready pod/ebs-dynamic-pod --timeout=180s
kubectl get pvc ebs-dynamic-pvc
kubectl get pv
kubectl describe pod ebs-dynamic-pod
```

Confirm the claim is `Bound`, the Pod is `Running`, and the PV is backed by the
EBS CSI driver. The selected node and volume topology should be compatible.

### Persistence test: replace the Pod, keep the claim

Write a record to the mounted filesystem, delete only the Pod, then recreate
the Pod with the existing PVC:

```bash
kubectl exec ebs-dynamic-pod -- sh -c 'date -u "+%Y-%m-%dT%H:%M:%SZ" >> /data/record.txt'
kubectl exec ebs-dynamic-pod -- cat /data/record.txt
kubectl delete pod ebs-dynamic-pod
kubectl wait --for=delete pod/ebs-dynamic-pod --timeout=120s
kubectl apply -f "Kubernetes-Volumes/Persistent Volumes/ebs-dynamic.yaml"
kubectl wait --for=condition=Ready pod/ebs-dynamic-pod --timeout=180s
kubectl exec ebs-dynamic-pod -- cat /data/record.txt
```

The record should still be present after the Pod is recreated because the PVC
and its EBS-backed PV were not deleted.

### Clean up and confirm automatic volume deletion

Capture the provisioned EBS volume ID while the claim and PV still exist:

```bash
kubectl get pvc ebs-dynamic-pvc
kubectl get pv
```

Then remove the consumer and claim. With the StorageClass reclaim policy set to
`Delete`, deleting the PVC causes the dynamically provisioned PV and its EBS
volume to be deleted by the CSI integration.

```bash
kubectl delete pod ebs-dynamic-pod
kubectl delete pvc ebs-dynamic-pvc
kubectl delete storageclass ebs-dynamic-sc
kubectl get pvc,pv
```

Use the volume ID noted above to verify in the AWS console or with the AWS CLI
that the EBS volume is no longer present. The StorageClass can be removed after
the claim; deleting it does not itself delete an EBS volume.

## Lab 3: Static Amazon EBS provisioning with `Retain`

### What this demonstrates

This lab binds an existing 3Gi `gp3` EBS volume to a manually defined PV and
PVC. The PV uses `storageClassName: manual-ebs` and `persistentVolumeReclaimPolicy:
Retain`; it is not dynamically provisioned by a StorageClass. The Pod mounts
the claim at `/data`.

The EBS volume `vol-0f6adf447d0e48658` was provisioned in `ap-south-2c`. The Pod
requires a node with `disktype=ssd`, and the node
`ip-172-31-20-220.ap-south-2.compute.internal` is labeled accordingly. Verify
the selected node is also in `ap-south-2c` before scheduling the Pod. The
custom label is a scheduling selector; it does not by itself encode or verify
an Availability Zone.

```yaml
spec:
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual-ebs
  csi:
    driver: ebs.csi.aws.com
    volumeHandle: vol-0f6adf447d0e48658
```

The Pod's required node affinity selects the labeled node:

```yaml
nodeSelectorTerms:
  - matchExpressions:
      - key: disktype
        operator: In
        values:
          - ssd
```

Manifest: [`ebs-static.yaml`](<./Persistent Volumes/ebs-static.yaml>)

### Prepare, deploy, and verify

Check node labels and AZs, then apply the `disktype=ssd` label to the intended
node. The command is idempotent if the label already has the desired value.

```bash
kubectl get nodes -L topology.kubernetes.io/zone,disktype -o wide
kubectl label node ip-172-31-20-220.ap-south-2.compute.internal disktype=ssd --overwrite
kubectl get node ip-172-31-20-220.ap-south-2.compute.internal --show-labels
kubectl apply -f "Kubernetes-Volumes/Persistent Volumes/ebs-static.yaml"
kubectl wait --for=jsonpath='{.status.phase}'=Bound pvc/ebs-static-pvc --timeout=180s
kubectl wait --for=condition=Ready pod/ebs-static-pod --timeout=180s
kubectl get pv ebs-static-pv
kubectl get pvc ebs-static-pvc
kubectl get pod ebs-static-pod -o wide
```

Confirm that the Pod landed on a node in the EBS volume's AZ and that the PVC
is bound to `ebs-static-pv`. If the volume was attached and mounted
successfully, the Pod should become `Running`; inspect its events if it does
not.

### Cleanup and retained data

Remove the Pod and claim before removing the PV. A PV with `Retain` does not
automatically delete the AWS volume when its PVC is deleted. Keep the EBS
volume if you intend to preserve its data; delete it separately in AWS only
when you have confirmed it is no longer needed.

```bash
kubectl delete pod ebs-static-pod
kubectl delete pvc ebs-static-pvc
kubectl get pv ebs-static-pv
kubectl delete pv ebs-static-pv
```

After deleting the PV object, verify the retained volume state in AWS and
perform any intended manual cleanup there. Remove the node label only if no
other workload depends on it:

```bash
kubectl label node ip-172-31-20-220.ap-south-2.compute.internal disktype-
```

## Real-world troubleshooting & lessons learned

### Availability Zone and node-affinity matching

Amazon EBS volumes are tied to the AZ in which they were created. A Pod can
remain `Pending` with `FailedScheduling` events if its eligible nodes are in a
different AZ from the volume. Check the Pod events, node labels/AZs, and PV
topology before changing scheduling rules:

```bash
kubectl describe pod ebs-static-pod
kubectl get nodes -L topology.kubernetes.io/zone,disktype -o wide
kubectl describe pv ebs-static-pv
kubectl get events --sort-by=.lastTimestamp
```

The static manifest selects `disktype=ssd`; the label must be applied to a node
in the volume's `ap-south-2c` AZ. Node affinity and volume topology must agree.
A label such as `disktype=ssd` is an operator-defined marker, not proof of
where a node runs.

### PersistentVolume finalizers and `Terminating`

A PV can stay in `Terminating` while it is still bound to a PVC or in use by a
Pod. Kubernetes protection finalizers prevent the storage object from being
removed while dependent resources still reference it. Use the dependency order:

```bash
kubectl delete pod ebs-static-pod
kubectl delete pvc ebs-static-pvc
kubectl get pv ebs-static-pv
kubectl describe pv ebs-static-pv
```

Wait for the Pod and PVC to be gone before deleting the PV. Do not remove
finalizers as a first response; first identify and resolve the remaining
consumer or claim.

### CSI attachment and reclaim-policy behavior

When a claim is bound but the Pod cannot mount its volume, inspect Pod events,
CSI controller/node pods, and `VolumeAttachment` objects:

```bash
kubectl describe pod ebs-dynamic-pod
kubectl get pods -n kube-system -o wide
kubectl get volumeattachments
kubectl describe pvc ebs-dynamic-pvc
```

The EBS CSI driver must be healthy and have the required IAM permissions to
create, attach, detach, and delete volumes. `Delete` on the dynamic StorageClass
requests automatic deletion of the dynamically provisioned EBS volume after
the claim is deleted. `Retain` on the static PV keeps the underlying EBS volume
after claim release, requiring deliberate manual cleanup in AWS. Verify the
result in both Kubernetes resources and the AWS volume inventory; deleting a
Kubernetes object is not proof that the cloud resource has been removed.

## Further reading

- [Amazon EKS EBS CSI driver](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html)
- [Kubernetes Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Kubernetes Volumes: `emptyDir`](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir)
