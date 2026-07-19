# Spot Render Ollama - Deployment Guide

> **PT-BR:** Guia completo de deploy do Ollama
> **EN-US:** Complete Ollama deployment guide

## 📋 Prerequisites

### For All Environments

- Kubernetes 1.25+
- kubectl configured with cluster access
- 20Gi+ available storage
- 4Gi+ RAM available for Ollama

### Local Development

- Docker Desktop with Kubernetes enabled
- OR Minikube/MicroK8s with ingress addon

### AWS Production

- EKS cluster with appropriate node groups
- AWS CLI configured
- EBS/EFS CSI driver installed
- (Optional) GPU node group with NVIDIA devices

## 🚀 Deployment

### Option 1: Using Kustomize (Recommended)

#### Local Deployment

```bash
# Clone repository
git clone https://github.com/raafa001/spot-render-ollama.git
cd spot-render-ollama

# Deploy to local cluster
kubectl apply -k kubernetes/overlays/local

# Verify deployment
kubectl get pods -n spot-ai

# Expected output:
# NAME                     READY   STATUS    RESTARTS   AGE
# ollama-xxxxxxxxx-xxxxx   1/1     Running   0          2m
```

#### AWS Deployment

```bash
# Ensure EFS storage class exists
kubectl get storageclass | grep efs

# If not, create EFS CSI driver
# See: https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html

# Deploy with AWS overlay
kubectl apply -k kubernetes/overlays/aws

# Verify GPU allocation
kubectl describe pod -n spot-ai -l app=ollama | grep -A10 "Allocated resources"
```

### Option 2: Using Raw Manifests

```bash
# Deploy base manifest directly
kubectl apply -f kubernetes/base/k8s-ollama.yaml

# This uses default values (CPU, hostpath storage)
# Suitable for quick testing
```

## 🔧 Post-Deployment Configuration

### 1. Wait for Model Download

The Llama 3.2 model (~2GB) is automatically pulled on first request. To verify:

```bash
# Check Ollama logs for model loading
kubectl logs -n spot-ai deploy/ollama --tail=20

# Look for:
# pulling manifest
# writing manifest
# success
```

Or manually trigger model loading:

```bash
# Exec into Ollama pod
kubectl exec -n spot-ai deploy/ollama -it -- /bin/sh

# Inside pod, run:
ollama list

# Or pull explicitly:
ollama pull llama3.2:latest
```

### 2. Verify API Access

```bash
# Local port-forward
kubectl port-forward -n spot-ai svc/ollama 11434:11434 &

# Test API
curl http://localhost:11434/api/tags

# Expected response:
# {"models":[{"name":"llama3.2:latest","size":2019393189,...}]}
```

### 3. Test Inference

```bash
# Simple chat test
curl -X POST http://localhost:11434/api/chat \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama3.2:latest",
    "messages": [{"role": "user", "content": "What is Kubernetes?"}],
    "stream": false
  }'
```

## 🌐 Connecting AI Agents

### Update Agent Configuration

Ensure AI agents have the correct Ollama endpoint:

```yaml
# In agent manifests or ConfigMaps
env:
  - name: OLLAMA_BASE_URL
    value: "http://ollama.spot-ai.svc.cluster.local:11434"
  - name: OLLAMA_MODEL
    value: "llama3.2:latest"
```

### Verify Agent Integration

```bash
# Check agent logs for Ollama connectivity
kubectl logs -n spot-ai deploy/ai-agent | grep -i "ollama"

# Should see periodic:
# Ollama: OK
# Cycle done. Sleeping 300s
```

## 🔄 Rollout & Updates

### Update Ollama Version

```bash
# Edit deployment image
kubectl edit deployment ollama -n spot-ai

# Change:
#   image: ollama/ollama:latest
# To desired version:
#   image: ollama/ollama:0.5.0

# Or via kubectl
kubectl set image deployment/ollama ollama=ollama/ollama:0.5.0 -n spot-ai

# Watch rollout
kubectl rollout status deployment/ollama -n spot-ai

# Verify
kubectl get pods -n spot-ai -l app=ollama
```

### Update Configuration

