# K3s Deployment Checklist

## Quick Deployment Checklist ✅

### Prerequisites
- [ ] Docker Desktop running (4+ GB RAM allocated)
- [ ] VS Code with Dev Containers extension
- [ ] GitHub Codespace with 4-core machine type

### Automated Deployment (Codespaces)
- [ ] Create new Codespace from repository
- [ ] Wait for container build (~2-3 minutes)
- [ ] Wait for cluster creation (~2-3 minutes)
- [ ] Verify with `kic pods`
- [ ] Validate with `kic check all`

### Manual Deployment Steps
- [ ] Set environment variables (`REPO_BASE`, `PATH`)
- [ ] Create Docker network (`docker network create k3d`)
- [ ] Start local registry (`k3d registry create registry.localhost --port 5500`)
- [ ] Pull base images (dotnet, webv-red)
- [ ] Clone application repos (imdb-app, webvalidate)
- [ ] Restore dependencies (`dotnet restore`)
- [ ] Create k3d cluster (`kic cluster create`)
- [ ] Build applications (`kic build imdb`, `kic build webv`)
- [ ] Deploy infrastructure (`kubectl apply -f deploy/bootstrap/`)
- [ ] Deploy applications (`kubectl apply -f deploy/apps/`)
- [ ] Deploy jumpbox (`kic cluster jumpbox`)

### Verification Steps
- [ ] All pods running (`kic pods`)
- [ ] Services accessible (`kic svc`)
- [ ] Endpoints responding (`kic check all`)
- [ ] Web UIs accessible:
  - [ ] IMDb API: <http://localhost:30080>
  - [ ] Heartbeat: <http://localhost:31080>
  - [ ] Prometheus: <http://localhost:30000>
  - [ ] Grafana: <http://localhost:32000> (admin/cse-labs)

### Testing
- [ ] Integration tests pass (`kic test integration`)
- [ ] Load tests complete (`kic test load`)
- [ ] Grafana dashboards show data
- [ ] Jumpbox access works (`kj`, `kje`)

### Troubleshooting Commands
- [ ] Check events: `kic events`
- [ ] View logs: `k9s` → select pod → 'l'
- [ ] Describe resources: `kubectl describe pod <name> -n <namespace>`
- [ ] Rebuild if needed: `kic cluster rebuild`

## Port Reference 🌐
- **30000**: Prometheus (Metrics)
- **30080**: IMDb API (Application)
- **31080**: Heartbeat (Health Checks)
- **32000**: Grafana (Dashboards)

## Essential Commands 🔧
```bash
# Cluster
kic cluster create|deploy|rebuild|delete

# Apps
kic build imdb|webv
kic pods|svc|ns|events
kic check all

# Testing
kic test integration|load

# Debug
kj        # jumpbox shell
kje       # run command in jumpbox
k9s       # kubernetes dashboard
```

## Recovery Commands 🔄
```bash
# Full reset
kic cluster rebuild

# Clean apps only
kic cluster clean
kic cluster deploy

# Complete cleanup
kic cluster delete
docker system prune -f
```
