# Task 11: Resolve Pod Deployment Issue

A junior DevOps team member encountered difficulties deploying a stack on the Kubernetes cluster. The pod fails to start, presenting errors. Let's troubleshoot and rectify the issue promptly.

There is a pod named webserver, and the container within it is named nginx-container, its utilizing the nginx:latest image.

Additionally, there's a sidecar container named sidecar-container using the ubuntu:latest image.

Identify and address the issue to ensure the pod is in the running state and the application is accessible.

Note: The kubectl utility on the jump-host has been configured to work with the Kubernetes cluster.

## Task Requirements

1. Troubleshoot the existing Pod named `webserver`.
2. Ensure the `nginx-container` uses `nginx:latest` and the `sidecar-container` uses `ubuntu:latest`.
3. Resolve the issue so the Pod reaches the `Running` state and the application is accessible.

## Solution

The Pod contained two containers: the main `nginx-container` and the `sidecar-container`. The initial status was `1/2` with `ImagePullBackOff`, so the Pod was not ready because one container could not download its image.

The detailed output revealed the exact problem: the main container was configured with `nginx:latests`, but that image tag does not exist. The sidecar was already running; its repeated log errors were secondary because nginx had not started and had not created the expected log files. The fix was to change only the image tag from `nginx:latests` to `nginx:latest`.

### 🔎 Step 1: Check the Pod status

```bash
kubectl get pod webserver
```

The initial status showed that only one of the two containers was ready:

```text
thor@jump-host ~$ kubectl get pod webserver
NAME        READY   STATUS             RESTARTS   AGE
webserver   1/2     ImagePullBackOff   0          4m42s
thor@jump-host ~$
```

> **Why:** `kubectl` is the Kubernetes command-line client. `get` retrieves resource information, `pod` selects the Pod resource type, and `webserver` selects the resource by name. `READY=1/2` means one of the two containers was ready. `ImagePullBackOff` means Kubernetes could not pull an image and was waiting longer between retry attempts.

### 🔎 Step 2: Inspect the Pod details and events

```bash
kubectl describe pod webserver
```

The description showed the invalid image reference and the pull errors:

```text
Name:             webserver
Namespace:        default
Priority:         0
Service Account:  default
Node:             jump-host/10.244.81.19
Start Time:       Thu, 23 Jul 2026 01:34:07 +0000
Labels:           app=web-app
Annotations:      <none>
Status:           Pending
IP:               10.22.0.9
IPs:
  IP: 10.22.0.9
Containers:
  nginx-container:
    Container ID:
    Image:          nginx:latests
    Image ID:
    Port:           <none>
    Host Port:      <none>
    State:          Waiting
      Reason:       ImagePullBackOff
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/log/nginx from shared-logs (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-47dsp (ro)
  sidecar-container:
    Container ID:  containerd://4f14ea9edbe8461e2208134a327a7c5937b8e8a55ca090e531c25d63dd2d7244
    Image:         ubuntu:latest
    Image ID:      docker.io/library/ubuntu@sha256:3131b4cc82a783df6c9df078f86e01819a13594b865c2cad47bd1bca2b7063bb
    Port:          <none>
    Host Port:     <none>
    Command:
      sh
      -c
      while true; do cat /var/log/nginx/access.log /var/log/nginx/error.log; sleep 30; done
    State:          Running
      Started:      Thu, 23 Jul 2026 01:34:11 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/log/nginx from shared-logs (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-47dsp (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True
  Initialized                 True
  Ready                       False
  ContainersReady             False
  PodScheduled                True
Events:
  Type     Reason     Age                   From               Message
  ----     ------     ----                  ----               -------
  Normal   Scheduled  4m57s                 default-scheduler  Successfully assigned default/webserver to jump-host
  Normal   Pulling    4m56s                 kubelet            Pulling image "ubuntu:latest"
  Normal   Pulled     4m53s                 kubelet            Successfully pulled image "ubuntu:latest" in 2.147s (2.147s including waiting). Image size: 41593622 bytes.
  Normal   Created    4m53s                 kubelet            Created container: sidecar-container
  Normal   Started    4m53s                 kubelet            Started container sidecar-container
  Normal   Pulling    111s (x5 over 4m56s)  kubelet            Pulling image "nginx:latests"
  Warning  Failed     110s (x5 over 4m56s)  kubelet            Failed to pull image "nginx:latests": rpc error: code = NotFound desc = failed to pull and unpack image "docker.io/library/nginx:latests": failed to resolve reference "docker.io/library/nginx:latests": docker.io/library/nginx:latests: not found
  Warning  Failed     110s (x5 over 4m56s)  kubelet            Error: ErrImagePull
  Warning  Failed     69s (x15 over 4m52s)  kubelet            ImagePullBackOff
  Normal   BackOff    54s (x16 over 4m52s)  kubelet            Back-off pulling image "nginx:latests"
thor@jump-host ~$
```