```bash
# Edit ConfigMap
kubectl edit configmap ollama-config -n spot-ai

# Or apply changes
kubectl apply -f kubernetes/base/k8s-ollama.yaml
```

## 📊 Monitoring

### View Logs

```bash
# Real-time logs
kubectl logs -n spot-ai deploy/ollama -f

# Last 100 lines
kubectl logs -n spot-ai deploy/ollama --tail=100

# With timestamps
kubectl logs -n spot-ai deploy/ollama -f --timestamps
```

### Check Resource Usage

```bash
# Pod metrics
kubectl top pod -n spot-ai

# Node resources (if metrics-server installed)
kubectl top node
```

### Check PVC Status

```bash
kubectl get pvc -n spot-ai
kubectl describe pvc ollama-models -n spot-ai
```

## 🛠️ Troubleshooting

### Common Issues

#### 1. PVC Stuck in Pending

```bash
# Check storage class exists
kubectl get storageclass

# For local, ensure hostpath is available
# For AWS, ensure EFS CSI driver is installed
```

#### 2. Pod ImagePullBackOff

```bash
# Check image exists
kubectl describe pod -n spot-ai -l app=ollama

# For Docker Desktop, image should pull fine
# Check registry access
```

#### 3. Ollama Crashing (OOM)

```bash
# Check pod status
kubectl get pod -n spot-ai -l app=ollama

# Check memory limits
kubectl describe pod -n spot-ai -l app=ollama | grep -A5 "Limits"

# Solution: Increase memory limits in deployment
```

#### 4. Model Not Loading

```bash
# Check model files exist
kubectl exec -n spot-ai deploy/ollama -- ls -la /models

# Manually pull model
kubectl exec -n spot-ai deploy/ollama -- ollama pull llama3.2:latest
```

#### 5. API Timeout

```bash
# Check Ollama is responding
kubectl exec -n spot-ai deploy/ollama -- curl -s http://localhost:11434/api/tags

# Check resource limits
kubectl describe deployment ollama -n spot-ai
```

### Debug Mode

```bash
# Enable debug logging (if supported)
kubectl exec -n spot-ai deploy/ollama -- sh -c "OLLAMA_DEBUG=1 ollama serve"

# Or check environment
kubectl exec -n spot-ai deploy/ollama -- env | grep OLLAMA
```

## 🧹 Cleanup

### Remove Ollama Deployment

```bash
# Delete all resources
kubectl delete -f kubernetes/base/k8s-ollama.yaml

# Or with kustomize
kubectl delete -k kubernetes/overlays/local

# Verify cleanup
kubectl get all -n spot-ai
```

### Keep PVC for Model Persistence

```bash
# Delete only deployment and service
kubectl delete deployment,service -n spot-ai -l app=ollama

# PVC remains with models
kubectl get pvc -n spot-ai

# Can redeploy later without re-downloading models
```

## 📋 Pre-Flight Checklist

Before going to production, verify:

- [ ] Ollama pod is `Running` and `Ready`
- [ ] PVC is `Bound`
- [ ] API responds to `/api/tags`
- [ ] Model `llama3.2:latest` is listed
- [ ] AI agents can reach Ollama internally
- [ ] Resource limits are appropriate
- [ ] Monitoring is configured
- [ ] Backup of PVC (if critical)
- [ ] Network policies applied (if production)

## 🔗 Quick Reference

```bash
# Common commands
kubectl get pods -n spot-ai                    # List pods
kubectl logs -n spot-ai deploy/ollama -f       # Follow logs
kubectl describe pod -n spot-ai -l app=ollama   # Pod details
kubectl top pod -n spot-ai                     # Resource usage
kubectl port-forward -n spot-ai svc/ollama 11434:11434  # Local access
kubectl exec -n spot-ai deploy/ollama -it -- /bin/sh  # Shell access

# Configuration
kubectl edit configmap ollama-config -n spot-ai    # Edit env vars
kubectl edit configmap ollama-env -n spot-ai        # Edit model config

# Troubleshooting
kubectl get events -n spot-ai --sort-by='.lastTimestamp' | tail -20
kubectl describe pvc ollama-models -n spot-ai
```

---

**Document Version:** 1.0.0
**Last Updated:** 2026-07-19
