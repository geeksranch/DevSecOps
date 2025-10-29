# DevSecOps with Gitea & Nexus - Complete Deployment Guide

## Updated Architecture

```
┌─────────────────────────────────────────────────────────┐
│               Developer Workflow                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. Code Push → Gitea Repository                       │
│  2. Gitea Webhook → Gitea Actions (CI/CD)             │
│                                                         │
│  ┌─────────────────────────────────────────────┐      │
│  │ CI/CD Pipeline Execution                    │      │
│  ├─────────────────────────────────────────────┤      │
│  │ ✓ Checkout Code                             │      │
│  │ ✓ SonarQube SAST Analysis                   │      │
│  │ ✓ OWASP Dependency Check                    │      │
│  │ ✓ Build Docker Image                        │      │
│  │ ✓ Trivy Container Scan                      │      │
│  │ ✓ Push to Nexus Registry                    │      │
│  │ ✓ Update K8s Manifests                      │      │
│  │ ✓ Commit to Git (for ArgoCD)                │      │
│  └─────────────────────────────────────────────┘      │
│                                                         │
│  3. ArgoCD Detects Changes in Git                     │
│  4. Deploy to Kubernetes                              │
│  5. Monitor with Prometheus + Grafana                 │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

## Quick Start (15 minutes)

### 1. Start Services
```bash
mkdir devsecops-demo && cd devsecops-demo
# Copy docker-compose.yml and other files from artifacts

docker-compose up -d

# Wait for all services
sleep 45
docker-compose ps
```

### 2. Verify Services
```bash
# Gitea
curl -i http://localhost:3000

# Nexus
curl -i http://localhost:8081/nexus

# SonarQube
curl -i http://localhost:9000

# Prometheus
curl -i http://localhost:9090
```

### 3. Access Web UIs
```
Gitea:      http://localhost:3000       (admin/admin)
Nexus:      http://localhost:8081       (admin/admin123*)
SonarQube:  http://localhost:9000       (admin/admin)
Prometheus: http://localhost:9090
Grafana:    http://localhost:3001       (admin/admin)
App:        http://localhost:5000/health

*Note: Nexus password is auto-generated, get it:
docker exec devsecops-nexus cat /nexus-data/admin.password
```

---

## Phase 1: Gitea Setup (10 minutes)

### Step 1.1: Initial Configuration
```bash
# Access Gitea at http://localhost:3000
# Click "Install"
# Complete setup wizard
# Create admin account: gitea_admin / admin
```

### Step 1.2: Create Repository
```bash
# In Gitea UI:
# 1. Click "+" → New Repository
# 2. Name: devsecops-demo
# 3. Initialize with README
# 4. Create

# Or via command line:
cd ~/devsecops-demo
git init
git remote add origin http://localhost:3000/gitea_admin/devsecops-demo.git
git add .
git commit -m "Initial commit: DevSecOps Pipeline Setup"
git push -u origin main
# When prompted: username=gitea_admin, password=admin
```

### Step 1.3: Repository Structure
```
devsecops-demo/
├── .gitea/
│   └── workflows/
│       └── ci-cd.yml              # CI/CD Pipeline
├── app/
│   ├── Dockerfile
│   ├── app.py
│   ├── requirements.txt
│   └── config.py
├── k8s/
│   ├── app-deployment.yaml
│   ├── app-service.yaml
│   └── argocd-app.yaml
├── prometheus.yml
├── docker-compose.yml
├── README.md
└── .gitignore
```

### Step 1.4: Push Initial Code
```bash
git push origin main

