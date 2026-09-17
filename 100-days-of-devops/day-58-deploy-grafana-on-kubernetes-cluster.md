# Day 58: Deploy Grafana on Kubernetes Cluster

The Nautilus DevOps teams is planning to set up a Grafana tool to collect and analyze analytics from some applications. They are planning to deploy it on Kubernetes cluster. Below you can find more details.

## Specific Requirements:

1. Create a deployment named `grafana-deployment-nautilus` using any grafana image for Grafana app. Set other parameters as per your choice.
2. Create `NodePort` type service with nodePort `32000` to expose the app.

`You do not need to make any configuration changes inside the Grafana app once deployed; just make sure you can access the Grafana login page.`

`Note:` The `kubectl` utility on the `jump-host` has been configured to work with the Kubernetes cluster.

## Solution

Grafana listens for HTTP traffic on container port `3000` by default. A Kubernetes NodePort Service makes that internal port reachable through port `32000` on the cluster node. The Deployment, Pod template, and Service must use matching labels so that the Service can discover the Grafana Pod.

### 📝 Step 1: Create the Deployment and Service Manifest

```bash
vi grafana.yaml
```

Add the following configuration:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana-deployment-nautilus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
        - name: grafana-container
          image: grafana/grafana:latest
          ports:
            - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: grafana-service
spec:
  type: NodePort
  selector:
    app: grafana
  ports:
    - protocol: TCP
      port: 3000
      targetPort: 3000
      nodePort: 32000
```

> **Why:** The Deployment maintains one Grafana Pod created from the official `grafana/grafana:latest` image. Its selector and Pod label both use `app: grafana`, establishing which Pod belongs to the Deployment. The Service uses the same selector to route TCP traffic to that Pod. `port: 3000` is the Service port, `targetPort: 3000` is Grafana's port inside the container, and `nodePort: 32000` publishes the application on the required port of every cluster node. The `---` separator allows both Kubernetes resources to remain in one readable manifest.

### 🚀 Step 2: Create the Kubernetes Resources

```bash
kubectl apply -f grafana.yaml
```

The command creates both resources:

```text
deployment.apps/grafana-deployment-nautilus created
service/grafana-service created
```

> **Why:** `kubectl apply` submits the declarative configuration to the Kubernetes API. The `-f` option identifies `grafana.yaml` as the manifest containing the Deployment and Service definitions.

### ⏳ Step 3: Wait for Grafana to Become Available

```bash
kubectl rollout status deployment/grafana-deployment-nautilus
```

Wait for the successful rollout message:

```text
deployment "grafana-deployment-nautilus" successfully rolled out
```

> **Why:** `kubectl rollout status` watches the named Deployment until its desired Pod is available. Waiting for completion avoids testing the NodePort while Grafana is still starting.

### 🔍 Step 4: Confirm the Service Endpoint

```bash
kubectl describe service grafana-service
```

The Service showed the required NodePort and a reachable backend:

```text
Selector:                 app=grafana
Type:                     NodePort
Port:                     <unset>  3000/TCP
TargetPort:               3000/TCP
NodePort:                 <unset>  32000/TCP
Endpoints:                10.22.0.9:3000
```

> **Why:** `kubectl describe service` provides a human-readable view of the Service configuration and its discovered endpoints. `NodePort: 32000/TCP` confirms the required external port. The non-empty `Endpoints` value proves that the `app: grafana` selector matched the Grafana Pod and that traffic will be forwarded to port `3000` in that Pod.

### ✅ Step 5: Verify the Grafana Login Page

First, test Grafana directly through the endpoint reported by the Service:

```bash
curl -I http://10.22.0.9:3000/login
```

Then verify the required NodePort:

```bash
curl -I http://localhost:32000/login
```

Both requests returned a successful response:

```text
HTTP/1.1 200 OK
Cache-Control: no-store
Content-Type: text/html; charset=UTF-8
```

> **Why:** `curl` sends an HTTP request to the Grafana login endpoint, while `-I` requests only the response headers so the test remains concise. The direct Pod endpoint test confirms that Grafana itself is listening on port `3000`. The second request follows the required path through node port `32000`, the Service, and finally the Pod. `HTTP/1.1 200 OK` and the HTML content type prove that the Grafana login page is accessible.

## Best Practices

- **Verify Service endpoints before testing external access.** A non-empty endpoint confirms that the Service selector matches a ready Pod and isolates routing problems from application startup problems.
- **Wait for the rollout to complete.** A newly created Pod can be running while its application is still initializing, so wait for the Deployment before testing the application.
- **Keep labels consistent.** Deployment selectors, Pod labels, and Service selectors must agree or the Service will have no backend endpoints.
- **Pin image versions in production.** The `latest` tag is acceptable for this lab, but a fixed Grafana version makes production deployments repeatable and prevents unexpected upgrades.
- **Use health probes for production workloads.** Readiness and liveness probes help Kubernetes distinguish between a running container and a Grafana instance that is actually ready to receive traffic.

### 📚 Official Documentation

- [Run Grafana Docker Image](https://grafana.com/docs/grafana/latest/setup-grafana/installation/docker/)
- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes Service and NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport)
- [kubectl describe](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_describe/)
