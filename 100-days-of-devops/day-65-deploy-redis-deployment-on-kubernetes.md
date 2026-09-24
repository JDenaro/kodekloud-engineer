# Day 65: Deploy Redis Deployment on Kubernetes

The Nautilus application development team observed some performance issues with one of the application that is deployed in Kubernetes cluster. After looking into number of factors, the team has suggested to use some in-memory caching utility for DB service. After number of discussions, they have decided to use Redis. Initially they would like to deploy Redis on kubernetes cluster for testing and later they will move it to production. Please find below more details about the task:

## Specific Requirements:

Create a redis deployment with following parameters:

1. Create a `config map` called `my-redis-config` having `maxmemory 2mb` in `redis-config`.
2. Name of the `deployment` should be `redis-deployment`, it should use `redis:alpine` image and container name should be `redis-container`. Also make sure it has only `1` replica.
3. The container should request for `1` CPU.
4. Mount `2` volumes:

   a. An Empty directory volume called `data` at path `/redis-master-data`.

   b. A configmap volume called `redis-config` at path `/redis-master`.

   c. The container should expose the port `6379`.
5. Finally, `redis-deployment` should be up and running.

`Note:` The `kubectl` utility on the `jump-host` has been configured to work with the Kubernetes cluster.

## Solution

The ConfigMap stores Redis configuration separately from the container image. Mounting it as a volume turns its `redis-config` key into `/redis-master/redis-config`. The Deployment then starts `redis-server` with that file, ensuring that Redis actually applies `maxmemory 2mb` rather than merely having the configuration available in the container.

### 📝 Step 1: Create the Redis Manifest

```bash
vi redis-deployment.yaml
```

Add the following configuration:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-redis-config
data:
  redis-config: |
    maxmemory 2mb

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-deployment
  labels:
    app: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis-container
          image: redis:alpine
          command:
            - redis-server
            - /redis-master/redis-config
          resources:
            requests:
              cpu: "1"
          ports:
            - containerPort: 6379
          volumeMounts:
            - name: data
              mountPath: /redis-master-data
            - name: redis-config
              mountPath: /redis-master
      volumes:
        - name: data
          emptyDir: {}
        - name: redis-config
          configMap:
            name: my-redis-config
```

> **Why:** `vi` creates a readable multi-document manifest. The first document defines `my-redis-config`; its `data` map contains a key named `redis-config` whose value is the Redis directive `maxmemory 2mb`. The `---` separator begins the Deployment document.
>
> The Deployment maintains exactly one Pod from the `redis:alpine` image. Its selector and Pod-template label both use `app: redis`, allowing the Deployment controller to identify the Pod it manages. `command` overrides the image's default command so Redis reads `/redis-master/redis-config` at startup. Without this command, mounting the ConfigMap would make the file available but would not automatically tell Redis to load it.
>
> The CPU request of `"1"` asks the Kubernetes scheduler to reserve one complete CPU core for the container. A request affects Pod placement and guaranteed scheduling capacity; because the challenge does not define a CPU limit, it does not set a maximum CPU usage. `containerPort: 6379` records the port exposed by Redis inside the Pod.
>
> The `data` volume uses `emptyDir`, which starts empty and exists for the lifetime of the Pod. It is mounted at `/redis-master-data`. The `redis-config` volume is populated from `my-redis-config` and mounted at `/redis-master`; Kubernetes therefore presents the ConfigMap key as `/redis-master/redis-config`. The matching names under `volumeMounts` and `volumes` connect each mount to its volume source.

### 🚀 Step 2: Create the ConfigMap and Deployment

```bash
kubectl apply -f redis-deployment.yaml
```

Kubernetes created both resources:

```text
configmap/my-redis-config created
deployment.apps/redis-deployment created
```

> **Why:** `kubectl apply` sends the declarative objects in the manifest to the Kubernetes API. The `-f` option identifies `redis-deployment.yaml` as the input file. Processing both documents together ensures the referenced ConfigMap exists while Kubernetes prepares the Redis Pod.

### ⏳ Step 3: Wait for the Deployment

```bash
kubectl rollout status deployment redis-deployment
```

The rollout completed successfully:

```text
deployment "redis-deployment" successfully rolled out
```

> **Why:** `kubectl rollout status` watches the Deployment until the updated replica becomes available. `deployment` is the resource type and `redis-deployment` is its name. A successful rollout means the Deployment controller created the requested Pod and Kubernetes reported it as available.

### 🔍 Step 4: Verify the Running Pod

```bash
kubectl get pods
```

The Redis Pod reached the required state:

```text
NAME                               READY   STATUS    RESTARTS   AGE
redis-deployment-c994d8d8d-xgwv2   1/1     Running   0          21s
```

> **Why:** `kubectl get pods` provides a direct status summary. `1/1` under `READY` means the Pod's only container is ready, while `Running` confirms that Kubernetes successfully pulled `redis:alpine` and started Redis.

### ⚙️ Step 5: Verify the ConfigMap

```bash
kubectl describe configmap my-redis-config
```

The description showed the required key and value:

```text
Data
====
redis-config:
----
maxmemory 2mb
```

> **Why:** `kubectl describe configmap` displays the resource metadata and stored entries in a human-readable format. The `redis-config` key and `maxmemory 2mb` value confirm that the configuration requested by the challenge was stored correctly.

### 📦 Step 6: Verify the Deployment Configuration

```bash
kubectl get deploy redis-deployment
kubectl describe deploy redis-deployment
```

`deploy` is the short resource name for `deployment`. The status showed one ready, current, and available replica:

```text
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
redis-deployment   1/1     1            1           57s
```

The live Pod template confirmed all required settings:

```text
Image:      redis:alpine
Port:       6379/TCP
Command:
  redis-server
  /redis-master/redis-config
