# Health Monitoring Utility (HMU) - Design Document

**Version:** 0.3.3
**Date:** 2026-07-17
**Author:** Martin Kecka & Gemini AI
**Category:** (U) Utility / Administrative
**Acronym:** HMU

---

## 1. Introduction & Purpose

The **Health Monitoring Utility (HMU)** is a core utility module designed to provide real-time, centralized health status, telemetry, and traffic routing visibility for the entire SERA AI ecosystem. Eventually, HMU will be refactored into a persistent, headless gRPC service that acts as the backend for the **Sera Admin Module (SAM)** web interface. 

The HMU now operates as a **persistent, headless gRPC backend** providing programmatic monitoring via the `hmu.proto` contract. It natively aggregates telemetry across Rancher Kubernetes, standalone VMs, and HAProxy load balancers.

This document also serves as the **Standard Operating Procedure (SOP) for replicating the K8s monitoring stack** across new global clusters (Prague, Slough, New York, etc.).

---

## 2. Core Architectural Principles 🏗️

1. **The Watcher (Read-Only Observability):** HMU is strictly a telemetry aggregator and observer. Unlike OMU (which mutates infrastructure) or AMU (which deploys logical apps), HMU evaluates state without making destructive changes.
2. **Multi-Platform Native Support:** Discovers and monitors dynamic deployments across on-premise hardware, Kubernetes clusters (Rancher), and global external load balancers (HAProxy).
3. **Smart Interoperability:** Automatically injects Keycloak tokens to resolve Smart Auth blocks when validating proxied Endpoints.
4. **Resilient Data Aggregation:** Designed to pull logs and metrics quickly to its own local PostgreSQL schema before upstream Kubernetes limits drop data (e.g., catching OOM errors before pod eviction).
5. **Ecosystem Compliance:** HMU fully integrates the `sera_standard_service.py` architecture for unified multi-service tracking and deep health dependency aggregation.

---

## 3. Database Schema & State Management 🗄️

HMU manages its own PostgreSQL schema (`hmu_utility_db`) natively supporting HA Routing (`write_dsn`/`read_dsn`) to persistently store scraped metrics, topologies, and events.

### 3.1. Metric & Log Aggregation
* **Event Catching:** Stores tracebacks, out-of-memory logs, and massive token batch payloads into PostgreSQL before Kubernetes log rotation drops them.
* **Topology State:** Constantly refreshes the physical topology maps between load balancers, clusters, and bare metal servers.
* **Performance Telemetry:** Retains time-series snapshots of GPU availability and Time-To-First-Token (TTFT) metrics for AI model endpoints.

### 3.2. Usage & Billing Aggregation (`UsageModelStat`)
* **30-Day Sliding Window:** A persistent `sync_daily_usage_stats` daemon recursively polls Prometheus and synchronizes AI inference billing metrics (token volume, inference requests, and GPU utilization) down to atomic daily intervals directly into PostgreSQL.

---

## 4. Enterprise Telemetry & Diagnostics (Core Sub-Systems) ⚙️

HMU is implemented as a persistent, headless gRPC service serving as the enterprise MLOps backend for the SAM UI. Key architectural capabilities include:

### 4.1. Advanced Topology & Workload Discovery
* **Broad Engine Support:** Dynamically discovers and monitors vLLM deployments, proprietary NVIDIA NIMs, and complex Triton Inference Server / Morpheus pipelines (via the `halo` namespace).
* **Deep Hardware & Storage Insight:** Parses OS and GPU telemetry natively. The `TopologyBuilder` leverages concurrent Kube-API remote execution (`nvidia-smi` and cgroup parsing) to precisely track VRAM usage per deployed model. It also natively queries PersistentVolumeClaims to extract `pvc_name`, `pvc_size`, and the underlying NetApp Trident PV string (`pvc_id_name`).
* **Active PVC Utilization Monitoring:** Synthesizes `kubelet_volume_stats_used_bytes` and `kubelet_volume_stats_capacity_bytes` to stream real-time disk consumption percentages directly for AI engine pods.

### 4.2. Granular Health & Latency Tracking
* **Smart Auth & Discovery:** HMU utilizes 'Smart Auth' to detect Envoy-proxied endpoints dynamically, injecting Keycloak tokens automatically to resolve authentication blocks during synthetic HTTP pings.
* **Segmented Latency (TTFT):** The `HealthCheckResult` breaks down end-to-end response times into precise `route_latency_ms`, `identity_latency_ms`, and `inference_latency_ms` (Time-To-First-Token), enabling operators to isolate bottlenecks immediately.
* **Deep Degraded Pod Telemetry:** Upgraded algorithms parse failing container statuses, streaming explicit Kubernetes crash loops, exit codes, restarts, and reasons natively over gRPC via `DegradedPodInfo`.
* **Persistent Scavenger Daemon:** A background task persistently scrapes vLLM and Triton pod logs, saving critical tracebacks, out-of-memory errors, and massive token batch payloads into PostgreSQL before Kubernetes log rotation drops them.

