# EKS Cluster with HPA, Karpenter, Prometheus, and Grafana

This guide provides step-by-step instructions to set up an Amazon EKS cluster integrated with Horizontal Pod Autoscaler (HPA), Karpenter for dynamic node provisioning, Prometheus for metrics collection, and Grafana for visualization. All external traffic is routed through an Ingress resource for centralized and efficient management.

---

## Step 1: Create an EKS Cluster with eksctl

### Why?
Creating an EKS cluster using eksctl simplifies the process by automating resource provisioning and configuration.

### How?
1. Create a configuration file for eksctl (`cluster-config.yaml`):
   ```yaml
   apiVersion: eksctl.io/v1alpha5
   kind: ClusterConfig
   metadata:
     name: prometheus-cluster
     region: us-east-1
   nodeGroups:
     - name: worker-nodes
       instanceType: t3.medium
       desiredCapacity: 3
       maxSize: 6
       minSize: 1
   ```

2. Create the cluster:
   ```bash
   eksctl create cluster -f cluster-config.yaml
   ```

3. Confirm the cluster is created:
   ```bash
   kubectl get nodes
   ```

---

## Step 2: Deploy an Ingress Controller

### Why?
An Ingress Controller is required to manage HTTP and HTTPS traffic to your cluster.

### How?
1. Add the NGINX Ingress Controller Helm repository:
   ```bash
   helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
   helm repo update
   ```

2. Install the Ingress Controller:
   ```bash
   helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace
   ```

3. Verify the deployment:
   ```bash
   kubectl get pods -n ingress-nginx
   kubectl get svc -n ingress-nginx
   ```
   - Note the external IP of the LoadBalancer.

---

## Step 3: Deploy Prometheus and Grafana

### Why?
Prometheus collects application and cluster metrics, while Grafana visualizes these metrics in customizable dashboards.

### How?
1. Add the Prometheus Helm repository:
   ```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm repo update
   ```

2. Install Prometheus and Grafana in the `monitoring` namespace:
   ```bash
   kubectl create namespace monitoring
   helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring
   ```

---

## Step 4: Expose Grafana Using Ingress

### Why?
Using an Ingress resource centralizes and simplifies routing external traffic to your services.

### How?
1. Ensure the Grafana service is of type `ClusterIP`:
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: prometheus-grafana
     namespace: monitoring
   spec:
     type: ClusterIP
     ports:
     - name: http
       port: 80
       targetPort: 3000
     selector:
       app.kubernetes.io/name: grafana
   ```
   Apply the service:
   ```bash
   kubectl apply -f grafana-service.yaml
   ```

2. Create an Ingress resource for Grafana:
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: grafana-ingress
     namespace: monitoring
     annotations:
       nginx.ingress.kubernetes.io/rewrite-target: /
   spec:
     rules:
     - http:
         paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: prometheus-grafana
               port:
                 number: 80
   ```

   Apply the Ingress resource:
   ```bash
   kubectl apply -f grafana-ingress.yaml
   ```

3. Verify Access:
   - Use the external IP of the LoadBalancer for the Ingress Controller to access Grafana. For example, open `http://<EXTERNAL-IP>` in your browser.

---

## Step 5: Configure the Rest of the Cluster

### 5.1 Deploy ADOT Collector

The AWS Distro for OpenTelemetry (ADOT) Collector gathers and exports metrics, logs, and traces from your cluster for observability.

1. Install ADOT Collector:
   ```bash
   helm repo add adot https://aws-observability.github.io/aws-otel-helm-charts
   helm repo update
   helm install adot adot/adot-collector -n monitoring
   ```

2. Verify the installation:
   ```bash
   kubectl get pods -n monitoring
   ```

3. Configure ADOT for Prometheus scraping and AWS X-Ray tracing by editing the `adot-collector-config.yaml`.

---

### 5.2 Deploy a Sample Application

Deploy a simple application to validate monitoring and scaling functionality.

1. Create a Kubernetes Deployment:
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: sample-app
     namespace: default
   spec:
       replicas: 3
       selector:
           matchLabels:
               app: sample
       template:
           metadata:
               labels:
                   app: sample
           spec:
               containers:
               - name: sample-app
                 image: hashicorp/http-echo:latest
                 args:
                 - "-text=Hello, World!"
                 ports:
                 - containerPort: 5678
   ```

   Apply the Deployment:
   ```bash
   kubectl apply -f sample-app.yaml
   ```

2. Expose the application:
   ```bash
   kubectl expose deployment sample-app --type=ClusterIP --port=80 --target-port=5678
   ```

---

### 5.3 Configure HPA and Karpenter

#### HPA
1. Enable the metrics server if not already installed:
   ```bash
   kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
   ```

2. Apply HPA to the sample app:
   ```yaml
   apiVersion: autoscaling/v2
   kind: HorizontalPodAutoscaler
   metadata:
     name: sample-app-hpa
     namespace: default
   spec:
     scaleTargetRef:
       apiVersion: apps/v1
       kind: Deployment
       name: sample-app
     minReplicas: 1
     maxReplicas: 5
     metrics:
     - type: Resource
       resource:
         name: cpu
         target:
           type: Utilization
           averageUtilization: 50
   ```

   Apply the HPA configuration:
   ```bash
   kubectl apply -f sample-app-hpa.yaml
   ```

#### Karpenter
1. Install Karpenter:
   ```bash
   helm repo add karpenter https://charts.karpenter.sh
   helm repo update
   helm install karpenter karpenter/karpenter --namespace karpenter --create-namespace
   ```

2. Apply required permissions for Karpenter to manage nodes:
   ```bash
   eksctl create iamserviceaccount        --name karpenter        --namespace karpenter        --cluster prometheus-cluster        --attach-policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy        --attach-policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly        --attach-policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy        --approve
   ```

3. Create a `Provisioner` resource to define scaling rules:
   ```yaml
   apiVersion: karpenter.sh/v1alpha5
   kind: Provisioner
   metadata:
     name: default
   spec:
     requirements:
       - key: "karpenter.k8s.aws/instance-category"
         operator: In
         values: ["t3", "m5"]
     limits:
       resources:
         cpu: "1000"
     provider:
       subnetSelector:
         karpenter.sh/discovery: "prometheus-cluster"
       securityGroupSelector:
         karpenter.sh/discovery: "prometheus-cluster"
   ```

   Apply the `Provisioner`:
   ```bash
   kubectl apply -f karpenter-provisioner.yaml
   ```

---

Let me know if further adjustments are needed!
