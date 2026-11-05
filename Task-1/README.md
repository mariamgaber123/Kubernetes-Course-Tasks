# Task 1: Multi-Container Pods (Init Container & Sidecar)

## 1. Overview

This task demonstrates how to deploy a multi-container application on a three-node Minikube Kubernetes cluster using an Init Container, a Main Container, and a Sidecar Container.

The application runs an Nginx web server that displays a welcome message with the current date and time. A Sidecar Container updates the HTML page every 5 seconds without restarting Nginx.

The task also demonstrates how Kubernetes handles Init Container failures and prevents application containers from starting until initialization succeeds.

## 2. Technologies Used

- Kubernetes v1.37.0
- Minikube v1.39.0
- Docker Desktop
- Calico CNI
- Nginx 1.27
- BusyBox 1.36
- kubectl

## 3. Cluster Configuration

| Configuration | Value |
|---|---|
| Cluster Name | mlops-course |
| Driver | Docker |
| Kubernetes Version | v1.37.0 |
| Nodes | 3 (1 Control Plane + 2 Workers) |
| CPUs per Node | 2 |
| Memory per Node | 2200 MB |
| CNI | Calico |

### Start the Cluster

```bash
minikube start -p mlops-course --nodes 3 --cni=calico --cpus 2 --memory 2200 --driver=docker
```

### Verify Cluster Nodes

```bash
kubectl --context=mlops-course get nodes
```

## 4. Kubernetes Resources

### Namespace

A dedicated namespace named `webapp` was created to isolate the application resources.

```bash
kubectl --context=mlops-course create namespace webapp
```

### Deployment Configuration

- Deployment Name: `web-deployment`
- Namespace: `webapp`
- Replicas: 3
- Pod Label: `app: web`
- Process Namespace Sharing: Enabled
- Volumes: None

### Container Architecture

Each Pod contains three containers:

| Container | Image | Role |
|---|---|---|
| init-done | busybox:1.36 | Executes initialization and must finish successfully before application containers start. |
| nginx | nginx:1.27 | Runs the Nginx web server and serves the HTML page. |
| time-updater | busybox:1.36 | Updates the HTML page with the current date and time every 5 seconds. |

### Architecture Flow

```text
Deployment: web-deployment
        |
        |-- Replica 1
        |     |-- Init: init-done
        |     |-- Main: nginx
        |     |-- Sidecar: time-updater
        |
        |-- Replica 2
        |     |-- Init: init-done
        |     |-- Main: nginx
        |     |-- Sidecar: time-updater
        |
        |-- Replica 3
              |-- Init: init-done
              |-- Main: nginx
              |-- Sidecar: time-updater
```

## 5. Implementation Details

### Init Container

The Init Container executes:

```bash
echo done
```

It must complete successfully before Kubernetes starts the main application containers.

### Main Container (Nginx)

The Nginx container:
1. Creates the initial HTML page.
2. Displays the welcome message and current timestamp.
3. Starts Nginx in the foreground.

### Sidecar Container (Time Updater)

The Sidecar:
1. Searches for the Nginx process using `pgrep`.
2. Retrieves the Nginx process ID.
3. Accesses the Nginx container filesystem through `/proc/$PID/root`.
4. Updates `index.html` with the current timestamp.
5. Repeats the operation every 5 seconds.

### Process Namespace Sharing

The Pod uses:

```yaml
shareProcessNamespace: true
```

This allows containers within the same Pod to see each other's processes.

The Sidecar also requires the `SYS_PTRACE` capability as specified in the task.

No shared volumes are configured.

## 6. Deployment Instructions

Apply the deployment configuration:

```bash
kubectl --context=mlops-course apply -f deployment.yaml
```

Check the Deployment:

```bash
kubectl --context=mlops-course get deployments -n webapp
```

Check the Pods:

```bash
kubectl --context=mlops-course get pods -n webapp
```

### Expected Result

```text
NAME             READY   UP-TO-DATE   AVAILABLE
web-deployment   3/3     3            3
```

Each Pod should reach:

```text
READY   STATUS
2/2     Running
```

**Note:** The Init Container is excluded from the ready container count because it terminates successfully after initialization. Therefore, each Pod reports `2/2 Running`, representing Nginx and the Sidecar.

## 7. Application Verification

### Port Forwarding

Expose the Nginx service locally:

```bash
kubectl --context=mlops-course port-forward deployment/web-deployment 8080:80 -n webapp
```

Open the application:

http://localhost:8080

### Expected Web Page

```text
hello in MLOPS MENA COMMUNITY : YYYY-MM-DD HH:MM:SS
```

The Sidecar updates the HTML file every 5 seconds.

Refreshing the browser displays the updated timestamp.

## 8. Init Container Failure Test

To test Kubernetes initialization behavior, the Init Container command was temporarily changed to:

```yaml
command: ["sh", "-c", "echo 'init failed'; exit 1"]
```

Apply the modified configuration:

```bash
kubectl --context=mlops-course apply -f deployment.yaml
```

Monitor the Pods:

```bash
kubectl --context=mlops-course get pods -n webapp -w
```

### Observed Results

The new Pod entered the following states:

```text
Init:0/1
Init:Error
Init:CrashLoopBackOff
```

### Verify Init Container Logs

```bash
kubectl --context=mlops-course logs POD_NAME -n webapp -c init-done
```

Expected output:

```text
init failed
```

### Inspect Pod Details

```bash
kubectl --context=mlops-course describe pod POD_NAME -n webapp
```

Important observations:

- Init Container terminated with `Reason: Error`.
- Exit Code was `1`.
- Restart Count increased as Kubernetes retried the failed container.
- Nginx remained in `Waiting` state.
- Sidecar remained in `Waiting` state.
- Pod initialization was not completed.

**Conclusion:** Kubernetes does not start application containers until all Init Containers complete successfully.

## 9. Recovery and Final Verification

The original Init Container command was restored:

```yaml
command: ["sh", "-c", "echo done"]
```

Apply the corrected configuration:

```bash
kubectl --context=mlops-course apply -f deployment.yaml
```

Check the rollout:

```bash
kubectl --context=mlops-course rollout status deployment/web-deployment -n webapp
```

Expected output:

```text
deployment "web-deployment" successfully rolled out
```

Final Pod verification:

```bash
kubectl --context=mlops-course get pods -n webapp
```

All three Pods returned to `2/2 Running`.

## 10. Project Structure

```text
Task-1/
│
├── deployment.yaml
└── README.md
```

## 11. Screenshots

Screenshots documenting the implementation and verification will be added below.

| Screenshot | Description |
|---|---|
| Cluster Nodes | Three Kubernetes nodes in Ready state. |
| Calico | Calico pods and DaemonSet running on all nodes. |
| Namespace | Successful creation of the webapp namespace. |
| Deployment | Three replicas successfully running. |
| Web Application | Nginx welcome page displayed in the browser. |
| Sidecar Verification | Updated timestamp after refreshing the page. |
| Init Failure | Pod showing Init:CrashLoopBackOff. |
| Init Logs | The `init failed` message and exit code 1. |
| Recovery | Deployment successfully rolled out with all three Pods ready. |

## 12. Final Result

The task was successfully implemented and verified.

- Three-node Minikube cluster configured with Calico.
- Three application replicas deployed successfully.
- Init Container initialization verified.
- Nginx web server running successfully.
- Sidecar updating the HTML page every 5 seconds.
- Init Container failure and its impact on application containers demonstrated.
- Deployment successfully recovered after restoring the original initialization command.

**Status: Completed Successfully.**