Requests:
  cpu:        1
Mounts:
  /redis-master from redis-config (rw)
  /redis-master-data from data (rw)
Volumes:
  data:
    Type:       EmptyDir
  redis-config:
    Type:       ConfigMap
    Name:       my-redis-config
```

> **Why:** `kubectl get deploy` confirms replica availability at a glance. `kubectl describe deploy` shows the live container image, command, exposed port, resource request, mounts, volume types, conditions, ReplicaSet, and events. Together these fields prove that Kubernetes accepted the complete Deployment specification rather than only confirming that a Pod exists.

### ✅ Step 7: Verify That Redis Loaded the Configuration

```bash
kubectl exec deployment/redis-deployment -- redis-cli CONFIG GET maxmemory
```

Redis returned:

```text
maxmemory
2097152
```

> **Why:** `kubectl exec` runs a command inside a Pod managed by `deployment/redis-deployment`. The `--` separator marks the end of `kubectl` options and the start of the in-container command. `redis-cli CONFIG GET maxmemory` asks the running Redis server for its effective `maxmemory` setting. The returned value `2097152` bytes equals `2 * 1024 * 1024`, proving that Redis read and applied the `maxmemory 2mb` directive from the mounted ConfigMap.

## Best Practices

- **Separate configuration from images.** ConfigMaps allow non-confidential settings to change without rebuilding the container image.
- **Make the application consume the configuration.** Mounting a ConfigMap only creates files or values; the application must still be started or configured to read them.
- **Use Secrets for confidential values.** ConfigMaps are intended for non-sensitive data and do not provide secrecy for passwords, tokens, or keys.
- **Distinguish requests from limits.** A CPU request guides scheduling and reserves capacity. A CPU limit constrains maximum usage; this challenge requires only the request.
- **Treat emptyDir as ephemeral.** Data in `emptyDir` survives individual container restarts but is removed when the Pod is removed from its node. Use persistent storage for production data that must survive Pod replacement.
- **Pin production image versions.** `redis:alpine` satisfies this challenge, but a specific tested Redis version or digest provides more predictable production deployments.
- **Add health probes in production.** Readiness and liveness probes allow Kubernetes to detect when Redis can serve traffic and when it needs recovery.

### 📚 Official Documentation

- [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Configure a Pod to Use a ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)
- [Volumes and emptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir)
- [Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Redis CONFIG GET](https://redis.io/docs/latest/commands/config-get/)
