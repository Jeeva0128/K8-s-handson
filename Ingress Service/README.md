# Shared AWS ALB Ingress Lab

This lab routes requests from one AWS Application Load Balancer (ALB) to
applications owned by two Kubernetes namespaces:

| Request path | Namespace | Service | Expected response |
| --- | --- | --- | --- |
| `/a` | `team-a` | `app-a-svc` | `Hello from App A LWM` |
| `/b` | `team-b` | `app-b-svc` | `Hello from App B LWM` |

Each team has a three-replica Deployment, a ClusterIP Service, and an Ingress.
Both Ingress resources use the `shared-alb` group, allowing the AWS Load
Balancer Controller to combine their rules on one ALB.

## Prerequisites

- An AWS Kubernetes cluster with the AWS Load Balancer Controller installed.
- The controller configured with the correct VPC, subnets, and IAM permissions.
- `kubectl` configured to use that cluster.

The manifests request an internet-facing ALB with IP targets. Applying them
can create billable AWS resources. Do not apply them to a production cluster
unless you have reviewed the impact. This lab is not intended for Minikube or
clusters without the AWS Load Balancer Controller.

## Deploy

Run these commands from the repository root. Create the namespaces before
deploying resources into them:

```bash
kubectl apply -f "Ingress Service/Ingress/namespaces.yaml"
kubectl apply -f "Ingress Service/Ingress/team-a.yaml"
kubectl apply -f "Ingress Service/Ingress/team-b.yaml"
```

Wait for both Deployments and inspect the Ingress resources:

```bash
kubectl rollout status deployment/app-a -n team-a
kubectl rollout status deployment/app-b -n team-b
kubectl get deployments,pods,services -n team-a
kubectl get deployments,pods,services -n team-b
kubectl get ingress -A
```

The ALB can take a few minutes to provision. Get its hostname from either
Ingress:

```bash
kubectl get ingress app-a-ingress -n team-a \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}{"\n"}'
```

Use the returned hostname to test both routes:

```bash
curl http://<alb-hostname>/a
curl http://<alb-hostname>/b
```

The responses should identify App A and App B respectively. The same ALB
hostname serves both paths.

## Troubleshooting

Check that each Service has endpoints and that the ALB controller has
processed the Ingress resources:

```bash
kubectl get endpointslice -n team-a -l kubernetes.io/service-name=app-a-svc
kubectl get endpointslice -n team-b -l kubernetes.io/service-name=app-b-svc
kubectl describe ingress app-a-ingress -n team-a
kubectl describe ingress app-b-ingress -n team-b
kubectl get events -n team-a --sort-by=.lastTimestamp
kubectl get events -n team-b --sort-by=.lastTimestamp
```

If the ALB hostname is missing or the targets are unhealthy, check the AWS
Load Balancer Controller logs, IAM permissions, subnet discovery, and the
cluster's VPC configuration.

## Cleanup

Delete the team resources first. Removing both Ingress resources allows the
controller to remove the shared ALB:

```bash
kubectl delete -f "Ingress Service/Ingress/team-a.yaml" --ignore-not-found
kubectl delete -f "Ingress Service/Ingress/team-b.yaml" --ignore-not-found
```

After confirming the namespaces are not used by other workloads, delete them:

```bash
kubectl delete -f "Ingress Service/Ingress/namespaces.yaml"
```