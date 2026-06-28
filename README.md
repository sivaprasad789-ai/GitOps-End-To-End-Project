# 🍄 GitOps End-to-End Project — Super Mario DevSecOps Pipeline

## 📖 Overview

This project demonstrates a **production-grade GitOps CI/CD pipeline** built around the classic Infinite Mario browser game (HTML5/JavaScript). It serves as a hands-on learning platform for end-to-end DevSecOps practices — covering source code security scanning, containerization, image vulnerability scanning, Kubernetes deployment, and GitOps-based continuous delivery via ArgoCD.

The application is a JavaScript port of the original Java Infinite Mario by Notch (Markus Persson), rendered using HTML5 Canvas and Audio elements, served via Apache Tomcat inside a Docker container.

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        DEVELOPER WORKSTATION                        │
│  git push → github.com/sivaprasad789-ai/GitOps-End-To-End-Project  │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ Push to main branch
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     GITHUB ACTIONS CI PIPELINE                      │
│                                                                     │
│  Job 1: [OPTIONAL] SonarQube SAST Scan                             │
│          └── sonar.projectKey=gitopsdevsecopspipeline               │
│                                                                     │
│  Job 2: Build & Push Docker Image                                   │
│          ├── docker build -t ramprasad789/gitops-devsecops-project  │
│          └── docker push → Docker Hub (tagged with VERSION)         │
│                                                                     │
│  Job 3: Container Image Scan (Trivy)                                │
│          ├── docker pull (image from Docker Hub)                    │
│          ├── docker save → .tar                                     │
│          └── trivy scan (CRITICAL,HIGH) on .tar artifact            │
│                                                                     │
│  Job 4: Update Kubernetes Manifests                                 │
│          ├── sed → update image tag in deployment.yaml              │
│          ├── echo VERSION → version.txt                             │
│          └── git commit + push → triggers ArgoCD sync              │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ deployment.yaml updated in Git
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    ARGOCD (GitOps Controller)                       │
│                                                                     │
│  Watches: deployment.yaml in GitHub repo                            │
│  Detects: image tag drift between Git (desired) vs AKS (live)      │
│  Action:  kubectl apply → rolling update on Kubernetes cluster      │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ Rolling deployment
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    KUBERNETES CLUSTER (AKS / EKS)                  │
│                                                                     │
│  Deployment: supermariogame-deployment (replicas: 1)                │
│  Container:  ramprasad789/gitops-devsecops-project:<VERSION>        │
│  Port:       containerPort 8080 (Tomcat)                            │
│  Service:    supermariogame-service (LoadBalancer)                  │
│              External port 8600 → Target port 8080                 │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
                    🎮 Super Mario Game → Browser
                       http://<EXTERNAL-IP>:8600
