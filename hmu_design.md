# Health Monitoring Utility (HMU) - Design & Setup Document

**Version:** 0.2
**Date:** 2026-05-09
**Category:** (U) Utility / Administrative
**Acronym:** HMU

---

## 1. Introduction & Purpose

The **Health Monitoring Utility (HMU)** is a core utility module designed to provide real-time, centralized health status, telemetry, and traffic routing visibility for the entire SERA AI ecosystem. Eventually, HMU will be refactored into a persistent, headless gRPC service that acts as the backend for the **Sera Admin Module (SAM)** web interface. 

However, **Phase 1** of the HMU is a standalone Python-based CLI application that directly queries the Kubernetes API and Prometheus TSDB via the Rancher Proxy to build a "single pane of glass" for operators.

This document currently serves as the **Standard Operating Procedure (SOP) for replicating the K8s monitoring stack** across new global clusters (Prague, Slough, New York, etc.).

---

## 2. Cluster Deployment Guide (The HMU Monitoring Setup SOP)

To enable the HMU CLI tools (`gpu-status`, `vllm-status`, `nim-status`, etc.), the target Kubernetes cluster must be properly instrumented so Rancher's Prometheus can discover and scrape the AI engines.

*Note: This guide is written for operators who may not be Kubernetes experts. Follow the verification steps carefully to ensure you do not cause an accidental outage.*

### Step 2.1: Pre-Flight Verification (Look Before You Leap)

Before changing any configurations, let's look at the current state of the cluster to see what needs to be fixed.

**1. View current Services and their labels:**
This command lists all AI services, their current labels, and their exposed port names.
[bash]
kubectl get svc -n sera-llm -o custom-columns='NAME:.metadata.name,LABELS:.metadata.labels,PORTS:.spec.ports[*].name'
[/bash]
*(Look for `gia-monitoring:enabled` in the labels. If it is missing, we need to add it.)*

**2. View current NIM metrics configuration:**
NVIDIA NIMs hide their metrics by default. Let's check if they are currently enabled (`true`) or disabled (`<none>`).
[bash]
kubectl get nimservice -n sera-llm -o custom-columns='NAME:.metadata.name,METRICS_ENABLED:.spec.metrics.enabled'
[/bash]

### Step 2.2: Labeling & Patching Open-Source (vLLM) Deployments

vLLM pods expose Prometheus metrics automatically, but we must ensure their Service port is correctly named `vllm` and labeled so our `ServiceMonitor` can discover it.

**1. Standardize the Service Port Name (If necessary):**
If the "PORTS" column in Step 2.1 did not say `vllm`, patch it using this command (replace the service name):
[bash]
kubectl patch svc <vllm-service-name> -n sera-llm --type='json' -p='[{"op": "replace", "path": "/spec/ports/0/name", "value":"vllm"}]'
[/bash]

**2. Label the Services:**
Apply the HMU opt-in label to all vLLM services.
[bash]
kubectl label svc <vllm-service-1> <vllm-service-2> -n sera-llm gia-monitoring=enabled
[/bash]

### Step 2.3: Instrumenting Proprietary NVIDIA NIM Deployments

**1. Enable Metrics in the NIM Blueprint:**
Patch the custom resources to instruct the Operator to turn metrics on.
[bash]
kubectl patch nimservice <nim-name-1> <nim-name-2> -n sera-llm --type=merge -p '{"spec":{"metrics":{"enabled":true}}}'
[/bash]

**2. Force Operator Rollout (Restart):**
The Operator will not restart running pods automatically for this change. We must force a rolling restart.
[bash]
kubectl rollout restart deployment <nim-deployment-1> <nim-deployment-2> -n sera-llm
[/bash]

**3. VERIFY THE RESTART (Crucial Step):**
You must ensure the new pods come back online successfully. Run this command to watch the rollout status live. It will block your terminal until all pods are fully healthy.
[bash]
kubectl rollout status deployment <nim-deployment-1> -n sera-llm
[/bash]
*(Repeat the rollout status command for each NIM deployment you restarted. If it hangs or says failed, press `Ctrl+C` and check `kubectl get pods -n sera-llm` to see what is crashing).*

**4. Label the NIM Services:**
Once healthy, apply the HMU label.
[bash]
kubectl label svc <nim-name-1> <nim-name-2> -n sera-llm gia-monitoring=enabled
[/bash]

### Step 2.4: Deploying Prometheus Discovery Monitors

Now that the services are labeled and outputting metrics, we tell Prometheus where to find them using `ServiceMonitor` objects.

**Apply the following manifest to the cluster:**
[bash]
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: gia-ai-vllm-monitor
  namespace: sera-llm
  labels:
    release: rancher-monitoring
spec:
  selector:
    matchLabels:
      gia-monitoring: enabled
  endpoints:
  - port: vllm
    path: /metrics
    interval: 15s
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: gia-ai-nim-monitor
  namespace: sera-llm
  labels:
    release: rancher-monitoring
