# Nexus Repository Manager - Complete Setup & Configuration

## Part 1: Nexus Basics

### 1.1 Access Nexus

```bash
# Open browser
http://localhost:8081/nexus

# Get initial admin password
docker exec devsecops-nexus cat /nexus-data/admin.password

# Login
Username: admin
Password: (from above)
```

### 1.2 First Time Setup

1. Sign in with initial credentials
2. Welcome screen appears
3. Click "Next >" to proceed through setup
4. Configure anonymous access (optional)
5. Finish setup

### 1.3 Change Admin Password

1. Click profile icon (top right)
2. Click "Set Password"
3. Enter new password
4. Confirm
5. Login with new password

---

## Part 2: Docker Repository Configuration

### 2.1 Create Docker Hosted Repository

**Via Web UI:**

1. Administration → Repositories
2. Click "Create Repository"
3. Select "docker (hosted)" recipe
4. Configure:
   - Name: `docker-hosted`
   - HTTP Port: `8082`
   - Enable Docker API: ✓
   - Allow redeploy: ✓
   - Blob store: `default`
5. Create repository

**Via API:**

```bash
curl -X POST \
  -u admin:admin123 \
  -H "Content-Type: application/json" \
  -d '{
    "name": "docker-hosted",
    "type": "hosted",
    "format": "docker",
    "storageId": "default",
    "hosted": {
      "deployPolicy": "ALLOW",
      "contentDisposition": "INLINE"
    },
    "docker": {
      "httpPort": 8082,
      "httpsPort": null,
      "forceBasicAuth": false
    }
  }' \
  http://localhost:8081/service/rest/v1/repositories/docker/hosted
```

### 2.2 Create Docker Proxy Repository (Optional)

For pulling from Docker Hub:

1. Administration → Repositories → Create Repository
2. Select "docker (proxy)" recipe
3. Configure:
   - Name: `docker-proxy`
   - HTTP Port: `8083`
   - Remote storage: `https://registry-1.docker.io`
4. Create repository

### 2.3 Create Docker Group Repository

Combines hosted and proxy repositories:

1. Administration → Repositories → Create Repository
2. Select "docker (group)" recipe
3. Configure:
   - Name: `docker-group`
   - HTTP Port: `8084`
   - Member repositories: 
     - `docker-hosted` (left available)
     - `docker-proxy` (add to group)
   - Add order
4. Create repository

---

## Part 3: Docker Configuration

### 3.1 Configure Docker Authentication

**Option 1: Docker Login**

```bash
# Login interactively
docker login localhost:8082
# Username: admin
# Password: <your-nexus-password>
```

**Option 2: Docker Config File**

```bash
# Create/update ~/.docker/config.json
cat > ~/.docker/config.json << 'EOF'
{
  "auths": {
    "localhost:8082": {
      "auth": "$(echo -n 'admin:admin123' | base64)"
    }
  }
}
EOF

# Make it read-only for security
chmod 600 ~/.docker/config.json
```

**Option 3: Environment Variables**

```bash
export DOCKER_USERNAME=admin
export DOCKER_PASSWORD=admin123
export DOCKER_REGISTRY=localhost:8082

docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD $DOCKER_REGISTRY
```

### 3.2 Test Docker Authentication

```bash
# Verify login worked
cat ~/.docker/config.json | jq '.auths'

# Test pull from Docker Hub (via proxy)
docker pull localhost:8084/library/ubuntu:latest

# Test push to Nexus
docker tag ubuntu:latest localhost:8082/ubuntu:test
docker push localhost:8082/ubuntu:test
```

---

## Part 4: Pushing Images to Nexus

### 4.1 Build & Push Process

```bash
# 1. Build image locally
docker build -t devsecops-python-app:v1.0 ./app

# 2. Tag for Nexus
docker tag devsecops-python-app:v1.0 localhost:8082/devsecops-python-app:v1.0
docker tag devsecops-python-app:v1.0 localhost:8082/devsecops-python-app:latest

# 3. Push to Nexus
docker push localhost:8082/devsecops-python-app:v1.0
docker push localhost:8082/devsecops-python-app:latest

# Expected output:
# The push refers to repository [localhost:8082/devsecops-python-app]
# abc123def456: Pushed
# v1.0: digest: sha256:... size: 5678
```

### 4.2 Verify Images in Nexus

**Via Web UI:**
1. Browse → docker-hosted
2. See pushed images
3. Click image to see details

**Via API:**

```bash
# List all images
curl -u admin:admin123 \
  http://localhost:8081/service/rest/v1/components?repository=docker-hosted

# List specific image
curl -u admin:admin123 \
  http://localhost:8081/service/rest/v1/components?repository=docker-hosted&name=devsecops-python-app

# Get image details
curl -u admin:admin123 \
  "http://localhost:8081/service/rest/v1/search?repository=docker-hosted&q=devsecops"
```

