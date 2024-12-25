
# EKS Cluster with HPA, Karpenter, Prometheus, and Grafana

This guide provides step-by-step instructions to set up an Amazon EKS cluster integrated with Horizontal Pod Autoscaler (HPA), Karpenter for dynamic node provisioning, Prometheus for metrics collection, and Grafana for visualization.

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
       desiredCapacity: 2
       maxSize: 4
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

## Step 2: Install Prometheus and Grafana

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

3. Access Grafana:
   ```bash
   # Retrieve the Grafana admin password:
   kubectl get secret -n monitoring prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode

   # Forward the Grafana port to access it locally:
   kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

   # Open Grafana at http://localhost:3000
   ```

---

## Step 3: Install ADOT Collector for Prometheus

### Why?
AWS Distro for OpenTelemetry (ADOT) integrates with Prometheus to collect metrics and forward them to Prometheus.

### How?
1. Deploy the ADOT Collector ConfigMap:
   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: adot-collector-config
     namespace: adot
   data:
     collector-config.yaml: |
       receivers:
         otlp:
           protocols:
             grpc:
             http:
       exporters:
         prometheus:
           endpoint: "0.0.0.0:9090"
       service:
         pipelines:
           metrics:
             receivers: [otlp]
             exporters: [prometheus]
   ```
   ```bash
   kubectl apply -f adot-collector-config.yaml
   ```

2. Install the ADOT Collector:
   ```bash
   helm repo add adot https://aws-observability.github.io/aws-otel-helm-charts
   helm repo update
   kubectl create namespace adot
   helm install adot-collector adot/aws-otel-collector -n adot --set configMap=adot-collector-config
   ```

---

## Step 4: Deploy the Sample Application

### Why?
The sample application emits metrics to test the integration between Prometheus, ADOT, and Grafana.

### How?
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  labels:
    app: sample-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: sample-app
        image: ghcr.io/open-telemetry/opentelemetry-demo:latest
        env:
        - name: OTEL_EXPORTER_OTLP_ENDPOINT
          value: "http://adot-collector.adot:4317"
        ports:
        - containerPort: 8080
```
```bash
kubectl apply -f sample-app.yaml
```

---

## Step 5: Configure Horizontal Pod Autoscaler (HPA)

### Why?
HPA ensures the application scales automatically based on resource usage, such as CPU or memory.

### How?
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: sample-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: sample-app
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```
```bash
kubectl apply -f sample-app-hpa.yaml
```

---

## Step 6: Configure Karpenter

### Why?
Karpenter dynamically provisions nodes to ensure sufficient capacity for pods scaled by HPA.

### How?
```yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: adot-provisioner
spec:
  provider:
    instanceTypes:
      - "t3.medium"
      - "m5.large"
  ttlSecondsAfterEmpty: 30
```
```bash
kubectl apply -f karpenter-provisioner.yaml
```

---

## Step 7: Generate Load

### Why?
Generating load tests the scaling capabilities of HPA and Karpenter under real-world-like conditions.

### How?
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: load-generator
spec:
  template:
    spec:
      containers:
      - name: load-generator
        image: busybox
        command:
        - /bin/sh
        - -c
        - |
          while true; do wget -q -O- http://sample-app; done
      restartPolicy: Never
  backoffLimit: 4
```
```bash
kubectl apply -f load-generator.yaml
```

---

## Step 8: View Metrics in Grafana

### Why?
Grafana provides a real-time visualization of metrics collected by Prometheus.

### How?
1. Access Grafana at http://localhost:3000.
2. Log in using the admin password retrieved earlier.
3. Add Prometheus as a data source:
   - Navigate to Configuration > Data Sources.
   - Add a new data source with the following details:
     - URL: `http://prometheus-server.monitoring.svc.cluster.local:9090`
     - Access: Server (Default).
4. Import a Kubernetes dashboard (e.g., ID 315) for visualizing cluster metrics.

---

## Step 9: Clean Up Resources

### Why?
Cleaning up resources prevents unnecessary costs and ensures a clean cluster state.

### How?
```bash
kubectl delete -f load-generator.yaml
kubectl delete -f sample-app-hpa.yaml
kubectl delete -f sample-app.yaml
kubectl delete -f karpenter-provisioner.yaml
helm uninstall adot-collector -n adot
helm uninstall prometheus -n monitoring
kubectl delete namespace adot
kubectl delete namespace monitoring
eksctl delete cluster --name=prometheus-cluster
```
