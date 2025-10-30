# Complete Integration: DevSecOps Pipeline

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                 DEVELOPER WORKFLOW                           │
└─────────────────────────────────────────────────────────────┘

1. CODE REPOSITORY (Gitea)
   ├─ Source code
   ├─ Kubernetes manifests
   ├─ CI/CD workflows
   └─ Documentation

         ↓ (git push)

2. GITEA ACTIONS (CI/CD)
   ├─ Checkout code
   ├─ SonarQube analysis
   ├─ Dependency scanning
   ├─ Build Docker image
   ├─ Trivy scan
   ├─ Push to Nexus
   ├─ Update manifests
   └─ Commit back to Gitea

         ↓ (detect changes)

3. NEXUS REGISTRY
   ├─ Container images
   ├─ Image tags/versions
   ├─ Vulnerability data
   └─ Storage management

         ↓ (image digest)

4. KUBERNETES + ArgoCD
   ├─ Pull image from Nexus
   ├─ Deploy pods
   ├─ Monitor health
   └─ Auto-sync on changes

         ↓ (running state)

5. MONITORING
   ├─ Prometheus metrics
   ├─ Grafana dashboards
   ├─ Application logs
   └─ Alerting

         ↑ (continuous feedback loop)

6. DEVELOPER NOTIFICATIONS
   ├─ Build status
   ├─ Deployment status
   ├─ Security alerts
   └─ Performance metrics
```

---

## Step 1: Initial Setup (30 minutes)

### 1.1 Start All Services

```bash
mkdir -p ~/devsecops-pipeline && cd ~/devsecops-pipeline

# Copy docker-compose.yml and all other configuration files
# From the provided artifacts

# Start services
docker-compose up -d

# Wait for initialization
sleep 60

# Verify all services
docker-compose ps
```

### 1.2 Verify Service Access

```bash
# Test connectivity
for service in gitea nexus sonarqube prometheus grafana; do
  echo "Testing $service..."
  case $service in
    gitea)
      curl -s http://localhost:3000 > /dev/null && echo "✓ Gitea" || echo "✗ Gitea"
      ;;
    nexus)
      curl -s http://localhost:8081/nexus/ > /dev/null && echo "✓ Nexus" || echo "✗ Nexus"
      ;;
    sonarqube)
      curl -s http://localhost:9000 > /dev/null && echo "✓ SonarQube" || echo "✗ SonarQube"
      ;;
    prometheus)
      curl -s http://localhost:9090 > /dev/null && echo "✓ Prometheus" || echo "✗ Prometheus"
      ;;
    grafana)
      curl -s http://localhost:3001 > /dev/null && echo "✓ Grafana" || echo "✗ Grafana"
      ;;
  esac
done
```

---

## Step 2: Gitea Configuration (15 minutes)

### 2.1 Initial Setup

```bash
# Access Gitea UI
# http://localhost:3000

# First access shows installation page
# Fill in database and admin credentials:
# Admin Username: gitea_admin
# Admin Password: admin
# Admin Email: admin@devsecops.local
# Site Title: DevSecOps Pipeline
# Repository Root Path: /data/gitea-repositories
# Click "Install Gitea"
```

### 2.2 Create Repository

```bash
# Via Gitea Web UI:
# 1. Click "+" → New Repository
# 2. Repository Name: devsecops-demo
# 3. Description: DevSecOps Pipeline Demo
# 4. Private/Public: Public
# 5. Initialize: Check "Initialize Repository"
# 6. Create Repository

# Via command line:
cd ~/devsecops-pipeline
git init
git remote add origin http://localhost:3000/gitea_admin/devsecops-demo.git
```

### 2.3 Push Initial Code

```bash
# Copy project files to repository
# app/
# k8s/
# .gitea/
# etc.

