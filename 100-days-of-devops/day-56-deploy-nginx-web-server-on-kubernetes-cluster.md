# Day 56: Deploy Nginx Web Server on Kubernetes Cluster

Some of the Nautilus team developers are developing a static website and they want to deploy it on Kubernetes cluster. They want it to be highly available and scalable. Therefore, based on the requirements, the DevOps team has decided to create a deployment for it with multiple replicas. Below you can find more details about it:

## Specific Requirements:

1. Create a deployment using `nginx` image with `latest` tag only and remember to mention the tag i.e `nginx:latest`. Name it as `nginx-deployment`. The container should be named as `nginx-container`, also make sure replica counts are `3`.
2. Create a `NodePort` type service named `nginx-service`. The nodePort should be `30011`.

`Note:` The `kubectl` utility on the `jump-host` has been configured to work with the Kubernetes cluster.

## Solution

A Deployment maintains the required three Nginx replicas and replaces failed Pods automatically. A NodePort Service selects those Pods through a shared label and exposes them through port `30011` on the Kubernetes node. The Deployment and Service were kept in separate manifests so that each resource could be understood and applied independently.

### 📝 Step 1: Create the Deployment Manifest

```bash
vi nginx-deployment.yaml
```

Add the following configuration:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx-container
          image: nginx:latest
          ports:
            - containerPort: 80
```

> **Why:** `apiVersion: apps/v1` selects the stable Deployment API, and `metadata.name` assigns the required name. `replicas: 3` defines the desired number of Pods. The Deployment's `selector.matchLabels` must match the `app: nginx` label in the Pod template so the controller knows which Pods it owns. The template creates the required `nginx-container` from `nginx:latest`, and `containerPort: 80` records the port where Nginx accepts HTTP traffic.

### 🌐 Step 2: Create the NodePort Service Manifest

```bash
vi nginx-service.yaml
```

Add the following configuration:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30011
```

> **Why:** `kind: Service` creates a stable network endpoint for the replaceable Pods. `type: NodePort` exposes the Service through every node, and `nodePort: 30011` reserves the exact required external port. The Service listens on `port: 80` and forwards TCP traffic to `targetPort: 80` on Pods whose `app: nginx` label matches its selector.

### 🚀 Step 3: Create the Deployment and Service

```bash
kubectl apply -f nginx-deployment.yaml
kubectl apply -f nginx-service.yaml
```

Kubernetes confirmed both resources:

```text
deployment.apps/nginx-deployment created
service/nginx-service created
```

> **Why:** Each `kubectl apply` command sends one declarative manifest to the Kubernetes API. The `-f` option identifies the file to apply. Applying the files separately keeps the creation results clear and avoids combining unrelated resource operations into a complex command.

### 🔄 Step 4: Wait for the Deployment Rollout

```bash
kubectl rollout status deployment/nginx-deployment
```

The rollout completed successfully:

```text
deployment "nginx-deployment" successfully rolled out
```

> **Why:** `kubectl rollout status` watches the specified Deployment until its desired replicas have been updated and become available. The `deployment/nginx-deployment` notation identifies the resource type and name.

### 🔍 Step 5: Verify the Deployment and Pods

```bash
kubectl get deployment nginx-deployment
kubectl get pods
```

The Deployment reported all three replicas as ready and available:

```text
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           93s
```

The controller created three running Pods:

```text
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-6586c5b5fb-dhtzl   1/1     Running   0          103s
nginx-deployment-6586c5b5fb-nrj7p   1/1     Running   0          103s
nginx-deployment-6586c5b5fb-sgplj   1/1     Running   0          103s
```

> **Why:** `kubectl get deployment` confirms the desired, current, and available replica counts. `READY 3/3`, `UP-TO-DATE 3`, and `AVAILABLE 3` prove the Deployment reached its intended state. `kubectl get pods` verifies that each individual replica is ready and running.

### 🔌 Step 6: Verify the NodePort Service and Endpoints

```bash
kubectl get service nginx-service
kubectl describe service nginx-service
```

The Service exposed port `80` through NodePort `30011`:

```text
NAME            TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
nginx-service   NodePort   10.43.45.28   <none>        80:30011/TCP   111s
```

Its details showed one endpoint for every Nginx replica:

```text
Selector:                 app=nginx
Type:                     NodePort
Port:                     80/TCP
TargetPort:               80/TCP
NodePort:                 30011/TCP
Endpoints:                10.22.0.9:80,10.22.0.10:80,10.22.0.11:80
```

> **Why:** `kubectl get service` confirms the Service type and compact `80:30011/TCP` port mapping. `kubectl describe service` shows the selector, target port, NodePort, and resolved endpoints. The three endpoint addresses prove that the Service found all three Pods through the matching `app=nginx` label.

### ✅ Step 7: Test the Website Through the NodePort

```bash
kubectl get nodes -o wide
```

The node information provided its internal address:

```text
NAME        STATUS   ROLES           VERSION        INTERNAL-IP
jump-host   Ready    control-plane   v1.34.1+k3s1   10.244.195.51
```

Use that node address and the required NodePort:

```bash
curl http://10.244.195.51:30011
```

The response contained the default Nginx page:

```html
<h1>Welcome to nginx!</h1>
```

> **Why:** `kubectl get nodes -o wide` adds node networking details, including `INTERNAL-IP`, to the normal node listing. A NodePort Service is reachable through a node address and its allocated port. `curl` sends an HTTP request to port `30011`; receiving the Nginx page proves that the node accepted the connection, the Service forwarded it to a ready endpoint, and the selected container served the response.

## Best Practices

- **Keep selectors and labels aligned.** The Deployment selector, Pod template label, and Service selector must identify the same workload. A mismatch leaves Pods unmanaged or a Service without endpoints.
- **Verify endpoints, not only the Service object.** A Service can exist with the correct port and still be unable to route traffic if its selector finds no ready Pods.
- **Use multiple replicas for availability.** Three replicas let the Service continue routing traffic when an individual Pod is restarted or replaced.
- **Use higher-level ingress for production HTTP traffic.** NodePort is appropriate for this lab, but production environments commonly use an Ingress, Gateway, or LoadBalancer with TLS and controlled external access.
- **Pin immutable image versions.** The challenge requires `nginx:latest`; production workloads should use a fixed version or digest to make rollouts predictable.

### 📚 Official Documentation

- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [kubectl apply](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_apply/)
- [kubectl rollout status](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/kubectl_rollout_status/)
