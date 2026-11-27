# Kubernetes Environment Variables

This lab demonstrates setting environment variables for a container directly
in a Pod specification, including a value populated from Pod metadata through
the Downward API. The example uses the `default` namespace. Run it only in a
cluster you are authorized to modify.

## Repository example

| Manifest | Purpose |
| --- | --- |
| [`environment-variables.yaml`](./environment-variables.yaml) | Runs a BusyBox Pod with literal environment variables and the Pod name injected from metadata. |
| [`configmap.yaml`](./configmap.yaml) | Defines non-sensitive application settings in a ConfigMap. |
| [`secret.yaml`](./secret.yaml) | Defines demo Secret values in Base64-encoded `data` fields. |
| [`secret-pod.yaml`](./secret-pod.yaml) | Runs a BusyBox Pod that reads the Secret values as environment variables. |

## ConfigMap

A ConfigMap stores non-sensitive configuration separately from a Pod. Apply
the example and inspect its contents:

```bash
kubectl apply -f "Kubernetes env/configmap.yaml"
kubectl get configmap app-config
kubectl describe configmap app-config
```

The manifest defines `APP_ENV` and `LOG_LEVEL`. A Pod can consume ConfigMap
keys as environment variables or as mounted files; this example only creates
the ConfigMap resource.

## Secret

A Secret is intended for sensitive values. In this example, `data` contains
Base64-encoded demo values. **Base64 is encoding, not encryption**; anyone who
can read the manifest or Secret can decode the values. Do not put real
credentials in this example or commit them to source control. Restrict Secret
access with RBAC and configure encryption at rest for your cluster.

The demo values in [`secret.yaml`](./secret.yaml) decode to:

| Key | Base64 value | Decoded demo value |
| --- | --- | --- |
| `username` | `ZGVtby11c2Vy` | `demo-user` |
| `password` | `cmVwbGFjZS1tZQ==` | `replace-me` |

You can decode the manifest values locally:

```bash
printf '%s' 'ZGVtby11c2Vy' | base64 --decode; echo
printf '%s' 'cmVwbGFjZS1tZQ==' | base64 --decode; echo
```

Apply the Secret, then create the BusyBox Pod that consumes it:

```bash
kubectl apply -f "Kubernetes env/secret.yaml"
kubectl apply -f "Kubernetes env/secret-pod.yaml"
kubectl wait --for=condition=Ready pod/secret-demo --timeout=120s
```

The Pod obtains both variables through `secretKeyRef`. This demo deliberately
prints the decoded values in its logs so you can observe the result:

```bash
kubectl logs secret-demo
```

This log output is safe only because the values are fictional placeholders.
Never print real Secret values to logs. To inspect just the Secret keys
without exposing their values, run:

```bash
kubectl describe secret app-secret
```

Delete the Secret example resources when finished:

```bash
kubectl delete -f "Kubernetes env/secret-pod.yaml" --ignore-not-found
kubectl delete -f "Kubernetes env/secret.yaml" --ignore-not-found
```

## Deploy and verify

Apply the manifest and wait for the Pod to become ready:

```bash
kubectl apply -f "Kubernetes env/environment-variables.yaml"
kubectl wait --for=condition=Ready pod/env-demo --timeout=120s
```

The container prints the configured values when it starts:

```bash
kubectl logs env-demo
```

You can also inspect the variables in the running container:

```bash
kubectl exec env-demo -- printenv APP_ENV
kubectl exec env-demo -- printenv LOG_LEVEL
kubectl exec env-demo -- printenv POD_NAME
```

`APP_ENV` and `LOG_LEVEL` are literal values in the manifest. `POD_NAME` uses
`valueFrom.fieldRef` to read `metadata.name` from the Pod. Avoid putting
passwords, tokens, or other sensitive values directly in a Pod manifest;
use a Kubernetes Secret and reference it instead.

## Cleanup

```bash
kubectl delete pod env-demo --ignore-not-found
```
