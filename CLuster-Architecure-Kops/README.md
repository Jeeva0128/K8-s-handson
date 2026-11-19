# Kubernetes architecture and AWS/kOps preparation

This is the AWS-focused lab. You will identify the Kubernetes control plane and
worker-node components, prepare an EC2 management host, verify IAM access, and
configure an S3 bucket for kOps state. It does **not** create a Kubernetes
cluster yet.

AWS resources can incur charges. Use an account and region you are permitted
to use, choose a unique bucket name, and delete resources when the lab is
complete. Do not copy the example names into a shared AWS account.

## Part 1: map the architecture

Before using AWS, write down what each component does:

| Component | Role |
| --- | --- |
| API server | Accepts and validates Kubernetes API requests. |
| etcd | Stores Kubernetes cluster state for the control plane. |
| Scheduler | Selects a worker node for unscheduled Pods. |
| Controller manager | Reconciles the desired and observed states. |
| kubelet | Runs and reports the Pods assigned to a node. |
| kube-proxy | Helps implement Service networking on a node. |
| Container runtime | Starts and manages containers. |

Trace this example on paper: a Deployment requests three replicas; the API
server stores it, controllers create Pods, the scheduler assigns nodes, and
kubelets start the containers.

## Part 2: prepare an EC2 management host

Create an Amazon Linux 2023 EC2 instance in the region you intend to use. For a
practice environment, record the instance ID, region, public address, and
attached IAM role. Keep SSH restricted to your own address.

Connect to the instance, then verify the operating system and identity:

```bash
cat /etc/os-release
aws --version
aws sts get-caller-identity
```

The identity should show the assumed IAM role, not a long-lived access key
embedded in a file. If `aws sts get-caller-identity` fails, fix the instance
profile or IAM permissions before continuing.

Install or verify the tools needed by the next kOps lab:

```bash
command -v aws
command -v kubectl
command -v kops
kubectl version --client
kops version
```

Use the official installation instructions for your Amazon Linux image if a
command is missing. Avoid downloading an unverified binary or placing
credentials in the repository.

## Part 3: create and secure the kOps state store

Choose values for your own session. The bucket name must be globally unique;
the state store region should match the region where you will run kOps:

```bash
export AWS_REGION=<your-aws-region>
export KOPS_STATE_BUCKET=<globally-unique-bucket-name>
```

Create the bucket using the syntax for your region. `us-east-1` does not use a
location constraint; other regions do:

```bash
aws s3api create-bucket \
  --bucket "$KOPS_STATE_BUCKET" \
  --region "$AWS_REGION" \
  $(test "$AWS_REGION" = us-east-1 || printf '%s\n' "--create-bucket-configuration" "LocationConstraint=$AWS_REGION")
```

If your shell does not accept the conditional argument above, run the
appropriate explicit command instead:

```bash
aws s3api create-bucket --bucket "$KOPS_STATE_BUCKET" --region us-east-1
aws s3api create-bucket --bucket "$KOPS_STATE_BUCKET" --region <other-region> \
  --create-bucket-configuration LocationConstraint=<other-region>
```

Enable the protections kOps state should have:

```bash
aws s3api put-bucket-versioning \
  --bucket "$KOPS_STATE_BUCKET" \
  --versioning-configuration Status=Enabled
aws s3api put-public-access-block \
  --bucket "$KOPS_STATE_BUCKET" \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
aws s3api put-bucket-encryption \
  --bucket "$KOPS_STATE_BUCKET" \
  --server-side-encryption-configuration \
  '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
```

Check each setting and confirm that the bucket is reachable from the EC2
instance:

```bash
aws s3api get-bucket-location --bucket "$KOPS_STATE_BUCKET"
aws s3api get-bucket-versioning --bucket "$KOPS_STATE_BUCKET"
aws s3api get-public-access-block --bucket "$KOPS_STATE_BUCKET"
aws s3api get-bucket-encryption --bucket "$KOPS_STATE_BUCKET"
aws s3 ls "s3://$KOPS_STATE_BUCKET"
```

The bucket is kOps' persistent configuration/state store. It is not the
Kubernetes control plane's etcd; etcd stores live cluster data inside the
cluster.

## Completion checklist

- [ ] The Kubernetes control-plane and worker-node roles are understood.
- [ ] `aws sts get-caller-identity` succeeds through the EC2 IAM role.
- [ ] `aws`, `kubectl`, and `kops` are available on the management host.
- [ ] The S3 bucket is private, encrypted, and versioned.
- [ ] The bucket region and name have been recorded for the next lab.

## Cleanup

Do not delete the bucket until you have finished any later kOps lab that uses
it. When it is no longer needed, remove its objects and then the bucket:

```bash
aws s3 rm "s3://$KOPS_STATE_BUCKET" --recursive
aws s3api delete-bucket --bucket "$KOPS_STATE_BUCKET" --region "$AWS_REGION"
```

Also stop or terminate the practice EC2 instance when you are finished, and
remove the IAM role only after confirming that no other resource uses it.

## Next step

Use the recorded bucket name as `s3://<bucket-name>` when configuring the kOps
cluster in the next lab. Keep the cluster name, DNS strategy, VPC, and
availability zones explicit before running `kops create cluster`.
