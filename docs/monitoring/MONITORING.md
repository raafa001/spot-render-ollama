# Spot Render Ollama - Monitoring Guide

> **PT-BR:** Guia completo de monitoramento para Ollama no Spot Render
> **EN-US:** Complete monitoring guide for Ollama in Spot Render

## 📊 Overview

This document describes the monitoring strategy, metrics, dashboards, and alerts for the Ollama deployment in the Spot Render platform.

## 🔍 Metrics to Monitor

### 1. Ollama Service Metrics

| Metric | Description | Type | Alert Threshold |
|--------|-------------|------|-----------------|
| `ollama_up` | Ollama service is running | Gauge | 0 = Down |
| `ollama_api_requests_total` | Total API requests | Counter | N/A |
| `ollama_api_request_duration_seconds` | Request latency | Histogram | p99 > 30s |
| `ollama_model_loaded` | Currently loaded model name | Gauge | N/A |
| `ollama_model_load_duration_seconds` | Time to load model | Histogram | > 5min |
| `ollama_active_connections` | Current active connections | Gauge | > 10 |

### 2. Kubernetes Resource Metrics

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `kube_pod_status_phase` | Pod phase (Running=1) | Not Running |
| `container_memory_working_set_bytes` | Memory usage | > 80% limit |
| `container_cpu_usage_seconds_total` | CPU usage | > 80% limit |
| `kube_pod_restart_total` | Pod restart count | > 2/hour |

### 3. Business Metrics

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `spotinho_messages_total` | Total chat messages | N/A |
| `spotinho_response_time_seconds` | Response time | p99 > 10s |
| `spotinho_errors_total` | Error count | > 10/hour |
| `spotinho_offline_total` | Offline events | > 0 |

## 📈 Prometheus Configuration

### Prometheus Scrape Config

```yaml
# prometheus-scrape-configs.yaml
scrape_configs:
  # Ollama metrics
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
      - source_labels: [__meta_kubernetes_pod_container_port_number]
        action: keep
        regex: "11434"
    metrics_path: '/api/tags'

  # AI Agents metrics
  - job_name: 'ai-agents'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_namespace]
        action: keep
        regex: spot-render-ai-agents
      - source_labels: [__meta_kubernetes_pod_label_app]
        action: keep
        regex: (sre|devops|self-healing)-agent
```

## 📊 Grafana Dashboards

### Dashboard 1: Ollama Overview

```json
{
  "dashboard": {
    "title": "Spot Render - Ollama Overview",
    "uid": "spot-render-ollama",
    "panels": [
      {
        "title": "Ollama Status",
        "type": "stat",
        "gridPos": {"x": 0, "y": 0, "w": 4, "h": 4},
        "targets": [
          {
            "expr": "up{job='ollama'}",
            "legendFormat": "Ollama Status"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "mappings": [
              {"type": "value", "options": {"0": {"text": "DOWN", "color": "red"}}},
              {"type": "value", "options": {"1": {"text": "UP", "color": "green"}}}
            ]
          }
        }
      },
      {
        "title": "API Request Rate",
        "type": "graph",
        "gridPos": {"x": 4, "y": 0, "w": 10, "h": 8},
        "targets": [
          {
            "expr": "rate(ollama_api_requests_total[5m])",
            "legendFormat": "Requests/sec"
          }
        ]
      },
      {
        "title": "Request Latency (p99)",
        "type": "graph",
        "gridPos": {"x": 14, "y": 0, "w": 10, "h": 8},
        "targets": [
          {
            "expr": "histogram_quantile(0.99, rate(ollama_api_request_duration_seconds_bucket[5m]))",
            "legendFormat": "p99 Latency"
          }
        ],
        "alert": {
          "name": "High Ollama Latency",
          "conditions": [
            {
              "evaluator": {"params": [30], "type": "gt"},
              "operator": {"type": "and"},
              "query": {"params": ["A", "5m", "now"]},
              "reducer": {"type": "avg"}
            }
          ],
          "frequency": "1m",
          "handler": 1,
          "message": "Ollama API p99 latency is above 30 seconds"
        }
      },
      {
        "title": "Memory Usage",
        "type": "graph",
        "gridPos": {"x": 0, "y": 8, "w": 8, "h": 8},
        "targets": [
          {
            "expr": "container_memory_working_set_bytes{pod=~'ollama-.*', namespace='spot-ai'}",
            "legendFormat": "Memory Usage"
          },
          {
            "expr": "kube_pod_container_resource_limits_memory_bytes{pod=~'ollama-.*', namespace='spot-ai'}",
            "legendFormat": "Memory Limit"
          }
        ]
      },
      {
        "title": "CPU Usage",
        "type": "graph",
        "gridPos": {"x": 8, "y": 8, "w": 8, "h": 8},
        "targets": [
          {
            "expr": "rate(container_cpu_usage_seconds_total{pod=~'ollama-.*', namespace='spot-ai'}[5m])",
            "legendFormat": "CPU Usage"
          }
        ]
      },
      {
        "title": "Loaded Model",
        "type": "stat",
        "gridPos": {"x": 16, "y": 8, "w": 8, "h": 4},
        "targets": [
          {
            "expr": "ollama_model_loaded",
            "legendFormat": "{{model}}"
          }
        ]
      },
      {
        "title": "Pod Restarts",
        "type": "stat",
        "gridPos": {"x": 16, "y": 12, "w": 8, "h": 4},
        "targets": [
          {
            "expr": "kube_pod_restart_total{namespace='spot-ai'}",
            "legendFormat": "Restarts"
          }
        ]
      }
    ]
  }
}
```

