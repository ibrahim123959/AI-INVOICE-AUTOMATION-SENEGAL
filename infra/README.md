---
# Infra - Infrastructure as Code

## 📖 Contexte

L'infrastructure (serveurs, bases de données, réseaux) était traditionnellement configurée manuellement : un administrateur se connecte en SSH, installe logiciels, configure services. Problèmes : non reproductible, erreurs humaines, documentation obsolète.

**Infrastructure as Code (IaC)** : Toute infrastructure définie en fichiers versionnés (comme du code). Avantages : reproductible, versionné, testable, documenté.

Ce module contient configurations pour déployer la plateforme sur différents environnements (dev local, staging, production).
```
┌─────────────────────────────────────────┐
│  Développeur / DevOps                   │
│  Execute : docker-compose up            │
└──────────────┬──────────────────────────┘
               ↓
┌──────────────────────────────────────────┐
│    ► INFRA ◄ (Ce module)                 │
│                                          │
│  docker/      → Images Docker            │
│  kubernetes/  → Manifests K8s (futur)    │
│  terraform/   → Provisioning cloud       │
└──────────────┬───────────────────────────┘
               ↓
┌──────────────────────────────────────────┐
│  Infrastructure déployée                 │
│  (Serveurs, BDD, Load Balancers)         │
└──────────────────────────────────────────┘
```

## 🏗️ Structure
```
infra/
│
├── docker/                      # Dockerfiles production-optimisés
│   ├── backend.Dockerfile       # Image backend (Python + FastAPI)
│   │                            # Multi-stage build (réduction taille)
│   │                            # Security: non-root user, scan vulns
│   ├── ml-engine.Dockerfile     # Image ML (PyTorch + Transformers)
│   │                            # Optionnel: Support GPU (CUDA)
│   └── frontend.Dockerfile      # Image frontend (Nginx + build Vite)
│                                # Serving static optimisé
│
├── kubernetes/                  # Déploiement K8s (production future)
│   ├── backend-deployment.yaml  # Deployment backend (3 replicas)
│   ├── ml-engine-deployment.yaml # Deployment ML (2 replicas)
│   ├── frontend-deployment.yaml  # Deployment frontend (2 replicas)
│   ├── postgres-statefulset.yaml # StatefulSet PostgreSQL
│   ├── services.yaml             # Services (exposition interne)
│   ├── ingress.yaml              # Ingress (exposition externe HTTPS)
│   ├── configmaps.yaml           # ConfigMaps (config non-sensible)
│   ├── secrets.yaml              # Secrets (BDD passwords, API keys)
│   └── hpa.yaml                  # Horizontal Pod Autoscaler
│
└── terraform/                   # Provisioning infrastructure cloud
    ├── main.tf                  # Config principale Terraform
    │                            # Provider (Render, AWS, GCP)
    │                            # Resources (instances, BDD, storage)
    ├── variables.tf             # Variables (environment, region, etc.)
    ├── outputs.tf               # Outputs (URLs, IPs, credentials)
    └── environments/            # Configs par environnement
        ├── staging.tfvars       # Variables staging
        └── production.tfvars    # Variables production
```

## 🐳 Docker

### Images Production vs Development

**Development (docker-compose.yml racine) :**
- Volumes montés (code live)
- Hot reload activé
- Logs verbose
- Pas d'optimisations build

