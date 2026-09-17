# Day 59: Troubleshoot Deployment issues in Kubernetes

Last week, the Nautilus DevOps team deployed a redis app on Kubernetes cluster, which was working fine so far. This morning one of the team members was making some changes in this existing setup, but he made some mistakes and the app went down. We need to fix this as soon as possible. Please take a look.

## Specific Requirements:

1. The deployment name is `redis-deployment`. The pods are not in running state right now, so please look into the issue and fix the same.

`Note:` The `kubectl` utility on the `jump-host` has been configured to work with the Kubernetes cluster.

## Solution

The Redis Pod was blocked in `ContainerCreating` because its required ConfigMap volume referenced the misspelled name `redis-cofig`. The Deployment also used the invalid image tag `redis:alpin`. Correcting both values allowed Kubernetes to create a new ReplicaSet and replace the broken Pod with a running one.

### 🔍 Step 1: Inspect the Deployment

```bash
kubectl get deployment redis-deployment
kubectl describe deployment redis-deployment
```

The Deployment had no available replicas and revealed two suspicious values:

```text
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
redis-deployment   0/1     1            0           33m

Image:      redis:alpin
Name:       redis-cofig
```

> **Why:** `kubectl get deployment` provides a concise view of the named Deployment. `READY 0/1` and `AVAILABLE 0` confirm that its desired Pod is not serving the application. `kubectl describe deployment` displays the Pod template, including its image, volume configuration, conditions, and recent events. This inspection exposed the incomplete Redis image tag and the suspicious ConfigMap name without changing the workload.

### 🧭 Step 2: Identify the Failing Pod

```bash
kubectl get pods
```

The Redis Pod had been unable to finish creating:

```text
NAME                                READY   STATUS              RESTARTS   AGE
redis-deployment-795ffcb56c-tm97b   0/1     ContainerCreating   0          34m
```

> **Why:** `kubectl get pods` lists the Pods and their current states. A Pod remaining in `ContainerCreating` for an extended time suggests that Kubernetes cannot prepare a prerequisite such as an image, volume, Secret, or ConfigMap.

### 🧩 Step 3: Find the Blocking Pod Event

```bash
kubectl describe pod redis-deployment-795ffcb56c-tm97b
```

The Pod events identified the immediate failure:

```text
Warning  FailedMount  MountVolume.SetUp failed for volume "config" : configmap "redis-cofig" not found
```

> **Why:** `kubectl describe pod` shows detailed container state, mounted volumes, conditions, and chronological events for the named Pod. The `FailedMount` event proves that Kubernetes could not mount the required `config` volume because no ConfigMap named `redis-cofig` existed. This prevented container creation before Kubernetes could encounter the invalid image tag.

### 📋 Step 4: Find the Correct ConfigMap Name

```bash
kubectl get configmaps
```

The cluster contained the intended ConfigMap:

```text
NAME               DATA   AGE
kube-root-ca.crt   1      54m
redis-config       2      37m
```

> **Why:** `kubectl get configmaps` lists the ConfigMaps in the current namespace. The output confirms that `redis-config` exists and that the Deployment referenced a misspelled name. Kubernetes resource references must match their target names exactly.

### 🛠️ Step 5: Correct the Deployment

```bash
kubectl edit deployment redis-deployment
```

Change the container image from:

```yaml
image: redis:alpin
```

to:

```yaml
image: redis:alpine
```

Change the ConfigMap reference from:

```yaml
configMap:
  name: redis-cofig
```

to:

```yaml
configMap:
  name: redis-config
```

Save and close the editor. Kubernetes confirms the change:

```text
deployment.apps/redis-deployment edited
```

> **Why:** `kubectl edit deployment` opens the live Deployment manifest in an editor and applies the saved changes through the Kubernetes API. Correcting `redis-cofig` allows the volume to mount from the existing ConfigMap. Correcting `redis:alpin` to `redis:alpine` gives Kubernetes a valid Redis image tag. Updating the Deployment's Pod template automatically creates a new ReplicaSet, so the broken Pod does not need to be deleted manually.

### ⏳ Step 6: Wait for the Corrected Rollout

```bash
kubectl rollout status deployment/redis-deployment
```

The update completed successfully:

```text
deployment "redis-deployment" successfully rolled out
```

> **Why:** `kubectl rollout status` watches the latest rollout of the named Deployment until it completes. Success means the new ReplicaSet created the desired replica and made it available.

### ✅ Step 7: Verify the Deployment and Pod

```bash
kubectl get deployment redis-deployment
kubectl get pods
kubectl describe deployment redis-deployment
```

The Deployment and replacement Pod became healthy:

```text
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
redis-deployment   1/1     1            1           39m

NAME                                READY   STATUS    RESTARTS   AGE
redis-deployment-5476b4ddd6-fvmjn   1/1     Running   0          34s
```

The final Deployment description also showed the corrected configuration:

```text
Image:      redis:alpine
Name:       redis-config
Available  True    MinimumReplicasAvailable
Progressing True   NewReplicaSetAvailable
```

> **Why:** The first command confirms that the Deployment has one ready, updated, and available replica. The second confirms that the replacement Pod is running without restarts. The final `kubectl describe deployment` verifies the corrected image and ConfigMap reference and shows that Kubernetes scaled the old ReplicaSet down after the new ReplicaSet became available.

## Best Practices

- **Read Pod events before changing resources.** Events such as `FailedMount` usually identify the immediate reason a Pod cannot start.
- **Verify referenced resources by name.** ConfigMap and Secret references are exact and case-sensitive, so confirm that the resource exists before editing the workload.
- **Inspect the complete Pod template.** Fixing only the first visible error can expose another one later; review the image, volumes, commands, and other dependencies together.
- **Edit the controller instead of its Pods.** Deployment-managed Pods are replaceable. Correct the Deployment so every replacement Pod receives the valid configuration.
- **Verify both rollout and runtime state.** A successful rollout plus a `Running` and ready Pod confirms that Kubernetes accepted the update and the workload recovered.

### 📚 Official Documentation

- [Debug Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/)
- [Configure a Pod to Use a ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)
- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [kubectl describe](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_describe/)
- [kubectl edit](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_edit/)
- [kubectl rollout status](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/kubectl_rollout_status/)
