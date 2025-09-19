# K3s Deployment Checklist (Updated from Testing)

## Quick Deployment Checklist ✅

### Prerequisites

- [ ] Docker Desktop running (4+ GB RAM allocated)
- [ ] VS Code with Dev Containers extension
- [ ] GitHub Codespace with 4-core machine type

### Automated Deployment (Codespaces) - ~5 minutes total

- [ ] Create new Codespace from repository
- [ ] Wait for container build (~2-3 minutes)
- [ ] Wait for cluster creation (~2-3 minutes)
- [ ] Verify with `kic pods`
- [ ] Validate with `kic check all`

### Manual Deployment Steps

**Environment Setup:**
- [ ] Set environment variables (`export REPO_BASE=$(pwd)` and `export PATH="$PATH:$REPO_BASE/cli"`)
- [ ] Create Docker network (`docker network create k3d`)
- [ ] Start local registry (`k3d registry create registry.localhost --port 5500`)
- [ ] Connect registry to network (`docker network connect k3d k3d-registry.localhost`)
- [ ] Pull base images (`docker pull mcr.microsoft.com/dotnet/aspnet:6.0-alpine`, etc.)

**Application Preparation:**
- [ ] Clone application repos:
  - [ ] `git clone https://github.com/cse-labs/imdb-app /workspaces/imdb-app`
  - [ ] `git clone https://github.com/microsoft/webvalidate /workspaces/webvalidate`
- [ ] Restore dependencies:
  - [ ] `dotnet restore /workspaces/webvalidate/src/webvalidate.sln`
  - [ ] `dotnet restore /workspaces/imdb-app/src/imdb.csproj`

**Cluster and Applications:**
- [ ] Create k3d cluster (`kic cluster create`)
- [ ] Verify cluster (`kubectl cluster-info`)
- [ ] Build applications (`kic build imdb`, `kic build webv`)
- [ ] Deploy infrastructure (`kubectl apply -f deploy/bootstrap/`)
- [ ] Deploy applications (`kubectl apply -f deploy/apps/`)
- [ ] Deploy jumpbox (`kic cluster jumpbox`)

### Known Issues and Fixes

- [ ] **WebV CrashLoopBackOff**: Check if WebV deployment has invalid `--prometheus` argument
  - If yes, remove it from `deploy/apps/webv/webv.yaml` and redeploy:
  - `kubectl apply -f deploy/apps/webv/webv.yaml`
  - `kubectl rollout restart deployment/webv -n imdb`

### Verification Steps - ~2 minutes

- [ ] All pods running (`kic pods`) - Should show all pods as 1/1 Running
- [ ] Services accessible (`kic svc`) - Should show NodePort services
- [ ] Endpoints responding (`kic check all`) - All should return HTTP 200/302
- [ ] Web UIs accessible:
  - [ ] IMDb API: <http://localhost:30080> (Swagger UI)
  - [ ] Heartbeat: <http://localhost:31080/heartbeat/17> (should return hex string)
  - [ ] Prometheus: <http://localhost:30000> (redirects to /graph)
  - [ ] Grafana: <http://localhost:32000> (admin/cse-labs)

### Testing - ~35 seconds total

- [ ] Integration tests pass (`kic test integration`) - ~5 seconds, generates 400/404 errors by design
- [ ] Load tests complete (`kic test load`) - ~30 seconds, shows real-time metrics
- [ ] Grafana dashboards show data (check IMDb dashboard for load test results)
- [ ] Jumpbox access works:
  - [ ] `kj` for interactive shell
  - [ ] `kje http imdb.imdb.svc.cluster.local:8080/version` for single commands

### Troubleshooting Commands

**Diagnostics:**
- [ ] Check events: `kic events`
- [ ] View logs: `k9s` → select pod → 'l'
- [ ] Describe resources: `kubectl describe pod <name> -n <namespace>`
- [ ] Check pod status: `kubectl get pods -A`
- [ ] View service endpoints: `kubectl get svc -A`

**Common Fixes:**
- [ ] Pod stuck in CrashLoopBackOff: Check logs and deployment args
- [ ] Service not responding: Verify port forwarding and service configuration
- [ ] Rebuild if needed: `kic cluster rebuild`

## Port Reference 🌐

- **30000**: Prometheus (Metrics)
- **30080**: IMDb API (Application) 
- **31080**: Heartbeat (Health Checks)
- **32000**: Grafana (Dashboards)

## Essential Commands 🔧

```bash
# Cluster Management
kic cluster create          # Create cluster (~2 minutes)
kic cluster deploy          # Deploy applications (~1 minute)
kic cluster rebuild         # Full rebuild (~3 minutes)
kic cluster delete          # Delete cluster (~30 seconds)

# Application Management  
kic build imdb             # Build IMDb app (~1 minute)
kic build webv             # Build WebValidate (~1 minute)
kic pods                   # List all pods
kic svc                    # List all services
kic check all              # Validate endpoints (~10 seconds)

# Testing
kic test integration       # Integration tests (~5 seconds)
kic test load              # Load tests (~30 seconds)

# Debugging
kj                         # jumpbox shell (interactive)
kje <command>              # run command in jumpbox
k9s                        # kubernetes dashboard
```

## Recovery Commands 🔄

```bash
# Full reset (preserves images)
kic cluster rebuild

# Clean apps only (~1 minute)
kic cluster clean
kic cluster deploy

# Complete cleanup (~2 minutes)
kic cluster delete
docker system prune -f
docker volume prune -f
```

## Deployment Timing ⏱️

**Full Manual Deployment from scratch:**
- Environment setup: ~2 minutes
- Application cloning and restore: ~3 minutes  
- Cluster creation: ~2 minutes
- Application builds: ~2 minutes
- Deployment: ~1 minute
- **Total: ~10 minutes**

**Quick operations:**
- Verification: ~2 minutes
- Testing: ~35 seconds
- Troubleshooting: ~1-5 minutes depending on issue

## Success Criteria ✅

**Deployment is successful when:**
- [ ] All pods show 1/1 Running status
- [ ] `kic check all` returns successful responses for all services
- [ ] All web UIs are accessible and responsive
- [ ] Integration tests complete without connection errors
- [ ] Load tests run for full 30 seconds
- [ ] Jumpbox commands work correctly
- [ ] Grafana shows live metrics during load tests

## Lessons Learned 📝

1. **WebV Issue**: The original deployment had an invalid `--prometheus` argument that caused CrashLoopBackOff
2. **Testing Order**: Run verification before testing to ensure all services are healthy
3. **Load Test Duration**: Load tests run for exactly 30 seconds as configured
4. **Grafana Data**: Live metrics appear immediately during load testing
5. **Recovery**: `kic cluster rebuild` is the fastest way to reset everything