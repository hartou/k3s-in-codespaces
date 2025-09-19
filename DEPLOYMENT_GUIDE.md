# K3s Cluster Deployment Instructions

> **Version**: 1.0
> **Target Environment**: GitHub Codespaces / Dev Containers
> **Kubernetes Distribution**: k3d (k3s in Docker)

## Overview

This document provides comprehensive instructions for deploying a complete k3s cluster with observability stack, sample applications, and development tools. The deployment is designed for development, learning, and testing purposes.

## 🏗️ Architecture Components

### Core Infrastructure

- **k3d Cluster**: Single-node Kubernetes cluster
- **Local Registry**: Container registry at `k3d-registry.localhost:5500`
- **Networking**: Custom k3d network with port forwarding
- **Storage**: Local volumes for Prometheus and Grafana data

### Application Stack

- **IMDb API**: Movie database REST API with Swagger UI
- **Heartbeat Service**: Health monitoring and validation service
- **WebV (WebValidate)**: Load testing and validation tool
- **Jumpbox**: Debugging pod with networking tools

### Observability Stack

- **Prometheus**: Metrics collection and alerting
- **Grafana**: Visualization dashboards
- **Fluent Bit**: Log aggregation and forwarding

## 🚀 Quick Start (Automated Deployment)

### Option 1: GitHub Codespaces (Recommended)

1. **Create Codespace**

   ```bash
   # Navigate to: https://github.com/your-org/k3s-in-codespaces
   # Click: Code > Codespaces > New Codespace
   # Select: 4-core machine type
   ```

2. **Wait for Automated Setup**
   - Container build and initialization (~2-3 minutes)
   - Cluster creation and app deployment (~2-3 minutes)
   - Total setup time: ~5 minutes

3. **Verify Deployment**

   ```bash
   # Check cluster status
   kic pods

   # Validate all services
   kic check all
   ```

### Option 2: Local Dev Container

1. **Prerequisites**

   ```bash
   # Required tools
   - Docker Desktop (with 4+ GB RAM allocated)
   - VS Code with Dev Containers extension
   - Git
   ```

2. **Clone and Open**

   ```bash
   git clone https://github.com/your-org/k3s-in-codespaces.git
   cd k3s-in-codespaces
   code .
   # VS Code will prompt to "Reopen in Container" - click Yes
   ```

## 🔧 Manual Deployment (Step-by-Step)

### Phase 1: Environment Preparation

1. **Set Environment Variables**

   ```bash
   export REPO_BASE=$(pwd)
   export PATH="$PATH:$REPO_BASE/cli"
   ```

2. **Create Docker Network**

   ```bash
   docker network create k3d
   ```

3. **Start Local Registry**

   ```bash
   k3d registry create registry.localhost --port 5500
   docker network connect k3d k3d-registry.localhost
   ```

4. **Pull Base Images**

   ```bash
   docker pull mcr.microsoft.com/dotnet/aspnet:6.0-alpine
   docker pull mcr.microsoft.com/dotnet/sdk:6.0
   docker pull ghcr.io/cse-labs/webv-red:latest
   ```

### Phase 2: Cluster Creation

1. **Create k3d Cluster**

   ```bash
   # Using kic CLI (recommended)
   kic cluster create

   # OR using k3d directly
   k3d cluster create --config .devcontainer/k3d.yaml
   ```

2. **Verify Cluster**

   ```bash
   kubectl cluster-info
   kubectl get nodes
   ```

### Phase 3: Application Preparation

1. **Clone Application Repositories**

   ```bash
   # IMDb Application
   git clone https://github.com/cse-labs/imdb-app /workspaces/imdb-app

   # WebValidate Tool
   git clone https://github.com/microsoft/webvalidate /workspaces/webvalidate
   ```

2. **Restore Dependencies**

   ```bash
   dotnet restore /workspaces/webvalidate/src/webvalidate.sln
   dotnet restore /workspaces/imdb-app/src/imdb.csproj
   ```

### Phase 4: Build Applications

1. **Build IMDb Application**

   ```bash
   kic build imdb
   ```

