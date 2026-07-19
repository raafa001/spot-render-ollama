# Spot Render Ollama

> **PT-BR:** Servidor Ollama com modelo Llama para inferência LLM no cluster Kubernetes
> **EN-US:** Ollama server with Llama model for LLM inference on Kubernetes cluster

## 📋 Visão Geral

Este repositório contém manifestos Kubernetes (Kustomize) para deploy do **Ollama** com o modelo **Llama 3.2** para inferência local de LLM, projetado para o ecossistema de agentes AI do Spot Render.

**Ollama** é um servidor LLM local que executa modelos como Llama, Mistral e outros diretamente na sua infraestrutura, oferecendo privacidade, economia de custos e baixa latência.

## 🏗️ Arquitetura

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

## 📁 Estrutura

```
spot-render-ollama/
├── kubernetes/
│   ├── base/
│   │   └── k8s-ollama.yaml       # Manifests base (todos ambientes)
│   └── overlays/
│       ├── local/                  # Docker Desktop/Minikube (CPU)
│       │   ├── kustomization.yaml
│       │   └── ollama-local-patch.yaml
│       └── aws/                    # EKS com GPU (NVIDIA)
│           ├── kustomization.yaml
│           └── ollama-aws-patch.yaml
├── docs/
│   ├── README.en.md               # Documentação em inglês
│   ├── README.pt.md               # Documentação em português
│   ├── ARCHITECTURE.md            # Detalhes da arquitetura
│   └── DEPLOYMENT.md              # Guia de deploy
└── README.md                      # Este arquivo
```

## 🚀 Quick Start

### Desenvolvimento Local (Docker Desktop K8s)

```bash
# Deploy do Ollama com recursos CPU
kubectl apply -k kubernetes/overlays/local

# Verificar status
kubectl get pods -n spot-ai

# Ver logs
kubectl logs -n spot-ai deploy/ollama -f
```

### AWS (EKS com GPU)

```bash
# Deploy do Ollama com recursos GPU
kubectl apply -k kubernetes/overlays/aws

# Verificar alocação GPU
kubectl describe pod -n spot-ai -l app=ollama | grep -A5 "Resources"
```

## 📦 Componentes

| Componente | Tipo | Descrição |
|------------|------|-----------|
| **Namespace** | `spot-ai` | Namespace isolado para componentes AI |
| **Deployment** | `ollama` | Servidor Ollama (1 réplica) |
| **ConfigMap** | `ollama-config` | Variáveis de ambiente |
| **ConfigMap** | `ollama-env` | Configuração do modelo |
| **PVC** | `ollama-models` | Storage persistente para modelos |
| **Service** | `ollama` | ClusterIP para acesso interno |
| **Service** | `ollama-headless` | Headless para DNS estável |

## 🔧 Configuração

### Variáveis de Ambiente

| Variável | Padrão | Descrição |
|----------|--------|-----------|
| `OLLAMA_HOST` | `0.0.0.0:11434` | Endereço do servidor |
| `OLLAMA_MODELS` | `/models` | Caminho de armazenamento dos modelos |
| `OLLAMA_NUM_PARALLEL` | `2` | Máx. requisições paralelas (local), `4` (AWS) |
| `OLLAMA_MAX_LOADED_MODELS` | `1` | Máx. modelos na memória (local), `2` (AWS) |
| `OLLAMA_MODEL` | `llama3.2:latest` | Modelo padrão a carregar |

### Limites de Recursos

| Ambiente | CPU | Memória | GPU |
|----------|-----|---------|-----|
| **Local** | 2 cores | 8Gi | ❌ Nenhuma |
| **AWS** | 4 cores | 16Gi | 1x NVIDIA GPU |

### Armazenamento

| Ambiente | Storage Class | Tamanho | Acesso |
|----------|---------------|---------|--------|
| **Local** | `hostpath` | 20Gi | ReadWriteOnce |
| **AWS** | `efs-sc` (EFS) | 50Gi | ReadWriteMany |

## 🌐 Endpoints da API

Após o deploy, o Ollama fica disponível em:

```bash
# Interno (de outros pods)
http://ollama.spot-ai.svc.cluster.local:11434

# Da máquina local (port-forward)
kubectl port-forward -n spot-ai svc/ollama 11434:11434
```

### Exemplos da API

```bash
# Listar modelos disponíveis
curl http://localhost:11434/api/tags

# Chat completion
curl -X POST http://localhost:11434/api/chat \
  -H "Content-Type: application/json" \
  -d '{"model": "llama3.2:latest", "messages": [{"role": "user", "content": "Olá!"}]}'

# Generate completion
curl -X POST http://localhost:11434/api/generate \
  -H "Content-Type: application/json" \
  -d '{"model": "llama3.2:latest", "prompt": "Por que o céu é azul?"}'
```

## 🤖 Integração com Agentes AI

Os agentes AI se conectam ao Ollama para:

1. **Decisões de Auto-Healing** - Diagnosticar problemas e recomendar correções
2. **Análise de Anomalias** - Entender padrões de erro
3. **Geração de Runbooks** - Criar documentação para incidentes
4. **Sumarização de Logs** - Condensar grandes outputs de logs

### Configuração dos Agentes

Os agentes devem usar:
```yaml
OLLAMA_BASE_URL: "http://ollama.spot-ai.svc.cluster.local:11434"
OLLAMA_MODEL: "llama3.2:latest"
```

## 🔒 Considerações de Segurança

- Ollama roda internamente sem autenticação (apenas interno ao cluster)
- Para produção, considere:
  - Network policies para restringir acesso
  - VPN/túnel para acesso externo
  - Rate limiting
  - Validação de modelo

## 🛠️ Troubleshooting

### Pod não inicia

```bash
# Verificar status do PVC
kubectl get pvc -n spot-ai

# Verificar storage class
kubectl get storageclass

# Ver eventos do pod
kubectl describe pod -n spot-ai -l app=ollama
```

### Modelo não carrega

```bash
# Ver logs do Ollama
kubectl logs -n spot-ai deploy/ollama --tail=50

# Puxar modelo manualmente (se necessário)
kubectl exec -n spot-ai deploy/ollama -- ollama pull llama3.2:latest
```

### Inferência lenta

- Local: Reduzir `OLLAMA_NUM_PARALLEL` para `1`
- AWS: Garantir que nó GPU tem recursos suficientes
- Verificar se limites de memória não estão sendo atingidos

## 📚 Documentação

- [Arquitetura](docs/ARCHITECTURE.md) - Arquitetura detalhada
- [Guia de Deploy](docs/DEPLOYMENT.md) - Deploy passo a passo
- [English Docs](docs/README.en.md) - Documentação completa em inglês
- [Docs em Português](docs/README.pt.md) - Esta documentação

## 🔗 Repositórios Relacionados

- [spot-render-ia-auto-agents](https://github.com/raafa001/spot-render-ia-auto-agents) - Agentes AI que usam Ollama
- [spot-render-teste-local](https://github.com/raafa001/spot-render-teste-local) - Configuração do cluster local

## 📄 Licença

MIT License - Veja arquivo LICENSE para detalhes.

## 🤝 Contribuindo

1. Fork o repositório
2. Crie uma branch de feature
3. Faça suas alterações
4. Envie um pull request

---

**Mantido por:** Spot Render Team
**Versão:** 1.0.0
**Última Atualização:** 2026-07-19
