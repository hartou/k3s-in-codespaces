# AKS Pet Store On-Premises Deployment Guide

> **Version**: 1.0
> **Target Environment**: On-Premises Kubernetes / Local Servers
> **Application**: Microsoft AKS Store Demo (Pet Store)
> **Compatibility**: Any Kubernetes distribution (K3s, K8s, MicroK8s, etc.)

## 🎯 Overview

This guide provides step-by-step instructions for deploying the Microsoft AKS Pet Store demo application on your on-premises Kubernetes cluster. The Pet Store is a comprehensive microservices e-commerce platform demonstrating modern cloud-native architecture patterns.

### What You'll Deploy

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
```bash
# Allow NodePort range (adjust for your firewall)
sudo ufw allow 30000:32767/tcp

# Or specific ports we'll use:
sudo ufw allow 30080/tcp  # Store Front
sudo ufw allow 30081/tcp  # Store Admin
sudo ufw allow 30092/tcp  # RabbitMQ Management (optional)
```

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

# Verify firewall rules
sudo ufw status

# Check if service is listening
curl -v http://YOUR-SERVER-IP:30080
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
# Deployment
kubectl create namespace pets
kubectl apply -f https://raw.githubusercontent.com/Azure-Samples/aks-store-demo/main/aks-store-all-in-one.yaml -n pets
kubectl apply -f pets-onprem-services.yaml

# Monitoring
kubectl get pods -n pets
kubectl get svc -n pets
kubectl logs <pod-name> -n pets

# Access
SERVER_IP=$(hostname -I | awk '{print $1}')
echo "Store Front: http://$SERVER_IP:30080"
echo "Admin Interface: http://$SERVER_IP:30081"

# Cleanup
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