### 4.3. Autonomous Model Benchmarking (`TestingManager`)
* **Feature Validation:** The backend autonomously builds and routes payloads to test `/chat/completions`, `/embeddings`, `/ranking`, `vision`, and `reasoning` endpoints. It now supports isolated feature checks for dynamic LoRA adapters via the `--lora` parameter.
* **Auto-Scaling Stress Tests:** Drives high-concurrency payload streams against models until encountering hardware saturation or latency cliffs, parsing detailed diagnostic errors directly into the gRPC response.

### 4.4. Cache & Infrastructure Observability
* **Independent Cache Auditing:** Completely decoupled from provisioning tools, HMU leverages its own Vault integration to independently query Valkey and Redis cache health, returning statuses via `GetValkeyClusterStatus` and `GetValkeyInspectStatus`.
* **External Dataplane Inspection:** HMU proxies HAProxy dataplane stats into the centralized aggregator, mapping internal cluster routing to external ingress VM realities.

---

## 5. Module Structure 📂

* `cli/`: Commands for inspecting metrics (e.g., `gpu_cmds.py`, `nim_cmds.py`, `vllm_cmds.py`, `usage_cmds.py`).
* `config/`: Contains `settings.yaml` to securely separate `system_services` (API and gRPC endpoints) from `external_services` (Vault, Keycloak, Rancher integrations).
* `core/`: The logic core, holding various monitors (`gpu_monitor.py`, `ai_engine_monitor.py`), the `testing_manager.py`, and `topology_builder.py`.
* `database/`: PostgreSQL ORM models to store latency and event telemetry.
* `grpc_api/`: The gRPC interfaces `hmu_service.py` to stream real-time data back to the UI.

---

## 6. Cluster Deployment Guide (The HMU Monitoring Setup SOP) 🛠️

To enable the HMU CLI tools (`gpu-status`, `vllm-status`, `nim-status`, etc.), the target Kubernetes cluster must be properly instrumented so Rancher's Prometheus can discover and scrape the AI engines.

*Note: This guide is written for operators who may not be Kubernetes experts. Follow the verification steps carefully to ensure you do not cause an accidental outage.*

### Step 6.1: Pre-Flight Verification (Look Before You Leap)

Before changing any configurations, let's look at the current state of the cluster to see what needs to be fixed.

**1. View current Services and their labels:**
This command lists all AI services, their current labels, and their exposed port names.
```bash
kubectl get svc -n sera-llm -o custom-columns='NAME:.metadata.name,LABELS:.metadata.labels,PORTS:.spec.ports[*].name'
```
*(Look for `gia-monitoring:enabled` in the labels. If it is missing, we need to add it.)*

**2. View current NIM metrics configuration:**
NVIDIA NIMs hide their metrics by default. Let's check if they are currently enabled (`true`) or disabled (`<none>`).
```bash
kubectl get nimservice -n sera-llm -o custom-columns='NAME:.metadata.name,METRICS_ENABLED:.spec.metrics.enabled'
```

### Step 6.2: Labeling & Patching Open-Source (vLLM) Deployments

vLLM pods expose Prometheus metrics automatically, but we must ensure their Service port is correctly named `vllm` and labeled so our `ServiceMonitor` can discover it.

**1. Standardize the Service Port Name (If necessary):**
If the "PORTS" column in Step 6.1 did not say `vllm`, patch it using this command (replace the service name):
```bash
kubectl patch svc <vllm-service-name> -n sera-llm --type='json' -p='[{"op": "replace", "path": "/spec/ports/0/name", "value":"vllm"}]'
```

**2. Label the Services:**
Apply the HMU opt-in label to all vLLM services.
```bash
kubectl label svc <vllm-service-1> <vllm-service-2> -n sera-llm gia-monitoring=enabled
```

### Step 6.3: Instrumenting Proprietary NVIDIA NIM Deployments

**1. Enable Metrics in the NIM Blueprint:**
Patch the custom resources to instruct the Operator to turn metrics on.
```bash
kubectl patch nimservice <nim-name-1> <nim-name-2> -n sera-llm --type=merge -p '{"spec":{"metrics":{"enabled":true}}}'
```

**2. Force Operator Rollout (Restart):**
The Operator will not restart running pods automatically for this change. We must force a rolling restart.
```bash
kubectl rollout restart deployment <nim-deployment-1> <nim-deployment-2> -n sera-llm
```

