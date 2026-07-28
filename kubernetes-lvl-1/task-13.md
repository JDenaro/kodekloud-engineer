# Task 13: Expose Application Using NodePort Service in Kubernetes

The Nautilus DevOps team has already deployed a ReplicaSet to host an application that requires a highly available infrastructure. Your task is to expose the application running in the existing ReplicaSet by creating a Kubernetes NodePort Service.

Follow the specifications below to create the Service and ensure the application pods are accessible:

A ReplicaSet named nginx-replicaset is already running in the cluster.

The pods managed by the ReplicaSet use the following labels:
Assign labels app as nginx_app, and type as front-end.

Create a NodePort Service named nginx-service to expose the application.

Set the NodePort to 30080.

Expose port 80 of the application.

**Note:** Do not delete or modify the configuration of the deployed ReplicaSet application.

## Task Requirements

1. Use the existing ReplicaSet named `nginx-replicaset`.
2. Select its Pods with the labels `app: nginx_app` and `type: front-end`.
3. Create a NodePort Service named `nginx-service`.
4. Set the NodePort to `30080`.
5. Expose application port `80`.
6. Do not delete or modify the ReplicaSet configuration.

## Solution

The application was already running in Pods managed by `nginx-replicaset`, so only a Service was needed. The Service manifest uses the Pods' labels as its selector and exposes port `80` through NodePort `30080`. The ReplicaSet was not changed.

### 📝 Step 1: Create the NodePort Service manifest

```bash
vi nginx-service.yaml
```

Inside the editor, press `i` and enter:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx_app
    type: front-end
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

Save and exit with `Esc`, `:wq`, and `Enter`.

> **Why:** `apiVersion: v1` selects the core Kubernetes API, and `kind: Service` identifies the resource being created. `metadata.name` gives the Service its required name. `type: NodePort` exposes the Service on a port of every cluster node. The `selector` matches Pods with both required labels, so traffic is sent to the Pods managed by the existing ReplicaSet. `port: 80` is the port exposed by the Service, `targetPort: 80` is the port used by the application container, and `nodePort: 30080` fixes the node-facing port requested by the challenge.

### 🚀 Step 2: Create the Service

```bash
kubectl apply -f nginx-service.yaml
```

The Service was created successfully:

```text
thor@jump-host ~$ vi nginx-service.yaml
thor@jump-host ~$ kubectl apply -f nginx-service.yaml
service/nginx-service created
thor@jump-host ~$
```

> **Why:** `kubectl apply` sends the resource definition to the Kubernetes API. The `-f` option tells `kubectl` to read the Service configuration from `nginx-service.yaml`. Applying the manifest creates the Service without deleting or changing the existing ReplicaSet.

### ✅ Step 3: Verify the Service

```bash
kubectl get service
```

The output confirmed that `nginx-service` is a NodePort Service exposing `80:30080/TCP`:

```text
thor@jump-host ~$ kubectl get service
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes      ClusterIP   10.43.0.1      <none>        443/TCP        57m
nginx-service   NodePort    10.43.125.72   <none>        80:30080/TCP   17s
thor@jump-host ~$
```

> **Why:** `get service` lists Services in the current namespace. `TYPE=NodePort` confirms the exposure method, and `80:30080/TCP` means the Service accepts traffic on port `80` and exposes it through node port `30080` using TCP. The assigned `CLUSTER-IP` is the internal address of the Service. The existing ReplicaSet remained untouched.

## Best Practices

- **Use selectors that match the application Pods.** A Service only routes traffic to Pods whose labels match every selector entry.
- **Keep workload and networking changes separate.** The ReplicaSet already provided the application Pods, so the task only required creating a Service.
- **Use an explicit NodePort when the port is part of the requirement.** Kubernetes can assign a random NodePort, but `nodePort: 30080` makes the exposed port predictable.
- **Understand the three port fields.** `port` is the Service port, `targetPort` is the application port, and `nodePort` is the port exposed on each node.
- **Avoid NodePort for unrestricted production exposure.** In production, an Ingress or cloud load balancer may provide better routing, security, and lifecycle management.
- **Check that the requested NodePort is available.** A node port cannot be reused by another Service in the same cluster.

### 📚 Official Documentation

- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [NodePort Services](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport)
- [Kubernetes ReplicaSets](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
- [`kubectl apply` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_apply/)
- [`kubectl get` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/)
