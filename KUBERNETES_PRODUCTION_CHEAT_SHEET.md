# Kubernetes Production Cheat Sheet

## Chapter 1: Essential Concepts & Cluster Anatomy

Before troubleshooting, you must understand the "Who's Who" of the cluster to isolate where a failure is occurring.

### 1.1 The Control Plane vs. The Data Plane

- **API Server**: The gateway. If `kubectl` commands time out, the API server is likely overloaded or the network is latent.
- **Etcd**: The "Brain." It stores the cluster state. If Etcd fails, the cluster becomes "read-only."
- **Kubelet**: The "Agent" on each node. If a node is `NotReady`, the Kubelet is usually crashed or the node has a "DiskPressure" issue.

### 1.2 Essential Management Commands

| Goal | Command |
|------|---------|
| **Global View** | `kubectl get pods -A -o wide` |
| **Resource Usage** | `kubectl top nodes` / `kubectl top pods` |
| **Real-time Events** | `kubectl get events -w --sort-by='.lastTimestamp'` |
| **Context Switch** | `kubectx <cluster-name>` (Recommended tool) |
| **Namespace Switch** | `kubens <namespace>` (Recommended tool) |

---

## Chapter 2: Common Production Issues & SOPs

Use these step-by-step resolutions when an alert fires.

### 2.1 Issue: CrashLoopBackOff

**Definition**: The pod starts, but the application process inside crashes immediately.

**SOP Resolution**:

1.  **Check Previous Logs**:
    ```bash
    kubectl logs <pod-name> -p
    ```
    *Why*: You need the logs from the failed instance, not the new one starting up.

2.  **Verify Environment**:
    ```bash
    kubectl describe pod <pod-name>
    ```
    Check the Environment section. Ensure ConfigMaps/Secrets are actually mounted.

3.  **Check Exit Code**:
    - **Exit 1**: Standard application crash (Check code/logs).
    - **Exit 137**: OOMKilled (See section 2.2).

### 2.2 Issue: OOMKilled (Exit Code 137)

**Definition**: The container exceeded its memory limit or the node ran out of RAM.

**SOP Resolution**:

1.  **Verify Reason**:
    ```bash
    kubectl describe pod <pod-name> | grep -i "Terminated"
    ```

2.  **Check Limits**:
    Compare `kubectl top pod` results with the limits in your YAML.

3.  **Immediate Fix**:
    Increase the `limits.memory` in the Deployment manifest.
    ```bash
    kubectl patch deployment <name> -p '{"spec":{"template":{"spec":{"containers":[{"name":"app","resources":{"limits":{"memory":"1Gi"}}}]}}}}'
    ```

### 2.3 Issue: ImagePullBackOff / ErrImagePull

**Definition**: Kubelet cannot fetch the container image.

**SOP Resolution**:

1.  **Check Registry Auth**:
    Ensure the `imagePullSecrets` exist in the namespace.
    ```bash
    kubectl get secret <secret-name>
    ```

2.  **Manual Test**:
    Attempt to pull the image manually on your local machine using `docker pull`.

3.  **Check Node DNS**:
    Sometimes nodes lose the ability to resolve external registry URLs (e.g., mcr.microsoft.com or gcr.io).

---

## Chapter 3: The Professional Troubleshooting Flow

Follow this 4-tier hierarchy to find the root cause of any incident.

### Tier 1: The "Describe" Phase
**Always start here.** It shows you the lifecycle of the object.

**Command**:
```bash
kubectl describe <resource> <name>
```
**Focus**: Look at the **Events** section at the bottom. It will tell you if the Scheduler failed to find a node or if the Volume failed to attach.

### Tier 2: The Connectivity Phase
If the pod is **Running** but not receiving traffic.

1.  **Check Service Selectors**:
    ```bash
    kubectl get svc <name> -o wide
    ```
    Ensure the selector labels match the Pod labels.

2.  **Check Endpoints**:
    ```bash
    kubectl get endpoints <service-name>
    ```
    If this is empty, your Service isn't "finding" any pods.

3.  **Bypass Ingress**:
    Use port-forwarding to see if the service works internally.
    ```bash
    kubectl port-forward svc/<name> 8080:80
    ```

### Tier 3: The Deep-Dive (Ephemeral Debugging)
If your container is "Distroless" (no shell), you cannot exec into it.

**SOP**: Use an ephemeral debug container to "spy" on the pod.
```bash
kubectl debug -it <pod-name> --image=nicolaka/netshoot --target=<container-name>
```
This allows you to run `tcpdump`, `curl`, and `netstat` as if you were inside the app container.

---

## Chapter 4: Advanced Concepts for Platform Engineering

To scale as a team, you must move from manual fixes to automated governance.

### 4.1 GitOps (The Gold Standard)
Stop using `kubectl apply` for production. Use a GitOps controller like **ArgoCD**.
**SOP**: All changes are made via Pull Request to a Git repo. The controller syncs the cluster to the Git state.

### 4.2 Admission Controllers (Policy as Code)
Use tools like **Kyverno** or **OPA Gatekeeper** to block "bad" deployments before they happen.
**Example Policy**: "No pod can be deployed without Resource Limits" or "No image can be pulled from public DockerHub."

### 4.3 Service Mesh (Istio)
Use for **mTLS** (automatic encryption between pods) and **Canary Deployments** (sending 5% of traffic to a new version to test stability).
