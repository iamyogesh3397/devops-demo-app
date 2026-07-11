# DevOps Demo App — End-to-End CI/CD Pipeline

A demo Node.js application deployed through a complete, DevOps pipeline:

**GitHub → GitHub Actions (CI/CD) → Docker Hub → Kubernetes (Minikube) → Prometheus + Grafana (monitoring)**

---

## Architecture

```
Developer
   │
   ▼
GitHub (source control)
   │
   ▼
GitHub Actions (CI: test → CD: build & push image)
   │
   ▼
Docker Hub (image registry)
   │
   ▼
Minikube / Kubernetes (deployment, 2 replicas, health checks)
   │
   ▼
Prometheus (scrapes /metrics) → Grafana (dashboards)
```

---

## Tools Used (all free)

| Purpose | Tool |
|---|---|
| Source control | GitHub |
| CI/CD | GitHub Actions |
| Container registry | Docker Hub |
| Local Kubernetes cluster | Minikube |
| Container runtime | Docker |
| Package manager for k8s apps | Helm |
| Metrics collection | Prometheus |
| Dashboards | Grafana |

---

## Project Structure

```
devops-demo-app/
├── app.js                      # Express app with /health and /metrics endpoints
├── test.js                     # Basic smoke test
├── package.json
├── Dockerfile
├── docker-compose.yml          # For local testing (app + Prometheus + Grafana)
├── prometheus.yml              # Prometheus config for docker-compose setup
├── .gitignore
├── .github/
│   └── workflows/
│       └── ci-cd.yml           # GitHub Actions CI/CD pipeline
└── k8s/
    ├── deployment.yaml         # Kubernetes Deployment (2 replicas, probes, prometheus annotations)
    └── service.yaml            # Kubernetes NodePort Service
```

---

## Prerequisites

Install these locally (all free, no signup required except GitHub/Docker Hub accounts):

- [Docker](https://docs.docker.com/get-docker/)
- [Minikube](https://minikube.sigs.k8s.io/docs/start/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm](https://helm.sh/docs/intro/install/)
- [Git](https://git-scm.com/downloads)
- Free [GitHub account](https://github.com/join)
- Free [Docker Hub account](https://hub.docker.com/signup)

---

## Setup Guide

### 1. Clone the repo

```bash
git clone https://github.com/iamyogesh3397/devops-demo-app.git
cd devops-demo-app
```

### 2. Run locally with Docker Compose (optional sanity check)

```bash
docker compose up --build
```
- App: http://localhost:3000
- Prometheus: http://localhost:9090
- Grafana: http://localhost:3001 (login `admin`/`admin`)

Stop with `Ctrl+C`, then `docker compose down`.

### 3. Set up CI/CD (GitHub Actions → Docker Hub)

1. Create a Docker Hub access token: **Docker Hub → Account Settings → Security → New Access Token** (permissions: Read & Write).
2. In your GitHub repo: **Settings → Secrets and variables → Actions → New repository secret**, add:
   - `DOCKERHUB_USERNAME` — your Docker Hub username
   - `DOCKERHUB_TOKEN` — the access token from step 1
3. Push to `main` — GitHub Actions will automatically run tests, then build and push the Docker image to Docker Hub.
4. Watch progress under the repo's **Actions** tab.

### 4. Deploy to Kubernetes (Minikube)

```bash
minikube start
```

Edit `k8s/deployment.yaml` and replace `<your-dockerhub-username>` with your actual Docker Hub username, then:

```bash
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml

kubectl get pods
kubectl get svc
```

Get the app URL (does not block the terminal):
```bash
minikube service devops-demo-app-service --url
```

Test it:
```bash
curl <the-url-from-above>
curl <the-url-from-above>/health
```

### 5. Install monitoring (Prometheus + Grafana via Helm)

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/prometheus
helm install grafana grafana/grafana
```

Get the Grafana admin password:
```bash
kubectl get secret grafana -o jsonpath="{.data.admin-password}" | base64 --decode; echo
```

Port-forward both (run each in the background so they don't block each other):
```bash
kubectl port-forward svc/grafana 3001:80 &
kubectl port-forward svc/prometheus-server 9090:80 &
```

Check running background jobs any time with:
```bash
jobs
```

- Grafana: http://localhost:3001 (user `admin`, password from above)
- Prometheus: http://localhost:9090

### 6. Connect Grafana to Prometheus and build a dashboard

1. Grafana → ☰ menu → **Connections → Data sources → Add data source → Prometheus**
2. URL: `http://prometheus-server` → **Save & test**
3. ☰ menu → **Dashboards → New → New Dashboard → Add visualization**
4. Choose the Prometheus data source, enter metric: `http_requests_total`
5. **Apply**, then **Save dashboard**

Generate traffic so the graph has data:
```bash
curl <app-url>
curl <app-url>/health
```

---

## Common Issues & Fixes

| Problem | Fix |
|---|---|
| `fatal: empty ident name` on commit | Run `git config --global user.name "..."` and `git config --global user.email "..."` |
| `password required` in Docker Hub login step | Re-check `DOCKERHUB_USERNAME`/`DOCKERHUB_TOKEN` secret names and values in repo Settings → Secrets |
| `malformed HTTP Authorization header` | Token has stray whitespace/newline — regenerate token, set secret via `gh secret set` instead of pasting into the web UI |
| Grafana login "incorrect" | Re-run the password decode command with `; echo` appended, or reset via `kubectl exec -it <grafana-pod> -- grafana-cli admin reset-admin-password <newpassword>` |
| Prometheus target DOWN / app not in Targets list | Confirm `prometheus.io/scrape`, `prometheus.io/port`, `prometheus.io/path` annotations exist in `k8s/deployment.yaml`, then `kubectl apply -f k8s/deployment.yaml` again |
| "No data" in Prometheus/Grafana graph | Metric only appears after at least one HTTP request — `curl` the app URL a few times, then refresh |
| Terminal freezes after `kubectl port-forward` | `Ctrl+Z` then `bg` to background it, or always launch with `&` at the end, or use a separate terminal tab |
| Pods `ImagePullBackOff` | Check the image name in `k8s/deployment.yaml` matches your Docker Hub repo exactly, and that the tag exists |
| Pods `CrashLoopBackOff` | Check logs: `kubectl logs <pod-name>` |
| Minikube stops / pods missing after restart | `minikube start` to resume; if deployment is gone, reapply: `kubectl apply -f k8s/deployment.yaml` |

---

## Useful Commands Reference

```bash
# Cluster
minikube start
minikube status
minikube stop
minikube delete

# Pods / deployments / services
kubectl get pods
kubectl get deployments
kubectl get svc
kubectl describe pod <pod-name>
kubectl logs <pod-name>

# Helm
helm list
helm uninstall <release-name>

# Background port-forwards
kubectl port-forward svc/<service-name> <local-port>:<remote-port> &
jobs
kill %1
```

---

## Next Steps / Extensions 

- Add image vulnerability scanning with [Trivy](https://github.com/aquasecurity/trivy) in the CI pipeline
- Add Slack/Discord webhook notifications on pipeline failure
- Add `staging` and `production` Kubernetes namespaces
- Add [ArgoCD](https://argo-cd.readthedocs.io/) for GitOps-style continuous deployment
- Manage the Kubernetes resources with Terraform instead of `kubectl apply`

---

## License

Free to use, modify, and share for learning purposes.