```

---

## 📁 Repository Structure

```
GitOps-End-To-End-Project/
│
├── .github/
│   └── workflows/
│       └── e2e-gitops.yaml          ← Full CI/CD pipeline definition
│
├── webapp/                          ← Mario game source (served by Tomcat)
│   ├── index.html                   ← Main game entry point (640x480 canvas)
│   ├── mario.min.js                 ← Minified production bundle
│   ├── enjine.min.js                ← Minified game engine bundle
│   ├── flipTest.html                ← Engine test page
│   ├── minTest.html                 ← Minified bundle test page
│   │
│   ├── Enjine/                      ← Custom JavaScript game engine
│   │   ├── application.js           ← App bootstrap & main loop
│   │   ├── gameCanvas.js            ← HTML5 Canvas wrapper
│   │   ├── keyboardInput.js         ← Keyboard event handling
│   │   ├── resources.js             ← Asset loader (images/sounds)
│   │   ├── gameTimer.js             ← Game loop timing
│   │   ├── camera.js                ← Viewport camera
│   │   ├── drawableManager.js       ← Render pipeline
│   │   ├── animatedSprite.js        ← Sprite animation
│   │   ├── collideable.js           ← Collision detection
│   │   └── state.js                 ← Game state machine
│   │
│   ├── code/                        ← Game logic
│   │   ├── setup.js                 ← Game initialisation
│   │   ├── character.js             ← Mario character (movement, states)
│   │   ├── enemy.js                 ← Enemy AI (Goombas, Koopas)
│   │   ├── levelGenerator.js        ← Procedural level generation
│   │   ├── levelState.js            ← Active gameplay state
│   │   ├── mapState.js              ← World map state
│   │   ├── titleState.js            ← Title screen state
│   │   ├── loadingState.js          ← Asset loading screen
│   │   ├── winState.js              ← Win screen state
│   │   ├── loseState.js             ← Game over state
│   │   ├── music.js                 ← Audio engine & MIDI playback
│   │   ├── fireball.js              ← Fireball projectile
│   │   ├── bulletBill.js            ← Bullet Bill enemy
│   │   ├── mushroom.js              ← Power-up mushroom
│   │   ├── fireFlower.js            ← Fire flower power-up
│   │   ├── shell.js                 ← Koopa shell physics
│   │   └── levelRenderer.js         ← Level tile rendering
│   │
│   ├── images/                      ← Sprite sheets & GIF assets
│   │   ├── mariosheet.png           ← Mario sprite sheet
│   │   ├── enemysheet.png           ← Enemy sprite sheet
│   │   ├── firemariosheet.png       ← Fire Mario sprites
│   │   ├── itemsheet.png            ← Items (coins, mushrooms)
│   │   ├── bgsheet.png              ← Background tiles
│   │   ├── mapsheet.png             ← World map tiles
│   │   └── worldmap.png             ← World map layout
│   │
│   └── sounds/                      ← Audio assets (MP3 + WAV)
│       ├── smwtitle.mp3             ← Title screen music
│       ├── smb3map1.mp3             ← World map music
│       ├── jump.mp3 / .wav          ← Jump sound
│       ├── coin.mp3 / .wav          ← Coin collect sound
│       ├── death.mp3 / .wav         ← Death sound
│       └── [18 additional sounds]   ← Full SFX library
│
├── Dockerfile                       ← Container build definition (Tomcat 9 / JRE 8 Alpine)
├── deployment.yaml                  ← Kubernetes Deployment + LoadBalancer Service
├── sonar-project.properties         ← SonarQube SAST configuration
├── version.txt                      ← Auto-incremented image version (current: 2)
└── demo/
    └── demo.PNG                     ← Project screenshot
```

---

## 🔄 End-to-End Pipeline Flow

### Step 1 — Code Push Triggers Pipeline

A `git push` to the `main` branch triggers the GitHub Actions workflow defined in `.github/workflows/e2e-gitops.yaml`.

The pipeline version is auto-calculated:
```bash
VERSION = $(( $(cat version.txt) + 1 ))
```

### Step 2 — [Optional] SonarQube SAST Scan

> Currently commented out in the workflow — enable by uncommenting the `sonarqube_sast_scan` job.

```yaml
- name: SonarQube Scan
  uses: sonarsource/sonarqube-scan-action@master
  env:
    SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

Project key: `gitopsdevsecopspipeline`

### Step 3 — Build & Push Docker Image

```bash
docker build -t docker.io/ramprasad789/gitops-devsecops-project:$VERSION .
docker push docker.io/ramprasad789/gitops-devsecops-project:$VERSION
```

The Dockerfile:
- Uses `tomcat:9.0.14-jre8-alpine` as base (minimal Alpine footprint)
- Clears the default Tomcat ROOT webapp
- Copies the `webapp/` directory into Tomcat's webapps
- Exposes port `8080`
- Starts Tomcat via `catalina.sh run`

### Step 4 — Container Image Vulnerability Scan (Trivy)

```bash
docker pull docker.io/ramprasad789/gitops-devsecops-project:$VERSION
docker save -o gitops-devsecops-project-latestdockerimage.tar <image>
trivy scan --severity CRITICAL,HIGH <image.tar>
```

Trivy scans the saved `.tar` artifact in tarball mode for `CRITICAL` and `HIGH` CVEs. The pipeline currently uses `exit-code: 0` (non-blocking scan — reports only). Change to `exit-code: 1` to enforce a hard gate.

### Step 5 — Update Kubernetes Manifests (GitOps Loop)

```bash
# Update image tag in deployment.yaml
sed -i "s|image: ramprasad789/gitops-devsecops-project:.*$|image: ramprasad789/gitops-devsecops-project:$VERSION|" deployment.yaml

# Update version tracking
echo $VERSION > version.txt

# Commit & push back to repo
git add deployment.yaml version.txt
git commit -m "Updated deployment yaml and version txt file with supermario image tag to $VERSION"
git push
```

