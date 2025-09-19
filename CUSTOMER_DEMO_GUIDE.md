# Customer Demo Guide - K3s in Codespaces

## 🎯 Demo Overview

This is a complete Kubernetes development environment running in GitHub Codespaces with:
- **Full k3s cluster** with real applications
- **Production-grade observability** (Prometheus + Grafana)
- **Sample microservices** (IMDb API + Load Testing)
- **Complete monitoring stack** with real-time metrics

## 🚀 Quick Access URLs

### 1. IMDb Movie Database API - **Primary Application**
**URL**: http://localhost:30080
- **What to Show**: Full-featured REST API with Swagger UI
- **Demo Points**:
  - Interactive API documentation
  - Search movies, actors, genres
  - Real database with thousands of movies
  - Production-ready .NET microservice

### 2. Grafana Dashboards - **Monitoring & Visualization**
**URL**: http://localhost:32000
- **Login**: admin / cse-labs
- **What to Show**:
  - Real-time application metrics
  - Beautiful dashboards showing request patterns
  - Performance monitoring
  - .NET runtime metrics
- **Demo Tip**: Run load test to show live metrics

### 3. Prometheus Metrics - **Metrics Collection**
**URL**: http://localhost:30000
- **What to Show**:
  - Raw metrics collection system
  - Query interface for custom metrics
  - System and application health data

### 4. Heartbeat Health Service
**URL**: http://localhost:31080/heartbeat/17
- **What to Show**: Simple health monitoring service
- **Demo Points**: Returns health check data

## 🎬 Demo Script for Customer

### Part 1: The Application (5 minutes)
1. **Open IMDb API**: http://localhost:30080
   ```
   "This is a complete movie database API running in Kubernetes.
   It has thousands of movies, actors, and provides full REST endpoints."
   ```

2. **Show Interactive Documentation**:
   - Click on any API endpoint (e.g., `/api/movies`)
   - Click "Try it out" → "Execute"
   - Show real data being returned

3. **Demonstrate Search**:
   - Try `/api/movies` with query parameter `q=matrix`
   - Try `/api/actors` with query parameter `q=keanu`

### Part 2: Real-Time Monitoring (5 minutes)
1. **Open Grafana**: http://localhost:32000 (admin/cse-labs)
   ```
   "This is production-grade monitoring. Every request to our API
   is being tracked in real-time."
   ```

2. **Show Default Dashboard**:
   - Point out request rates, response times, error rates
   - Show beautiful visualizations

3. **Generate Load** (in terminal):
   ```bash
   kic test load
   ```
   - Switch back to Grafana
   - Show metrics changing in real-time
   - Demonstrate the spike in traffic

### Part 3: The Infrastructure (3 minutes)
1. **Show Kubernetes Cluster**:
   ```bash
   kic pods
   ```
   ```
   "This entire environment is running in Kubernetes with
   multiple microservices, monitoring, and logging."
   ```

2. **Show Services**:
   ```bash
   kic svc
   ```

3. **Open Prometheus**: http://localhost:30000
   - Show the metrics collection system
   - Query for custom metrics like `ImdbAppDuration_bucket`

## 🔥 Key Selling Points to Emphasize

### 1. **Complete Production Environment**
- "This isn't just a demo - this is a real production-ready environment"
- "Full Kubernetes cluster with monitoring, logging, and applications"
- "Everything you see here can run in production"

### 2. **Developer Experience**
- "One click in GitHub Codespaces gets you this entire environment"
- "No local setup, no 'it works on my machine' issues"
- "Same environment for every developer"

### 3. **Real Observability**
- "This is enterprise-grade monitoring with Prometheus and Grafana"
- "Real-time metrics, alerting, dashboards"
- "See exactly what's happening in your applications"

### 4. **Cloud-Native Best Practices**
- "Health checks, proper resource limits, service discovery"
- "12-factor app patterns, containerization"
- "Ready for production deployment"

## 🎯 Interactive Demo Activities

### Activity 1: API Testing
1. Open http://localhost:30080
2. Navigate to `/api/movies` endpoint
3. Click "Try it out"
4. Add query parameter: `q=star wars`
5. Execute and show results

### Activity 2: Real-Time Monitoring
1. Open Grafana: http://localhost:32000 (admin/cse-labs)
2. Show baseline metrics
3. Run in terminal: `kic test load`
4. Watch metrics spike in real-time
5. Show request volume, response times, error rates

### Activity 3: System Exploration
1. Show cluster status: `kic pods`
2. Show all services: `kic svc`
3. Test individual endpoints: `kic check all`
4. Show logs with k9s: `k9s` (optional for technical audience)

## 🚨 Demo Tips

### Before Starting
- Ensure all services are running: `kic check all`
- Have all browser tabs ready
- Test the load test command once

### During Demo
- **Start with the application** - show business value first
- **Move to monitoring** - show operational excellence
- **End with infrastructure** - show technical sophistication
- **Be ready for questions** about scalability, security, deployment

### Troubleshooting
- If any service is down: `kic cluster rebuild`
- If ports aren't accessible: Check Codespaces port forwarding
- If Grafana login fails: Try admin/admin or admin/cse-labs

## 🎨 What Makes This Special

1. **Zero Setup Time**: "From zero to full environment in 60 seconds"
2. **Production Patterns**: "This is how Netflix, Google, Amazon run their services"
3. **Developer Productivity**: "Focus on code, not infrastructure"
4. **Cost Effective**: "Pay only when developing, not 24/7 infrastructure"
5. **Scalable**: "Every developer gets their own isolated environment"

## 📊 Technical Highlights

- **Kubernetes**: Real k3s cluster with multiple namespaces
- **Observability**: Prometheus metrics + Grafana dashboards
- **Applications**: .NET Core microservices with proper health checks
- **Networking**: Service discovery, load balancing, ingress
- **Storage**: Persistent volumes for data
- **Security**: RBAC, network policies, resource limits

## 🎤 Closing Points

"This represents the future of software development - cloud-native, observable, and instantly available. Your developers can focus on building features instead of fighting with infrastructure."

---

**Ready to demo!** All services are running and accessible. Start with the IMDb API to show business value, then move to Grafana to show operational excellence!
