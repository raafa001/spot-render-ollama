# Spot Render Ollama - Architecture

> **PT-BR:** Documentação da arquitetura do Ollama no Spot Render
> **EN-US:** Ollama architecture documentation for Spot Render

## 🏗️ High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Spot Render Platform                            │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                         Kubernetes Cluster                             │   │
│  │                                                                       │   │
│  │   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐           │   │
│  │   │  SRE Agent  │     │ DevOps Agent│     │Self-Healing │           │   │
│  │   │             │     │             │     │   Agent      │           │   │
│  │   └──────┬──────┘     └──────┬──────┘     └──────┬──────┘           │   │
│  │          │                    │                    │                  │   │
│  │          └────────────────────┼────────────────────┘                  │   │
│  │                               ▼                                         │   │
│  │                    ┌─────────────────┐                                  │   │
│  │                    │  Inter-Agent    │                                  │   │
│  │                    │  Communication  │                                  │   │
│  │                    │  (ConfigMaps)   │                                  │   │
│  │                    └────────┬────────┘                                  │   │
│  │                             │                                           │   │
│  │                    ┌────────▼────────┐                                  │   │
│  │                    │   LLM Gateway   │                                  │   │
│  │                    │  (Ollama API)   │                                  │   │
│  │                    └────────┬────────┘                                  │   │
│  │                             │                                           │   │
│  │          ┌──────────────────┼──────────────────┐                      │   │
│  │          ▼                  ▼                  ▼                      │   │
│  │   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐                │   │
│  │   │   Llama     │   │   Mistral   │   │   Other     │                │   │
│  │   │   3.2       │   │   (future)  │   │   Models    │                │   │
│  │   └─────────────┘   └─────────────┘   └─────────────┘                │   │
│  │          │                                     │                        │   │
│  │          └─────────────────┬─────────────────┘                        │   │
│  │                            ▼                                          │   │
│  │                    ┌───────────────┐                                 │   │
│  │                    │  Persistent   │                                 │   │
│  │                    │  Volume       │                                 │   │
│  │                    │  (Models)     │                                 │   │
│  │                    └───────────────┘                                 │   │
│  │                                                                      │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 🔄 Data Flow

### 1. Monitoring Flow (SRE Agent)

```
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│   Kubernetes │      │  SRE Agent   │      │    Ollama    │
│   API Server │─────▶│  (DaemonSet) │─────▶│   (LLM)      │
│              │      │              │      │              │
│  Metrics:    │      │ Snapshots:   │      │ Analysis:    │
│  - pods      │      │ - errors     │      │ - diagnose   │
│  - events    │      │ - latency    │      │ - explain    │
│  - resources │      │ - saturation │      │ - suggest    │
└──────────────┘      └──────────────┘      └──────────────┘
```

### 2. Decision Flow (Self-Healing Agent)

```
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│   Snapshot   │      │ Self-Healing │      │    Ollama    │
│   Store      │─────▶│   Agent      │─────▶│   (LLM)      │
│              │      │              │      │              │
│  Problem:    │      │ Decision:    │      │ Reasoning:   │
│  - OOMKill   │      │ - restart    │      │ - cause      │
│  - CrashLoop │      │ - scale      │      │ - solution   │
│  - Latency   │      │ - config     │      │ - prevention │
└──────────────┘      └──────┬───────┘      └──────────────┘
                             │
                             ▼
                      ┌──────────────┐
                      │   Action     │
                      │  Executor    │
                      │              │
                      │ kubectl:     │
                      │ - delete pod │
                      │ - scale      │
                      │ - patch      │
                      └──────────────┘
```

### 3. Communication Flow

```
┌──────────────────────────────────────────────────────────────┐
│                  Inter-Agent Communication                    │
│                                                              │
│  ConfigMaps (spot-render-ai-agents namespace):               │
│                                                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│  │ agent-messages  │  │ agent-shared-   │  │  agent-      │ │
│  │                 │  │ state           │  │  config     │ │
│  │ - SRE → DevOps  │  │                 │  │             │ │
│  │ - DevOps → Heal │  │ - active issues│  │ - intervals │ │
│  │ - Heal → SRE    │  │ - agent status │  │ - thresholds │ │
│  └─────────────────┘  └─────────────────┘  └─────────────┘ │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

## 🗄️ Storage Architecture

### Local Development

```
┌─────────────────────────────────────────────────────────────┐
│                    Docker Desktop Node                       │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                    hostpath PV                         │  │
│  │  ┌──────────────────────────────────────────────────┐ │  │
│  │  │              /mnt/data/ollama-models             │ │  │
│  │  │                                                   │ │  │
│  │  │   ┌─────────────┐  ┌─────────────┐               │ │  │
│  │  │   │ llama3.2/   │  │ mistral/    │  ...          │ │  │
│  │  │   │   2GB       │  │   4GB       │               │ │  │
│  │  │   └─────────────┘  └─────────────┘               │ │  │
│  │  └──────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### AWS Production