### Step 6 — ArgoCD Detects Drift & Syncs to Kubernetes

ArgoCD continuously watches the repository. When `deployment.yaml` is updated with the new image tag, ArgoCD detects the drift between the desired state (Git) and the live state (Kubernetes), and automatically applies the updated manifest — triggering a rolling update with zero downtime.

---

## 🐳 Docker Details

**Base Image:** `tomcat:9.0.14-jre8-alpine`

```dockerfile
FROM tomcat:9.0.14-jre8-alpine
LABEL maintainer="github.com/asecurityguru"

RUN rm -rf /usr/local/tomcat/webapps/ROOT/*
COPY webapp/ /usr/local/tomcat/webapps/ROOT/
RUN ln -sf /bin/bash /bin/sh

EXPOSE 8080
CMD ["catalina.sh", "run"]
```

| Detail | Value |
|---|---|
| Base OS | Alpine Linux (minimal) |
| Web Server | Apache Tomcat 9.0.14 |
| Java Runtime | JRE 8 |
| Container Port | 8080 |
| Image Registry | Docker Hub |
| Image Name | `ramprasad789/gitops-devsecops-project` |
| Tagging Strategy | Auto-incremented integer version |

---

## ☸️ Kubernetes Manifests

### Deployment (`deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: supermariogame-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: supermariogame
  template:
    metadata:
      labels:
        app: supermariogame
    spec:
      containers:
      - image: ramprasad789/gitops-devsecops-project:3
        name: supermariogame-container
        ports:
        - containerPort: 8080
```

### Service (`deployment.yaml` — Service section)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: supermariogame-service
spec:
  selector:
    app: supermariogame
  ports:
  - protocol: TCP
    port: 8600
    targetPort: 8080
  type: LoadBalancer
```

The service exposes the game externally on port **8600**, routing traffic to Tomcat on port **8080** inside the container.

---

## ✅ Prerequisites

### Local Tools

| Tool | Version | Purpose |
|---|---|---|
| Git | 2.40+ | Source control |
| Docker Desktop | 24.0+ | Local build & test |
| kubectl | 1.28+ | Kubernetes CLI |
| Azure CLI / AWS CLI | Latest | Cloud cluster access |
| ArgoCD CLI (optional) | 2.9+ | ArgoCD management |

### GitHub Secrets Required

| Secret | Description |
|---|---|
| `DOCKERHUB_USERNAME` | Docker Hub account username |
| `DOCKERHUB_TOKEN` | Docker Hub access token |
| `GIT_EMAIL` | Git commit email for manifest updates |
| `GIT_USERNAME` | Git commit username for manifest updates |
| `SONAR_HOST_URL` | SonarQube server URL *(optional)* |
| `SONAR_TOKEN` | SonarQube authentication token *(optional)* |

---

## 🚀 How to Run Locally

### Option 1 — Docker (Fastest)

```bash
# Clone the repository
git clone https://github.com/sivaprasad789-ai/GitOps-End-To-End-Project.git
cd GitOps-End-To-End-Project

# Build the image
docker build -t mario-game:local .

# Run the container
docker run -d -p 8080:8080 --name mario mario-game:local

# Open the game
open http://localhost:8080
```

### Option 2 — Pull from Docker Hub

```bash
docker pull ramprasad789/gitops-devsecops-project:2
docker run -d -p 8080:8080 ramprasad789/gitops-devsecops-project:2
open http://localhost:8080
```

### Option 3 — Deploy to Kubernetes

```bash
# Apply manifests
kubectl apply -f deployment.yaml

# Watch rollout
kubectl rollout status deployment/supermariogame-deployment

# Get external IP
kubectl get svc supermariogame-service

# Open game
open http://<EXTERNAL-IP>:8600
```

### Option 4 — Set up ArgoCD GitOps (Full Pipeline)