### Dashboard 2: AI Agents Overview

```json
{
  "dashboard": {
    "title": "Spot Render - AI Agents Overview",
    "uid": "spot-render-ai-agents",
    "panels": [
      {
        "title": "Agent Status",
        "type": "table",
        "gridPos": {"x": 0, "y": 0, "w": 24, "h": 4},
        "targets": [
          {
            "expr": "kube_pod_metadata_labels{namespace='spot-render-ai-agents'}",
            "format": "table"
          }
        ]
      },
      {
        "title": "SRE Agent - Error Detection",
        "type": "graph",
        "gridPos": {"x": 0, "y": 4, "w": 8, "h": 8},
        "targets": [
          {
            "expr": "rate(sre_agent_errors_detected_total[5m])",
            "legendFormat": "Errors/sec"
          }
        ]
      },
      {
        "title": "DevOps Agent - Operations",
        "type": "graph",
        "gridPos": {"x": 8, "y": 4, "w": 8, "h": 8},
        "targets": [
          {
            "expr": "rate(devops_agent_operations_total[5m])",
            "legendFormat": "Operations/sec"
          }
        ]
      },
      {
        "title": "Self-Healing Agent - Remediations",
        "type": "graph",
        "gridPos": {"x": 16, "y": 4, "w": 8, "h": 8},
        "targets": [
          {
            "expr": "rate(self_healing_agent_remediations_total[5m])",
            "legendFormat": "Remediations/sec"
          }
        ]
      }
    ]
  }
}
```

## 🚨 Alert Rules

### Prometheus Alert Rules (prometheus-alerts.yaml)