**Production (infra/docker/*.Dockerfile) :**
- Code copié dans image (immutable)
- Multi-stage builds (réduction taille 50-70%)
- Security hardening (non-root user, scan)
- Optimisations build (caching layers)

### Backend Dockerfile
```dockerfile
# infra/docker/backend.Dockerfile

# Stage 1: Builder (compile dependencies)
FROM python:3.11-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# Stage 2: Runtime (image finale légère)
FROM python:3.11-slim
WORKDIR /app

# Security: Non-root user
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

# Copy dependencies from builder
COPY --from=builder /root/.local /home/appuser/.local
ENV PATH=/home/appuser/.local/bin:$PATH

# Copy application code
COPY --chown=appuser:appuser backend/app ./app

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s \
  CMD python -c "import requests; requests.get('http://localhost:8000/health')"

# Run application
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Avantages :**
- Taille finale : ~200MB (vs ~800MB sans multi-stage)
- Security : User non-root (principe least privilege)
- Health checks : K8s/Docker détecte si unhealthy → restart auto

### Build et Push Images
```bash
# Build toutes les images
cd infra/docker

docker build -f backend.Dockerfile -t saisie-backend:v1.0.0 ../../backend
docker build -f ml-engine.Dockerfile -t saisie-ml:v1.0.0 ../../ml-engine
docker build -f frontend.Dockerfile -t saisie-frontend:v1.0.0 ../../frontend

# Tag pour registry
docker tag saisie-backend:v1.0.0 registry.example.com/saisie-backend:v1.0.0

# Push vers registry (Docker Hub, GCR, ECR, etc.)
docker push registry.example.com/saisie-backend:v1.0.0
```

## ☸️ Kubernetes (Production Future)

### Pourquoi Kubernetes ?

**Problème avec serveurs simples :**
- Un serveur down = application down (SPOF - Single Point of Failure)
- Scaling manuel (ajouter serveurs = travail humain)
- Pas de load balancing automatique
- Pas de self-healing (crash = intervention manuelle)

**Kubernetes résout :**
- **High Availability** : 3+ replicas backend, si 1 down les autres prennent relais
- **Auto-scaling** : Charge haute → K8s ajoute pods automatiquement
- **Self-healing** : Pod crash → K8s restart automatiquement
- **Zero-downtime deployments** : Rolling updates (nouveaux pods démarrés avant arrêt anciens)

### Architecture K8s Prévue
```
┌──────────────────────────────────────────────────┐
│              INGRESS (HTTPS)                     │
│  saisie-auto.example.com → Load Balancer         │
└──────────────┬───────────────────────────────────┘
               ↓
┌──────────────────────────────────────────────────┐
│            FRONTEND SERVICE                       │
│  2 Pods Nginx (replicas: 2)                      │
└──────────────┬───────────────────────────────────┘
               ↓
┌──────────────────────────────────────────────────┐
│            BACKEND SERVICE                        │
│  3 Pods FastAPI (replicas: 3)                    │
│  HPA: Scale 3-10 pods selon CPU                  │
└──────────────┬───────────────────────────────────┘
               ↓
┌─────────────────────┬────────────────────────────┐
│  ML ENGINE SERVICE  │  POSTGRESQL STATEFULSET    │
│  2 Pods (replicas:2)│  1 Pod + Persistent Volume │
└─────────────────────┴────────────────────────────┘
```

### Exemple Deployment Backend
```yaml
# kubernetes/backend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 3  # 3 instances backend
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: registry.example.com/saisie-backend:v1.0.0
        ports:
        - containerPort: 8000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: url
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:  # K8s vérifie santé pod
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:  # K8s vérifie si prêt recevoir trafic
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
```

### Déploiement K8s
```bash
# Appliquer tous manifests
kubectl apply -f kubernetes/

# Vérifier déploiement
kubectl get pods
# NAME                        READY   STATUS    RESTARTS   AGE
# backend-5d4c8b9f7d-8xk2p   1/1     Running   0          2m
# backend-5d4c8b9f7d-j9m4n   1/1     Running   0          2m
# backend-5d4c8b9f7d-p3r8t   1/1     Running   0          2m

# Vérifier services
kubectl get svc
# NAME       TYPE           EXTERNAL-IP   PORT(S)
# backend    ClusterIP      10.0.1.5      8000/TCP
# frontend   LoadBalancer   34.123.45.67  80:30080/TCP

# Logs backend (tous pods)
kubectl logs -l app=backend --tail=100 -f

# Scale manuel si besoin
kubectl scale deployment backend --replicas=5
```

## 🏗️ Terraform (Provisioning Cloud)

### Pourquoi Terraform ?

Créer infrastructure manuellement (clicks dans console AWS/GCP) :
- Temps : 1-2h par environnement
- Erreurs : Oubli config, incohérences
- Documentation : Obsolète rapidement
- Reproduction : Impossible identique

**Terraform automatise :**
- Infrastructure définie en code (`.tf` files)
- `terraform apply` → infrastructure créée en 10min
- Reproductible : Même code = même infra
- Versionné : Git track changements infra

### Exemple Provisioning Render.com
```hcl
# terraform/main.tf
terraform {
  required_providers {
    render = {
      source  = "render-oss/render"
      version = "~> 1.0"
    }
  }
}

provider "render" {
  api_key = var.render_api_key
}

# Backend service
resource "render_service" "backend" {
  name   = "saisie-auto-backend"
  type   = "web_service"
  region = "frankfurt"  # EU region
  
  runtime = "docker"
  docker_context = "../backend"
  dockerfile_path = "../infra/docker/backend.Dockerfile"
  
  env_vars = {
    DATABASE_URL = render_postgres.database.connection_string
    OCR_PROVIDER = var.ocr_provider
  }
  
  scaling = {
    min_instances = 2
    max_instances = 10
  }
}

# PostgreSQL database
resource "render_postgres" "database" {
  name    = "saisie-auto-db"
  plan    = "standard"  # 4GB RAM, 50GB storage
  region  = "frankfurt"
  version = "15"
}

# ML Engine service
resource "render_service" "ml_engine" {
  name   = "saisie-auto-ml"
  type   = "web_service"
  region = "frankfurt"
  
  runtime = "docker"
  docker_context = "../ml-engine"
  
  env_vars = {
    MODEL_NAME = "paraphrase-multilingual-MiniLM-L12-v2"
  }
  
  scaling = {
    min_instances = 1
    max_instances = 3
  }
}

# Outputs (URLs, credentials)
output "backend_url" {
  value = render_service.backend.url
}

output "database_connection_string" {
  value     = render_postgres.database.connection_string
  sensitive = true
}
```

### Usage Terraform
```bash
# Initialiser (télécharge providers)
cd terraform
terraform init

# Planifier changements (dry-run)
terraform plan -var-file=environments/staging.tfvars

# Appliquer changements (créer infra)
terraform apply -var-file=environments/production.tfvars

# Voir état actuel
terraform show

# Détruire infrastructure (attention!)
terraform destroy -var-file=environments/staging.tfvars
```

## 🔒 Sécurité

### Secrets Management

**Jamais commiter secrets :**
```bash
# .gitignore
*.tfvars
secrets.yaml
.env
```

**Utiliser outils secrets :**
- **Kubernetes** : Secrets objects (encodés base64)
- **Terraform** : Variables sensibles (`sensitive = true`)
- **Cloud providers** : Secrets Manager (AWS), Secret Manager (GCP)

### Exemple K8s Secret
```yaml
# kubernetes/secrets.yaml (JAMAIS commiter)
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
type: Opaque
data:
  url: cG9zdGdyZXNxbDovL3VzZXI6cGFzc0Bob3N0L2Ri  # base64
  password: c3VwZXJzZWNyZXQ=  # base64
```
```bash
# Créer secret depuis fichier
kubectl create secret generic postgres-secret --from-env-file=.env

# Utiliser depuis déploiement
env:
- name: DATABASE_URL
  valueFrom:
    secretKeyRef:
      name: postgres-secret
      key: url
```

## 📊 Monitoring (Futur)

**Stack recommandée :**
- **Prometheus** : Collecte métriques (CPU, RAM, requêtes/sec)
- **Grafana** : Dashboards visuels
- **Loki** : Logs centralisés
- **AlertManager** : Alertes (Slack, email)

**Exemple dashboard Grafana :**
- Requêtes API/sec
- Latence moyenne backend
- Taux erreurs 5xx
- Utilisation BDD (connections, queries/sec)
- Coût infrastructure ($/jour)

## ❓ FAQ

**Q : Docker Compose suffit pas pour production ?**  
R : Pour pilote SCC (1-5 users), oui. Pour scaling (100+ users), Kubernetes nécessaire.

**Q : Coût Kubernetes vs serveur simple ?**  
R : K8s plus cher (3+ nodes vs 1 serveur), mais robuste. Coût ~$200-500/mois vs ~$50-100/mois.

**Q : Terraform obligatoire ?**  
R : Non, mais fortement recommandé. Sans Terraform = config manuelle (fragile).

**Q : Peut-on migrer Docker Compose → K8s facilement ?**  
R : Oui, outil `kompose` convertit docker-compose.yml en manifests K8s automatiquement.

---

*Version : 0.1.0*  
*Infrastructure : Docker, Kubernetes, Terraform*  
*Providers supportés : Render, AWS, GCP*