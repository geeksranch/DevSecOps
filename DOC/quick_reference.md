# DevSecOps - Quick Reference

## 📋 Project Structure

```
devsecops-demo/
├── .gitea/
│   └── workflows/
│       └── ci-cd.yml              # Gitea Actions workflow
├── app/
│   ├── Dockerfile
│   ├── app.py
│   ├── requirements.txt
│   └── config.py
├── k8s/
│   ├── app-deployment.yaml
│   ├── app-service.yaml
│   └── argocd-app.yaml
├── docker-compose.yml
├── prometheus.yml
├── README.md
└── .gitignore
```

---

## 🚀 Quick Start (5 minutes)

```bash
# 1. Start services
docker-compose up -d

# 2. Wait for initialization
sleep 45

# 3. Access services
Services available at:
- Gitea:      http://localhost:3000       (gitea_admin/admin)
- Nexus:      http://localhost:8081       (admin/admin123)
- SonarQube:  http://localhost:9000       (admin/admin)
- App:        http://localhost:5000/health
```

---

## 🔀 Gitea Commands

### Repository Setup
```bash
# Initialize repository
cd devsecops-demo
git init
git remote add origin http://localhost:3000/gitea_admin/devsecops-demo.git

# Push to Gitea
git add .
git commit -m "Initial commit"
git push -u origin main
# Credentials: gitea_admin / admin

# Clone repository
git clone http://localhost:3000/gitea_admin/devsecops-demo.git

# Via SSH (after adding SSH key)
git clone ssh://git@localhost:222/gitea_admin/devsecops-demo.git
```

### SSH Key Setup
```bash
# Generate SSH key
ssh-keygen -t ed25519 -f ~/.ssh/gitea_key -N ""

# Add to Gitea:
# 1. http://localhost:3000
# 2. Profile → SSH/GPG Keys → Add Key
# 3. Paste public key content

# Test connection
ssh -p 222 git@localhost -i ~/.ssh/gitea_key
```

### CI/CD Workflow
```bash
# Create workflow file
mkdir -p .gitea/workflows
touch .gitea/workflows/ci-cd.yml

# Add workflow content (see full guide)
git add .gitea/workflows/ci-cd.yml
git commit -m "ci: Add Gitea Actions workflow"
git push origin main

# Trigger pipeline
# Make any push to main branch
echo "# Test" >> README.md
git add README.md
git commit -m "test: trigger pipeline"
git push origin main

# Monitor pipeline
# Gitea UI → Repository → Actions → View workflow run
```

### Repository Settings
```bash
# Add secrets to CI/CD
# 1. Repository Settings → Secrets
# 2. Add NEXUS_URL: nexus:8082
# 3. Add NEXUS_USERNAME: admin
# 4. Add NEXUS_PASSWORD: admin123
# 5. Add SONAR_TOKEN: <your-sonar-token>

# View webhook logs
# Repository Settings → Webhooks → Edit → Recent Deliveries
```

---

## 📦 Nexus Registry Commands

### Docker Login
```bash
# Get Nexus password (first time)
docker exec devsecops-nexus cat /nexus-data/admin.password

# Login to Nexus
docker login localhost:8082
# Username: admin
# Password: <from above or set your own>
```

### Build & Push Images
```bash
# Build image locally
docker build -t devsecops-python-app:v1.0 ./app

# Tag for Nexus registry
docker tag devsecops-python-app:v1.0 localhost:8082/devsecops-python-app:v1.0
docker tag devsecops-python-app:v1.0 localhost:8082/devsecops-python-app:latest

# Push to Nexus
docker push localhost:8082/devsecops-python-app:v1.0
docker push localhost:8082/devsecops-python-app:latest

# Verify push
docker image ls | grep nexus
```

### Pull & Run Images
```bash
# Pull from Nexus
docker pull localhost:8082/devsecops-python-app:latest

# Run container
docker run -p 5001:5000 localhost:8082/devsecops-python-app:latest

# Test application
curl http://localhost:5001/health
```

### Nexus API Operations
```bash
# List all repositories
curl -u admin:admin123 \
  http://localhost:8081/service/rest/v1/repositories

# List images in docker-hosted
curl -u admin:admin123 \
  http://localhost:8081/service/rest/v1/components?repository=docker-hosted | jq '.'

# Get image components
curl -u admin:admin123 \
  http://localhost:8081/service/rest/v1/search?repository=docker-hosted

# Delete image
curl -X DELETE -u admin:admin123 \
  http://localhost:8081/service/rest/v1/components/docker-hosted/devsecops-python-app
```

### Nexus Configuration
```bash
# Access Nexus
http://localhost:8081/nexus

# Create repository
# Administration → Repositories → Create Repository
# Recipe: docker (hosted)
# Name: docker-hosted
# HTTP Port: 8082
# Blob Store: default

# View repository status
# Browse → docker-hosted → View contents

# Change admin password
# Sign in → Profile → Change Password
```

