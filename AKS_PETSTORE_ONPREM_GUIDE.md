# AKS Pet Store On-Premises Deployment Guide

> **Version**: 1.2  
> **Target Environment**: On-Premises Kubernetes / Local Servers  
> **Application**: Microsoft AKS Store Demo (Pet Store)  
> **Compatibility**: Any Kubernetes distribution (K3s, K8s, MicroK8s, etc.)  
> **Red Hat Ready**: Optimized for RHEL 9, includes RHEL/CentOS/Fedora instructions

## 🎯 Overview

This guide provides step-by-step instructions for deploying the Microsoft AKS Pet Store demo application on your on-premises Kubernetes cluster. The Pet Store is a comprehensive microservices e-commerce platform demonstrating modern cloud-native architecture patterns.

**🔴 RHEL 9 Optimized**: This guide is specifically optimized for Red Hat Enterprise Linux 9 with cgroup v2, Podman/container tools, and modern SELinux policies.### What You'll Deploy

- **9 Microservices**: Complete polyglot e-commerce platform
- **Vue.js Frontend**: Customer store and admin interfaces
- **Backend Services**: Node.js, Golang, and Rust microservices
- **Data Layer**: MongoDB database and RabbitMQ message queue
- **Simulation**: Automated customer and worker services for realistic demos

## 🏗️ Architecture Overview

### Frontend Applications
- **Store Front** (Vue.js) - Customer shopping interface
- **Store Admin** (Vue.js) - Administrative management interface

### Backend Microservices
- **Order Service** (Node.js) - Order processing and management
- **Product Service** (Rust) - Product catalog CRUD operations
- **Makeline Service** (Golang) - Order fulfillment processing

### Infrastructure Services
- **MongoDB** - Persistent database with StatefulSet
- **RabbitMQ** - Message queue for order processing

### Simulation Services
- **Virtual Customer** (Rust) - Automated order generation
- **Virtual Worker** (Rust) - Automated order processing

## 🛠️ Prerequisites

### Required Infrastructure
- **Kubernetes Cluster**: Any distribution (K3s, K8s, MicroK8s, etc.)
- **kubectl**: Kubernetes command-line tool
- **Storage**: Dynamic persistent volume provisioning
- **Network**: NodePort access (ports 30000-32767)

### Resource Requirements
- **CPU**: Minimum 2 cores, Recommended 4+ cores
- **Memory**: Minimum 4GB RAM, Recommended 8GB+ RAM
- **Storage**: 10GB+ available for persistent volumes
- **Network**: Internet access for pulling container images

### Firewall Configuration

#### For Ubuntu/Debian (ufw):
```bash
# Allow NodePort range
sudo ufw allow 30000:32767/tcp

# Or specific ports we'll use:
sudo ufw allow 30080/tcp  # Store Front
sudo ufw allow 30081/tcp  # Store Admin
sudo ufw allow 30092/tcp  # RabbitMQ Management (optional)
```

#### For Red Hat/CentOS/Fedora (firewalld):
```bash
# Allow NodePort range
sudo firewall-cmd --permanent --add-port=30000-32767/tcp
sudo firewall-cmd --reload

# Or specific ports we'll use:
sudo firewall-cmd --permanent --add-port=30080/tcp  # Store Front
sudo firewall-cmd --permanent --add-port=30081/tcp  # Store Admin
sudo firewall-cmd --permanent --add-port=30092/tcp  # RabbitMQ Management
sudo firewall-cmd --reload

# Verify firewall rules
sudo firewall-cmd --list-ports

# Alternative: Add to public zone specifically
sudo firewall-cmd --zone=public --permanent --add-port=30080-30092/tcp
sudo firewall-cmd --reload
```

#### For Red Hat Enterprise Linux (RHEL) - SELinux Considerations:
```bash
# Check SELinux status
sestatus

# If SELinux is enforcing, allow container access
sudo setsebool -P container_manage_cgroup true

# Allow container networking (if needed)
sudo semanage port -a -t container_port_t -p tcp 30080-30092

# Or temporarily disable SELinux (not recommended for production)
# sudo setenforce 0
```

