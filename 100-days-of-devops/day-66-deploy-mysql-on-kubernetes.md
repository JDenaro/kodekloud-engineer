# Day 66: Deploy MySQL on Kubernetes

A new MySQL server needs to be deployed on Kubernetes cluster. The Nautilus DevOps team was working on to gather the requirements. Recently they were able to finalize the requirements and shared them with the team members to start working on it. Below you can find the details:

## Specific Requirements:

1.) Create a PersistentVolume `mysql-pv`, its capacity should be `250Mi`, set other parameters as per your preference.
2.) Create a PersistentVolumeClaim to request this PersistentVolume storage. Name it as `mysql-pv-claim` and request a `250Mi` of storage. Set other parameters as per your preference.
3.) Create a deployment named `mysql-deployment`, use any mysql image as per your preference. Mount the PersistentVolume at mount path `/var/lib/mysql`.
4.) Create a `NodePort` type service named `mysql` and set nodePort to `30007`.
5.) Create a secret named `mysql-root-pass` with key `password`, another named `mysql-user-pass` with keys `username` and `password`, and one named `mysql-db-url` with key `database`. The challenge-provided values are intentionally omitted from this repository guide.
6.) Define these container environment variables using `secretKeyRef`:

   a.) `MYSQL_ROOT_PASSWORD` from `mysql-root-pass`, key `password`.

   b.) `MYSQL_DATABASE` from `mysql-db-url`, key `database`.

   c.) `MYSQL_USER` from `mysql-user-pass`, key `username`.

   d.) `MYSQL_PASSWORD` from `mysql-user-pass`, key `password`.

`Note:` The `kubectl` utility on the `jump-host` has been configured to work with the Kubernetes cluster.

## Solution

The PV supplies storage, the PVC requests it, and the Deployment mounts the bound claim into MySQL's data directory. Three Secrets supply the initialization values. The lab initially stalled because the `mysql-user-pass` Secret lacked its `password` key; recreating it with two separate `--from-literal` arguments allowed the Pod to start.

### 🔐 Step 1: Create the Secrets

Run the following commands on the jump host. Replace the placeholders with the values supplied by the current lab before running them:

```bash
kubectl create secret generic mysql-root-pass --from-literal=password='<ROOT_PASSWORD>'
```

```bash
kubectl create secret generic mysql-user-pass --from-literal=username='<MYSQL_USER>' --from-literal=password='<USER_PASSWORD>'
```

```bash
kubectl create secret generic mysql-db-url --from-literal=database='<DATABASE_NAME>'
```

> **Why:** `kubectl create secret generic` creates a Secret containing arbitrary key-value pairs. Each `--from-literal=key=value` is a separate shell argument; the space between the two options for `mysql-user-pass` is essential. Without it, the shell concatenates both pieces into one argument. In the failed attempt, `kubectl describe secret mysql-user-pass` showed only a 47-byte `username` key and no `password` key. The Pod event then reported `couldn't find key password in Secret default/mysql-user-pass`. With the space, the corrected Secret had both required keys.

### 📝 Step 2: Create the PV, PVC, Deployment, and Service Manifest

```bash
vi mysql-deployment.yaml
```

Add the following YAML:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 250Mi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/mysql-data
    type: DirectoryOrCreate

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  volumeName: mysql-pv
  resources:
    requests:
      storage: 250Mi

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql-container
          image: mysql:8.0
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-root-pass
                  key: password
            - name: MYSQL_DATABASE
              valueFrom:
                secretKeyRef:
                  name: mysql-db-url
                  key: database
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-user-pass
                  key: username
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-user-pass
                  key: password
          volumeMounts:
            - name: mysql-storage
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-storage
          persistentVolumeClaim:
            claimName: mysql-pv-claim

---
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  type: NodePort
  selector:
    app: mysql
  ports:
    - protocol: TCP
      port: 3306
      targetPort: 3306
      nodePort: 30007
