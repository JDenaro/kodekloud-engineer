# Day 57: Print Environment Variables

The Nautilus DevOps team is working on to setup some pre-requisites for an application that will send the greetings to different users. There is a sample deployment, that needs to be tested. Below is a scenario which needs to be configured on Kubernetes cluster. Please find below more details about it.

## Specific Requirements:

1. Create a `pod` named `print-envars-greeting`.
2. Configure spec as, the container name should be `print-env-container` and use `bash` image.
3. Create three environment variables:

   a. `GREETING` and its value should be `Welcome to`

   b. `COMPANY` and its value should be `Stratos`

   c. `GROUP` and its value should be `Industries`
4. Use command `["/bin/sh", "-c", 'echo "$(GREETING) $(COMPANY) $(GROUP)"']` (please use this exact command), also set its `restartPolicy` policy to `Never` to avoid crash loop back.
5. You can check the output using `kubectl logs -f print-envars-greeting` command.

`Note:` The `kubectl` utility on the `jump-host` has been configured to work with the Kubernetes cluster.

## Solution

Kubernetes can inject environment variables directly into a container through the `env` field. References written as `$(VARIABLE)` inside a container's `command` or `args` are expanded from those configured values before the process starts. The command in this challenge prints one greeting and exits, so `restartPolicy: Never` allows the Pod to remain successfully completed instead of restarting the container.

### 📝 Step 1: Create the Pod Manifest

```bash
vi print-envars-greeting.yaml
```

Add the following configuration:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: print-envars-greeting
spec:
  containers:
    - name: print-env-container
      image: bash
      env:
        - name: GREETING
          value: "Welcome to"
        - name: COMPANY
          value: "Stratos"
        - name: GROUP
          value: "Industries"
      command: ["/bin/sh", "-c", 'echo "$(GREETING) $(COMPANY) $(GROUP)"']
  restartPolicy: Never
```

> **Why:** `metadata.name` assigns the required Pod name, while `containers[].name` and `image` configure `print-env-container` with the Bash image. Each `env` entry defines one variable and its literal value. The exact `command` starts `/bin/sh`, uses `-c` to execute the following command string, and prints the three Kubernetes-expanded `$(VARIABLE)` references. The Pod-level `restartPolicy: Never` prevents Kubernetes from restarting this one-time container after it exits.

### 🚀 Step 2: Create the Pod

```bash
kubectl apply -f print-envars-greeting.yaml
```

Kubernetes confirmed the creation:

```text
pod/print-envars-greeting created
```

> **Why:** `kubectl apply` submits the declarative Pod configuration to the Kubernetes API. The `-f` option identifies `print-envars-greeting.yaml` as the manifest to apply.

### 🔍 Step 3: Check the Completed Pod

```bash
kubectl get pod print-envars-greeting
```

The Pod completed without restarting:

```text
NAME                    READY   STATUS      RESTARTS   AGE
print-envars-greeting   0/1     Completed   0          60s
```

> **Why:** `kubectl get pod` displays the current state of the named Pod. `Completed` is the expected successful state because the container only prints one message and exits. `READY 0/1` means no container process remains active, and `RESTARTS 0` confirms that `restartPolicy: Never` prevented another execution.

### ✅ Step 4: Verify the Expanded Greeting

```bash
kubectl logs -f print-envars-greeting
```

The container printed the expected message:

```text
Welcome to Stratos Industries
```

> **Why:** `kubectl logs` retrieves the container's standard output. The `-f` option follows the log stream until it ends. Because the container had already completed, the command displayed its single line and returned. The final text proves that Kubernetes expanded `$(GREETING)`, `$(COMPANY)`, and `$(GROUP)` using the values defined in the Pod manifest.

## Best Practices

- **Use `Never` for one-time commands.** A container that performs a finite task should not be restarted after successful completion unless retry behavior is explicitly required.
- **Do not mistake `Completed` for a failure.** For short-lived Pods, `Completed` indicates that the command exited successfully; a persistent `Running` state is not required.
- **Use ConfigMaps for reusable non-sensitive configuration.** Direct `env` values are clear for this small lab, while ConfigMaps are easier to manage when several workloads share application settings.
- **Use Secrets for confidential values.** Environment variables that contain passwords, tokens, or keys should reference Kubernetes Secrets instead of storing plaintext values in a Pod manifest.
- **Remember the Kubernetes expansion syntax.** References in `command` and `args` use `$(VARIABLE)`, which differs from the usual shell form `$VARIABLE`.

### 📚 Official Documentation

- [Define Environment Variables for a Container](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/)
- [Define a Command and Arguments for a Container](https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/)
- [Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [kubectl logs](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/)