```yaml
groups:
  - name: ollama-alerts
    interval: 30s
    rules:
      # Ollama Down
      - alert: OllamaDown
        expr: up{job="ollama"} == 0
        for: 1m
        labels:
          severity: critical
          component: ollama
        annotations:
          summary: "Ollama service is down"
          description: "Ollama has been down for more than 1 minute"
          runbook_url: "https://docs.spot-render.local/runbooks/ollama-down"

      # High Latency
      - alert: OllamaHighLatency
        expr: histogram_quantile(0.99, rate(ollama_api_request_duration_seconds_bucket[5m])) > 30
        for: 5m
        labels:
          severity: warning
          component: ollama
        annotations:
          summary: "Ollama API latency is high"
          description: "p99 latency is {{ $value }} seconds"
          runbook_url: "https://docs.spot-render.local/runbooks/ollama-high-latency"

      # High Memory Usage
      - alert: OllamaHighMemory
        expr: (container_memory_working_set_bytes / kube_pod_container_resource_limits_memory_bytes) > 0.9
        for: 5m
        labels:
          severity: warning
          component: ollama
        annotations:
          summary: "Ollama memory usage is high"
          description: "Memory usage is at {{ $value | humanizePercentage }}"
          runbook_url: "https://docs.spot-render.local/runbooks/ollama-high-memory"

      # Model Not Loaded
      - alert: OllamaModelNotLoaded
        expr: absent(ollama_model_loaded)
        for: 2m
        labels:
          severity: critical
          component: ollama
        annotations:
          summary: "No model loaded in Ollama"
          description: "Ollama has no model loaded"
          runbook_url: "https://docs.spot-render.local/runbooks/ollama-no-model"

      # Pod Restarting Frequently
      - alert: OllamaPodRestarting
        expr: increase(kube_pod_restart_total{namespace="spot-ai"}[1h]) > 2
        for: 1m
        labels:
          severity: warning
          component: ollama
        annotations:
          summary: "Ollama pod is restarting frequently"
          description: "Pod has restarted {{ $value }} times in the last hour"
          runbook_url: "https://docs.spot-render.local/runbooks/ollama-restarting"

  - name: ai-agents-alerts
    interval: 30s
    rules:
      # SRE Agent Down
      - alert: SREAgentDown
        expr: absent(kube_pod_status_phase{namespace="spot-render-ai-agents", pod=~"sre-agent-.*", phase="Running"})
        for: 2m
        labels:
          severity: critical
          component: sre-agent
        annotations:
          summary: "SRE Agent is down"
          description: "SRE Agent has been down for more than 2 minutes"
          runbook_url: "https://docs.spot-render.local/runbooks/sre-agent-down"

      # DevOps Agent Down
      - alert: DevOpsAgentDown
        expr: absent(kube_pod_status_phase{namespace="spot-render-ai-agents", pod=~"devops-agent-.*", phase="Running"})
        for: 2m
        labels:
          severity: critical
          component: devops-agent
        annotations:
          summary: "DevOps Agent is down"
          description: "DevOps Agent has been down for more than 2 minutes"
          runbook_url: "https://docs.spot-render.local/runbooks/devops-agent-down"

      # Self-Healing Agent Down
      - alert: SelfHealingAgentDown
        expr: absent(kube_pod_status_phase{namespace="spot-render-ai-agents", pod=~"self-healing-agent-.*", phase="Running"})
        for: 2m
        labels:
          severity: critical
          component: self-healing-agent
        annotations:
          summary: "Self-Healing Agent is down"
          description: "Self-Healing Agent has been down for more than 2 minutes"
          runbook_url: "https://docs.spot-render.local/runbooks/self-healing-agent-down"

      # High Error Rate
      - alert: AIAgentsHighErrorRate
        expr: rate(ai_agents_errors_total[5m]) > 0.1
        for: 5m
        labels:
          severity: warning
          component: ai-agents
        annotations:
          summary: "AI Agents error rate is high"
          description: "Error rate is {{ $value }} per second"
          runbook_url: "https://docs.spot-render.local/runbooks/ai-agents-high-error-rate"

      # Ollama Unreachable from Agents
      - alert: OllamaUnreachableFromAgents
        expr: ai_agent_ollama_unavailable == 1
        for: 2m
        labels:
          severity: warning
          component: ai-agents
        annotations:
          summary: "AI Agents cannot reach Ollama"
          description: "AI Agents have been unable to reach Ollama for 2 minutes"
          runbook_url: "https://docs.spot-render.local/runbooks/ollama-unreachable"
```

## 🔔 Notification Configuration

### Slack Integration