2. **Build WebValidate**

   ```bash
   # Modify Dockerfile to skip tests during build
   sed -i "s/RUN dotnet test//g" /workspaces/webvalidate/Dockerfile
   kic build webv
   ```

### Phase 5: Deploy Infrastructure

1. **Create Namespaces**

   ```bash
   kubectl apply -f deploy/bootstrap/namespaces.yaml
   ```

2. **Deploy Fluent Bit (Logging)**

   ```bash
   kubectl apply -f deploy/bootstrap/fluentbit/
   ```

3. **Deploy Prometheus (Metrics)**

   ```bash
   kubectl apply -f deploy/bootstrap/prometheus/
   ```

4. **Deploy Grafana (Visualization)**

   ```bash
   kubectl apply -f deploy/bootstrap/grafana/
   ```

5. **Deploy Heartbeat Service**

   ```bash
   kubectl apply -f deploy/bootstrap/heartbeat/
   kubectl apply -f deploy/bootstrap/webv-heartbeat/
   ```

### Phase 6: Deploy Applications

1. **Deploy IMDb Application**

   ```bash
   kubectl apply -f deploy/apps/namespace.yaml
   kubectl apply -f deploy/apps/imdb/
   ```

2. **Deploy WebV (Load Testing)**

   ```bash
   kubectl apply -f deploy/apps/webv/
   ```

3. **Deploy Jumpbox**

   ```bash
   kic cluster jumpbox
   ```

## 🔍 Verification and Testing

### 1. Check Cluster Status

```bash
# List all pods
kic pods

# Check services
kic svc

# View namespaces
kic ns

# Check events
kic events
```

### 2. Validate Endpoints

```bash
# Check all service endpoints
kic check all

# Individual service checks
curl http://localhost:30080/healthz    # IMDb Health
curl http://localhost:31080/heartbeat/17  # Heartbeat
curl http://localhost:30000/metrics   # Prometheus
curl http://localhost:32000/          # Grafana
```

### 3. Run Integration Tests

```bash
# Integration tests (will generate some 400/404 errors by design)
kic test integration

# Load tests (30 seconds of load)
kic test load
```

### 4. Access Web UIs

| Service    | URL                          | Credentials          |
|------------|------------------------------|---------------------|
| IMDb API   | <http://localhost:30080>     | None (Swagger UI)   |
| Heartbeat  | <http://localhost:31080>     | None                |
| Prometheus | <http://localhost:30000>     | None                |
| Grafana    | <http://localhost:32000>     | admin / cse-labs    |

## 🛠️ Development Workflow

### Code-Build-Test Cycle

1. **Make Code Changes**

   ```bash
   # Edit application code in /workspaces/imdb-app/
   ```

2. **Rebuild and Deploy**

   ```bash
   kic build imdb
   ```

3. **Test Changes**

   ```bash
   # Quick health check
   curl http://localhost:30080/version

   # Full integration test
   kic test integration
   ```

### Debugging

1. **Access Jumpbox**

   ```bash
   # Interactive shell in cluster
   kj

   # Run single command in jumpbox
   kje http imdb.imdb.svc.cluster.local:8080/version
   ```

2. **View Logs with k9s**

   ```bash
   k9s
   # Press '0' to show all namespaces
   # Navigate to pods and press 'l' to view logs
   ```

3. **Monitor with Grafana**
   - Open <http://localhost:32000>
   - Default dashboard shows IMDb application metrics
   - View request patterns, error rates, and performance

## 🔧 Troubleshooting

### Common Issues and Solutions

1. **Cluster Creation Fails**

   ```bash
   # Delete and recreate
   kic cluster delete
   kic cluster create
   ```

2. **Port Conflicts**

   ```bash
   # Check what's using the ports
   netstat -tulpn | grep -E ':(30000|30080|31080|32000)'

   # Kill conflicting processes or change ports in k3d.yaml
   ```

3. **Image Pull Errors**

   ```bash
   # Rebuild local registry
   docker rm -f k3d-registry.localhost
   k3d registry create registry.localhost --port 5500
   docker network connect k3d k3d-registry.localhost
   ```

