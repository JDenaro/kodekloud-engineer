# Task 10: Set Up Time Check Pod in Kubernetes

The Nautilus DevOps team needs a time check pod created in a specific Kubernetes namespace for logging purposes. Initially, it's for testing, but it may be integrated into an existing cluster later. Here's what's required:

## Task Requirements

1. Create a pod called `time-check` in the `datacenter` namespace. The pod should contain a container named `time-check`, utilizing the busybox image with the latest tag (specify as `busybox:latest`).
2. Create a config map named `time-config` with the data `TIME_FREQ=8` in the same namespace.
3. Configure the time-check container to execute the command: `while true; do date; sleep $TIME_FREQ;done`. Ensure the result is written `/opt/data/time/time-check.log`. Also, add an environmental variable `TIME_FREQ` in the container, fetching its value from the config map `TIME_FREQ` key.
4. Create a volume `log-volume` and mount it at `/opt/data/time` within the container.

Note: The `kubectl` utility on the jump-host has been configured to work with the Kubernetes cluster.

## Solution

This task creates three related pieces: the `datacenter` namespace, the `time-config` ConfigMap, and the `time-check` Pod. The ConfigMap stores the interval as configuration, and the Pod reads that value into the `TIME_FREQ` environment variable.

The Pod uses an `emptyDir` volume named `log-volume`. This is a Pod volume, not a separate top-level Kubernetes resource. Kubernetes creates it when the Pod starts and mounts it at `/opt/data/time`, where the loop writes the time-check log.

### 🔎 Step 1: Review the available namespaces

```bash
kubectl get ns
```

The initial namespace list did not contain `datacenter`:

```text
thor@jump-host ~$ kubectl get ns
NAME              STATUS   AGE
default           Active   72m
kube-node-lease   Active   72m
kube-public       Active   72m
kube-system       Active   72m
thor@jump-host ~$
```

> **Why:** `get` retrieves Kubernetes resources, and `ns` is the short form of the `namespace` resource type. Listing the namespaces showed that the required `datacenter` namespace was not present, so it had to be created before the namespaced ConfigMap and Pod.

### 🏗️ Step 2: Create the `datacenter` namespace

```bash
kubectl create ns datacenter
```

The namespace was created successfully:

```text
thor@jump-host ~$ kubectl create ns datacenter
namespace/datacenter created
thor@jump-host ~$
```

> **Why:** `create` requests a new Kubernetes resource, `ns` identifies the Namespace resource type, and `datacenter` is the required namespace name. Kubernetes resources such as ConfigMaps and Pods are namespaced, so the namespace must exist before those resources can be created in it.

### ⚙️ Step 3: Create the ConfigMap manifest

```bash
vi time-config.yaml
```

Inside the editor, press `i`, enter the following YAML, press `Esc`, type `:wq`, and press `Enter`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: time-config
  namespace: datacenter
data:
  TIME_FREQ: "8"
```

Apply the manifest:

```bash
kubectl apply -f time-config.yaml
```

The ConfigMap was created successfully:

```text
thor@jump-host ~$ vi time-config.yaml
thor@jump-host ~$ kubectl apply -f time-config.yaml
configmap/time-config created
thor@jump-host ~$
```

> **Why:** `vi` is a terminal text editor; `i` enters insert mode, `Esc` leaves it, and `:wq` saves and exits. `apiVersion: v1` selects the core API, `kind: ConfigMap` identifies the configuration resource, and `metadata.namespace` places it in `datacenter`. The `data` section stores the key-value pair `TIME_FREQ: "8"`. `kubectl apply` reads the resource from `time-config.yaml` using the `-f` file option and sends it to the cluster.

### 📝 Step 4: Create the time-check Pod manifest

```bash
vi time-check-pod.yaml
```

Inside the editor, press `i`, enter the following YAML, press `Esc`, type `:wq`, and press `Enter`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: time-check
  namespace: datacenter
spec:
  containers:
    - name: time-check
      image: busybox:latest
      env:
        - name: TIME_FREQ
          valueFrom:
            configMapKeyRef:
              name: time-config
              key: TIME_FREQ
      command:
        - /bin/sh
        - -c
        - 'while true; do date >> /opt/data/time/time-check.log; sleep $TIME_FREQ; done'
      volumeMounts:
        - name: log-volume
          mountPath: /opt/data/time
  volumes:
    - name: log-volume
      emptyDir: {}
```