```

> **Why:** `vi` creates the manifest, with `---` separating its four Kubernetes resources. The PV offers `250Mi` using a directory on the lab node; `DirectoryOrCreate` creates that directory if needed. `ReadWriteOnce` and `storageClassName: manual` match between the PV and PVC, while `volumeName: mysql-pv` requests this specific PV. `Retain` tells Kubernetes to keep the underlying volume data when the claim is released.
>
> The Deployment maintains one Pod and mounts the claim at MySQL's `/var/lib/mysql` data directory. Its `app: mysql` Pod label matches both the Deployment selector and the Service selector. Each `secretKeyRef` maps one environment variable to an exact Secret name and key. The `NodePort` Service accepts TCP traffic at port `30007` on the node and forwards it through Service port `3306` to container port `3306`.

### 🚀 Step 3: Apply the Manifest

```bash
kubectl apply -f mysql-deployment.yaml
```

The command created `mysql-pv`, `mysql-pv-claim`, `mysql-deployment`, and the `mysql` Service.

> **Why:** `kubectl apply` submits the desired objects to the Kubernetes API. The `-f` option specifies the manifest file. Keeping the four related resources together makes their names and references easier to inspect.

### 💾 Step 4: Check the Storage Binding

```bash
kubectl get pv mysql-pv
kubectl get pvc mysql-pv-claim
```

Both resources reached `Bound`: the PV reported a claim of `default/mysql-pv-claim`, and the PVC reported volume `mysql-pv` with capacity `250Mi`.

> **Why:** `kubectl get pv` inspects the offered PersistentVolume; `kubectl get pvc` inspects the claim consumed by the Pod. `Bound` on both confirms that the claim successfully matched the intended volume. The exact PVC name matters: `mysql-pv-claim` was created, while a lookup for `mysql-pvc` would return `NotFound`.

### ✅ Step 5: Verify MySQL and the NodePort

```bash
kubectl rollout status deploy mysql-deployment
kubectl get pods
kubectl get service mysql
kubectl describe service mysql
```

The rollout completed successfully, and the Pod reached `1/1 Running`. The Service showed `NodePort` with `3306:30007/TCP`; its endpoint was `10.22.0.10:3306` in this lab run.

> **Why:** `kubectl rollout status` waits for an available Deployment replica; `deploy` is a short resource name for Deployment. `kubectl get pods` confirms that the MySQL container is ready. `kubectl get service` confirms the required external port, and `kubectl describe service` confirms that the Service selector found the Pod and forwards to its MySQL port. The full Deployment name is `mysql-deployment`; a shortened name such as `mysql-deploy` refers to a different, nonexistent object.

### The Secret error encountered in this run

The first `mysql-user-pass` Secret was created with no space between its two `--from-literal` options. The shell passed them as a single argument, and the Secret ended up with just one 47-byte `username` key. The Pod could not start and reported `CreateContainerConfigError`. Its event gave the precise cause:

```text
Error: couldn't find key password in Secret default/mysql-user-pass
```

The Secret was recreated with the two options separated by a space, as shown in Step 1. `kubectl describe secret mysql-user-pass` then showed a 13-byte `username` key and a 10-byte `password` key, and the existing Pod started. In a new lab run, following Step 1 correctly from the start avoids this error.

## Best Practices

- **Check Secret keys before starting dependent workloads.** `kubectl describe secret` reveals key names and lengths without revealing values. It catches a missing `password` key before Pod creation.
- **Keep shell options separate.** Each `--from-literal` must be a distinct argument. A missing space can silently combine them into one value.
- **Read Pod events for startup errors.** `CreateContainerConfigError` means Kubernetes could not assemble the container configuration; the Pod events pinpointed the missing Secret key.
- **Use a storage backend suited to production.** This lab's `hostPath` works on its single node, but it ties data to that node. A production database usually uses a suitable PersistentVolumeClaim and storage provisioner.
- **Keep credentials out of Git.** The example uses placeholders; use the exact challenge values only in the temporary lab. For real workloads, prefer an approved secret-management workflow over command-line literals that may appear in shell history.

### 📚 Official Documentation

- [kubectl create secret generic](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/kubectl_create_secret_generic/)
- [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Persistent Volumes and Claims](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Services and NodePort](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Debug Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/)
