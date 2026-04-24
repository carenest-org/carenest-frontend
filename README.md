# CareNest Frontend

React-based frontend application for the CareNest healthcare platform. Built with Vite and served via Nginx.

---

## 🔄 CI/CD Pipeline

Every push to `main` triggers the full CI/CD pipeline:

```
① SonarQube  →  ② Build  →  ③ Snyk  →  ④ Docker+Trivy+Push  →  ⑤ Approval  →  ⑥ Deploy
```

| Stage | Tool | Purpose | Fail Condition |
|---|---|---|---|
| ① SonarQube | SonarQube | Static code analysis | Quality gate fails |
| ② Build | Node.js 20 + Vite | Install deps & build (`dist/`) | Build errors |
| ③ Snyk | Snyk CLI | Dependency vulnerability scan | HIGH/CRITICAL vulns |
| ④ Docker | Docker + Trivy | Build image, scan, push | HIGH/CRITICAL image vulns |
| ⑤ Approval | GitHub Environments | Manual production gate | Reviewer rejects |
| ⑥ Deploy | GitOps (yq) | Update manifest repo | Git push fails |

### Frontend-Specific Notes
- Stage ② uses `has-build-step: true` to run `npm run build` (Vite)
- Docker uses **multi-stage build**: `node:20-alpine` (build) → `nginx:alpine` (serve)

### Pipeline Behavior
- **Push to `main`**: Full pipeline (CI + CD)
- **Pull request to `main`**: CI only (SonarQube → Build → Snyk)
- **Tag `v*`**: Full pipeline with semantic version tag on Docker image

---

## 🐳 Docker Image Tagging

Every successful pipeline push produces **3 tags**:

```
jayadevarun2003/carenest-frontend:latest
jayadevarun2003/carenest-frontend:sha-abc1234
jayadevarun2003/carenest-frontend:v1.2.0     ← only when git tag exists
```

The `sha-*` tag is used for deployment — it's immutable and traceable to the exact commit.

---

## 🔒 Security Scanning Flow

```
Source Code ──► SonarQube (SAST)
                    │
Dependencies ──► Snyk (SCA)
                    │
Docker Image ──► Trivy (Container Scan)
                    │
              All Pass? ──► Push to DockerHub
```

- **SonarQube**: Checks code quality, bugs, vulnerabilities, code smells
- **Snyk**: Scans `package.json` / `package-lock.json` for known CVEs
- **Trivy**: Scans the final Docker image (Nginx + static assets) for vulnerabilities

---

## 🐳 Multi-Stage Docker Build

```dockerfile
# Stage 1: Build with Node.js
FROM node:20-alpine AS builder
  → npm ci
  → npm run build  (Vite → dist/)

# Stage 2: Serve with Nginx
FROM nginx:alpine
  → Copy custom nginx.conf
  → Copy dist/ to /usr/share/nginx/html
  → Expose port 80
```

The Nginx configuration includes:
- Gzip compression for all text assets
- SPA fallback (`try_files $uri $uri/ /index.html`)
- Static asset caching (1 year, immutable)

---

## 🔐 Required GitHub Secrets

| Secret | Description |
|---|---|
| `SONAR_HOST_URL` | SonarQube server URL |
| `SONAR_TOKEN` | SonarQube authentication token |
| `SNYK_TOKEN` | Snyk API token |
| `DOCKERHUB_USERNAME` | DockerHub username |
| `DOCKERHUB_TOKEN` | DockerHub access token |
| `GH_PAT` | GitHub PAT with repo write access to `carenest-manifest` |

---

## 🛠️ Local Development

```bash
# Install dependencies
npm ci

# Run dev server (Vite HMR)
npm run dev

# Build for production
npm run build

# Preview production build
npm run preview
```

### Environment Variables
See `.env.example` for required environment variables.

---

## 📦 Docker

```bash
# Build locally
docker build -t carenest-frontend .

# Run locally
docker run -p 80:80 carenest-frontend
```