---

## 🔍 SonarQube Integration

### Configure SonarQube
```bash
# Access SonarQube
http://localhost:9000

# Login: admin/admin

# Create project
# Projects → Create Project
# Key: python-app
# Main branch: main
```

### Generate SonarQube Token
```bash
# 1. Login to http://localhost:9000
# 2. User menu → My Account → Security
# 3. Generate Token → Name: gitea-ci
# 4. Copy token

# Add to Gitea secrets
# Repository → Settings → Secrets
# Add SONAR_TOKEN: <your-token>
```

### View Analysis Results
```bash
# Check code quality
curl http://localhost:9000/api/measures/search_history \
  ?component=python-app | jq '.'

# Get quality gate status
curl http://localhost:9000/api/qualitygates/project_status \
  ?projectKey=python-app | jq '.projectStatus.status'

# View issues
curl http://localhost:9000/api/issues/search \
  ?componentKeys=python-app | jq '.issues'
```

---

## 🐳 Application Testing

### Test Application Endpoints
```bash
# Health check
curl http://localhost:5000/health

# Get app info
curl http://localhost:5000/api/v1/info | jq '.'

# Process text
curl -X POST http://localhost:5000/api/v1/process \
  -H "Content-Type: application/json" \
  -d '{"text":"hello world"}'

# Analyze text
curl -X POST http://localhost:5000/api/v1/analyze \
  -H "Content-Type: application/json" \
  -d '{"text":"hello world"}'

# Get metrics
curl http://localhost:5000/api/v1/metrics | jq '.metrics'
```

### Load Testing
```bash
# Simple test with curl
curl -i http://localhost:5000/health

# Batch requests
for i in {1..10}; do
  curl -s http://localhost:5000/health | jq '.status'
done

# Using Apache Bench
ab -n 100 -c 10 http://localhost:5000/health

# Using wrk (if installed)
wrk -t4 -c100 -d30s http://localhost:5000/health
```

---

## 📊 Monitoring Commands

### Prometheus Queries
```bash
# Access Prometheus
http://localhost:9090

# Check service status
up

# Request rate (per second)
rate(http_requests_total[5m])

# Error rate
rate(http_requests_total{status=~"5.."}[5m])

# Response time (p95)
histogram_quantile(0.95, http_request_duration_seconds)

# Memory usage
process_resident_memory_bytes

# CPU usage
rate(process_cpu_seconds_total[1m])
```

### Grafana Setup
```bash
# Access Grafana
http://localhost:3001

# Login: admin/admin

# Add data source
# Configuration → Data Sources → Add Prometheus
# URL: http://prometheus:9090

# Create dashboard
# Create → Dashboard → Add Panel → Select Prometheus
```

---

## ⚙️ Kubernetes Commands

### Cluster Verification
```bash
# Check cluster info
kubectl cluster-info

# Get nodes
kubectl get nodes

# Get all resources
kubectl get all -A
```

### Application Deployment
```bash
# Create namespace
kubectl create namespace production

# Apply manifests
kubectl apply -f k8s/

# Check deployment
kubectl get deployment -n production
kubectl get pods -n production
kubectl get svc -n production

# View pod logs
kubectl logs -f deployment/python-app -n production

# Port forward
kubectl port-forward svc/python-app -n production 8080:80

# Scale deployment
kubectl scale deployment python-app --replicas=5 -n production

# Describe pod
kubectl describe pod <pod-name> -n production
```

---

## 🚀 ArgoCD Commands

### Installation
```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for deployment
kubectl -n argocd rollout status deployment/argocd-server
```

### Access ArgoCD
```bash
# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo

# Port forward
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

# Access: https://localhost:8080
```

### Application Management
```bash
# Login to ArgoCD CLI
argocd login localhost:8080 --username admin --password <password>

# Create application
argocd app create python-app \
  --repo http://gitea:3000/gitea_admin/devsecops-demo.git \
  --path k8s \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace production

# List applications
argocd app list

# Get application status
argocd app get python-app

# Sync application
argocd app sync python-app

# Wait for sync
argocd app wait python-app --timeout 300

# Delete application
argocd app delete python-app
```

---

## 📋 Complete Workflow Commands

### Developer Workflow
```bash
# 1. Clone repository
git clone http://localhost:3000/gitea_admin/devsecops-demo.git
cd devsecops-demo

# 2. Create feature branch
git checkout -b feature/new-endpoint

# 3. Make code changes
echo "# New code" >> app/app.py

# 4. Commit changes
git add app/app.py
git commit -m "feat: add new endpoint"

# 5. Push to Gitea
git push origin feature/new-endpoint

# 6. Create Pull Request in Gitea
# Repository → Pull Requests → New Pull Request
# Set base: main, compare: feature/new-endpoint
# Create

# 7. Pipeline runs automatically
# Monitor in: Repository → Actions

# 8. After approval, merge PR
# Pull Request → Merge

# 9. Merge to main triggers deployment
# ArgoCD detects change → Auto deploys to Kubernetes
```