**3. VERIFY THE RESTART (Crucial Step):**
You must ensure the new pods come back online successfully. Run this command to watch the rollout status live. It will block your terminal until all pods are fully healthy.
```bash
kubectl rollout status deployment <nim-deployment-1> -n sera-llm
```
*(Repeat the rollout status command for each NIM deployment you restarted. If it hangs or says failed, press `Ctrl+C` and check `kubectl get pods -n sera-llm` to see what is crashing).*

**4. Label the NIM Services:**
Once healthy, apply the HMU label.
```bash
kubectl label svc <nim-name-1> <nim-name-2> -n sera-llm gia-monitoring=enabled
```

### Step 6.4: Deploying Prometheus Discovery Monitors

Now that the services are labeled and outputting metrics, we tell Prometheus where to find them using `ServiceMonitor` objects.

**Apply the following manifest to the cluster:**
```bash
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
```

### Step 6.5: HMU Application Configuration (`settings.yaml`)

HMU requires a Rancher API key and the internal Cluster ID to successfully route queries through the Rancher Proxy.

**1. Retrieve the internal `cluster_id` via CLI:**
Rancher tags every namespace with a project ID that includes the cluster ID. We can extract it easily with this command:
```bash
kubectl get namespace default -o jsonpath='{.metadata.annotations.field\.cattle\.io/projectId}' | cut -d: -f1 ; echo
```
*(Expected output: A string looking something like `c-m-v2sk8v8v`)*

**2. Generate a Rancher API Key:**
* Log into the Rancher Web UI.
* Click your profile picture/icon in the top right -> **Account & API Keys**.
* Click **Create API Key**.
* Copy the generated `Bearer Token` (It will look like `token-xxxxx:yyyyyyyyyyy`).

**3. Update `settings.yaml`:**
Open your HMU `settings.yaml` file and add the new cluster location to the `external_services` block under `rancher_clusters`:
```yaml
external_services:
  rancher_clusters:
    slo:
      url: "https://<rancher-fqdn>"
      cluster_id: "<cluster_id_from_step_1>"
      token: "<bearer_token_from_step_2>"
```

### Step 6.6: Final Validation & Testing

Run these CLI commands from the HMU utility server to verify data is flowing correctly.

**1. Refresh the Physical Topology:**
Forces the database to rebuild the K8s mapping.
```bash
python3 health_monitor_utility.py refresh-map
```

**2. Verify vLLM Telemetry:**
```bash
python3 health_monitor_utility.py vllm-status
```
*(Expected: A table showing active requests, DISK GiB, KV cache %, and TPS for vLLM models in the new cluster).*

**3. Check Token Billing Usage:**
```bash
python3 health_monitor_utility.py usage-stats
```

---

## 7. Roadmap & Future Enhancements 🚀

While the core HMU gRPC backend is active, the following enterprise capabilities remain on the development roadmap:

### 7.1. Long-Term Data Warehousing
HMU natively polls Prometheus for usage and billing rollups to daily intervals (via `sync_daily_usage_stats`). However, broader enterprise metrics (like specific latency degradation patterns) will eventually leverage the Patroni cluster for long-term historical and chargeback analysis beyond Prometheus's standard retention window.

### 7.2. Asynchronous Event Caching
* **Kubernetes Event Capture:** Scrape the Kubernetes Events API to catch infrastructure-level failures (e.g., `OOMKilled`, `NodeNotReady`) that cause a pod to vanish before it can log an error.

### 7.3. External HAProxy Config Inspector (Data Plane API)
Auditing the external VMs to ensure physical traffic routing aligns with the internal Kubernetes state is crucial.
* **HAProxy Data Plane API:** The external HAProxy VMs (Prague, Slough, NYC) will be configured to expose the official HAProxy Data Plane REST API. HMU will query this API to retrieve the exact routing configuration as structured JSON, rather than relying on SSH and regex file parsing.
* **Firewall Prerequisites:** Network ACLs must be updated to allow traffic on the Data Plane API port from the HMU utility server to the external HAProxy VMs.

### 7.4. AI-Specific Service Level Indicators (SLIs)
HMU will explicitly define and aggregate the core metrics required to chart business-level AI SLAs:
* **Cache Hit Ratios:** Tracking prefix/KV cache hit rates over time to validate the computational efficiency of prompt caching.

### 7.5. Separation of Observability and Provisioning
While Redis and Valkey global cache auditing logic has been centralized into HMU (`valkey-cluster`, `valkey-inspect`, `redis-inspect`), future roadmap items include:
* **PostgreSQL:** Extract all read-only DB monitoring (e.g., `db-activity`, `db-status`, `db-cluster`) from legacy scripts and into HMU to monitor the global Patroni deployments and local utility databases.
* **Vector Databases:** Implement global inspection/auditing for both local Chroma DBs and production Qdrant clusters directly within HMU.