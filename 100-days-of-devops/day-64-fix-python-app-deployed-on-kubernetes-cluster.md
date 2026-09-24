# Day 64: Fix Python App Deployed on Kubernetes Cluster

One of the DevOps engineers was trying to deploy a python app on Kubernetes cluster. Unfortunately, due to some mis-configuration, the application is not coming up. Please take a look into it and fix the issues. Application should be accessible on the specified nodePort.

## Specific Requirements:

1. The deployment name is `python-deployment-devops`, its using `poroko/flask-demo-app` image. The deployment and service of this app is already deployed.
2. nodePort should be `32345` and targetPort should be python flask app's default port.

`Note:` The `kubectl` utility on the `jump-host` has been configured to work with the Kubernetes cluster.

## Solution

The application had two independent configuration errors. The Deployment referenced the nonexistent image `poroko/flask-app-demo` instead of the required `poroko/flask-demo-app`, leaving its Pod in `ImagePullBackOff`. After correcting the image, the Service still forwarded traffic to port `8080`, while the Flask application listened on its default port `5000`. Correcting both settings restored the complete request path through NodePort `32345`.

### 🔍 Step 1: Inspect the Existing Resources

```bash
kubectl get deployment python-deployment-devops
kubectl get pods
kubectl get services
```

The initial state showed an unavailable Deployment, a Pod that could not pull its image, and an existing NodePort Service:

```text
NAME                       READY   UP-TO-DATE   AVAILABLE
python-deployment-devops   0/1     1            0

NAME                                        READY   STATUS
python-deployment-devops-7dd8f6ddf8-fxqz4   0/1     ImagePullBackOff

NAME                    TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)
python-service-devops   NodePort   10.43.90.53   <none>        8080:32345/TCP
```

> **Why:** `kubectl get deployment` summarizes the desired and available replicas; `0/1` confirmed that the application was unavailable. `kubectl get pods` exposed the immediate `ImagePullBackOff` symptom. `kubectl get services` confirmed that `python-service-devops` already had type `NodePort` and used the required external port `32345`, so recreating the Service was unnecessary.

### 🧭 Step 2: Diagnose the Image Pull Failure

```bash
kubectl describe deployment python-deployment-devops
kubectl describe pod python-deployment-devops-7dd8f6ddf8-fxqz4
```

The Deployment contained the wrong image name:

```text
Image:  poroko/flask-app-demo
Port:   5000/TCP
```

The Pod events identified the direct cause:

```text
Failed to pull image "poroko/flask-app-demo"
pull access denied, repository does not exist or may require authorization
Error: ErrImagePull
Error: ImagePullBackOff
```

> **Why:** `kubectl describe deployment` displays the active Pod template, including the container name, image, declared port, labels, replica state, and rollout events. `kubectl describe pod` adds container state and kubelet events. `ImagePullBackOff` means Kubernetes could not pull the image and is delaying repeated attempts. Here the event explicitly showed that `poroko/flask-app-demo` did not resolve, while the challenge required `poroko/flask-demo-app`.

### 🛠️ Step 3: Correct the Deployment Image

```bash
kubectl edit deployments.apps python-deployment-devops
```

Change the container image from:

```yaml
image: poroko/flask-app-demo
```

to:

```yaml
image: poroko/flask-demo-app
```

Save and close the editor. Kubernetes confirmed the update:

```text
deployment.apps/python-deployment-devops edited
```

> **Why:** `kubectl edit` opens the live Deployment definition in an editor and updates it after the file is saved. `deployments.apps` explicitly identifies the Deployment resource in the `apps` API group, while `python-deployment-devops` is its name. Changing the Pod-template image causes the Deployment controller to create a new ReplicaSet and replace the broken Pod with one based on the valid image.

### ⏳ Step 4: Verify the Deployment Rollout

```bash
kubectl rollout status deployment python-deployment-devops
kubectl get pods
```

The corrected Deployment completed its rollout, and the replacement Pod became ready:

```text
deployment "python-deployment-devops" successfully rolled out

NAME                                        READY   STATUS    RESTARTS
python-deployment-devops-6fdb46f474-t9pzt   1/1     Running   0
```