## 🏠 K3s Local Installation Guide

If you don't have Kubernetes installed yet, K3s is the easiest option for local development and testing. Here's how to install and configure K3s:

### Option 1: Standard K3s Installation (Recommended)

#### Install K3s Server
```bash
# For RHEL 9 - Install K3s with optimized settings
curl -sfL https://get.k3s.io | sh -

# RHEL 9 Prerequisites - Install container tools and SELinux policies
sudo dnf install -y container-selinux selinux-policy-base container-tools podman

# RHEL 9 specific - Install K3s SELinux policy
sudo dnf install -y https://rpm.rancher.io/k3s/latest/common/centos/9/noarch/k3s-selinux-1.4-1.el9.noarch.rpm

# For RHEL 8 (legacy)
# sudo dnf install -y https://rpm.rancher.io/k3s/latest/common/centos/8/noarch/k3s-selinux-1.2-2.el8.noarch.rpm

# RHEL 7 (legacy - not recommended)
# sudo yum install -y container-selinux selinux-policy-base
# sudo yum install -y https://rpm.rancher.io/k3s/latest/common/centos/7/noarch/k3s-selinux-0.2-1.el7_8.noarch.rpm

# RHEL 9 specific - Configure for cgroup v2 (default in RHEL 9)
# K3s automatically detects cgroup v2, no additional configuration needed

# Check installation status
sudo systemctl status k3s

# Verify cluster is running
sudo k3s kubectl get nodes

# Start K3s on boot (Red Hat systems)
sudo systemctl enable k3s

# RHEL 9 - Verify cgroup version being used
cat /sys/fs/cgroup/cgroup.controllers
```

#### Configure kubectl Access
```bash
# Copy kubeconfig for regular user access
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

# Update server address if needed (for remote access)
# sed -i 's/127.0.0.1/YOUR-SERVER-IP/g' ~/.kube/config

# Test kubectl access
kubectl get nodes
```

### Option 2: K3s with Custom Configuration

#### Create K3s Configuration
```bash
# Create K3s config directory
sudo mkdir -p /etc/rancher/k3s

# Create custom configuration
sudo cat > /etc/rancher/k3s/config.yaml << 'EOF'
# Disable Traefik (optional - we'll use NodePort)
disable:
  - traefik

# Enable features we need
cluster-init: true

# Set node name
node-name: "k3s-petstore"

# Kubelet configuration
kubelet-arg:
  - "max-pods=110"

# API server configuration
kube-apiserver-arg:
  - "service-node-port-range=30000-32767"
EOF

# Install K3s with custom config
curl -sfL https://get.k3s.io | sh -
```

#### Verify Installation
```bash
# Check K3s service
sudo systemctl status k3s

# Check nodes
kubectl get nodes -o wide

# Check system pods
kubectl get pods -A
```

### Option 3: K3d (K3s in Docker) - Alternative

If you prefer Docker-based deployment:

```bash
# Install K3d
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash

# Create K3s cluster in Docker
k3d cluster create petstore \
  --port "30080:30080@agent:0" \
  --port "30081:30081@agent:0" \
  --port "30092:30092@agent:0" \
  --agents 1

# Update kubeconfig
k3d kubeconfig merge petstore --kubeconfig-switch-context

# Verify cluster
kubectl get nodes
```

### K3s Post-Installation Configuration

#### Install Local Storage Provisioner (If Needed)
```bash
# K3s includes local-path provisioner by default
kubectl get storageclass

# If you need additional storage options:
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.24/deploy/local-path-storage.yaml
```

#### Install Metrics Server (For HPA)
```bash
# Install metrics-server for horizontal pod autoscaling
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Patch metrics-server for local development (insecure TLS)
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'

# Verify metrics-server
kubectl top nodes
```

#### Configure Container Runtime (Optional)
```bash
# Check current runtime
kubectl get nodes -o wide

# K3s uses containerd by default - no changes needed
# For Docker runtime (if needed):
# curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--docker" sh -
```