> **Why:** `describe` displays detailed resource information, including container states, image references, mounts, conditions, and recent events. The important evidence was `Image: nginx:latests` and the event saying that `docker.io/library/nginx:latests` was not found. `ubuntu:latest` was pulled successfully and `sidecar-container` was `Running`, so the sidecar was not the image-pull problem.

### 🔎 Step 3: Inspect the sidecar logs

```bash
kubectl logs webserver -c sidecar-container
```

The sidecar repeatedly reported that the nginx log files did not exist:

```text
thor@jump-host ~$ kubectl logs webserver -c sidecar-container
cat: /var/log/nginx/access.log: No such file or directory
cat: /var/log/nginx/error.log: No such file or directory
cat: /var/log/nginx/access.log: No such file or directory
cat: /var/log/nginx/error.log: No such file or directory
cat: /var/log/nginx/access.log: No such file or directory
cat: /var/log/nginx/error.log: No such file or directory
...
thor@jump-host ~$
```

> **Why:** `logs` prints the logs for a container in a Pod. `webserver` selects the Pod, and `-c sidecar-container` selects the sidecar instead of the main nginx container. These messages were a consequence of the main container not starting: nginx had not created `/var/log/nginx/access.log` or `/var/log/nginx/error.log` yet. The logs did not indicate that the sidecar image itself was invalid.

### 🛠️ Step 4: Correct the nginx image tag

```bash
kubectl edit pod webserver
```

In the editor, locate the `nginx-container` definition and change only this value:

```yaml
image: nginx:latests
```

to:

```yaml
image: nginx:latest
```

Save and exit with `Esc`, `:wq`, and `Enter`.

The edit completed successfully:

```text
thor@jump-host ~$ kubectl edit pod webserver
pod/webserver edited
thor@jump-host ~$
```

> **Why:** `edit` retrieves the live Pod definition and opens it in the configured editor. The only required change was correcting the nonexistent `latests` tag to the valid `latest` tag. The image field can be updated on an existing Pod, and Kubernetes then retries the image pull. The container names, sidecar command, shared volume, and other working settings were left unchanged.

### ✅ Step 5: Verify that both containers are running

```bash
kubectl get pod webserver
```

The final output confirmed that both containers were ready and the Pod was running:

```text
thor@jump-host ~$ kubectl get pod webserver
NAME        READY   STATUS    RESTARTS   AGE
webserver   2/2     Running   0          8m52s
thor@jump-host ~$
```

> **Why:** `READY=2/2` confirms that both `nginx-container` and `sidecar-container` are ready. `STATUS=Running` confirms that the Pod is operational. Once nginx started successfully, it could create its log files in the shared volume, allowing the sidecar's log-reading loop to work as intended.

## Root Cause

The root cause was a typo in the main container's image tag:

```text
nginx:latests  →  invalid image reference
nginx:latest   →  valid image reference
```

The sidecar was healthy enough to run, but its log-reading command produced `No such file or directory` because the nginx container never started. Correcting the image reference resolved both symptoms.

## Best Practices

- **Read Pod events first when an image will not start.** `kubectl describe pod` exposes the exact image reference and the kubelet's pull error.
- **Check image names and tags character by character.** A small typo such as `latests` causes `ErrImagePull` and `ImagePullBackOff`.
- **Separate primary and secondary symptoms.** The sidecar log errors were caused by the missing nginx log files, not by an invalid Ubuntu image.
- **Change only the broken field.** The sidecar, shared volume, container names, and commands were already correct, so they were left unchanged.
- **Use a shared volume for sidecar communication.** Both containers mounted `shared-logs` at `/var/log/nginx`, allowing the sidecar to read files created by nginx after the main container started.
- **Confirm all containers are ready.** A Pod with two containers should report `2/2`, not merely `Running` with one ready container.

### 📚 Official Documentation

- [`kubectl get` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/)
- [`kubectl describe` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_describe/)
- [`kubectl logs` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/)
- [`kubectl edit` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_edit/)
- [Kubernetes container images](https://kubernetes.io/docs/concepts/containers/images/)
- [Kubernetes sidecar containers](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)
