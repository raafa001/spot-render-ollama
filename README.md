# Spot Render Ollama

> **PT-BR:** Repositório para deploy do Ollama com modelo Llama para inferência LLM no cluster Kubernetes (local e AWS)
> **EN-US:** Repository for deploying Ollama with Llama model for LLM inference on Kubernetes cluster (local and AWS)

## 📋 Descrição / Description

Este repositório contém os manifestos Kubernetes (Kustomize) para deploy do Ollama + Llama 3.2 para inferência de modelos de linguagem localmente.

This repository contains Kubernetes manifests (Kustomize) for deploying Ollama + Llama 3.2 for local LLM inference.

## 🏗️ Estrutura / Structure

```
spot-render-ollama/
├── kubernetes/
│   ├── base/
│   │   └── k8s-ollama.yaml      # Base manifests
│   └── overlays/
│       ├── local/                # Docker Desktop/Minikube (CPU only)
│       └── aws/                  # EKS with GPU (NVIDIA)
├── docs/
└── README.md
```

## 🚀 Quick Start

### Local (Docker Desktop K8s)

```bash
kubectl apply -k kubernetes/overlays/local
```

### AWS (EKS with GPU)

```bash
kubectl apply -k kubernetes/overlays/aws
```

## 📦 Componentes / Components

| Component | Description |
|-----------|-------------|
| **Namespace** | `spot-ai` |
| **Deployment** | Ollama server |
| **ConfigMap** | Ollama environment variables |
| **PVC** | Persistent storage for models (20Gi) |
| **Service** | ClusterIP + Headless |

## 🔧 Configuração / Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `OLLAMA_MODEL` | `llama3.2:latest` | Model to load |
| `OLLAMA_NUM_PARALLEL` | `2` | Parallel requests |
| `OLLAMA_MAX_LOADED_MODELS` | `1` | Max models in memory |

## 🌐 Endpoint

- **Internal:** `http://ollama.spot-ai.svc.cluster.local:11434`
- **From agents:** `http://ollama.spot-ai.svc.cluster.local:11434`

## 📚 Documentação / Documentation

- [docs/README.en.md](docs/README.en.md) - English documentation
- [docs/README.pt.md](docs/README.pt.md) - Documentação em português

## 🔗 Integração / Integration

Os agentes de IA usam este Ollama para:
- Decisões de auto-healing
- Análise de anomalias
- Geração de runbooks

AI agents use this Ollama for:
- Self-healing decisions
- Anomaly analysis
- Runbook generation