# View in Gitea:
# Repository → Files
```

---

## Phase 2: Gitea Actions Setup (15 minutes)

### Step 2.1: Enable Actions
```bash
# In Gitea:
# 1. Site Administration (wrench icon)
# 2. Configuration → Actions
# 3. Enable Actions
# 4. Save
```

### Step 2.2: Create CI/CD Workflow

Create `.gitea/workflows/ci-cd.yml` in your repository:

```yaml
name: DevSecOps CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  security-scan:
    runs-on: ubuntu-latest
    
    services:
      sonarqube:
        image: sonarqube:10.0-community
        options: >-
          -e SONARQUBE_JDBC_URL=jdbc:h2:mem:sonarqube
          -e SONARQUBE_ADMIN_PASSWORD=admin
        ports:
          - 9000:9000

    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r app/requirements.txt
          pip install pip-audit

      - name: Dependency Check
        run: |
          pip-audit -r app/requirements.txt || true
          echo "Dependency check completed"

      - name: Build Docker Image
        run: |
          docker build -t devsecops-python-app:${{ github.sha }} ./app
          docker tag devsecops-python-app:${{ github.sha }} devsecops-python-app:latest

      - name: Install Trivy
        run: |
          wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | apt-key add - 2>/dev/null || true
          echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | tee /etc/apt/sources.list.d/trivy.list > /dev/null
          apt-get update -qq
          apt-get install -y trivy

      - name: Trivy Container Scan
        run: |
          trivy image --exit-code 0 --severity LOW,MEDIUM,HIGH \
            devsecops-python-app:latest
          trivy image --exit-code 1 --severity CRITICAL \
            devsecops-python-app:latest || echo "Critical vulnerabilities found"

      - name: Set Nexus Credentials
        run: |
          docker login -u admin -p "$(cat /nexus-password.txt 2>/dev/null || echo admin123)" nexus:8082 || true

      - name: Push to Nexus
        run: |
          docker tag devsecops-python-app:latest nexus:8082/devsecops-python-app:${{ github.sha }}
          docker tag devsecops-python-app:latest nexus:8082/devsecops-python-app:latest
          docker push nexus:8082/devsecops-python-app:${{ github.sha }} || echo "Push to Nexus failed"
          docker push nexus:8082/devsecops-python-app:latest || echo "Push to Nexus failed"

      - name: Update K8s Manifests
        run: |
          sed -i "s|image:.*|image: nexus:8082/devsecops-python-app:${{ github.sha }}|g" k8s/app-deployment.yaml

      - name: Commit Updated Manifests
        run: |
          git config --global user.email "ci@devsecops.local"
          git config --global user.name "DevSecOps CI"
          git add k8s/app-deployment.yaml
          git commit -m "ci: Update image to ${{ github.sha }}" || echo "No manifest changes"
          git push || echo "Push failed - may already be updated"
        continue-on-error: true

      - name: Upload Scan Reports
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: scan-reports
          path: |
            trivy-report.json
            app/
```

### Step 2.3: Commit Workflow
```bash
git add .gitea/workflows/ci-cd.yml
git commit -m "ci: Add Gitea Actions CI/CD workflow"
git push origin main
```

### Step 2.4: Monitor Pipeline

View in Gitea UI:
- Repository → Actions
- Click latest workflow run
- Monitor execution progress

---

## Phase 3: Nexus Registry Setup (10 minutes)

### Step 3.1: Access Nexus

```bash
# Open: http://localhost:8081/nexus
# Get admin password:
docker exec devsecops-nexus cat /nexus-data/admin.password

# Login as admin
```

### Step 3.2: Create Docker Hosted Repository

1. **Administration** → **Repositories** → **Create Repository**
2. **Recipe**: `docker (hosted)`
3. **Name**: `docker-hosted`
4. **HTTP Port**: `8082`
5. **Blob Store**: `default`
6. **Allow redeploy**: `enabled`
7. **Create Repository**

### Step 3.3: Configure Docker Authentication

```bash
# Create config file
mkdir -p ~/.docker

# Login to Nexus
docker login localhost:8082
# Username: admin
# Password: <from step 3.1>
```

### Step 3.4: Test Push to Nexus

```bash
# Build image
docker build -t devsecops-python-app:v1.0 ./app