### K3s Cluster Management

#### Useful K3s Commands
```bash
# Start/Stop K3s
sudo systemctl start k3s
sudo systemctl stop k3s

# Enable/Disable auto-start
sudo systemctl enable k3s
sudo systemctl disable k3s

# View K3s logs
sudo journalctl -u k3s -f

# Check K3s configuration
sudo cat /etc/rancher/k3s/k3s.yaml
```

#### Add Worker Nodes (Optional)
```bash
# On master node - get token
sudo cat /var/lib/rancher/k3s/server/node-token

# On worker nodes - join cluster
curl -sfL https://get.k3s.io | K3S_URL=https://MASTER-IP:6443 \
  K3S_TOKEN=NODE-TOKEN sh -

# Verify nodes
kubectl get nodes
```

### K3s Troubleshooting

#### Common Issues and Solutions

**1. Permission Denied**
```bash
# Fix kubeconfig permissions
sudo chown $(id -u):$(id -g) ~/.kube/config
chmod 600 ~/.kube/config
```

**2. Pod Startup Issues**
```bash
# Check system resources
df -h
free -h

# Check K3s logs
sudo journalctl -u k3s --no-pager -l

# Restart K3s if needed
sudo systemctl restart k3s
```

**3. Network Issues**
```bash
# Check CNI
kubectl get pods -n kube-system | grep -E "(flannel|calico|cilium)"

# Check node status
kubectl describe nodes
```

**4. Storage Issues**
```bash
# Check storage class
kubectl get storageclass

# Check local-path provisioner
kubectl get pods -n local-path-storage
```

**5. RHEL 9 Specific Issues**
```bash
# SELinux blocking container operations (RHEL 9 has stricter policies)
sudo setsebool -P container_manage_cgroup true
sudo setsebool -P container_use_cephfs true
sudo ausearch -m avc -ts recent | grep k3s

# Check if firewalld is blocking traffic
sudo firewall-cmd --list-all

# RHEL 9 - Fix container runtime issues
sudo dnf update container-selinux container-tools

# Check Red Hat subscription status (RHEL 9)
sudo subscription-manager status
sudo subscription-manager list --available

# RHEL 9 - cgroup v2 specific issues (rare, but possible)
# K3s handles cgroup v2 automatically in RHEL 9
mount | grep cgroup
systemctl --version  # Should be 249+ for proper cgroup v2 support

# RHEL 9 - Check container storage configuration
podman info | grep -A5 storage

# Fix systemd-resolved conflicts (if using custom DNS)
sudo systemctl disable systemd-resolved
sudo systemctl stop systemd-resolved

# RHEL 9 - Container networking debugging
sudo podman network ls
sudo firewall-cmd --get-active-zones
```

### K3s Uninstallation (If Needed)

```bash
# Uninstall K3s completely
sudo /usr/local/bin/k3s-uninstall.sh

# Clean up remaining files
sudo rm -rf /etc/rancher/k3s
sudo rm -rf /var/lib/rancher/k3s
rm -rf ~/.kube
```

### K3s Performance Tuning

#### Resource Limits
```bash
# Check current limits
kubectl describe node | grep -A 5 "Allocated resources"

# Adjust K3s resource limits if needed
sudo systemctl edit k3s

# Add under [Service]:
[Service]
Environment="K3S_NODE_NAME=petstore-node"
Environment="K3S_KUBELET_ARGS=--max-pods=50"
```

#### Network Performance
```bash
# For high-performance networking
sudo cat >> /etc/rancher/k3s/config.yaml << 'EOF'
flannel-backend: "host-gw"  # Higher performance than VXLAN
EOF

sudo systemctl restart k3s
```

### K3s vs Other Kubernetes Options