```bash
# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Register the application
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gitops-devsecops-project
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/sivaprasad789-ai/GitOps-End-To-End-Project.git
    targetRevision: main
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF

# Get ArgoCD admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

---

## 🎮 Application — Game Details

The Mario game is a full HTML5 JavaScript port of **Infinite Mario Bros** — originally written in Java by Notch (Markus Persson, creator of Minecraft).

### Game Engine (`webapp/Enjine/`)

A custom JavaScript engine with the following systems:

| Module | Responsibility |
|---|---|
| `application.js` | Bootstrap, main game loop |
| `gameCanvas.js` | HTML5 Canvas 2D context management |
| `keyboardInput.js` | Real-time keyboard event capture |
| `resources.js` | Async image and sound asset loading |
| `camera.js` | Scrolling viewport camera |
| `gameTimer.js` | Delta-time based game loop |
| `drawableManager.js` | Z-ordered render pipeline |
| `animatedSprite.js` | Frame-based sprite animation |
| `collideable.js` | AABB collision detection |
| `state.js` | Game state machine base class |

### Game States

| State | Description |
|---|---|
| `loadingState.js` | Asset preloading with progress |
| `titleState.js` | Title screen |
| `mapState.js` | World map navigation |
| `levelState.js` | Active gameplay (physics, enemies, collectibles) |
| `winState.js` | Level complete screen |
| `loseState.js` | Game over screen |

### Controls

| Key | Action |
|---|---|
| ← → Arrow | Move left / right |
| ↑ Arrow / Space | Jump |
| Shift | Run / Fire |

---

## 🔐 Security Hardening (DevSecOps)

### Current Security Controls in Pipeline

| Control | Tool | Stage | Status |
|---|---|---|---|
| SAST (Static Analysis) | SonarQube | Pre-build | ⚠️ Configured, commented out |
| Container image scanning | Trivy (tarball mode) | Post-build | ✅ Active |
| Severity gate | CRITICAL + HIGH | Scan | ⚠️ Non-blocking (exit-code: 0) |
| Secrets management | GitHub Secrets | CI | ✅ Active |
| Version pinning | Auto-increment | Build | ✅ Active |

### Recommended Hardening (Not Yet Applied)

**1. Enforce Trivy Hard Gate**
```yaml
# Change in e2e-gitops.yaml:
exit-code: '1'   # Fail pipeline on CRITICAL/HIGH CVEs
```

**2. Upgrade Base Image**
```dockerfile
# Current (outdated):
FROM tomcat:9.0.14-jre8-alpine

# Recommended:
FROM tomcat:10.1-jre21-alpine
```

**3. Run as Non-Root**
```dockerfile
RUN addgroup -S mario && adduser -S mario -G mario
USER mario
```

**4. Kubernetes Security Context**
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
```

**5. Enable SonarQube SAST**

Uncomment the `sonarqube_sast_scan` job in `.github/workflows/e2e-gitops.yaml` and set `SONAR_HOST_URL` + `SONAR_TOKEN` in GitHub Secrets.

**6. Add Network Policy**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: mario-default-deny
spec:
  podSelector:
    matchLabels:
      app: supermariogame
  policyTypes: [Ingress, Egress]
  ingress:
  - ports:
    - port: 8080
```

---

## 🔧 CI/CD Pipeline Jobs Summary

| Job | Depends On | Runner | Key Steps |
|---|---|---|---|
| `sonarqube_sast_scan` *(disabled)* | — | ubuntu-latest | Checkout → SonarQube scan |
| `build_push_supermario_docker_image` | sonarqube *(disabled)* | ubuntu-latest | Checkout → Docker login → Build → Push |
| `run_container_image_scan_on_supermario_docker_image` | build_push | ubuntu-latest | Pull image → Save tar → Trivy scan |
| `update_k8s_yaml_version_file_with_latest_image_tag` | image_scan | ubuntu-latest | Git pull → sed update → Commit → Push |

---

## 📌 Key Configuration Files

| File | Purpose |
|---|---|
| `.github/workflows/e2e-gitops.yaml` | Full GitHub Actions CI/CD pipeline |
| `Dockerfile` | Tomcat 9 Alpine container definition |
| `deployment.yaml` | Kubernetes Deployment + LoadBalancer Service |
| `version.txt` | Current image version (auto-incremented by pipeline) |
| `webapp/index.html` | Game entry point — 640×480 Canvas |



---

*Secure by design. Automated by default. Deployed with GitOps.*