# Push to Gitea
git add .
git commit -m "Initial commit: DevSecOps Pipeline Setup"
git push -u origin main
# Username: gitea_admin
# Password: admin
```

### 2.4 Enable Actions

```bash
# Via Gitea Web UI:
# 1. Login as gitea_admin
# 2. Administration (wrench icon)
# 3. Configuration → Actions
# 4. Enable: ✓ "Enable Actions"
# 5. Click "Save"
```

---

## Step 3: Nexus Configuration (20 minutes)

### 3.1 Initial Login

```bash
# Get Nexus password
NEXUS_PASS=$(docker exec devsecops-nexus cat /nexus-data/admin.password)
echo "Nexus admin password: $NEXUS_PASS"

# Access Nexus
# http://localhost:8081/nexus
# Login: admin / $NEXUS_PASS
```

### 3.2 Change Admin Password

```bash
# Via Nexus Web UI:
# 1. Sign in with initial password
# 2. Click profile icon (top right)
# 3. Click "Set Password"
# 4. Enter new password: admin123
# 5. Confirm
# 6. Click "Change password"
```

### 3.3 Create Docker Hosted Repository

```bash
# Via Nexus Web UI:
# 1. Administration → Repositories
# 2. Create Repository
# 3. Recipe: docker (hosted)
# 4. Repository Name: docker-hosted
# 5. HTTP Port: 8082
# 6. Enable: Docker API, Allow redeploy
# 7. Blob Store: default
# 8. Create Repository
```

### 3.4 Configure Docker Authentication

```bash
# Test Docker login
docker login localhost:8082
# Username: admin
# Password: admin123

# Verify config
cat ~/.docker/config.json | jq '.auths."localhost:8082"'
```

---

## Step 4: GitHub Actions Workflow Setup (10 minutes)

### 4.1 Create Workflow File

```bash
# In your local repository
mkdir -p .gitea/workflows
cat > .gitea/workflows/ci-cd.yml << 'EOF'
# (Copy complete workflow from the ci-cd.yml artifact)
EOF

git add .gitea/workflows/ci-cd.yml
git commit -m "ci: Add Gitea Actions CI/CD workflow"
git push origin main
```

### 4.2 Add Repository Secrets

```bash
# Via Gitea Web UI:
# 1. Repository → Settings → Secrets
# 2. Add NEXUS_URL: nexus:8082
# 3. Add NEXUS_USERNAME: admin
# 4. Add NEXUS_PASSWORD: admin123
# 5. Add SONAR_HOST_URL: http://sonarqube:9000
# 6. Add GITEA_TOKEN: (generate from Gitea user settings)
```

### 4.3 Generate Gitea Token

```bash
# Via Gitea Web UI:
# 1. Profile → Settings → Applications
# 2. Manage Access Tokens
# 3. Generate New Token
# 4. Token Name: ci-automation
# 5. Scopes: repo, write:repository
# 6. Generate Token
# 7. Copy token to Gitea secrets as GITEA_TOKEN
```

---

## Step 5: First Pipeline Execution (5 minutes)

### 5.1 Trigger Pipeline

```bash
# Make a simple change to trigger pipeline
echo "# Test" >> README.md
git add README.md
git commit -m "test: trigger CI/CD pipeline"
git push origin main
```

### 5.2 Monitor Pipeline

```bash
# Via Gitea Web UI:
# 1. Repository → Actions
# 2. Click latest workflow run
# 3. Monitor real-time execution

# View logs for each step:
# Click step name → View logs
```

### 5.3 Verify Artifacts

```bash
# After successful pipeline:

# 1. Check image in Nexus
curl -u admin:admin123 \
  http://localhost:8081/service/rest/v1/components?repository=docker-hosted

# 2. Verify K8s manifest updated
git pull origin main
grep "image:" k8s/app-deployment.yaml

# 3. Check SonarQube analysis
curl http://localhost:9000/api/measures/search_history?component=python-app
```

---

## Step 6: Kubernetes & ArgoCD Deployment (20 minutes)

### 6.1 Enable Kubernetes in Docker Desktop

```bash
# Settings → Kubernetes → Enable Kubernetes
# Wait for cluster initialization (5-10 minutes)