| Feature | K3s | K8s (kubeadm) | MicroK8s | K3d |
|---------|-----|---------------|-----------|-----|
| **Installation Time** | ~2 minutes | ~15 minutes | ~5 minutes | ~1 minute |
| **Resource Usage** | Low (512MB) | High (2GB+) | Medium (1GB) | Very Low (Docker) |
| **Production Ready** | ✅ Yes | ✅ Yes | ✅ Yes | ❌ Dev only |
| **Multi-node** | ✅ Easy | ✅ Complex | ✅ Easy | ✅ Docker-based |
| **Storage** | ✅ Built-in | ⚙️ Manual | ✅ Built-in | ✅ Built-in |
| **Load Balancer** | ✅ Built-in | ❌ Manual | ✅ Add-on | ✅ Built-in |
| **Best For** | Edge/IoT/Dev | Enterprise | Ubuntu/Snap | Development |

**Recommendation**: Use **K3s** for this Pet Store deployment - it's the easiest to install and manage while being production-ready.

## 🚀 Quick Start Deployment

### Step 1: Create Namespace
```bash
# Create dedicated namespace for the pet store
kubectl create namespace pets

# Verify namespace creation
kubectl get namespaces
```

### Step 2: Deploy Application Stack
```bash
# Deploy all microservices from Microsoft's official manifest
kubectl apply -f https://raw.githubusercontent.com/Azure-Samples/aks-store-demo/main/aks-store-all-in-one.yaml -n pets

# Verify deployment started
kubectl get pods -n pets --watch
```

### Step 3: Create On-Premises Service Configuration
```bash
# Create on-premises specific service configuration
cat > pets-onprem-services.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: store-front-nodeport
  namespace: pets
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080
  selector:
    app: store-front
---
apiVersion: v1
kind: Service
metadata:
  name: store-admin-nodeport
  namespace: pets
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 8081
      nodePort: 30081
  selector:
    app: store-admin
---
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq-management-nodeport
  namespace: pets
spec:
  type: NodePort
  ports:
    - port: 15672
      targetPort: 15672
      nodePort: 30092
  selector:
    app: rabbitmq
EOF

# Apply the on-premises service configuration
kubectl apply -f pets-onprem-services.yaml
```

### Step 4: Wait for Deployment Completion
```bash
# Monitor pod startup (should take 2-5 minutes)
kubectl get pods -n pets

# Wait for all pods to be Running (expected: 9 pods)
kubectl wait --for=condition=ready pod --all -n pets --timeout=300s
```

### Step 5: Get Access URLs
```bash
# Get your server's IP address
SERVER_IP=$(hostname -I | awk '{print $1}')

# Display access URLs
echo "🏪 Pet Store Front: http://$SERVER_IP:30080"
echo "👨‍💼 Admin Interface: http://$SERVER_IP:30081"
echo "📊 RabbitMQ Management: http://$SERVER_IP:30092 (guest/guest)"
```

## 📊 Deployment Verification

### Check Pod Status
```bash
# All pods should be Running
kubectl get pods -n pets

# Expected output:
NAME                                READY   STATUS    RESTARTS   AGE
mongodb-0                           1/1     Running   0          5m
rabbitmq-0                          1/1     Running   0          5m
order-service-75685c56d8-xxx        1/1     Running   0          5m
makeline-service-89ddf67b5-xxx      1/1     Running   0          5m
product-service-5cb9549676-xxx      1/1     Running   0          5m
store-front-7d9cfcd7-xxx            1/1     Running   0          5m
store-admin-f847666f5-xxx           1/1     Running   0          5m
virtual-customer-5ff7665484-xxx     1/1     Running   0          5m
virtual-worker-7848d8cd74-xxx       1/1     Running   0          5m
```

### Check Service Status
```bash
# Verify NodePort services
kubectl get svc -n pets

# Test HTTP connectivity
SERVER_IP=$(hostname -I | awk '{print $1}')
curl -I http://$SERVER_IP:30080  # Should return HTTP 200
curl -I http://$SERVER_IP:30081  # Should return HTTP 200
```

### Validate Application Function
```bash
# Test from within cluster (if available)
kubectl run test-pod --rm -i --tty --image=busybox -- /bin/sh
# Inside the test pod:
wget -qO- http://store-front.pets.svc.cluster.local
exit
```