spec:
  selector:
    matchLabels:
      gia-monitoring: enabled
  endpoints:
  - port: api
    path: /v1/metrics
    interval: 15s
EOF
[/bash]

### Step 2.5: HMU Application Configuration (`settings.yaml`)

HMU requires a Rancher API key and the internal Cluster ID to successfully route queries through the Rancher Proxy.

**1. Retrieve the internal `cluster_id` via CLI:**
Rancher tags every namespace with a project ID that includes the cluster ID. We can extract it easily with this command:
[bash]
kubectl get namespace default -o jsonpath='{.metadata.annotations.field\.cattle\.io/projectId}' | cut -d: -f1 ; echo
[/bash]
*(Expected output: A string looking something like `c-m-v2sk8v8v`)*

**2. Generate a Rancher API Key:**
* Log into the Rancher Web UI.
* Click your profile picture/icon in the top right -> **Account & API Keys**.
* Click **Create API Key**.
* Copy the generated `Bearer Token` (It will look like `token-xxxxx:yyyyyyyyyyy`).

**3. Update `settings.yaml`:**
Open your HMU `settings.yaml` file and add the new cluster location to the `rancher_clusters` block:
[yaml]
rancher_clusters:
  slo:
    url: "https://<rancher-fqdn>"
    cluster_id: "<cluster_id_from_step_1>"
    token: "<bearer_token_from_step_2>"
[/yaml]

### Step 2.6: Final Validation & Testing

Run these CLI commands from the HMU utility server to verify data is flowing correctly.

**1. Refresh the Physical Topology:**
Forces the database to rebuild the K8s mapping.
[bash]
python health_monitor_utility.py refresh-map
[/bash]

**2. Verify vLLM Telemetry:**
[bash]
python health_monitor_utility.py vllm-status
[/bash]
*(Expected: A table showing active requests, KV cache %, and TPS for vLLM models in the new cluster).*

**3. Verify NIM Telemetry:**
[bash]
python health_monitor_utility.py nim-status
[/bash]
*(Expected: A table showing active NIM models with a "✅ Active" status).*

**4. Check for Metric Discovery (Troubleshooting):**
If a status command returns empty, verify that Prometheus actually sees the new metrics.
[bash]
python health_monitor_utility.py discover-metrics vllm
[/bash]
*(Expected: A list of metric names. If empty, Prometheus has not picked up the ServiceMonitor yet. Wait 2 minutes and try again).*

---

## 3. Future State: Enterprise Telemetry & Diagnostics (Phase 2)

Once the CLI stack is validated across all global deployments, HMU will transition into a persistent, headless gRPC service. To make HMU a true enterprise MLOps backend for the SAM UI, the following capabilities will be implemented:

### 3.1. Telemetry Data Warehouse (Patroni Aggregator)
Prometheus is optimal for real-time scraping but lacks the retention for long-term historical and chargeback analysis. HMU will leverage the 7-node Patroni PostgreSQL cluster for data warehousing:
* **Interval Downsampling:** A background scheduler will aggregate instantaneous metrics every 5-15 minutes. 
* **Weekly Rollups:** After 7 days, the high-fidelity interval data will be rolled up into hourly summaries to maintain long-term performance trends without bloating the database.

### 3.2. Asynchronous Error & Event Caching
To provide operators with immediate context during an outage without requiring manual `kubectl` log tails, HMU will proactively cache critical states:
* **Targeted Log Scraping:** Continuously query AI engine pods for known critical failure strings (e.g., `CUDA OutOfMemory`, `Triton Error`, `NCCL Timeout`).
* **Kubernetes Event Capture:** Scrape the Kubernetes Events API to catch infrastructure-level failures (e.g., `OOMKilled`, `NodeNotReady`) that cause a pod to vanish before it can log an error.

### 3.3. External HAProxy Config Inspector (Data Plane API)
Auditing the external VMs to ensure physical traffic routing aligns with the internal Kubernetes state is crucial.
* **HAProxy Data Plane API:** The external HAProxy VMs (Prague, Slough, NYH) will be configured to expose the official HAProxy Data Plane REST API. HMU will query this API to retrieve the exact routing configuration as structured JSON, rather than relying on SSH and regex file parsing.
* **Firewall Prerequisites:** Network ACLs must be updated to allow traffic on the Data Plane API port from the HMU utility server (`ukslo-crp-ai01`) to the external HAProxy VMs.

### 3.4. AI-Specific Service Level Indicators (SLIs)
HMU will explicitly define and aggregate the core metrics required to chart business-level AI SLAs:
* **Time-To-First-Token (TTFT):** The most critical metric for perceived LLM performance.
* **Cache Hit Ratios:** Tracking prefix/KV cache hit rates over time to validate the computational efficiency of prompt caching.