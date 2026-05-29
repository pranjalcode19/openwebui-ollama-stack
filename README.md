# OpenWebUI + Ollama — Production K8s Stack

Self-hosted LLM chat interface (Open WebUI) backed by Ollama, deployed on Kubernetes with persistent storage, health checks, and CI/CD.

## Stack

| Component | Purpose |
|---|---|
| **Open WebUI** | Browser-based chat UI (like ChatGPT, self-hosted) |
| **Ollama** | LLM inference server (runs llama3.2 locally) |
| **Kubernetes** | Orchestration — PVC, Ingress, PDB, rolling updates |
| **GitHub Actions** | CI: manifest validation + Trivy security scan + deploy |

## Architecture

```
User → Ingress → OpenWebUI (port 8080) → Ollama ClusterIP (port 11434) → llama3.2
                      ↓                         ↓
               PVC: /app/backend/data    PVC: /root/.ollama (20Gi models)
```

## Run Locally

```bash
docker compose up
```

Open `http://localhost:3000` — sign up, select `llama3.2`, start chatting.

## Deploy to Kubernetes

```bash
# 1. Create namespace
kubectl apply -f k8s/namespace.yaml

# 2. Deploy Ollama (pulls llama3.2 via initContainer — takes ~5 min first time)
kubectl apply -f k8s/ollama/
kubectl rollout status deployment/ollama -n ai-platform --timeout=300s

# 3. Set secret before deploying Open WebUI
kubectl create secret generic openwebui-secret \
  --from-literal=WEBUI_SECRET_KEY=your-secret-here \
  -n ai-platform

# 4. Deploy Open WebUI
kubectl apply -f k8s/openwebui/
kubectl rollout status deployment/openwebui -n ai-platform --timeout=120s

# 5. Access via Ingress or port-forward
kubectl port-forward svc/openwebui 3000:80 -n ai-platform
```

## Key Design Decisions

**initContainer for model pull**
Ollama's initContainer pulls `llama3.2` before the main container starts. First deploy takes ~5 minutes; subsequent restarts skip the pull since the model is cached on the PVC.

**20Gi PVC for Ollama**
LLM models are large (llama3.2 = ~2GB, larger models can be 8-40GB). PVC ensures models survive pod restarts — without this, every restart re-downloads the model.

**Ingress timeouts extended to 300s**
LLM inference for long prompts can take 30-120 seconds. Default nginx timeout is 60s — without increasing `proxy-read-timeout`, long responses get cut off mid-stream.

**OLLAMA_BASE_URL uses full cluster DNS**
`http://ollama.ai-platform.svc.cluster.local:11434` — explicit namespace in the URL prevents DNS resolution issues when services are in different namespaces.

**Trivy in CI**
Scans both Ollama and Open WebUI images for HIGH/CRITICAL CVEs on every push. `exit-code: 0` means it reports but doesn't block — set to `1` to enforce hard gates.

## Monitoring

```bash
# Check both services are running
kubectl get pods -n ai-platform

# Tail Open WebUI logs
kubectl logs -f deployment/openwebui -n ai-platform

# Tail Ollama logs (shows model loading, inference requests)
kubectl logs -f deployment/ollama -n ai-platform

# Check PVC usage
kubectl get pvc -n ai-platform
```

## Adding More Models

```bash
# Exec into Ollama pod and pull additional models
kubectl exec -it deployment/ollama -n ai-platform -- ollama pull mistral
kubectl exec -it deployment/ollama -n ai-platform -- ollama pull codellama
```

Models appear automatically in Open WebUI's model selector.