## 🌐 Accessing the Application

### Customer Store Front
- **URL**: `http://YOUR-SERVER-IP:30080`
- **Purpose**: Customer shopping experience
- **Features**:
  - Browse pet products
  - Add items to cart
  - Place orders
  - View order status

### Admin Interface
- **URL**: `http://YOUR-SERVER-IP:30081`
- **Purpose**: Store management and analytics
- **Features**:
  - View incoming orders
  - Manage product catalog
  - Monitor order queues
  - View sales analytics

### RabbitMQ Management (Optional)
- **URL**: `http://YOUR-SERVER-IP:30092`
- **Credentials**: guest / guest
- **Purpose**: Message queue monitoring
- **Features**:
  - View message queues
  - Monitor queue performance
  - Debug message flow

## 🔧 Configuration Options

### Alternative Service Types

#### Option 1: LoadBalancer (If Available)
```bash
# If your cluster has LoadBalancer support (MetalLB, etc.)
kubectl patch svc store-front -n pets -p '{"spec":{"type":"LoadBalancer"}}'
kubectl patch svc store-admin -n pets -p '{"spec":{"type":"LoadBalancer"}}'

# Get LoadBalancer IPs
kubectl get svc -n pets
```

#### Option 2: Ingress Controller
```bash
# Example Ingress configuration (adjust for your ingress controller)
cat > pets-ingress.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: petstore-ingress
  namespace: pets
spec:
  rules:
  - host: petstore.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: store-front
            port:
              number: 80
  - host: admin.petstore.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: store-admin
            port:
              number: 80
EOF

kubectl apply -f pets-ingress.yaml
```

#### Option 3: Port Forwarding (Development)
```bash
# Forward ports to localhost (run in background)
kubectl port-forward -n pets svc/store-front 8080:80 &
kubectl port-forward -n pets svc/store-admin 8081:80 &

# Access via localhost
echo "Store Front: http://localhost:8080"
echo "Admin Interface: http://localhost:8081"
```

### Persistent Storage Configuration

#### Check Current Storage
```bash
# View persistent volumes and claims
kubectl get pv,pvc -n pets

# Storage should be automatically provisioned
```

#### Custom Storage Class (If Needed)
```bash
# Example local storage class
cat > local-storage-class.yaml << 'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
EOF

kubectl apply -f local-storage-class.yaml
```

## 🎬 Demo Scenarios

### Scenario 1: Customer Shopping Experience
1. **Browse Products**: Open store front, view pet product catalog
2. **Add to Cart**: Select products, adjust quantities
3. **Place Order**: Complete checkout process
4. **Monitor Processing**: Watch order move through system

### Scenario 2: Administrative Management
1. **View Orders**: Open admin interface, see incoming orders
2. **Manage Inventory**: Add/edit products in catalog
3. **Monitor Queues**: Check order processing status
4. **View Analytics**: Examine sales data and trends

### Scenario 3: System Architecture Demo
1. **Show Microservices**: Explain service decomposition
2. **Demonstrate Event-Driven**: Show RabbitMQ message flow
3. **Display Scalability**: Show horizontal scaling capability
4. **Highlight Resilience**: Demonstrate pod restart/recovery

### Scenario 4: Live Simulation
1. **Virtual Customers**: Automatic order generation
2. **Virtual Workers**: Automatic order processing
3. **Real-time Updates**: Live order flow in admin interface
4. **System Load**: Demonstrate handling multiple concurrent orders

## 🐛 Troubleshooting

### Common Issues and Solutions

#### 1. Pods Not Starting
```bash
# Check pod status and events
kubectl describe pods -n pets

# Common causes:
# - Insufficient resources
# - Image pull failures
# - Storage provisioning issues

# Solutions:
kubectl delete pod <pod-name> -n pets  # Restart pod
kubectl scale deployment <deployment-name> --replicas=0 -n pets
kubectl scale deployment <deployment-name> --replicas=1 -n pets
```