### CI/CD Pipeline Monitoring
```bash
# View pipeline in Gitea
# http://localhost:3000/gitea_admin/devsecops-demo/actions

# View logs
curl http://localhost:3000/api/v1/repos/gitea_admin/devsecops-demo/actions/runs

# Check Nexus for image
curl -u admin:admin123 \
  http://localhost:8081/service/rest/v1/components?repository=docker-hosted

# Check K8s deployment
kubectl get pods -n production -o wide

# Check ArgoCD sync status
argocd app get python-app
```

---

## 🔐 Security Checklist

- ✅ Change default passwords
  ```bash
  # Gitea: Profile → Settings
  # Nexus: Profile → Change Password
  # SonarQube: Administration → Security → Users
  ```

- ✅ Configure SSH keys for Git
  ```bash
  ssh-keygen -t ed25519 -f ~/.ssh/gitea_key -N ""
  # Add to Gitea profile
  ```

- ✅ Store secrets in CI/CD
  ```bash
  # Gitea: Repository → Settings → Secrets
  # Add NEXUS_URL, NEXUS_USERNAME, NEXUS_PASSWORD
  ```

- ✅ Enable HTTPS (Production)
  ```yaml
  # Create certificates
  # Mount in docker-compose for Gitea
  ```

- ✅ Configure image pull secrets (Production)
  ```bash
  kubectl create secret docker-registry nexus-secret \
    --docker-server=nexus:8082 \
    --docker-username=admin \
    --docker-password=admin123 \
    -n production
  ```

---

## 🔧 Troubleshooting

### Gitea Issues
```bash
# Check Gitea logs
docker logs -f devsecops-gitea

# Restart Gitea
docker restart devsecops-gitea

# Check database connection
docker logs devsecops-gitea-postgres

# Reset admin password
docker exec devsecops-gitea gitea admin user change-password admin "newpassword"
```

### Nexus Issues
```bash
# Check Nexus logs
docker logs -f devsecops-nexus

# Get admin password
docker exec devsecops-nexus cat /nexus-data/admin.password

# Reset Nexus
docker-compose down -v
docker-compose up -d nexus
```

### Pipeline Issues
```bash
# Check runner status
docker logs -f devsecops-gitea-runner

# Check workflow logs
# Gitea UI → Repository → Actions → Click workflow

# Trigger manual retry
# Actions page → Re-run workflow
```

### Docker Push Failures
```bash
# Verify credentials
docker logout localhost:8082
docker login localhost:8082

# Test Docker connection
curl http://localhost:8082/v2/

# Check repository exists
curl -u admin:admin123 http://localhost:8081/service/rest/v1/repositories
```

---

## 📚 Service Endpoints Reference

| Service | Type | URL | Port | Credentials |
|---------|------|-----|------|-------------|
| Gitea Web | Web | http://localhost:3000 | 3000 | gitea_admin/admin |
| Gitea SSH | SSH | ssh://git@localhost | 222 | SSH key |
| Nexus | Web | http://localhost:8081/nexus | 8081 | admin/admin123 |
| Nexus Registry | Docker | localhost:8082 | 8082 | admin/admin123 |
| SonarQube | Web | http://localhost:9000 | 9000 | admin/admin |
| Prometheus | Web | http://localhost:9090 | 9090 | None |
| Grafana | Web | http://localhost:3001 | 3001 | admin/admin |
| Application | HTTP | http://localhost:5000 | 5000 | None |
| ArgoCD | Web | https://localhost:8080 | 8080 | admin/<generated> |

---

## 📖 Quick File Locations

| File | Location | Purpose |
|------|----------|---------|
| Workflow | `.gitea/workflows/ci-cd.yml` | Gitea Actions pipeline |
| App Code | `app/app.py` | Python Flask application |
| Dockerfile | `app/Dockerfile` | Container image build |
| K8s Manifest | `k8s/app-deployment.yaml` | Kubernetes deployment |
| Docker Compose | `docker-compose.yml` | Local services |
| Prometheus Config | `prometheus.yml` | Metrics collection |

---

## 🎯 Next Steps

1. ✅ Start Docker Compose stack
2. ✅ Create Gitea repository
3. ✅ Enable Gitea Actions
4. ✅ Create Nexus repository
5. ✅ Setup CI/CD workflow
6. ✅ Test pipeline execution
7. ✅ Deploy to Kubernetes
8. ✅ Setup ArgoCD
9. ✅ Monitor with Prometheus/Grafana
10. ✅ Complete DevSecOps pipeline ready! 🚀