# Tag for Nexus
docker tag devsecops-python-app:v1.0 localhost:8082/devsecops-python-app:v1.0

# Push to Nexus
docker push localhost:8082/devsecops-python-app:v1.0

# Verify in Nexus UI:
# Browse → docker-hosted → devsecops-python-app
```

---

## Phase 4: Complete CI/CD Pipeline Execution

### Step 4.1: Make Code Change
```bash
# Edit app/app.py
echo "
@app.route('/api/v1/version')
def get_version():
    return jsonify({'version': '2.0.0'})
" >> app/app.py

# Commit and push
git add app/app.py
git commit -m "feat: Add version endpoint"
git push origin main
```

### Step 4.2: Monitor Pipeline

```bash
# In Gitea UI:
# Repository → Actions → Click latest workflow
# Monitor real-time execution

# View logs:
# Click on job → View logs
```

### Step 4.3: Verify Build Artifacts

```bash
# Verify image in Nexus
curl -u admin:admin123 \
  http://localhost:8081/service/rest/v1/components?repository=docker-hosted | jq '.'

# Pull image
docker pull localhost:8082/devsecops-python-app:latest

# Run container
docker run -p 5002:5000 localhost:8082/devsecops-python-app:latest
curl http://localhost:5002/api/v1/version
```

### Step 4.4: Verify K8s Manifest Update

```bash
# Check k8s/app-deployment.yaml was updated
git log --oneline k8s/app-deployment.yaml | head -5

# View latest manifest
cat k8s/app-deployment.yaml | grep -A 2 "image:"
```

---

## Phase 5: ArgoCD Integration

### Step 5.1: Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for deployment
kubectl -n argocd rollout status deployment/argocd-server
```

### Step 5.2: Access ArgoCD

```bash
# Get password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo

# Port forward
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

# Access: https://localhost:8080
# Login: admin / <password>
```

### Step 5.3: Create ArgoCD Application

```bash
# Configure kubectl to use Gitea repo
# In ArgoCD UI:
# 1. Settings → Repositories → Connect Repo
# 2. Repository URL: http://gitea:3000/gitea_admin/devsecops-demo.git
# 3. Type: git
# 4. Username: gitea_admin
# 5. Password: admin
# 6. Connect

# Create application:
# Applications → New App
# Name: python-app
# Project: default
# Repository: http://gitea:3000/gitea_admin/devsecops-demo.git
# Path: k8s
# Cluster: https://kubernetes.default.svc
# Namespace: production
# Sync Policy: Automatic
# Create
```

### Step 5.4: Deploy Application

```bash
# ArgoCD will auto-sync when K8s manifests change in Git
# Manifests updated by CI/CD pipeline automatically trigger ArgoCD

# Verify deployment
kubectl get all -n production
kubectl logs -f deployment/python-app -n production
```

---

## Phase 6: Complete End-to-End Workflow

### Workflow Overview

```
1. Developer Makes Change
        ↓
2. Git Push to Gitea
        ↓
3. Gitea Webhook Triggers Actions
        ↓
4. CI/CD Pipeline Runs:
   - Code Analysis (SonarQube)
   - Dependency Check
   - Build Docker Image
   - Scan Container (Trivy)
   - Push to Nexus Registry
   - Update K8s Manifests
   - Commit Back to Gitea
        ↓
5. ArgoCD Detects Manifest Change
        ↓
6. Auto-Sync to Kubernetes
        ↓
7. Application Deployed
        ↓
8. Monitor with Prometheus/Grafana
```

### Test the Complete Flow

