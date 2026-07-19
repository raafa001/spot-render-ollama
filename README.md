# Spot Render Ollama

> **PT-BR:** Servidor Ollama com modelo Llama para inferência LLM no cluster Kubernetes
> **EN-US:** Ollama server with Llama model for LLM inference on Kubernetes cluster

## 📋 Overview

This repository contains Kubernetes manifests (Kustomize) for deploying **Ollama** with **Llama 3.2** model for local LLM inference, designed for the Spot Render AI agents ecosystem.

**Ollama** is a local LLM server that runs models like Llama, Mistral, and others directly on your infrastructure, providing privacy, cost savings, and low latency inference.

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Kubernetes Cluster                        │
│                                                                 │
│  ┌──────────────┐         ┌──────────────┐                     │
│  │  AI Agents   │────────▶│    Ollama     │                     │
│  │  (spot-ai)   │         │  (spot-ai)   │                     │
│  └──────────────┘         └──────────────┘                     │
│                                   │                             │
│                            ┌──────┴──────┐                      │
│                            │   Llama 3.2 │                      │
│                            │   (2GB)     │                      │
│                            └─────────────┘                      │
│                                   │                             │
│                            ┌──────┴──────┐                      │
│                            │    PVC      │                      │
│                            │  (models)   │                      │
│                            └─────────────┘                      │
└─────────────────────────────────────────────────────────────────┘
```

## 📁 Structure

```
spot-render-ollama/
├── kubernetes/
│   ├── base/
│   │   └── k8s-ollama.yaml       # Base manifests (all environments)
│   └── overlays/
│       ├── local/                  # Docker Desktop/Minikube (CPU)
│       │   ├── kustomization.yaml
│       │   └── ollama-local-patch.yaml
│       └── aws/                    # EKS with GPU (NVIDIA)
│           ├── kustomization.yaml
│           └── ollama-aws-patch.yaml
├── docs/
│   ├── README.en.md               # English documentation
│   ├── README.pt.md               # Portuguese documentation
│   ├── ARCHITECTURE.md            # Architecture details
│   └── DEPLOYMENT.md              # Deployment guide
└── README.md                      # This file
```

## 🚀 Quick Start

### Local Development (Docker Desktop K8s)

```bash
# Deploy Ollama with CPU resources
kubectl apply -k kubernetes/overlays/local

# Check status
kubectl get pods -n spot-ai

# View logs
kubectl logs -n spot-ai deploy/ollama -f
```

### AWS (EKS with GPU)

```bash
# Deploy Ollama with GPU resources
kubectl apply -k kubernetes/overlays/aws

# Verify GPU allocation
kubectl describe pod -n spot-ai -l app=ollama | grep -A5 "Resources"
```

## 📦 Components

| Component | Type | Description |
|-----------|------|-------------|
| **Namespace** | `spot-ai` | Isolated namespace for AI components |
| **Deployment** | `ollama` | Ollama server (1 replica) |
| **ConfigMap** | `ollama-config` | Environment variables |
| **ConfigMap** | `ollama-env` | Model configuration |
| **PVC** | `ollama-models` | Persistent storage for downloaded models |
| **Service** | `ollama` | ClusterIP for internal access |
| **Service** | `ollama-headless` | Headless for stable DNS |

## 🔧 Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `OLLAMA_HOST` | `0.0.0.0:11434` | Server bind address |
| `OLLAMA_MODELS` | `/models` | Model storage path |
| `OLLAMA_NUM_PARALLEL` | `2` | Max parallel requests (local), `4` (AWS) |
| `OLLAMA_MAX_LOADED_MODELS` | `1` | Max models in memory (local), `2` (AWS) |
| `OLLAMA_MODEL` | `llama3.2:latest` | Default model to load |

### Resource Limits

| Environment | CPU | Memory | GPU |
|--------------|-----|--------|-----|
| **Local** | 2 cores | 8Gi | ❌ None |
| **AWS** | 4 cores | 16Gi | 1x NVIDIA GPU |

### Storage

| Environment | Storage Class | Size | Access |
|-------------|---------------|------|--------|
| **Local** | `hostpath` | 20Gi | ReadWriteOnce |
| **AWS** | `efs-sc` (EFS) | 50Gi | ReadWriteMany |

## 🌐 API Endpoints

Once deployed, Ollama is available at:

```bash
# Internal (from other pods)
http://ollama.spot-ai.svc.cluster.local:11434

# From local machine (port-forward)
kubectl port-forward -n spot-ai svc/ollama 11434:11434
```

### API Examples

```bash
# List available models
curl http://localhost:11434/api/tags

# Chat completion
curl -X POST http://localhost:11434/api/chat \
  -H "Content-Type: application/json" \
  -d '{"model": "llama3.2:latest", "messages": [{"role": "user", "content": "Hello!"}]}'

# Generate completion
curl -X POST http://localhost:11434/api/generate \
  -H "Content-Type: application/json" \
  -d '{"model": "llama3.2:latest", "prompt": "Why is the sky blue?"}'
```

## 🤖 Integration with AI Agents

AI agents connect to Ollama for:

1. **Self-Healing Decisions** - Diagnosing issues and recommending fixes
2. **Anomaly Analysis** - Understanding error patterns
3. **Runbook Generation** - Creating documentation for incidents
4. **Log Summarization** - condensing large log outputs

### Agent Configuration

Agents should use:
```yaml
OLLAMA_BASE_URL: "http://ollama.spot-ai.svc.cluster.local:11434"
OLLAMA_MODEL: "llama3.2:latest"
```

## 🔒 Security Considerations

- Ollama runs internally without authentication (cluster-internal only)
- For production, consider:
  - Network policies to restrict access
  - VPN/tunnel for external access
  - Rate limiting
  - Model validation

## 🛠️ Troubleshooting

### Pod not starting

```bash
# Check PVC status
kubectl get pvc -n spot-ai

# Check storage class
kubectl get storageclass

# View pod events
kubectl describe pod -n spot-ai -l app=ollama
```

### Model not loading

```bash
# Check Ollama logs
kubectl logs -n spot-ai deploy/ollama --tail=50

# Manually pull model (if needed)
kubectl exec -n spot-ai deploy/ollama -- ollama pull llama3.2:latest
```

### Slow inference

- Local: Reduce `OLLAMA_NUM_PARALLEL` to `1`
- AWS: Ensure GPU node has enough resources
- Check memory limits are not being hit

## 📚 Documentation

- [Architecture](docs/ARCHITECTURE.md) - Detailed architecture
- [Deployment Guide](docs/DEPLOYMENT.md) - Step-by-step deployment
- [English Docs](docs/README.en.md) - Full English documentation
- [Docs em Português](docs/README.pt.md) - Documentação completa em português

## 🔗 Related Repositories

- [spot-render-ia-auto-agents](https://github.com/raafa001/spot-render-ia-auto-agents) - AI agents that use Ollama
- [spot-render-teste-local](https://github.com/raafa001/spot-render-teste-local) - Local cluster configuration

## 📄 License

MIT License - See LICENSE file for details.

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request

---

**Maintained by:** Spot Render Team
**Version:** 1.0.0
**Last Updated:** 2026-07-19
