# Kubernetes Health Probes

This lab demonstrates how Kubernetes uses container probes to decide whether
to restart a container, send traffic to a Pod, or wait for an application to
finish starting. The examples are standalone Pods in the `default` namespace.
Run them only in a cluster you are authorized to modify.

## Probe types

| Probe | What it checks | On failure |
| --- | --- | --- |
| Liveness | Whether the container should continue running | Kubelet restarts the container after the configured consecutive failures |
| Readiness | Whether the container can receive traffic | The Pod is marked not ready and is removed from matching Service endpoints; the container is not restarted |
| Startup | Whether a slow-starting container has finished initialization | Other probes are held until startup succeeds; repeated failure causes a restart |

Use liveness for a process that is stuck and cannot recover, not for a
temporary dependency outage. A readiness check should represent whether this
instance can serve requests. A startup probe is useful when the application
needs more time before liveness/readiness checks are meaningful.

## Repository files

| Manifest | Purpose |
| --- | --- |
| [`liveness.yaml`](./liveness.yaml) | NGINX Pod with an HTTP liveness check on `/` port `80` |
| [`liveness-busybox.yaml`](./liveness-busybox.yaml) | BusyBox Pod with an exec liveness check for `/tmp/healthy` |
| [`probe.yaml`](./probe.yaml) | Draft probe example; see the configuration warning below before applying |

## Lab 1: HTTP liveness probe with NGINX

The NGINX container serves its default page on port `80`. The manifest's HTTP
GET to `/` checks that the web server responds. If the check fails three times
in a row, Kubernetes restarts the container.

```yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3
```

### Deploy and verify

```bash
kubectl apply -f "Kubernetes Probes/liveness.yaml"
kubectl wait --for=condition=Ready pod/liveness-nginx --timeout=120s
kubectl get pod liveness-nginx
kubectl describe pod liveness-nginx
```

Check that the Pod is `Running` and `Ready`, and that its restart count remains
`0`. In `describe`, inspect the liveness probe configuration and events.

<p align="center">
  <img
    src="../docs/Screenshot%20From%202026-11-26%2016-55-15.png"
    alt="kubectl describe output for the running NGINX Pod and its successful HTTP liveness probe configuration"
    width="100%"
  />
</p>

## Lab 2: Exec liveness probe with BusyBox

The BusyBox container creates `/tmp/healthy` when it starts. The liveness probe
runs `cat /tmp/healthy`; a successful read means the check exits with status
zero. The file is created by the container command in the checked-in manifest:

```yaml
command: ["sh", "-c", "touch /tmp/healthy && sleep 3600"]
livenessProbe:
  exec:
    command: ["cat", "/tmp/healthy"]
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 3
```

### Deploy and verify

```bash
kubectl apply -f "Kubernetes Probes/liveness-busybox.yaml"
kubectl wait --for=condition=Ready pod/liveness-busybox --timeout=120s
kubectl exec liveness-busybox -- cat /tmp/healthy
kubectl get pod liveness-busybox
kubectl describe pod liveness-busybox
```

The file is empty, so `cat` prints no content; its successful exit status is
the important result. With the current manifest, the Pod should remain ready
without its restart count increasing.

## Troubleshooting: liveness failure and restart

The BusyBox screenshots capture a liveness failure where the probe reports
that `/tmp/healthy` does not exist, followed by container restarts. Kubernetes
restarts a container after the probe fails consecutively up to
`failureThreshold`; a single failed check does not immediately restart it.

<p align="center">
  <img
    src="../docs/Screenshot%20From%202026-11-26%2017-13-03.png"
    alt="kubectl describe events showing the BusyBox liveness check failing because /tmp/healthy was missing and the container being restarted"
    width="100%"
  />
</p>

The current checked-in manifest runs `touch /tmp/healthy` before sleeping, so a
fresh Pod created from this file should not reproduce that failure. If it
does, verify the live Pod spec and check that the probe path matches the file
created by the container:

```bash
kubectl get pod liveness-busybox -o yaml
kubectl exec liveness-busybox -- ls -l /tmp/healthy
kubectl describe pod liveness-busybox
kubectl logs liveness-busybox --previous
```

Use the restart count and recent events together. `kubectl logs --previous`
shows logs from the prior container instance when that instance is available.

<p align="center">
  <img
    src="../docs/Screenshot%20From%202026-11-26%2017-11-35.png"
    alt="BusyBox terminal session followed by Pod status showing its restart count"
    width="100%"
  />
</p>

## Configuration warning: `probe.yaml`

[`probe.yaml`](./probe.yaml) is not a working health-check example as currently
written. It uses `httpd:latest` with probes on port `8080` and paths
`/healthz`, `/ready`, and `/startup`. The stock image does not configure those
application endpoints or listen on that port by default. In addition, the
startup probe must succeed before the other probes can run. Applying this
manifest as-is can leave the Pod failing its probes and restarting; use an
application configured to serve those endpoints, or align the checks with the
actual container port and health paths before applying it.

## Cleanup

Delete only the Pods created for this lab:

```bash
kubectl delete pod liveness-nginx liveness-busybox --ignore-not-found
```