#### 2. Services Not Accessible
```bash
# Verify service endpoints
kubectl get svc -n pets
kubectl get endpoints -n pets

# Check if NodePort is accessible
netstat -tulpn | grep :30080

# Test connectivity
telnet YOUR-SERVER-IP 30080
```

#### 3. Database Connection Issues
```bash
# Check MongoDB status
kubectl logs mongodb-0 -n pets

# Check persistent volume
kubectl describe pvc mongodb-pvc -n pets

# Restart MongoDB if needed
kubectl delete pod mongodb-0 -n pets
```

#### 4. Message Queue Problems
```bash
# Check RabbitMQ logs
kubectl logs rabbitmq-0 -n pets

# Access RabbitMQ management UI
# http://YOUR-SERVER-IP:30092 (guest/guest)

# Check queue status
kubectl exec rabbitmq-0 -n pets -- rabbitmqctl list_queues
```

### Performance Issues

#### Resource Monitoring
```bash
# Check resource usage
kubectl top nodes
kubectl top pods -n pets

# Check resource limits
kubectl describe deployments -n pets | grep -A 3 -B 3 resources
```

#### Scaling Services
```bash
# Scale individual services
kubectl scale deployment order-service --replicas=2 -n pets
kubectl scale deployment product-service --replicas=2 -n pets

# Monitor scaling
kubectl get pods -n pets --watch
```

### Network Troubleshooting

#### Internal Connectivity
```bash
# Test service-to-service communication
kubectl run debug-pod --rm -i --tty --image=busybox -n pets -- /bin/sh

# Inside debug pod:
nslookup mongodb.pets.svc.cluster.local
wget -qO- http://order-service.pets.svc.cluster.local:3000/health
```

#### External Connectivity
```bash
# Check NodePort accessibility
sudo netstat -tulpn | grep :30080

# For Red Hat systems - Check firewalld
sudo firewall-cmd --list-ports
sudo firewall-cmd --list-all

# Verify firewall rules are active
sudo firewall-cmd --state

# Check if service is listening
curl -v http://YOUR-SERVER-IP:30080

# Red Hat specific - Check SELinux denials
sudo ausearch -m avc -ts recent | grep -i denied

# Temporarily allow all traffic (troubleshooting only)
sudo firewall-cmd --set-default-zone=trusted
# Remember to reset: sudo firewall-cmd --set-default-zone=public
```

## 🔒 Security Considerations

### Production Hardening
```bash
# Create security policies (example)
cat > pets-network-policy.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: pets-network-policy
  namespace: pets
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: pets
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: pets
EOF

# Apply security policies (adjust as needed)
# kubectl apply -f pets-network-policy.yaml
```

### RBAC Configuration
```bash
# Create service account and RBAC (example)
cat > pets-rbac.yaml << 'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: petstore-sa
  namespace: pets
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: petstore-role
  namespace: pets
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch"]
EOF

# kubectl apply -f pets-rbac.yaml
```

## 📈 Performance Tuning

### Resource Optimization
```bash
# Optimize resource requests/limits
cat > pets-resources-patch.yaml << 'EOF'
spec:
  template:
    spec:
      containers:
      - name: store-front
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
EOF

# Apply resource patches
kubectl patch deployment store-front -n pets --patch-file pets-resources-patch.yaml
```

### Horizontal Pod Autoscaler
```bash
# Enable autoscaling (requires metrics-server)
kubectl autoscale deployment order-service -n pets --cpu-percent=70 --min=1 --max=5
kubectl autoscale deployment product-service -n pets --cpu-percent=70 --min=1 --max=3

# Check autoscaler status
kubectl get hpa -n pets
```

## 🧹 Cleanup and Maintenance

### Complete Cleanup
```bash
# Delete entire pet store deployment
kubectl delete namespace pets

# Clean up service configuration file
rm -f pets-onprem-services.yaml
```

### Partial Cleanup
```bash
# Delete only applications (keep namespace)
kubectl delete deployment,statefulset,service,configmap,secret --all -n pets

# Or delete specific components
kubectl delete -f https://raw.githubusercontent.com/Azure-Samples/aks-store-demo/main/aks-store-all-in-one.yaml -n pets
```

