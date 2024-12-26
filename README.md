
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
   helm install ingress-nginx ingress-nginx/ingress-nginx      --namespace ingress-nginx --create-namespace
   ```

3. Verify the deployment:
   ```bash
   kubectl get pods -n ingress-nginx
   kubectl get svc -n ingress-nginx
   ```
   - Note the external IP or DNS name of the LoadBalancer.

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
     - host: grafana.example.com
       http:
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

3. Update DNS:
   - Point your domain (`grafana.example.com`) to the external IP of the Ingress Controller LoadBalancer.

4. Verify Access:
   - Open `http://grafana.example.com` in your browser.

---

## Step 5: Enable HTTPS for Grafana

### Why?
Using HTTPS secures communication between your browser and Grafana.

### How?
1. Install **Cert-Manager**:
   ```bash
   kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.yaml
   ```

2. Create a ClusterIssuer for Let’s Encrypt:
   ```yaml
   apiVersion: cert-manager.io/v1
   kind: ClusterIssuer
   metadata:
     name: letsencrypt
   spec:
     acme:
       server: https://acme-v02.api.letsencrypt.org/directory
       email: your-email@example.com
       privateKeySecretRef:
         name: letsencrypt-private-key
       solvers:
       - http01:
           ingress:
             class: nginx
   ```

   Apply the ClusterIssuer:
   ```bash
   kubectl apply -f cluster-issuer.yaml
   ```

3. Update the Ingress resource for TLS:
   Add a TLS section to the Ingress resource:
   ```yaml
   spec:
     tls:
     - hosts:
       - grafana.example.com
       secretName: grafana-tls
   ```

   Apply the updated Ingress:
   ```bash
   kubectl apply -f grafana-ingress.yaml
   ```

4. Verify HTTPS:
   - Access Grafana at `https://grafana.example.com`.

---

## Step 6: Configure the Rest of the Cluster

- **Deploy ADOT Collector**: Follow Step 4 from the original guide.
- **Deploy the Sample Application**: Follow Step 5 from the original guide.
- **Configure HPA and Karpenter**: Follow Steps 6 and 7 from the original guide.

---

Let me know if you’d like to adjust this further or need a downloadable version!