# Verify
kubectl cluster-info
kubectl get nodes
```

### 6.2 Create Production Namespace

```bash
kubectl create namespace production

# Label namespace for ArgoCD
kubectl label namespace production argocd.argoproj.io/managed-by=argocd
```

### 6.3 Create Image Pull Secret

```bash
kubectl create secret docker-registry nexus-credentials \
  --docker-server=nexus:8082 \
  --docker-username=admin \
  --docker-password=admin123 \
  -n production

# Verify
kubectl get secret nexus-credentials -n production -o yaml
```

### 6.4 Deploy Application

```bash
# Apply K8s manifests
kubectl apply -f k8s/

# Verify deployment
kubectl get deployment -n production
kubectl get pods -n production
kubectl get svc -n production

# Port-forward to test
kubectl port-forward svc/python-app -n production 8080:80

# Test application
curl http://localhost:8080/health
```

### 6.5 Install ArgoCD

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for deployment
kubectl -n argocd rollout status deployment/argocd-server

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo

# Port-forward
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

# Access: https://localhost:8080
# Login: admin / <password>
```

### 6.6 Create ArgoCD Application

```bash
# In ArgoCD UI:
# 1. Applications → New App
# 2. Application Name: python-app
# 3. Project: default
# 4. Repository: http://gitea:3000/gitea_admin/devsecops-demo.git
# 5. Revision: main
# 6. Path: k8s
# 7. Destination Cluster: https://kubernetes.default.svc
# 8. Destination Namespace: production
# 9. Sync Policy: Automatic
# 10. Create

# Or via CLI
argocd app create python-app \
  --repo http://gitea:3000/gitea_admin/devsecops-demo.git \
  --path k8s \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace production \
  --auto-prune \
  --self-heal
```

---

## Step 7: Complete End-to-End Test

### 7.1 Test Full Workflow

```bash
# 1. Make code change
echo "# New feature" >> app/app.py

# 2. Commit and push
git add app/app.py
git commit -m "feat: Add new feature"
git push origin main

# 3. Monitor pipeline
# Gitea UI → Actions → Watch workflow

# 4. Check Nexus for new image
# Nexus UI → Browse → docker-hosted → See new tags

# 5. Verify K8s deployment
kubectl rollout status deployment/python-app -n production

# 6. Check ArgoCD sync
argocd app get python-app

# 7. Test updated application
kubectl port-forward svc/python-app -n production 8080:80
curl http://localhost:8080/health
```

### 7.2 Monitor with Prometheus & Grafana

```bash
# Access Grafana
# http://localhost:3001
# Login: admin/admin

# Add Prometheus data source
# Configuration → Data Sources → Add Prometheus
# URL: http://prometheus:9090

# Create dashboard
# Create → Dashboard → Add Panel
# Query: up
# Visualize
```

---

## Step 8: Monitoring & Alerts

### 8.1 Configure Prometheus Queries

```bash
# Access Prometheus
# http://localhost:9090

# Query examples:
# - up (service status)
# - rate(http_requests_total[5m]) (request rate)
# - histogram_quantile(0.95, http_request_duration_seconds) (p95 latency)
```

### 8.2 Create Grafana Dashboards

```bash
# Common dashboard panels:
# 1. Service Status (up metric)
# 2. Request Rate (rate metric)
# 3. Error Rate (5xx responses)
# 4. Response Time (histogram_quantile)
# 5. Container CPU/Memory
```

### 8.3 Setup Alerting

```yaml
# prometheus.yml - Add alerting rules
rule_files:
  - 'alerts.yml'

# Create alerts.yml
groups:
  - name: devsecops-alerts
    rules:
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
        for: 5m
```

---

## Step 9: Security Best Practices

### 9.1 Change Default Credentials

```bash
# Gitea
# Login → Settings → Change Password

# Nexus
# Profile → Change Password

# SonarQube
# Administration → Security → Users → Change Password
```

### 9.2 Enable HTTPS (Production)