### 4.3 Pull Images from Nexus

```bash
# Pull image
docker pull localhost:8082/devsecops-python-app:v1.0

# Run container
docker run -p 5001:5000 \
  localhost:8082/devsecops-python-app:v1.0

# Test application
curl http://localhost:5001/health
```

---

## Part 5: Automated Pushing via CI/CD

### 5.1 Configure Nexus Credentials in Gitea

Add to repository secrets (Gitea UI):

1. Repository Settings → Secrets
2. Add secret: `NEXUS_PASSWORD`
3. Value: `<your-nexus-password>`

### 5.2 Use in Gitea Actions

In `.gitea/workflows/ci-cd.yml`:

```yaml
- name: Login to Nexus Registry
  run: |
    echo "${{ secrets.NEXUS_PASSWORD }}" | docker login \
      -u admin \
      --password-stdin \
      nexus:8082

- name: Push to Nexus
  run: |
    docker tag myapp:latest nexus:8082/myapp:latest
    docker push nexus:8082/myapp:latest
```

### 5.3 Docker Push with Retry Logic

```bash
# Function to push with retries
push_to_nexus() {
  local image=$1
  local retries=3
  
  for ((i=1; i<=retries; i++)); do
    echo "Pushing $image (attempt $i/$retries)..."
    if docker push "$image"; then
      echo "✓ Successfully pushed: $image"
      return 0
    fi
    
    if [ $i -lt $retries ]; then
      echo "⚠️ Push failed, retrying in 10 seconds..."
      sleep 10
    fi
  done
  
  echo "❌ Failed to push $image after $retries attempts"
  return 1
}

# Usage
push_to_nexus "localhost:8082/devsecops-python-app:v1.0"
```

---

## Part 6: Advanced Configuration

### 6.1 Cleanup Policies

Automatically remove old images:

1. Administration → Repository → docker-hosted
2. Cleanup Policies → Create New
3. Configure:
   - Keep N releases: `5`
   - Keep releases from last N days: `30`
4. Save

### 6.2 Security Configuration

**Enable SSL/TLS:**

```yaml
# docker-compose.yml
nexus:
  volumes:
    - ./certs/nexus.crt:/var/lib/nexus/ssl/nexus.crt
    - ./certs/nexus.key:/var/lib/nexus/ssl/nexus.key
  environment:
    - NEXUS_CONTEXT=nexus
```

**Enable Authentication:**

1. Administration → Security → Realms
2. Ensure "Docker Bearer Token Realm" is active
3. Save

### 6.3 User & Permissions

**Create CI/CD User:**

1. Administration → Security → Users
2. Create user:
   - ID: `ci-user`
   - First Name: CI
   - Last Name: Pipeline
   - Email: ci@devsecops.local
   - Password: `<generate>`
3. Roles: Select `nx-admin` (or custom role)

**Create Custom Role:**

1. Administration → Security → Roles
2. Create Role:
   - Role ID: `docker-pusher`
   - Role Name: Docker Pusher
   - Privileges:
     - `nx-repository-admin-*-*-*`
     - `nx-repository-view-docker-*-*`

### 6.4 Blob Store Configuration

View storage:

```bash
# Check blob store size
curl -u admin:admin123 \
  http://localhost:8081/service/rest/v1/blobstores

# Monitor storage
du -sh /nexus-data/blobs/default/*
```

---

## Part 7: Nexus API Operations

### 7.1 List Repositories

```bash
# All repositories
curl -u admin:admin123 \
  http://localhost:8081/service/rest/v1/repositories

# Docker repositories only
curl -u admin:admin123 \
  http://localhost:8081/service/rest/v1/repositories | \
  jq '.[] | select(.format=="docker")'
```

### 7.2 Search Images

```bash
# Search by name
curl -u admin:admin123 \
  "http://localhost:8081/service/rest/v1/search?repository=docker-hosted&q=devsecops"

# Search with filters
curl -u admin:admin123 \
  "http://localhost:8081/service/rest/v1/search?repository=docker-hosted&name=devsecops-python-app"
```

### 7.3 Delete Images

```bash
# Delete specific image version
curl -X DELETE -u admin:admin123 \
  http://localhost:8081/service/rest/v1/components/docker-hosted/<component-id>

# Delete by tag
curl -u admin:admin123 \
  http://localhost:8081/service/rest/v1/search?repository=docker-hosted&tag=v1.0 | \
  jq '.items[].id' | \
  xargs -I {} curl -X DELETE -u admin:admin123 \
  http://localhost:8081/service/rest/v1/components/docker-hosted/{}
```

### 7.4 Component Details