### Update Deployment
```bash
# Update to latest version
kubectl delete -f https://raw.githubusercontent.com/Azure-Samples/aks-store-demo/main/aks-store-all-in-one.yaml -n pets
kubectl apply -f https://raw.githubusercontent.com/Azure-Samples/aks-store-demo/main/aks-store-all-in-one.yaml -n pets

# Rolling update individual services
kubectl rollout restart deployment/store-front -n pets
kubectl rollout status deployment/store-front -n pets
```

## 📚 Additional Configuration

### Monitoring Integration
```bash
# Add Prometheus monitoring (if available)
cat > pets-servicemonitor.yaml << 'EOF'
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: petstore-metrics
  namespace: pets
spec:
  selector:
    matchLabels:
      app: order-service
  endpoints:
  - port: http
EOF

# kubectl apply -f pets-servicemonitor.yaml
```

### Backup Configuration
```bash
# Backup persistent data
kubectl exec mongodb-0 -n pets -- mongodump --archive | gzip > mongodb-backup-$(date +%Y%m%d).gz

# Backup configuration
kubectl get all,configmap,secret,pvc -n pets -o yaml > pets-backup-$(date +%Y%m%d).yaml
```

## 📋 Quick Reference

### Essential Commands
```bash
# RHEL 9 Optimized Installation
# 1. Install prerequisites
sudo dnf install -y container-selinux selinux-policy-base container-tools podman
sudo dnf install -y https://rpm.rancher.io/k3s/latest/common/centos/9/noarch/k3s-selinux-1.4-1.el9.noarch.rpm

# 2. Configure firewall for RHEL 9
sudo firewall-cmd --permanent --add-port=30080-30092/tcp
sudo firewall-cmd --reload

# 3. Install and configure K3s
curl -sfL https://get.k3s.io | sh -
mkdir -p ~/.kube && sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

# 4. Deploy Pet Store
kubectl create namespace pets
kubectl apply -f https://raw.githubusercontent.com/Azure-Samples/aks-store-demo/main/aks-store-all-in-one.yaml -n pets
kubectl apply -f pets-onprem-services.yaml

# 5. Monitor and Access
kubectl get pods -n pets
kubectl get svc -n pets
kubectl logs <pod-name> -n pets

# 6. Get Access URLs
SERVER_IP=$(hostname -I | awk '{print $1}')
echo "🏪 Store Front: http://$SERVER_IP:30080"
echo "👨‍💼 Admin Interface: http://$SERVER_IP:30081"

# 7. RHEL 9 Specific Verification
sestatus  # Check SELinux status
cat /sys/fs/cgroup/cgroup.controllers  # Verify cgroup v2
sudo firewall-cmd --list-ports  # Verify firewall rules

# 8. Cleanup (when done)
kubectl delete namespace pets
```

### Port Reference
- **30080**: Store Front (Customer Interface)
- **30081**: Store Admin (Management Interface)
- **30092**: RabbitMQ Management (Optional)

### Default Credentials
- **RabbitMQ**: guest / guest (if management UI enabled)

---

## 🎉 Success!

If all steps completed successfully, you now have a fully functional e-commerce platform running on your on-premises Kubernetes cluster! The Pet Store demonstrates modern microservices architecture, event-driven design, and cloud-native deployment patterns.

**Next Steps:**
- Explore the customer shopping experience
- Try the administrative interface
- Monitor the automated simulation
- Consider integrating with your existing monitoring stack
- Plan for production hardening and security

For questions or issues, refer to the troubleshooting section above or consult the [original AKS Store Demo documentation](https://learn.microsoft.com/en-us/samples/azure-samples/aks-store-demo/aks-store-demo/).

---

*This guide provides comprehensive instructions for deploying the AKS Pet Store on any on-premises Kubernetes environment. Adapt the networking and storage configurations to match your specific infrastructure setup.*