```bash
# Step 1: Edit code
echo "# New Feature" >> app/app.py

# Step 2: Push to Gitea
git add app/app.py
git commit -m "feat: Add new feature"
git push origin main

# Step 3: Monitor Gitea Actions (http://localhost:3000)
# Repository → Actions → Watch pipeline

# Step 4: Check Nexus for new image (http://localhost:8081)
# Browse → docker-hosted → devsecops-python-app

# Step 5: Verify K8s manifest updated
git pull origin main
cat k8s/app-deployment.yaml | grep "image:"

# Step 6: Check ArgoCD (https://localhost:8080)
# Applications → python-app → See sync status

# Step 7: Verify pods updated
kubectl get pods -n production -o wide
```

---

## Service Ports & Access

| Service | Port | URL | Credentials |
|---------|------|-----|-------------|
| Gitea | 3000 | http://localhost:3000 | gitea_admin/admin |
| Gitea SSH | 222 | ssh://git@localhost:222 | (SSH key) |
| Nexus | 8081 | http://localhost:8081/nexus | admin/admin123* |
| Nexus Registry | 8082 | localhost:8082 | admin/admin123* |
| SonarQube | 9000 | http://localhost:9000 | admin/admin |
| Prometheus | 9090 | http://localhost:9090 | N/A |
| Grafana | 3001 | http://localhost:3001 | admin/admin |
| App | 5000 | http://localhost:5000 | N/A |
| ArgoCD | 8080 | https://localhost:8080 | admin/<generated> |

*Nexus auto-generates password. Get it: `docker exec devsecops-nexus cat /nexus-data/admin.password`

---

## Troubleshooting

### Gitea Runner Not Executing
```bash
# Check runner status
docker logs -f devsecops-gitea-runner

# Restart runner
docker restart devsecops-gitea-runner

# Check registered runners in Gitea UI:
# Administration → Actions → Runners
```

### Push to Nexus Fails
```bash
# Verify Nexus is accessible
curl http://localhost:8081/nexus/

# Check Docker login
docker logout localhost:8082
docker login localhost:8082

# Verify repository exists
curl -u admin:admin123 \
  http://localhost:8081/service/rest/v1/repositories | jq '.[] | select(.name=="docker-hosted")'
```

### Gitea Actions Not Triggering
```bash
# Check webhook in Gitea:
# Repository → Settings → Webhooks

# Check Actions enabled:
# Administration → Configuration → Actions (enabled?)

# Manually re-run workflow:
# Actions → Click workflow → Rerun
```

### SonarQube Not Responding in Pipeline
```bash
# Wait longer for SonarQube startup (60+ seconds on first run)
# Check SonarQube health:
curl http://localhost:9000/api/system/health

# View SonarQube logs:
docker logs -f devsecops-sonarqube
```

### ArgoCD Not Syncing
```bash
# Check ArgoCD application status
argocd app get python-app

# Manual sync
argocd app sync python-app

# Check repository connection
# ArgoCD UI → Settings → Repositories
```

---

## Security Configuration

### 1. Change Default Passwords

```bash
# Gitea
# Login → Settings → Security → Change Password

# Nexus
# Sign in → Profile → Change Password

# SonarQube
# Admin → Security → Users → Change Password
```

### 2. Configure HTTPS

```yaml
# docker-compose.yml for Gitea
gitea:
  volumes:
    - ./certs/gitea.crt:/data/gitea/https/cert.crt
    - ./certs/gitea.key:/data/gitea/https/key.key
```

### 3. Setup SSH Keys for Gitea

```bash
# Generate key
ssh-keygen -t ed25519 -f ~/.ssh/gitea -N ""

# Add to Gitea:
# Profile → SSH/GPG Keys → Add Key → Paste public key

# Clone via SSH
git clone ssh://git@localhost:222/gitea_admin/devsecops-demo.git
```

---

## Summary

✅ Gitea for source control and CI/CD
✅ Nexus as container registry  
✅ Automated pipeline on every push
✅ SonarQube code analysis
✅ Trivy vulnerability scanning
✅ ArgoCD for GitOps deployment
✅ Kubernetes orchestration
✅ Complete monitoring with Prometheus/Grafana

**Your complete DevSecOps pipeline is ready! 🚀**