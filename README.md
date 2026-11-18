# Kubernetes Nginx Deployment & Autoscaling Project

This project demonstrates how to deploy and manage the fundamental building blocks of Kubernetes (**Pod**, **Deployment**, **Service**, and **Horizontal Pod Autoscaler (HPA)**) using a declarative approach with YAML configuration files.

## 📁 Project Structure

The repository contains the following configuration files:

- `nginx_pod.yaml`: Defines a standalone Nginx Pod.
- `deployment.yaml`: Defines an Nginx Deployment with 3 replicas and CPU resource requests.
- `service.yaml`: Exposes the application to the outside world using a `NodePort` service.
- `hpa.yaml`: Defines an HPA that automatically scales the number of Pods based on CPU utilization (between 1 and 10 replicas).

## 🛠️ Prerequisites

To run this project in your local environment, you need to have the following tools installed:

- **Docker Desktop** (with Kubernetes enabled under Settings > Kubernetes) or **Minikube**
- **kubectl** (Kubernetes command-line tool)

### 1. Installing `kubectl` on macOS

Install using Homebrew through the terminal:

```bash
brew install kubectl
```

### 2. Installing `kubectl` on Windows

You can install it using **Winget** or **Chocolatey** through PowerShell or Command Prompt (CMD). If Docker Desktop is already installed, `kubectl` is generally included.

```powershell
# Install using Winget (Recommended)
winget install -e --id Kubernetes.kubectl

# Alternatively, install using Chocolatey
choco install kubernetes-cli
```

### Cluster Verification (macOS & Windows)

After installation, verify that your Kubernetes cluster is running:

```bash
kubectl cluster-info
kubectl get nodes
```

## 🚀 Installation and Execution Steps

### 1. Creating a Standalone Pod (Optional Test)

To start a single Nginx Pod:

```bash
kubectl apply -f nginx_pod.yaml
kubectl get pods
```

### 2. Creating a Deployment

To run the application with 3 replicas and enable self-healing (automatic recovery):

```bash
kubectl apply -f deployment.yaml
```

To check the deployment status:

```bash
kubectl get deployments
kubectl get pods
```

### 3. Exposing the Application Using a NodePort Service

To expose the Pods to the outside world through a fixed port and enable load balancing:

```bash
kubectl apply -f service.yaml
kubectl get services
```

To access the application through your browser:

- **Docker Desktop (macOS & Windows):** `http://localhost:30007`
- **Minikube:** Run the following command to obtain the service URL:

```bash
minikube service nginx-service --url
```

## 📈 Setting Up Horizontal Pod Autoscaler (HPA)

For the HPA to retrieve CPU utilization metrics, **Metrics Server** must be installed in the cluster.

### 1. Installing Metrics Server (For Local Clusters)

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

To bypass TLS certificate verification errors in local development environments, apply the appropriate patch for your operating system:

**macOS / Linux Terminal:**

```bash
kubectl patch deployment metrics-server -n kube-system --type='json' -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'
```

**Windows (PowerShell / CMD):**

```powershell
kubectl patch deployment metrics-server -n kube-system --type="json" -p="[{\\"op\\": \\"add\\", \\"path\\": \\"/spec/template/spec/containers/0/args/-\\", \\"value\\": \\"--kubelet-insecure-tls\\"}]"
```

Verify that metrics are being collected (this may take 30–60 seconds):

```bash
kubectl top nodes
```

### 2. Enabling HPA

Start the HPA, which automatically increases the number of Pods up to a maximum of 10 when CPU utilization exceeds 80%, and scales down to a minimum of 1 when there is little or no load:

```bash
kubectl apply -f hpa.yaml
kubectl get hpa
```

### 3. Load Testing (Autoscaling Test)

To test automatic scaling, start monitoring the HPA in one terminal:

```bash
kubectl get hpa -w
```

Open a second terminal window (macOS Terminal or Windows PowerShell) and start a temporary load-generator Pod that continuously sends requests to the service:

```bash
kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://nginx-service:8080; done"
```

## 📄 YAML Configuration Files

The contents of the configuration files in the repository are shown below.

**`nginx_pod.yaml`**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: webserver
      image: nginx:latest
      ports:
        - containerPort: 80
```

**`deployment.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
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
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
```

**`service.yaml`**

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
    - port: 8080
      targetPort: 80
      nodePort: 30007
```

**`hpa.yaml`**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 80
```

## 🧹 Cleanup

To delete all the resources created in the cluster, execute the following commands:

```bash
kubectl delete -f hpa.yaml
kubectl delete -f service.yaml
kubectl delete -f deployment.yaml
kubectl delete -f nginx_pod.yaml
```