```yaml
# alertmanager-config.yaml
global:
  resolve_timeout: 5m
  slack_api_url: "<SLACK_WEBHOOK_URL>"

route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'slack-notifications'
  routes:
    - match:
        severity: critical
      receiver: 'slack-critical'
      group_wait: 10s
    - match:
        severity: warning
      receiver: 'slack-warning'

receivers:
  - name: 'slack-notifications'
    slack_configs:
      - channel: '#alerts-spot-render'
        send_resolved: true
        title: |
          [{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}
        text: |
          {{ range .Alerts }}
          *{{ .Annotations.summary }}*
          {{ .Annotations.description }}
          {{ if .Annotations.runbook_url }}
          Runbook: {{ .Annotations.runbook_url }}
          {{ end }}
          {{ end }}

  - name: 'slack-critical'
    slack_configs:
      - channel: '#alerts-critical'
        send_resolved: true
        title: |
          🚨 [CRITICAL] {{ .GroupLabels.alertname }}
        text: |
          *{{ .Annotations.summary }}*
          Cluster: {{ .Labels.cluster }}
          Service: {{ .Labels.service }}
          {{ .Annotations.description }}
```

### PagerDuty Integration (for critical alerts)

```yaml
  - name: 'pagerduty-critical'
    pagerduty_configs:
      - service_key: "<PAGERDUTY_SERVICE_KEY>"
        severity: critical
        details:
          - key: "cluster"
            value: "{{ .Labels.cluster }}"
          - key: "service"
            value: "{{ .Labels.service }}"
```

## 📝 Runbooks

### Runbook: Ollama Down

```markdown
# Runbook: Ollama Service Down

## Severity: Critical

## Symptoms
- Ollama API returns 503 or connection refused
- `up{job="ollama"}` metric is 0
- Agents cannot process LLM requests

## Diagnosis
1. Check Ollama pod status:
   ```bash
   kubectl get pods -n spot-ai -l app=ollama
   kubectl describe pod -n spot-ai -l app=ollama
   ```

2. Check pod logs:
   ```bash
   kubectl logs -n spot-ai -l app=ollama --tail=100
   ```

3. Verify PVC is mounted:
   ```bash
   kubectl describe pod -n spot-ai -l app=ollama | grep -A5 "Volumes:"
   ```

4. Check resource limits (OOM):
   ```bash
   kubectl top pod -n spot-ai -l app=ollama
   ```

## Resolution
1. If pod is CrashLoopBackOff:
   - Check memory limits and increase if needed
   - Verify model files exist in PVC

2. If pod is Pending:
   - Check PVC status: `kubectl get pvc -n spot-ai`
   - Verify storage class exists

3. If image pull error:
   - Verify image exists: `docker pull ollama/ollama:latest`
   - Check registry credentials

4. Restart deployment:
   ```bash
   kubectl rollout restart deployment/ollama -n spot-ai
   kubectl rollout status deployment/ollama -n spot-ai
   ```

## Prevention
- Monitor memory usage and set appropriate limits
- Use PVC for model persistence
- Set up proper health checks
```

## 🔧 Custom Metrics Implementation

### Adding Custom Metrics to Ollama

Ollama exposes metrics via its API. To add Prometheus metrics:

```python
# ollama-exporter.py
from prometheus_client import Counter, Histogram, Gauge, start_http_server

# Define metrics
ollama_requests = Counter('ollama_api_requests_total', 'Total API requests')
ollama_latency = Histogram('ollama_api_request_duration_seconds', 'Request latency')
ollama_model_loaded = Gauge('ollama_model_loaded', 'Loaded model', ['model'])
ollama_errors = Counter('ollama_api_errors_total', 'API errors', ['error_type'])

# Expose metrics endpoint
start_http_server(9111)
```

## 📊 Grafana Dashboard Import

To import the dashboard:

1. Open Grafana
2. Click "+" > "Import"
3. Upload the JSON file or paste the dashboard JSON
4. Select Prometheus data source
5. Click "Import"

## 🔍 Quick Reference Commands

```bash
# Check Ollama status
kubectl get pods -n spot-ai

# View logs
kubectl logs -n spot-ai deploy/ollama -f

# Check resource usage
kubectl top pod -n spot-ai

# Check PVC
kubectl get pvc -n spot-ai

# Test API
curl http://ollama.spot-ai.svc.cluster.local:11434/api/tags

# Port forward for local testing
kubectl port-forward -n spot-ai svc/ollama 11434:11434

# Restart deployment
kubectl rollout restart deployment/ollama -n spot-ai
```

---

**Document Version:** 1.0.0
**Last Updated:** 2026-07-19