Apply the Pod manifest:

```bash
kubectl apply -f time-check-pod.yaml
```

The Pod was created successfully:

```text
thor@jump-host ~$ vi time-check-pod.yaml
thor@jump-host ~$ kubectl apply -f time-check-pod.yaml
pod/time-check created
thor@jump-host ~$
```

> **Why:** `metadata.namespace` places the Pod in `datacenter`. The `env` block creates the container variable `TIME_FREQ`; `valueFrom.configMapKeyRef.name` selects `time-config`, and `key` selects its `TIME_FREQ` entry. The `command` starts BusyBox's shell, runs the infinite loop, appends each `date` result to `/opt/data/time/time-check.log`, and sleeps for the number of seconds stored in `TIME_FREQ`. `volumeMounts` connects the container path to the named `log-volume`, while `volumes` defines that volume with `emptyDir: {}`.

### ✅ Step 5: Verify the Pod in the correct namespace

The first command queried the default namespace and correctly found no Pods there:

```bash
kubectl get pods
```

```text
thor@jump-host ~$ kubectl get pods
No resources found in default namespace.
thor@jump-host ~$
```

The Pod was then found in its actual namespace:

```bash
kubectl get pods --namespace datacenter
```

```text
thor@jump-host ~$ kubectl get pods --namespace datacenter
NAME         READY   STATUS    RESTARTS   AGE
time-check   1/1     Running   0          12s
thor@jump-host ~$
```

> **Why:** `get pods` lists Pods in the current namespace, which defaults to `default` when no namespace is specified. The `--namespace datacenter` option changes the scope of the request and confirms that `time-check` exists in the required namespace. `1/1` means its one container is ready, and `Running` confirms that the loop is active.

### Volume versus PersistentVolume

`log-volume` is an `emptyDir` volume defined inside the Pod. Kubernetes creates it automatically when the Pod starts, and it remains available if the container restarts. Its contents are deleted when the Pod itself is deleted or replaced.

A `PersistentVolume` is different: it represents storage that exists independently from an individual Pod. A workload normally requests that storage through a `PersistentVolumeClaim`, and the claim is then mounted into the Pod. Use a persistent volume when the log must survive Pod replacement; use `emptyDir` for temporary data tied to the Pod's lifetime, as required by this testing task.

## Best Practices

- **Create the namespace first.** Namespaced resources cannot be placed in `datacenter` until the namespace exists.
- **Use ConfigMaps for non-sensitive configuration.** `TIME_FREQ` is a configuration value, so storing it in `time-config` avoids hard-coding the interval into the Pod definition.
- **Reference ConfigMap keys explicitly.** `configMapKeyRef` makes it clear which ConfigMap and key provide the `TIME_FREQ` environment variable.
- **Use the correct namespace on every resource.** Both `time-config` and `time-check` explicitly specify `datacenter`, avoiding accidental creation in `default`.
- **Understand `emptyDir` lifetime.** It is useful for temporary Pod-local data, but it is not a substitute for persistent storage.
- **Use a PersistentVolumeClaim for durable logs.** If the log must survive Pod deletion or rescheduling, provision persistent storage instead of relying on `emptyDir`.
- **Use an appropriate controller for production logging.** A standalone Pod is suitable for this lab, but production workloads are usually managed by a Deployment, StatefulSet, or dedicated logging agent.

### 📚 Official Documentation

- [Kubernetes Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- [Kubernetes ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Configure a Pod to use a ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)
- [Configure a Pod to use a volume](https://kubernetes.io/docs/tasks/configure-pod-container/configure-volume-storage/)
- [Kubernetes Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [`kubectl apply` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_apply/)