```bash
# Get component details
curl -u admin:admin123 \
  http://localhost:8081/service/rest/v1/components?repository=docker-hosted | \
  jq '.items[] | {name, version, tags: .assets[].name}'

# Pretty print results
curl -u admin:admin123 \
  http://localhost:8081/service/rest/v1/components?repository=docker-hosted | \
  jq '.items[] | {
    repository: .repository,
    name: .name,
    version: .version,
    tags: [.assets[].name]
  }' | less
```

---

## Part 8: Backup & Restore

### 8.1 Backup Nexus Data

```bash
# Using Docker
docker exec devsecops-nexus bash -c \
  'tar -czf /nexus-data/nexus-backup-$(date +%Y%m%d).tar.gz /nexus-data'

# Copy backup locally
docker cp devsecops-nexus:/nexus-data/nexus-backup-*.tar.gz ./

# Or use Nexus UI:
# Administration → Maintenance → Backup
```

### 8.2 Restore Nexus

```bash
# Stop Nexus
docker stop devsecops-nexus

# Restore from backup
docker cp ./nexus-backup-20240101.tar.gz devsecops-nexus:/nexus-data/
docker exec devsecops-nexus bash -c \
  'tar -xzf /nexus-data/nexus-backup-*.tar.gz -C /'

# Start Nexus
docker start devsecops-nexus
```

---

## Part 9: Troubleshooting

### Issue: Docker Login Fails

```bash
# Check credentials
curl -u admin:admin123 \
  http://localhost:8081/service/rest/v1/status

# Reset auth config
docker logout localhost:8082
rm ~/.docker/config.json
docker login localhost:8082
```

### Issue: Push to Nexus Fails

```bash
# Check repository exists
curl -u admin:admin123 \
  http://localhost:8081/service/rest/v1/repositories | \
  jq '.[] | select(.format=="docker")'

# Test Docker connectivity
curl -i http://localhost:8082/v2/

# Check storage space
du -sh /nexus-data/
```

### Issue: Nexus Not Responding

```bash
# Check logs
docker logs -f devsecops-nexus

# Check Java process
docker exec devsecops-nexus ps aux | grep java

# Restart Nexus
docker restart devsecops-nexus

# Wait for startup (can take 1-2 minutes)
sleep 60
curl http://localhost:8081/nexus/
```

### Issue: Authentication Required for Pull

```bash
# Ensure Docker is logged in
docker login localhost:8082

# Use full image name
docker pull localhost:8082/devsecops-python-app:latest

# Check if anonymous pull is allowed:
# Administration → Repositories → docker-hosted → Anonymous Access
```

---

## Part 10: Integration with Kubernetes

### 10.1 Create Image Pull Secret

```bash
# Create secret in Kubernetes
kubectl create secret docker-registry nexus-credentials \
  --docker-server=nexus:8082 \
  --docker-username=admin \
  --docker-password=admin123 \
  --docker-email=ci@devsecops.local \
  -n production

# Verify secret
kubectl get secret nexus-credentials -n production -o yaml
```

### 10.2 Use in Kubernetes Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: python-app
  namespace: production
spec:
  template:
    spec:
      imagePullSecrets:
      - name: nexus-credentials
      
      containers:
      - name: app
        image: nexus:8082/devsecops-python-app:latest
        imagePullPolicy: Always
```

### 10.3 Verify Pod Can Pull Images

```bash
# Check pod status
kubectl get pods -n production

# View pull events
kubectl describe pod <pod-name> -n production | grep -A 5 "Events"

# View image pull logs
kubectl logs <pod-name> -n production
```

---

## Part 11: Performance & Monitoring

### 11.1 Monitor Disk Usage

```bash
# Check Nexus data directory
du -sh /nexus-data/

# Check blob stores
du -sh /nexus-data/blobs/default/*

# Monitor in real-time
watch -n 5 du -sh /nexus-data/blobs/default/
```

### 11.2 View System Status

```bash
# System status
curl -u admin:admin123 \
  http://localhost:8081/service/rest/v1/status | jq '.'

# Repository statistics
curl -u admin:admin123 \
  http://localhost:8081/service/rest/v1/repositories | \
  jq '.[] | {name, type, format}'
```

### 11.3 Optimize Storage

```bash
# Cleanup policies (automatic)
# Administration → Cleanup Policies → Create

# Manual compact
curl -X POST -u admin:admin123 \
  http://localhost:8081/service/rest/v1/blobstores/default/compact
```

---

## Summary: Nexus in DevSecOps Pipeline

| Stage | Nexus Role | Command |
|-------|------------|---------|
| Build | - | Build Docker image locally |
| Scan | - | Scan with Trivy |
| Push | Registry | `docker push nexus:8082/app:tag` |
| Deploy | Registry | K8s pulls from Nexus |
| Monitor | Metrics | Track image pulls & disk usage |

**Nexus is the central container registry for your DevSecOps pipeline! 🎯**