> **Why:** `kubectl rollout status` watches the Deployment until its updated replica becomes available. The resource type `deployment` and resource name are passed as separate arguments. `kubectl get pods` then confirms that the replacement Pod is `1/1 Running`, proving that Kubernetes successfully pulled and started `poroko/flask-demo-app`.

### 🌐 Step 5: Diagnose the Service Port Mapping

```bash
kubectl describe service python-service-devops
```

The Service initially reported:

```text
Type:        NodePort
Port:        8080/TCP
TargetPort:  8080/TCP
NodePort:    32345/TCP
Endpoints:   10.22.0.10:8080
```

The Deployment had already shown that the container used port `5000`, and a direct request to that Pod port returned the application response:

```bash
curl http://10.22.0.10:5000
```

```text
Hello World Pyvo 1!
```

> **Why:** `kubectl describe service` shows the Service type, selector, exposed port, destination port, NodePort, and selected endpoints. `nodePort: 32345` was already correct. However, `targetPort: 8080` made Kubernetes send traffic to port `8080` on the Pod, even though the Flask process listened on its default port `5000`. The successful direct `curl` request to the Pod IP on port `5000` confirmed the application's actual listening port and isolated the remaining problem to the Service mapping.

### 🔧 Step 6: Correct the Service Target Port

```bash
kubectl edit service python-service-devops
```

Keep the existing Service port and NodePort, but change `targetPort` from `8080` to `5000`:

```yaml
ports:
  - nodePort: 32345
    port: 8080
    protocol: TCP
    targetPort: 5000
```

Kubernetes confirmed the update:

```text
service/python-service-devops edited
```

> **Why:** Editing only `targetPort` applies the smallest necessary change. `nodePort: 32345` is the port clients use on a cluster node. `port: 8080` is the Service's stable internal port and does not need to equal the application port. `targetPort: 5000` is the destination port on the selected Pod, so it must match the Flask process. The resulting traffic path is `Node:32345 -> Service:8080 -> Pod:5000`.

### ✅ Step 7: Verify the Service and Application

```bash
kubectl describe service python-service-devops
curl http://localhost:32345/
```

The corrected Service pointed to the Flask port:

```text
Type:        NodePort
Port:        8080/TCP
TargetPort:  5000/TCP
NodePort:    32345/TCP
Endpoints:   10.22.0.10:5000
```

The NodePort request reached the application successfully:

```text
Hello World Pyvo 1!
```

> **Why:** The updated endpoint ending in `:5000` proves that the Service now forwards to the correct Pod port. `curl http://localhost:32345/` tests the complete external access path through the required NodePort rather than bypassing the Service. Receiving the Flask response confirms that the Deployment, Pod, Service selector, endpoint, `targetPort`, and `nodePort` all work together.

## Best Practices

- **Read Pod events before changing resources.** `kubectl describe pod` usually explains states such as `ImagePullBackOff`, `CrashLoopBackOff`, and `ContainerCreating` more precisely than the short Pod listing.
- **Verify image names and tags exactly.** Reordered words or misspelled repository names point Kubernetes to a different image reference and can prevent the container from starting.
- **Follow the traffic path one layer at a time.** Confirm the application works on the Pod port before troubleshooting the Service and its external NodePort.
- **Understand the three Service ports.** `nodePort` accepts traffic on the node, `port` exposes the Service inside the cluster, and `targetPort` identifies the destination port on the Pod.
- **Make the smallest justified correction.** The Service type, Service port, NodePort, and selectors were already correct, so only `targetPort` needed modification.
- **Prefer declarative manifests in maintained environments.** `kubectl edit` is convenient for a small lab repair, but production changes should normally be reviewed and stored in version-controlled manifests.
- **Add readiness probes for real applications.** A readiness probe prevents a Service from routing traffic to a container until the application is actually prepared to answer requests.

### 📚 Official Documentation

- [Kubernetes Images](https://kubernetes.io/docs/concepts/containers/images/)
- [Debug Running Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/)
- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [kubectl edit](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_edit/)
- [kubectl rollout status](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/kubectl_rollout_status/)
- [Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