```yaml
# docker-compose.yml
gitea:
  volumes:
    - ./certs/gitea.crt:/data/gitea/https/cert.crt
    - ./certs/gitea.key:/data/gitea/https/key.key

nexus:
  environment:
    - NEXUS_CONTEXT=nexus
    # Add HTTPS configuration
```

### 9.3 Setup SSH Keys

```bash
# Generate key
ssh-keygen -t ed25519 -f ~/.ssh/gitea_key -N ""

# Add to Gitea
# Profile → SSH/GPG Keys → Add Key

# Use for authentication
git clone ssh://git@localhost:222/gitea_admin/devsecops-demo.git
```

### 9.4 Configure Backup

```bash
# Gitea backup
docker exec devsecops-gitea gitea dump -c /data/gitea/conf/app.ini

# Nexus backup
# Administration → Maintenance → Backup

# SonarQube backup
# Administration → Configuration → Backup
```

---

## Step 10: Troubleshooting

### Issue: Pipeline Not Triggering

```bash
# Check Actions enabled
# Gitea Administration → Configuration → Actions

# Check webhook
# Repository → Settings → Webhooks → Recent Deliveries

# Manual re-run
# Actions page → Rerun workflow
```

### Issue: Image Push Fails

```bash
# Verify Docker login
docker logout localhost:8082
docker login localhost:8082

# Check repository exists
curl -u admin:admin123 \
  http://localhost:8081/service/rest/v1/repositories | jq '.[] | select(.name=="docker-hosted")'

# Verify Nexus connectivity
telnet localhost 8082
```

### Issue: K8s Deployment Fails

```bash
# Check image pull
kubectl describe pod <pod-name> -n production

# Create pull secret if needed
kubectl create secret docker-registry nexus-credentials \
  --docker-server=nexus:8082 \
  --docker-username=admin \
  --docker-password=admin123 \
  -n production

# Verify image exists
curl -u admin:admin123 \
  http://localhost:8081/service/rest/v1/components?repository=docker-hosted
```

### Issue: ArgoCD Not Syncing

```bash
# Check repository connection
argocd repo list

# Test Git access
git clone http://gitea:3000/gitea_admin/devsecops-demo.git /tmp/test

# Manual sync
argocd app sync python-app --force
```

---

## Quick Reference URLs

| Component | URL | Credentials |
|-----------|-----|-------------|
| Gitea | http://localhost:3000 | gitea_admin/admin |
| Gitea SSH | ssh://git@localhost:222 | SSH key |
| Nexus | http://localhost:8081/nexus | admin/admin123 |
| Nexus Registry | localhost:8082 | docker login |
| SonarQube | http://localhost:9000 | admin/admin |
| Prometheus | http://localhost:9090 | None |
| Grafana | http://localhost:3001 | admin/admin |
| App | http://localhost:5000 | None |
| ArgoCD | https://localhost:8080 | admin/<generated> |

---

## Success Checklist

- ✅ All services running (docker-compose ps)
- ✅ Gitea repository created and accessible
- ✅ Gitea Actions enabled
- ✅ Nexus repositories configured
- ✅ Docker login successful
- ✅ CI/CD workflow file created
- ✅ First pipeline executed successfully
- ✅ Image pushed to Nexus
- ✅ Kubernetes cluster ready
- ✅ Application deployed to K8s
- ✅ ArgoCD syncing manifests
- ✅ Monitoring with Prometheus/Grafana
- ✅ End-to-end pipeline working

**Your complete DevSecOps pipeline is operational! 🚀**

---

## Next Steps

1. **Customize Workflow**: Adjust pipeline based on your needs
2. **Add Security Scanning**: Configure Falco, OWASP ZAP
3. **Setup Notifications**: Slack, email alerts
4. **Scale Infrastructure**: Multi-node Kubernetes
5. **Implement Policies**: OPA/Gatekeeper for compliance
6. **Optimize Performance**: Network policies, resource limits
7. **Disaster Recovery**: Backup & restore procedures
8. **Documentation**: Update wiki/documentation