```
┌─────────────────────────────────────────────────────────────────────┐
│                            AWS Region                                │
│                                                                       │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                        VPC (10.0.0.0/16)                       │  │
│  │                                                                       │  │
│  │   ┌─────────────────────────────────────────────────────────┐  │  │
│  │   │                    EKS Cluster                            │  │  │
│  │   │                                                                 │  │  │
│  │   │   ┌─────────────┐        ┌─────────────────────────────┐ │  │  │
│  │   │   │  EFS File   │◀──────▶│      Ollama Pod             │ │  │  │
│  │   │   │  System     │        │                             │ │  │  │
│  │   │   │             │        │  ┌─────────────────────────┐ │ │  │  │
│  │   │   │  Models:    │        │  │  /models mount         │ │ │  │  │
│  │   │   │  - llama3.2 │        │  │                         │ │ │  │  │
│  │   │   │  - mistral  │        │  │  Persistent across      │ │ │  │  │
│  │   │   │  - etc.     │        │  │  pod restarts           │ │ │  │  │
│  │   │   └─────────────┘        │  └─────────────────────────┘ │ │  │  │
│  │   │                          └─────────────────────────────┘ │  │  │
│  │   └─────────────────────────────────────────────────────────┘  │  │
│  │                                                                       │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

## 🔌 Network Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              Cluster Network                              │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │                    spot-render-ai-agents namespace                  │ │
│  │                                                                        │ │
│  │   sre-agent (Pod)──────┐                                             │ │
│  │                        │                                             │ │
│  │   devops-agent (Pod)───┼──── ConfigMaps ────▶│                      │ │
│  │                        │                       │                      │ │
│  │   self-healing (Pod)───┘                       │                      │ │
│  │                                                  │                      │ │
│  └──────────────────────────────────────────────────┼────────────────────┘ │
│                                                     │                      │
│  ┌──────────────────────────────────────────────────▼────────────────────┐ │
│  │                          spot-ai namespace                             │ │
│  │                                                                        │ │
│  │                   ┌─────────────────┐                                  │ │
│  │                   │     Ollama      │                                  │ │
│  │                   │   (Deployment)  │                                  │ │
│  │                   │                 │                                  │ │
│  │                   │  ClusterIP:     │                                  │ │
│  │                   │  10.103.x.x     │                                  │ │
│  │                   └────────┬────────┘                                  │ │
│  │                            │                                            │ │
│  │                            │ (Port 11434)                              │ │
│  │                            ▼                                            │ │
│  │                   ┌─────────────────┐                                  │ │
│  │                   │   Ollama API    │                                  │ │
│  │                   │   /api/chat     │                                  │ │
│  │                   │   /api/generate │                                  │ │
│  │                   │   /api/tags     │                                  │ │
│  │                   └─────────────────┘                                  │ │
│  │                                                                        │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

## 🧩 Component Details

### Ollama Deployment

```yaml
# Resource Allocation
Resources:
  Local:  CPU: 2 cores, Memory: 8Gi
  AWS:    CPU: 4 cores, Memory: 16Gi, GPU: 1x NVIDIA

# Scaling
- Replicas: 1 (single model instance)
- Note: Ollama loads model into memory; multiple replicas = multiple model copies

# Health Checks
Readiness: HTTP GET /api/tags (30s delay, 10s interval)
Liveness:  HTTP GET /api/tags (60s delay, 30s interval)
```

### Model Management

```
Model Location: /models (inside container)

Storage Requirements:
- Llama 3.2: ~2GB
- Mistral: ~4GB
- Total PVC: 20Gi (local), 50Gi (AWS)

Model Preloading:
- OLLAMA_MODEL env var specifies default model
- Model loaded on first request
- Stays in memory until pod restart
```

## 🔄 Update & Maintenance

### Updating Ollama Version

```bash
# 1. Update image in k8s-ollama.yaml
spec:
  template:
    spec:
      containers:
        - name: ollama
          image: ollama/ollama:latest  # Change to new version

# 2. Apply changes
kubectl apply -f kubernetes/base/k8s-ollama.yaml

# 3. Watch rollout
kubectl rollout status deploy/ollama -n spot-ai
```

### Updating Models

```bash
# Connect to Ollama pod
kubectl exec -it -n spot-ai deploy/ollama -- /bin/bash

# Pull new model
ollama pull llama3.2:latest

# List models
ollama list

# Remove old model
ollama rm llama3.2:previous-version
```

## 📊 Monitoring

### Key Metrics to Monitor

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `ollama_requests_total` | Total API requests | N/A |
| `ollama_request_duration_seconds` | Request latency | p99 > 30s |
| `ollama_model_loaded` | Currently loaded model | none |
| Pod memory usage | Memory pressure | > 80% limit |
| Pod restarts | Unexpected restarts | > 2/hour |

### Prometheus Scraping

```yaml
# Add to prometheus scrape config
- job_name: 'ollama'
  kubernetes_sd_configs:
    - role: pod
  relabel_configs:
    - source_labels: [__meta_kubernetes_pod_namespace]
      action: keep
      regex: spot-ai
    - source_labels: [__meta_kubernetes_pod_label_app]
      action: keep
      regex: ollama
```

## 🛡️ Security Model

### Current State (Development)

```
┌─────────────────────────────────────────────────────────────┐
│                      Development Mode                         │
│                                                              │
│  ✓ Cluster-internal communication only                      │
│  ✓ No external access (ClusterIP)                           │
│  ✓ No authentication (trusting cluster network)             │
│  ✗ No network policies                                       │
│  ✗ No rate limiting                                          │
└─────────────────────────────────────────────────────────────┘
```

### Production Hardening (TODO)

```
┌─────────────────────────────────────────────────────────────┐
│                     Production Mode (Planned)                 │
│                                                              │
│  ✓ Cluster-internal communication only                      │
│  ✓ No external access (ClusterIP)                           │
│  ✓ No authentication (trusting cluster network)             │
│  ○ Network policies (restrict to agent namespaces)           │
│  ○ Rate limiting (per-agent limits)                          │
│  ○ TLS for API (self-signed cert)                           │
│  ○ Audit logging                                             │
└─────────────────────────────────────────────────────────────┘
```

## 🔗 Related Architecture Docs

- [AI Agents Architecture](../spot-render-ia-auto-agents/docs/ARCHITECTURE.md)
- [Spot Render Platform Overview](../spot-render-portal/docs/ARCHITECTURE.md)

---

**Document Version:** 1.0.0
**Last Updated:** 2026-07-19