4. **Application Not Responding**

   ```bash
   # Check pod status
   kubectl get pods -A

   # Describe failing pods
   kubectl describe pod <pod-name> -n <namespace>

   # View pod logs
   kubectl logs <pod-name> -n <namespace>
   ```

### Resource Issues

1. **Insufficient Memory**

   ```bash
   # Check resource usage
   kubectl top nodes
   kubectl top pods -A

   # Restart cluster with more resources
   kic cluster rebuild
   ```

2. **Storage Issues**

   ```bash
   # Check persistent volumes
   kubectl get pv,pvc -A

   # Clean up if needed
   kubectl delete pvc --all -A
   ```

## 🧹 Cleanup and Reset

### Full Cluster Reset

```bash
# Complete cluster rebuild (preserves container images)
kic cluster rebuild
```

### Selective Cleanup

```bash
# Remove applications only
kic cluster clean

# Redeploy applications
kic cluster deploy
```

### Complete Cleanup

```bash
# Delete everything
kic cluster delete
docker system prune -f
docker volume prune -f
```

## 🔒 Security Considerations

### Development Environment Security

- **Non-Production**: This setup is for development only
- **Privileged Containers**: Required for Docker-in-Docker
- **No TLS**: Internal services use HTTP for simplicity
- **Default Credentials**: Change Grafana password for sensitive use

### Network Security

- **Isolated Network**: Uses custom Docker network
- **Port Forwarding**: Only necessary ports exposed
- **Service Mesh Ready**: Prepared for Istio/Linkerd integration

## 📊 Monitoring and Observability

### Prometheus Metrics

- **Application Metrics**: Custom IMDb application metrics
- **System Metrics**: Kubernetes cluster health
- **Performance Metrics**: Request latency and throughput

### Grafana Dashboards

- **IMDb Application**: Request patterns, errors, performance
- **.NET Runtime**: Memory, GC, threading metrics
- **Custom Dashboards**: Add your own via Grafana UI

### Log Management

- **Fluent Bit**: Centralized log collection
- **Structured Logs**: JSON formatted application logs
- **Log Forwarding**: Ready for external log systems

## 🔄 CI/CD Integration

### Automated Testing

```bash
# Add to your pipeline
kic cluster create
kic build imdb
kic build webv
kic cluster deploy
kic test integration
kic test load
```

### Multi-Environment Support

- **Branch Deployments**: Each branch can have its own cluster
- **PR Previews**: Deploy applications per pull request
- **Integration Testing**: Automated validation in pipelines

## 📚 Additional Resources

### Documentation

- [Original README](./README.md) - Detailed usage instructions
- [Project Analysis](./PROJECT_ANALYSIS_CLEAN.md) - Comprehensive project overview
- [curl.http](./curl.http) - REST API examples

### Community and Support

- **GitHub Discussions**: Feature requests and questions
- **Issues**: Bug reports and troubleshooting
- **CSE-Labs**: Microsoft Customer Success Engineering resources

---

## 📋 Quick Reference

### Essential Commands

```bash
# Cluster Management
kic cluster create          # Create cluster
kic cluster deploy          # Deploy applications
kic cluster rebuild         # Full rebuild
kic cluster delete          # Delete cluster

# Application Management
kic build imdb             # Build IMDb app
kic build webv             # Build WebValidate
kic pods                   # List all pods
kic svc                    # List all services
kic check all             # Validate endpoints

# Testing
kic test integration      # Integration tests
kic test load            # Load tests

# Debugging
kj                       # Jumpbox shell
kje <command>           # Run command in jumpbox
k9s                     # Kubernetes dashboard
```

### Port Reference

- **30000**: Prometheus (Metrics)
- **30080**: IMDb API (Application)
- **31080**: Heartbeat (Health Checks)
- **32000**: Grafana (Dashboards)

---

*This deployment guide provides a complete reference for setting up and managing the k3s cluster environment. For questions or issues, please refer to the project's GitHub Discussions or Issues.*
