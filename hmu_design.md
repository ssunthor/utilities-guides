# Health Monitor Utility (HMU) - Architecture & Design Document

**Doc_Type:** Architecture & Design
**Category:** (U) Utility
**Name:** Health Monitor Utility
**Acronym:** HMU
**Version:** 0.4.2
**Date:** 2026-10-12
**Author:** Martin Kecka & Gemini AI
**Status:** Active

---

## 1. Introduction & Purpose 📜

The **Health Monitor Utility (HMU)** is a dedicated administrative microservice and CLI control plane within the SERA/GIA AI ecosystem engineered to deliver real-time **Global Telemetry Aggregation**, **Physical Topology Introspection**, **Tiered Synthetic Health Probing**, and **Autonomous Prompt Analytics** across multi-region AI Factory clusters (**Prague (`PRG`)**, **Slough (`SLO`)**, and **New York (`NYC`)**).

Operating as an autonomous, standalone observability control plane, HMU provides engineers and operators with an interactive Click CLI for real-time GPU/node diagnostics, model benchmarking, cache inspection, and log scavenging. Concurrently, it serves as the primary observability and telemetry backend engine powering the **Sera Admin Module (SAM)** web interface. HMU acts as **The Watcher**—enforcing a strict read-only boundary that observes, tests, and aggregates system health without mutating infrastructure state.

HMU persists its operational state and metrics in HA PostgreSQL (`sam_admin_prod_db`), stores raw prompt payloads in a local disk vault (`/ai/assets/vault/health_monitor_utility/payloads`), and autonomously classifies AI traffic into business categories using a dedicated Gemma-4 cognitive engine.

Its primary operational capabilities include:
- **Multi-Cluster Hardware & Engine Telemetry:** Real-time scraping of NVIDIA DCGM GPU metrics, Host OS metrics, vLLM V1 engine metrics, NVIDIA NIM metrics, and NVIDIA Triton/Morpheus (Halo) pipelines via Rancher-proxied Prometheus instances.
- **Physical Topology & Remote VRAM Mapping:** Automated discovery of Kubernetes nodes, GPU indices, PersistentVolumeClaims (NetApp Trident), and per-model MiB VRAM consumption via concurrent remote `kubectl exec` (`nvidia-smi` + `cgroup` parsing), focusing on physical GPU hardware state and memory saturation (contrasting with CMU's declarative GitOps and manifest drift tracking).
- **Tiered Synthetic Health Checks:** High-concurrency, GIL-isolated (`ProcessPoolExecutor`) end-to-end synthetic probing across **KUBE**, **VM**, **F5**, and **INT-LB** routing layers with segmented latency tracking (`route_latency_ms`, `identity_latency_ms`, `inference_latency_ms` / TTFT) and Smart DNS fallback.
- **Autonomous Prompt Scavenging & Self-Learning Classification:** Real-time streaming of vLLM pod logs to capture raw prompt payloads onto disk, extract token counts, LoRA adapters, sampling parameters, and consumer `client_id` tags, and autonomously classify workloads into business categories using **Gemma-4 E4B** (`google/gemma-4-E4B-it`).
- **Global Cache & Ingress Auditing:** Multi-cluster inspection of global 9-node Valkey clusters, local Redis instances (`__GIA_DB_OWNER__` / `__SERA_DB_OWNER__` tagging), Kubernetes HAProxy Ingress controllers, and external VM HAProxy CSV statistics.
- **Isolated Model Testing & Auto-Scaling Stress Benchmarks:** Programmatic validation of `/v1/chat/completions`, `/v1/embeddings`, `/v1/ranking`, vision, reasoning, and LoRA adapters, alongside 1:1 concurrency stress benchmarking.

---

## 2. Architecture & Ecosystem Integration 🏗️

HMU operates on a client-server architecture, exposing dual gRPC servicers backed by a multi-worker thread pool (`hmu-grpc.service`), while providing operators with a standalone Click CLI and autonomous background telemetry daemons.

### 2.1. Dual Servicer & Core Architectural Principles
- **Dual Servicer Model:**
  1. **`HealthMonitorUtility` (`hmu.proto`):** Domain servicer exposing 12 RPCs for high-frequency hardware, engine, cache, traffic, and topology telemetry, tiered synthetic health probing, and auto-scaling benchmarks.
  2. **`SeraAppService` (`sera_standard.proto`):** Implements the universal SERA ecosystem standard contract (maintained centrally by AMU) for self-reporting HMU's metadata, deep health matrix, documentation, and changelog.
- **The Watcher (Strict Read-Only Observability):** HMU is purely a telemetry aggregator, diagnostic prober, and observer. It strictly evaluates state without executing destructive mutations.
- **Multi-Platform Heterogeneous Observability:** Unifies telemetry across bare-metal GPU nodes (NVIDIA DCGM), Kubernetes AI workloads (vLLM, NVIDIA NIM, Triton, Morpheus), PersistentVolumeClaims (NetApp Trident), external VM load balancers (HAProxy CSV), and global Valkey/Redis cache clusters.
- **GIL-Isolated High-Concurrency Probing:** Offloads CPU-intensive TLS handshakes and synthetic HTTP probing to a pre-warmed 12-worker multiprocessing pool (`ProcessPoolExecutor`), completely bypassing Python Global Interpreter Lock (GIL) contention to ensure microsecond-level timing accuracy.
- **Decoupled Autonomous Intelligence:** Gathers telemetry independently without relying on upstream schedulers, salvages pod logs into PostgreSQL and local disk before Kubernetes log rotation drops them, and autonomously learns workload classifications via a dedicated local Gemma-4 E4B inference loop.
- **High-Availability Persistence Isolation:** Utilizes split read/write SQLAlchemy connection pools (`write_dsn` on `:5432` and `read_dsn` on `:5433`) with `pool_pre_ping=True` and connection-hold prevention.
- **Dynamic Provenance Tracing:** Propagates and logs `x-client-id` metadata headers (`[SAM]`, `[CLI]`, `[HMU]`) across all RPC invocations for end-to-end operation auditing.

### 2.2. Ecosystem Integration & Data Flow

```mermaid
graph TD
    subgraph Clients ["Clients, Operators & Control Plane"]
        SAM["Sera Admin Module (SAM UI)"]
        CLI["HMU Click CLI (hmu_cli.py)"]
        AMU["Application Management Utility (AMU)"]
    end

    subgraph HMU_Service ["HMU gRPC Server (0.0.0.0:50101)"]
        GRPC["gRPC Servicers<br/>(HealthMonitorUtility & SeraAppService)"]
        
        subgraph Core_Monitors ["Core Domain Monitors"]
            GM["GpuMonitor & NodeMonitor"]
            AEM["AiEngineMonitor"]
            PC["PayloadClassifier<br/>(Scavenger + Auto-Classifier)"]
            TM["TopologyMonitor & TopologyBuilder"]
            AHM["ActiveHealthMonitor<br/>(12-Worker ProcessPool)"]
            TRM["TrafficMonitor"]
            CM["CacheMonitor"]
            TEST["TestingManager"]
        end
    end

    subgraph Persistence ["State, Storage & Secrets"]
        PG_W[("PostgreSQL Primary<br/>(write_dsn :5432)")]
        PG_R[("PostgreSQL Replica<br/>(read_dsn :5433)")]
        DISK[("Local Vault Disk<br/>(/ai/assets/vault/.../payloads)")]
        REGEX["In-Memory Shadow Copy<br/>(core/database/regex.py)"]
        VAULT["HashiCorp Vault<br/>(vault-aif.gt-t.net:8200)"]
    end

    subgraph Global_Infra ["Global Multi-Cluster Infrastructure (PRG | SLO | NYC)"]
        RANCHER["Rancher K8s API Proxy & Prometheus<br/>(sera-llm, halo, gpu-operator, thirdlaw-platform)"]
        HAPROXY["External VM HAProxy (:9000/;csv)<br/>& K8s Ingress Controllers"]
        VALKEY["Global Valkey Cluster (9 Nodes)<br/>& Local Redis Instances"]
        ENDPOINTS["AI Model Endpoints<br/>(KUBE / VM / F5 / INT-LB)"]
        LLM_SEC["Gemma-4 E4B Endpoint<br/>(aif-llm-gemma4-e4b.aif.gt-t.net)"]
    end

    SAM -->|gRPC :50101| GRPC
    CLI -->|gRPC :50101| GRPC
    AMU -->|GetAppInfo / GetDeepHealth| GRPC

    GRPC --> GM
    GRPC --> AEM
    GRPC --> TM
    GRPC --> AHM
    GRPC --> TRM
    GRPC --> CM
    GRPC --> TEST
    AEM --> PC

    GRPC & Core_Monitors -->|AppRole Auth| VAULT
    PC & TM & AEM -->|SQLWriter| PG_W
    AEM & TM & PC -->|SQLReader| PG_R
    PC <-->|R/W .txt & .json| DISK
    PC <-->|Bi-Directional Sync| REGEX

    GM & AEM & TM & PC -->|Bearer Token| RANCHER
    TRM -->|HTTP Basic / PromQL| HAPROXY
    CM -->|Redis Protocol :6379| VALKEY
    AHM & TEST -->|Static API Key sk_aif_...| ENDPOINTS
    PC -->|Generator Prompt| LLM_SEC
```

---

## 3. Module File Map 📂

```plaintext
/ai/utilities/health_monitor_utility
├── bootstrap_helper.py                  # Virtual environment bootstrap and dependency verification helper
├── health_monitor_utility.py            # Main executable entrypoint; configures env vars, loads YAML, initializes logging & CLI
├── requirements.txt                     # Python package dependencies (grpcio, protobuf, SQLAlchemy, click, hvac, redis, etc.)
├── README.md                            # Repository overview and links to central documentation
├── cli/
│   ├── __init__.py                      # CLI package initializer
│   ├── hmu_cli.py                       # Root Click group (NestedHelpGroup), stub injection, and start-grpc-server command
│   ├── utils.py                         # MetadataClientInterceptor (x-client-id), gRPC stub factories, JSON printer, error decorator
│   └── commands/
│       ├── __init__.py                  # Commands package initializer
│       ├── cache_cmds.py                # CLI: valkey-cluster, valkey-inspect, redis-inspect
│       ├── gpu_cmds.py                  # CLI: gpu-status (global DCGM telemetry)
│       ├── halo_cmds.py                 # CLI: halo-status (NVIDIA Triton inference telemetry)
│       ├── health_cmds.py               # CLI: model-health (tiered KUBE -> VM -> F5 -> INT-LB synthetic checks)
│       ├── kube_cmds.py                 # CLI: model-config (untruncated engine parameters) and raw-pod (100% raw K8s JSON)
│       ├── log_cmds.py                  # CLI: payload-history, payload-stream (BatchDescriptor scanner), cleanup-ghosts
│       ├── map_cmds.py                  # CLI: cluster-map (physical GPU topology + VRAM) and refresh-map (streaming sync)
│       ├── module_cmds.py               # CLI: module info and module check-health (SERA Standard gRPC consumer)
│       ├── nim_cmds.py                  # CLI: nim-status (NVIDIA NIM telemetry)
│       ├── node_cmds.py                 # CLI: node-status (single-node Host OS + GPU hardware drill-down)
│       ├── prom_cmds.py                 # CLI: discover-metrics (Prometheus metric name regex search)
│       ├── prompt_cmds.py               # CLI group 'prompt': overview, list, largest, usage, content, category
│       ├── test_cmds.py                 # CLI: test-model (isolated feature tests) and benchmark-model (streamed stress test)
│       ├── traffic_cmds.py              # CLI: traffic-status (K8s HAProxy Ingress) and vm-traffic (external VM HAProxy CSV)
│       ├── usage_cmds.py                # CLI: usage-stats (daily inference and token billing rollups)
│       └── vllm_cmds.py                 # CLI: vllm-status (real-time and historical trend tables for vLLM pods)
├── config/
│   └── settings.yaml                    # Core configuration: gRPC server, DB DSNs, Vault AppRole, Rancher clusters, HAProxy, LLMs
├── core/
│   ├── __init__.py                      # Core package initializer
│   ├── constants.py                     # DECOMMISSIONED_MODELS, MODEL_QUIRKS, and LEGACY_MODEL_NAME_MAP static definitions
│   ├── health_manager.py                # ProcessPoolExecutor (12 workers) synthetic health check engine & Smart DNS fallback
│   ├── kube_manager.py                  # Rancher K8s REST API client with connection pooling and thread-safe Vault token refresh
│   ├── llm_interface.py                 # vLLMClient wrapper with Vault API key injection and exponential backoff retry logic
│   ├── prometheus_manager.py            # Rancher-proxied Prometheus query/query_range client with automatic 401/403 token retry
│   ├── testing_manager.py               # Feature validation and auto-scaling concurrency benchmark engine
│   ├── topology_builder.py              # K8s pod/PVC parser, remote kubectl exec VRAM mapper, and DB topology state synchronizer
│   ├── vault_manager.py                 # Legacy hvac helper functions for Vault secret listing and retrieval
│   ├── clients/
│   │   ├── haproxy_client.py            # HAProxyVMClient for fetching and parsing raw CSV statistics from external VMs
│   │   └── vault_client.py              # VaultClient (AppRole login, get_llm_key, get_rancher_tokens with proxy bypass)
│   ├── database/
│   │   ├── database.py                  # SQLAlchemy read/write engine initialization, QueuePool config, and get_db_session
│   │   ├── models.py                    # ORM Models: HardwareNode, HardwareGpu, AiDeployment, PromptCategory, PromptSignature, TokenPayload, UsageModelStat
│   │   └── regex.py                     # Auto-generated bi-directional shadow copy of prompt classification regex signatures
│   ├── io_handlers/
│   │   ├── sql_reader.py                # Read-only SQLAlchemy query abstractions (nodes, deployments, payloads, quantiles, clients)
│   │   └── sql_writer.py                # Write/mutation SQLAlchemy abstractions (topology sync, bulk payload upsert, client backfill)
│   └── monitors/
│       ├── active_health_monitor.py     # Multi-cluster orchestrator for execute_health_checks and location/routing health rollups
│       ├── ai_engine_monitor.py         # Orchestrator for vLLM, NIM, Halo status, prompt analytics, and daily usage stats daemon
│       ├── cache_monitor.py             # Global Valkey 9-node cluster auditor, global DB inspector, and local Redis DB scanner
│       ├── gpu_monitor.py               # Multi-cluster DCGM GPU and Host OS metric aggregator with historical range queries
│       ├── node_monitor.py              # Targeted single-node OS and GPU real-time + historical Prometheus query engine
│       ├── payload_classifier.py        # Background daemons for vLLM log scavenging, disk storage, and LLM auto-classification
│       ├── topology_monitor.py          # Thread-locked RefreshClusterMap generator, GetClusterMap, and GetModelConfig handler
│       └── traffic_monitor.py           # K8s HAProxy Ingress Prometheus metric aggregator and external VM HAProxy stats collector
├── docs/
│   ├── CHANGELOG.yaml                   # Structured release history following Ecosystem Documentation Standards
│   └── hmu_design.md                    # Symlink to /ai/assets/guides/utilities_guides/hmu_design.md
├── grpc_api/
│   ├── __init__.py                      # gRPC API package initializer
│   ├── generate_stubs.py                # Automated mtime-aware protoc stub compiler for hmu.proto and sera_standard.proto
│   ├── hmu_service.py                   # HealthMonitorUtilityServicer implementation mapping all 21 HMU RPCs
│   ├── sera_standard_service.py         # SeraAppServiceServicer implementation (GetAppInfo, GetDeepHealth, GetDocs, GetChangelog)
│   ├── services.py                      # gRPC server bootstrapper (ThreadPoolExecutor, HealthServicer, Server Reflection)
│   └── protos/
│       └── hmu.proto                    # Protocol Buffers v3 contract defining HealthMonitorUtility service and messages
├── prompts/
│   └── classification_prompts.py        # GENERATOR_PROMPT and EVALUATOR_PROMPT templates for LLM payload categorization
├── scripts/                             # Dumping ground for one-time operational, repair, and test scripts (ephemeral, not part of runtime)
└── utils/
    ├── logging_setup.py                 # MultiProcessSafeDailyLogger (14-day retention) and console/file handler setup
    └── rpc_transaction_logger.py        # JSON transaction logger and retention pruner
```

---

## 4. Core Processing Lifecycle ⚙️

### 4.1. Boot & Initialization Sequence
1. **Environment Sanitization (`health_monitor_utility.py`):** Sets `GRPC_ENABLE_FORK_SUPPORT="0"` and `GRPC_VERBOSITY="ERROR"` prior to importing `grpc` to prevent C-core fork deadlocks and log spam when spawning multiprocessing workers.
2. **Configuration Loading & Normalization:** Loads `config/settings.yaml` and normalizes all cluster keys in `rancher_clusters` and `vm_haproxy` to uppercase, migrating legacy `nyh`/`nyc` keys strictly to `NYC`.
3. **Logging Setup (`utils/logging_setup.py`):** Configures `MultiProcessSafeDailyLogger` writing to `/ai/assets/logs/hmu_utility/hmu.log.YYYY-MM-DD` (`DEBUG`) and `hmu_err.log.YYYY-MM-DD` (`WARNING`) with a 14-day retention policy. Console output is set to `INFO` in server mode (`start-grpc-server`) and `WARNING` in CLI mode.
4. **Dynamic Stub Compilation (`grpc_api/generate_stubs.py`):** When `start-grpc-server` is invoked, HMU compares the modification timestamps (`mtime`) of `hmu.proto` and `sera_standard.proto` against `/ai/assets/protos/health_monitor_utility/generated/`. If missing or outdated, it compiles the stubs via `grpc_tools.protoc` and re-executes the Python process via `os.execv(sys.executable, [sys.executable, "-u"] + sys.argv)`.
5. **Database Initialization (`core/database/database.py`):** Initializes dual SQLAlchemy `QueuePool` engines (`_write_engine` on port `5432`, `_read_engine` on port `5433`, `pool_size=5`, `max_overflow=10`, `pool_pre_ping=True`) and runs `Base.metadata.create_all()` on the write engine.
6. **Vault Credential Hydration (`grpc_api/hmu_service.py`):** Instantiates `VaultClient` using AppRole credentials (`role_id` and `secret_id`), fetches Rancher bearer tokens from `secret/rancher/{loc}/system/gia_ai`, and binds them to `KubeManager` and `PrometheusManager` instances for `PRG`, `SLO`, and `NYC`.
7. **Process Pool Pre-Warming (`core/health_manager.py`):** `ActiveHealthMonitor` calls `prewarm_health_check_pool(max_workers=12)` during startup, spawning 12 persistent worker processes before heavy memory allocations occur, reducing idle RAM footprint by ~85% and eliminating per-request fork overhead.
8. **Background Daemon Launch:** `AiEngineMonitor.start_background_tasks()` spawns the **Usage Stats Sync Daemon**, **Startup Regex/DB Backfill Thread**, **Payload Scavenger Daemon**, and **Auto-Classifier Daemon**.

### 4.2. Runtime Request Lifecycle
1. **Inbound gRPC Request:** Client (`SAM` or `hmu_cli.py`) sends a request with metadata header `x-client-id: HMU` (injected via `MetadataClientInterceptor`).
2. **Location Sanitization:** `HealthMonitorUtilityService._sanitize_loc()` normalizes any location parameter (`nyh`/`nyc` -> `NYC`, others -> `.upper()`).
3. **Concurrent Multi-Cluster Execution:** Monitor classes (`GpuMonitor`, `AiEngineMonitor`, `TrafficMonitor`, `ActiveHealthMonitor`) fan out requests across all configured clusters (`PRG`, `SLO`, `NYC`) concurrently using `ThreadPoolExecutor` (or `ProcessPoolExecutor` for synthetic health checks).
4. **Auto-Healing Token Retry:** If any Rancher Kubernetes or Prometheus HTTP request returns `401` or `403`, `KubeManager.get_with_retry()` acquires `self._vault_lock` (with a 60-second cooldown to prevent thundering-herd Vault queries), refreshes tokens from Vault, updates `self.session.auth`, and transparently retries the request.
5. **Response Serialization:** Results and any per-cluster `ClusterError` warnings are aggregated into the protobuf response message and returned to the caller.

---

## 5. Detailed Business Logic, Rules & HITL 🧠

### 5.1. Tiered Synthetic Health Check Engine (`core/health_manager.py` & `active_health_monitor.py`)
- **Trigger Condition:** Invoked on-demand via `RunHealthChecks` RPC (`model-health` CLI).
- **Target Discovery:** Queries Kubernetes Ingresses across `sera-llm`, `halo`, and `thirdlaw-platform` namespaces, plus `LoadBalancer` Services in `halo`. Filters out internal Envoy routes (`thirdlaw-aif-llm-vllm`) and platform UIs (`ui.`, `dl.`, `otel`).
- **Four-Layer Probing Matrix:**
  1. **`KUBE` Layer:** Direct cluster Ingress FQDN (`https://<host>.<loc>.gt-t.net`). No auth header unless flagged as Envoy.
  2. **`VM` Layer:** Regional external HAProxy VM VIP (`https://aif-llm-<model_id>.<loc>.gt-t.net`). Injects Vault static API key (`Authorization: Bearer sk_aif_...`).
  3. **`F5` Layer:** Global F5 GTM load-balanced VIP (`https://aif-llm-<model_id>.aif.gt-t.net`). Injects Vault static API key.
  4. **`INT-LB` Layer:** Internal MetalLB IP for Halo Triton services (`http://<lb_ip>:8000`).
- **Three-Stage Sequential Probe (per target inside 12-worker `ProcessPoolExecutor`):**
  1. **Readiness Probe (`route_latency_ms`):** `GET /health` (vLLM), `/v1/health/ready` (NIM), or `/v2/health/ready` (Triton/INT-LB) with 5s timeout.
     - **Smart DNS Fallback:** If `NameResolutionError` occurs on `aif-llm-<model>`, HMU dynamically rewrites the target URL to `aif-llm-vllm-<model>`, appends ` (Fallback)` to `endpoint_url`, and retries immediately.
     - **Auth Detection:** If response contains `"login-pf"` or HTTP `401`/`403`, halts and sets message to `"Auth Blocked (DNS/Proxy Misroute)"` (if `KUBE` without auth) or `"Auth Blocked"`.
  2. **Identity Probe (`identity_latency_ms`):** `GET /v1/models` with 5s timeout. Extracts `data[0].id` as `detected_name`.(Skipped for `INT-LB`, which marks identity and inference OK upon readiness pass).
  3. **Inference Probe (`inference_latency_ms` / TTFT):** `POST` with 10s timeout using `detected_name`:
     - Embeddings (`"embed"` in name): `POST /v1/embeddings` with `{"model": detected_name, "input": ["ping"], "input_type": "query"}`.
     - Rerankers (`"rerank"` in name): `POST /v1/ranking` with `{"model": detected_name, "query": {"text": "ping"}, "passages": [{"text": "pong"}]}`.
     - Chat/Default: `POST /v1/chat/completions` with `{"model": detected_name, "messages": [{"role": "user", "content": "ping"}], "max_tokens": 1}`.
     - HTTP `200` or `429` (rate-limited but alive) marks `inference_ok = True` and `"All Tests Passed"`.
- **Degraded Pod & Status Rollup Rules:**
  - Pods in `sera-llm` and `halo` that are not `Running` + `Ready=True` are parsed for container state (`waiting` reason/message, `terminated` exitCode/reason, or `NotReady`), `restartCount`, and attached as `DegradedPodInfo` alongside flag `"{N} POD DEGRADED"`.
  - Models listed in `DECOMMISSIONED_MODELS` (`gemma3-4b`, `llama-31-8b`, `vllm-qwen3-32b`, `nemotron-3-nano`) receive the `"DECOMMISSIONED"` flag.
  - **`engine_status`:** Evaluates per-datacenter health (`EngineLocationHealth`, where a location is healthy if *any* layer in that datacenter passes readiness, identity, and inference). Set to `"OK"` if all locations are healthy, `"DEGRADED"` if some are healthy, or `"FAILED"` if none are healthy.
  - **`routing_status`:** Evaluates *all* endpoints globally (including `VM` and `F5`). Set to `"OK"` if all pass, `"DEGRADED"` if at least one passes, or `"FAILED"` if all fail.

### 5.2. Physical Topology & Remote VRAM Mapping (`core/topology_builder.py`)
- **Trigger Condition:** Invoked via `RefreshClusterMap` streaming RPC (`refresh-map` CLI). Protected by non-blocking `threading.Lock()` (`self._sync_lock`); concurrent requests immediately yield `success=False` ("Topology sync is already in progress").
- **Step-by-Step Execution:**
  1. Fetches all pods and PVCs in `sera-llm` and `halo`, mapping container IDs (`containerID` stripped of `containerd://` or `docker://`) to pod names and PVC names to storage capacities and NetApp Trident volume IDs (`spec.volumeName`).
  2. Queries `gpu-operator` pods, `DCGM_FI_DEV_GPU_UTIL`, and `kube_node_spec_unschedulable`. Nodes with `kube_node_spec_unschedulable == 1.0` are suffixed with ` [🛑 CORDONED]` and have their reported GPU utilization forced to `0.0%`.
  3. **Remote `kubectl exec` VRAM Mapping:** Spawns a 20-worker `ThreadPoolExecutor` that executes a bash script inside every running `nvidia-driver-daemonset*` pod in the `gpu-operator` namespace. The script runs `nvidia-smi --query-compute-apps=gpu_bus_id,pid,used_memory,process_name` and `nvidia-smi --query-gpu=pci.bus_id,index`, correlates each PID's `/proc/$pid/cgroup` 64-character hex container ID back to the Kubernetes pod name, and sums exact MiB VRAM usage per `(pod_name, gpu_index)`.
  4. **Triton Log Config Salvaging:** For `Triton` pods, HMU salvages existing `TRITON_*` keys from the database to avoid spamming the Kubelet log API; for new Triton pods, it fetches the first `50,000` bytes (`limitBytes=50000`) of pod logs to parse the ASCII startup configuration table (`| Option | Value |`).
  5. **Model Identity Normalization:** Strips vendor prefixes (`google/`, `nvidia/`) while preserving exact model casing and suffixes (e.g., `-it`), then applies `LEGACY_MODEL_NAME_MAP` (`gemma-4-31b` -> `gemma-4-31B-it`, `gemma-4-e4b` -> `gemma-4-E4B-it`, `gemma-3-27b` -> `gemma-3-27b-it`, `nanonets-ocr2-3b` -> `Nanonets-OCR2-3B`, etc.).
  6. **GPU Pinning vs. Floating Models:** Inspects `NVIDIA_VISIBLE_DEVICES` and `CUDA_VISIBLE_DEVICES`. Explicit GPU indices bind `AiDeployment` to specific `HardwareGpu` records; pods without explicit device pins (or floating across the node) store `gpu_id=None`, inject `"node_id": target_node.id` into `engine_config`, and sum VRAM across all GPUs on the node.

### 5.3. Prompt Load Scoring & Quantile Analytics (`core/monitors/ai_engine_monitor.py`)
- **Prometheus Quantiles:** Queries `histogram_quantile(0.50 | 0.90 | 0.99, sum by (le, model_name) (rate(vllm:request_prompt_tokens_bucket[Xh])))` and `vllm:request_generation_tokens_bucket`.
- **Quadratic Load Score Formula:** To identify which prompt category places the heaviest quadratic attention burden on KV cache and GPU compute, HMU calculates:
  $$\text{Category Load Score} = \text{classified\_payloads} \times (\text{avg\_prompt\_tokens})^2$$
  The category with the highest score is returned as `top_category_load` in `PromptOverviewResponse`.

### 5.4. State Changes, Terminal Outputs & HITL
* **Status Rollup Enums:** `engine_status` and `routing_status` evaluate multi-region and global endpoint telemetry to yield deterministic status values: `"OK"`, `"DEGRADED"`, or `"FAILED"`.
* **HITL & Learning Loops:** *Not applicable for this module.* HMU operates as a fully autonomous read-only telemetry and probing engine without interactive Human-in-the-Loop approval queues. (Autonomous LLM classification, manual regex curation in `core/database/regex.py`, and diagnostic JSON discovery dumps are detailed in Section 6 and Section 7).

---

## 6. AI, LLM & Machine Learning Execution 🤖

HMU utilizes an internal LLM-driven self-learning loop inside `PayloadClassifier` (`core/monitors/payload_classifier.py`) to categorize incoming prompt traffic across the AI Factory.

### 6.1. LLM Client Configuration (`core/llm_interface.py`)
- **Target Model:** Uses the `secondary` engine alias defined in `settings.yaml`: **`google/gemma-4-E4B-it`** hosted at `https://aif-llm-gemma4-e4b.aif.gt-t.net/v1/chat/completions`.
- **Inference Parameters:** `temperature=0.1` (override during classification; config default `0.0`), `max_tokens=8192`, `timeout_seconds=180`, `response_format={"type": "json_object"}`, and `chat_template_kwargs={"enable_thinking": False}`.
- **Resilience:** `vLLMClient.get_chat_completion()` implements a 3-attempt exponential backoff retry loop ($2^1\text{s}, 2^2\text{s}$).

### 6.2. Autonomous Classification & Signature Generation Loop
1. **Batch Selection:** Every 5 minutes (or 10 seconds if the previous batch was full), `_run_auto_classifier_daemon` queries PostgreSQL for up to `50` `TokenPayload` records where `category_id IS NULL`.
2. **Prompt Truncation:** Reads the raw prompt file from `/ai/assets/vault/health_monitor_utility/payloads/{loc}/{model}/{request_id}.txt`. If length exceeds `1000` characters, truncates to `prompt_text[:500] + "\n...\n" + prompt_text[-500:]`.
3. **Generator Execution (`GENERATOR_PROMPT`):** Instructs Gemma-4 E4B to output strict JSON with:
   - `category_name`: 1 to 3 words maximum capturing the specific business domain (e.g., `"Fortinet Analysis"`, `"Debt Collection Analysis"`).
   - `unique_phrase`: A verbatim 5–15 word static phrase from the core system instructions appearing before any dynamic variables.
4. **Deterministic Prefix Regex Construction:**
   - HMU locates `unique_phrase` inside the full `prompt_text` (`idx = prompt_text.find(phrase) + len(phrase)`), slices the static prefix `prompt_text[:idx]`, and constructs a deterministic Python-escaped regex: `"^" + re.escape(prefix_to_match)`. (If the phrase is not found verbatim, falls back to `"^" + re.escape(prompt_text[:150])`).
5. **Backfill Validation Gate:**
   - Tests the newly generated `regex_pat` (`re.IGNORECASE | re.MULTILINE`) against all currently unclassified payloads on disk.
   - **Pass (`backfill_count > 1`):** Commits the new `PromptCategory` and `PromptSignature` (`created_by_llm=True`), updates all matched `TokenPayload` records, and writes the updated ruleset to `core/database/regex.py` via `_sync_regex_file()`.
   - **Fail (`backfill_count <= 1`):** Rolls back the transaction, deletes the category if it has `0` active signatures, assigns the payload to `"Unclassifiable"`, and writes the JSON dump to `/ai/utilities/health_monitor_utility/discovery/{request_id}.json`.

---

## 7. Data Models, State, Storage & Auditing 🗄️

### 7.1. PostgreSQL Schema (`core/database/models.py`)
HMU persists state in `sam_admin_prod_db` across 7 SQLAlchemy ORM tables:

| Table Name | ORM Class | Primary Key / Constraints | Key Columns & Purpose |
| :--- | :--- | :--- | :--- |
| `hardware_nodes` | `HardwareNode` | `id` (PK), `hostname` (Unique) | `location`, `rancher_url`, `cluster_id`, `hostname`, `infra_pods` (JSON), `last_updated_at`. Represents physical K8s GPU worker nodes. |
| `hardware_gpus` | `HardwareGpu` | `id` (PK), `node_id` (FK) | `gpu_index` (String), `gpu_uuid`. Cascades delete when parent `HardwareNode` is wiped during topology refresh. |
| `ai_deployments` | `AiDeployment` | `id` (PK), `gpu_id` (FK, Nullable) | `engine_type` (`vLLM`, `NIM`, `Triton`, `Morpheus`), `model_name`, `current_pod_name`, `engine_config` (JSON), `vram_used_mb` (Integer). `gpu_id=NULL` indicates a shared floating model on `engine_config["node_id"]`. |
| `prompt_categories` | `PromptCategory` | `id` (PK), `name` (Unique) | `name`, `description`. Stores manual and LLM-learned workload categories. |
| `prompt_signatures` | `PromptSignature` | `id` (PK), `category_id` (FK) | `regex_pattern` (`String(1000)`), `is_active` (Boolean), `created_by_llm` (Boolean), `created_from_request_id`. |
| `token_payloads` | `TokenPayload` | `id` (PK), `uix_request_id` (Unique) | `request_id` (`chatcmpl-...`), `pod_name`, `model_name`, `cluster_location`, `token_count`, `max_tokens`, `temperature`, `lora_request`, `sampling_params` (JSON), `timestamp` (Indexed), `category_id` (FK), `client_id`. Retained for a sliding 24-hour window. |
| `usage_model_stats` | `UsageModelStat` | `id` (PK), `uix_date_model` (Unique) | `date` (Indexed), `model_name` (Indexed), `inferences`, `input_tokens`, `output_tokens`, `active_gpus`. Daily billing & token rollups. |

### 7.2. Bi-Directional In-Memory Shadow Cache (`core/database/regex.py`)
- To prevent hitting PostgreSQL on every log line during high-throughput vLLM log scavenging, `core/database/regex.py` maintains an in-memory dictionary `REGEX_SIGNATURES`.
- On startup, `_sync_db_from_regex_file()` synchronizes `regex.py` into `prompt_categories` and `prompt_signatures`, prunes unreferenced DB signatures, wipes existing `category_id` assignments (`category_id = None`), and re-evaluates all on-disk payloads for a clean-slate backfill.
- Whenever the Auto-Classifier learns a new signature, `_sync_regex_file()` regenerates `regex.py` and reloads the module via `importlib.reload(shadow_regex)`.

### 7.3. File-Based Payload Storage & Auditing
- **Raw Prompt Storage:** `/ai/assets/vault/health_monitor_utility/payloads/{cluster_location}/{model_name}/{request_id}.txt`. Automatically pruned after 24 hours (`86,400` seconds) by the scavenger daemon.
- **Unclassifiable Discovery Dumps:** `/ai/utilities/health_monitor_utility/discovery/{request_id}.json`.
- **RPC Transaction Auditing (`utils/rpc_transaction_logger.py`):** Supports writing structured JSON audit logs (`hmu_txn_YYYY-MM-DDTHH-MM-SS-mmmZ.json`) in `/ai/assets/logs/hmu_utility/transactions/` with automatic retention pruning (`retain_count: 50`).

---

## 8. APIs, Observability & Communication Contracts 📡

HMU exposes two gRPC services on `0.0.0.0:50101` alongside standard gRPC Health Checking (`grpc.health.v1.Health`) and Server Reflection:

### 8.1. `hmu.HealthMonitorUtility` RPC Contract (`grpc_api/protos/hmu.proto`)

| RPC Method | Request Message | Response Message | Description |
| :--- | :--- | :--- | :--- |
| `GetGpuStatus` | `GetGpuStatusRequest` (`history_hours`) | `GpuStatusResponse` | Returns real-time DCGM GPU telemetry (`GpuDetails`), Host OS telemetry (`NodeOsDetails`), and optional time-series history (`GpuHistorySeries`) across all clusters. |
| `GetNodeStatus` | `GetNodeStatusRequest` (`node_name`, `cluster_location`, `history_hours`) | `NodeStatusResponse` | Returns targeted Host OS and GPU metrics plus historical series for a single node. |
| `GetVllmStatus` | `GetVllmStatusRequest` (`history_hours`, `model_filter`) | `VllmStatusResponse` | Returns active vLLM pod metrics (running/waiting reqs, prompt/gen TPS, KV cache %, hit rate, preemptions, PVC disk GiB) or historical trends. |
| `GetNimStatus` | `GetNimStatusRequest` | `NimStatusResponse` | Returns NVIDIA NIM pod health, pending requests, exec/s, prompt/gen TPS, and PVC disk GiB. |
| `GetHaloStatus` | `GetHaloStatusRequest` (`history_hours`, `model_filter`) | `HaloStatusResponse` | Returns NVIDIA Triton pod metrics (queue depth, success/fail/exec TPS, input/queue/compute/output/total latency ms, disk GiB) or history. |
| `GetTrafficStatus` | `GetTrafficStatusRequest` | `TrafficStatusResponse` | Returns K8s HAProxy Ingress frontend sessions/conn-rate and backend queue depth, healthy pods, P99 latency, and 4xx/5xx rates. |
| `GetVmHaproxyStats` | `GetVmHaproxyStatsRequest` (`location`, `include_raw`) | `VmHaproxyStatsResponse` | Queries external HAProxy VMs (`;csv`), returning curated `llm` routes (`CuratedVmRoute`) and optional raw CSV JSON. |
| `GetIngressStatus` | `GetIngressStatusRequest` (`cluster_location`) | `IngressStatusResponse` | Lists Kubernetes Ingress resources (`name`, `ingress_class`, `hosts`, `address`, `ports`) in `sera-llm`. |
| `GetClusterMap` | `GetClusterMapRequest` | `ClusterMapResponse` | Returns the persisted physical topology (`TopologyNode` -> `TopologyGpu` -> `TopologyModel` with `vram_used_mb`, plus `shared_floating_models`). |
| `RefreshClusterMap` | `RefreshClusterMapRequest` | `stream RefreshClusterMapResponse` | Streams real-time progress steps while rebuilding the global K8s node/GPU/model/VRAM topology in PostgreSQL. |
| `GetModelConfig` | `GetModelConfigRequest` (`model_filter`, `engine_filter`) | `ModelConfigResponse` | Returns untruncated JSON engine configurations (`ModelConfigInstance`) for deployed vLLM, NIM, Triton, and Morpheus pods. |
| `GetRawPod` | `GetRawPodRequest` (`pod_name`, `cluster_location`) | `RawPodResponse` | Fetches and returns the 100% raw Kubernetes Pod API JSON object from `sera-llm` or `halo`. |
| `RunHealthChecks` | `RunHealthChecksRequest` | `HealthChecksResponse` | Executes concurrent synthetic probes across `KUBE`, `VM`, `F5`, and `INT-LB` layers, returning `ModelHealthSummary` records. |
| `RunFeatureTest` | `FeatureTestRequest` (`url`, `model`, `test_type`, `include_logprobs`, `lora`) | `FeatureTestResponse` | Executes an isolated feature test against an endpoint and returns HTTP status, latency, full JSON response, and a reproducible `curl` command. |
| `RunBenchmark` | `BenchmarkRequest` (`url`, `model`, `mode`, `steps`, `enable_thinking`, `input_tokens`, `output_tokens`) | `stream BenchmarkStepResponse` | Streams `INFO`, `PROGRESS`, and `SUMMARY` messages during multi-step concurrent load testing. |
| `GetPromptOverview` | `GetPromptOverviewRequest` (`history_hours`, `model_filter`) | `PromptOverviewResponse` | Returns 24h token quantiles (`P50`/`P90`/`P99`), total prompts, global avg tokens, top categories (volume/size/load), client volumes, active models/LoRAs, and category metadata. |
| `GetPromptList` | `GetPromptListRequest` (filters: `model`, `pod`, `category`, `lora`, `client`, `min_tokens`, `sort_by_tokens`, `limit`, `offset`) | `PromptListResponse` | Returns paginated `TopPayload` records matching the filter criteria along with `total_matched_records`. |
| `GetEnginePromptStats` | `GetEnginePromptStatsRequest` (`history_hours`, `model_filter`, `limit`, `min_tokens`) | `EnginePromptStatsResponse` | Returns 1h token quantiles and top largest batch requests (`largest_prompts`) for real-time engine monitoring. |
| `GetPromptPayload` | `GetPromptPayloadRequest` (`request_id`) | `GetPromptPayloadResponse` | Reads the full raw prompt text from disk alongside `sampling_params_json`, `category_name`, `cluster_location`, `model_name`, and `pod_name`. |
| `GetPayloadHistory` | `GetPayloadHistoryRequest` (`tail_count`, `filter`, `pod_name`, `cluster_location`) | `PayloadHistoryResponse` | Fetches pod logs filtered by `RAW`, `ERRORS_ONLY`, or `PERFORMANCE`. |
| `DiscoverMetrics` | `DiscoverMetricsRequest` (`search_string`) | `DiscoverMetricsResponse` | Searches Prometheus across all clusters for metric names matching `.*{search_string}.*`. |
| `GetValkeyClusterStatus` | `GetValkeyClusterStatusRequest` | `ValkeyClusterStatusResponse` | Audits all 9 global Valkey nodes for role (`MASTER`/`SLAVE`), memory usage, connected clients, ops/sec, and max DBs. |
| `GetValkeyInspectStatus`| `GetValkeyInspectStatusRequest` | `ValkeyInspectStatusResponse` | Inspects occupied Valkey databases globally (`SLO`/`NYC`/`PRG`), active masters, key/connection counts, and `__GIA_DB_OWNER__` tags. |
| `GetRedisInspectStatus` | `GetRedisInspectStatusRequest` (`env`) | `RedisInspectStatusResponse` | Audits a target Redis instance (`0-15` DBs) for free/occupied DBs, key counts, client names, and owner flags. |
| `GetUsageStats` | `UsageStatsRequest` (`days_back`) | `UsageStatsResponse` | Returns daily `UsageModelStatMessage` billing rollups (`inferences`, `input_tokens`, `output_tokens`, `active_gpus`). |

### 8.2. `sera.standard.v1.SeraAppService` Contract (`grpc_api/sera_standard_service.py`)
HMU implements the universal SERA standard contract (`sera.standard.v1.SeraAppService`), centrally maintained and deployed by the Application Management Utility (AMU). It provides standardized endpoints (`GetAppInfo`, `GetDeepHealth`, `GetFeedback`, `GetChangelog`, `GetDocs`) enabling AMU and the SAM UI to self-discover HMU's metadata, deep TCP socket dependency health matrix, changelog history, and design documentation without requiring custom integration logic.

---

## 9. User Interface (UI) & Visualizations 🖥️

*Not applicable for this module.* HMU is a headless gRPC backend and CLI utility; visual dashboards and administrative UI views are rendered upstream by the **Sera Admin Module (SAM)** frontend.

---

## 10. Background Services & Workers 🛠️

HMU runs four background daemon threads (initialized via `AiEngineMonitor.start_background_tasks()`) and one persistent multiprocessing pool:

| Worker / Daemon | Location | Trigger / Interval | Concurrency & Locking | Operational Responsibility |
| :--- | :--- | :--- | :--- | :--- |
| **Startup Regex & DB Backfill Worker** | `PayloadClassifier.start_background_tasks` | Once on server boot (async daemon thread) | Single background thread; DB transaction | Syncs `core/database/regex.py` to Postgres, prunes orphaned signatures/categories, resets `TokenPayload.category_id = None`, and backfills categories across all existing `.txt` files on disk before starting the scavenger and classifier threads. |
| **Payload Scavenger Daemon** | `PayloadClassifier._run_payload_scavenger_daemon` | Every `600` seconds (10 minutes) | Single daemon thread; bulk upsert (`chunk_size=5000`) | Streams logs (`tail_lines=150000` on first run including `previous=True`, `10000` thereafter) from all running `vllm` pods in `sera-llm`. Parses `chatcmpl-` IDs, `prompt_token_ids` length (`> 16` tokens), `SamplingParams`, `lora_request`, and `POST /v1/chat/completions?client=...&req_id=...`. Saves raw prompts to disk, upserts `TokenPayload` (`on_conflict_do_update` for `client_id`), backfills unpopulated `client_id`s, and purges DB records and `.txt` files older than 24 hours. |
| **Auto-Classifier Self-Learning Daemon** | `PayloadClassifier._run_auto_classifier_daemon` | Every `300` seconds (or `10`s between full 50-record batches) | Single daemon thread; sequential LLM calls | Queries up to `50` unclassified `TokenPayload` records (`category_id IS NULL`), calls Gemma-4 E4B (`secondary` LLM) with `GENERATOR_PROMPT`, builds escaped prefix regex signatures, validates against historical payloads (`backfill_count > 1`), updates `regex.py`, or dumps unclassifiable payloads to `discovery/`. |
| **Daily Usage Stats Sync Daemon** | `AiEngineMonitor._run_usage_stats_sync_daemon` | Every `3600` seconds (1 hour) | Single daemon thread | Queries Prometheus over a 14-day sliding window (`step="1d"`) for `increase(vllm:prompt_tokens_total[24h])`, `increase(vllm:generation_tokens_total[24h])`, and `increase(vllm:request_prompt_tokens_bucket{le="+Inf"}[24h])`, correlates active GPU counts from `AiDeployment`, and upserts per-model and `"TOTAL (All Models)"` records into `usage_model_stats`. |
| **Health Check Worker Pool** | `core.health_manager._HEALTH_CHECK_POOL` | Pre-warmed at boot; invoked on `RunHealthChecks` | `ProcessPoolExecutor(max_workers=12)` protected by `_POOL_LOCK` | Executes GIL-isolated TLS handshakes and HTTP probes across KUBE, VM, F5, and INT-LB targets. Automatically recreates the pool if `_broken` is detected. |

---

## 11. CLI, Tooling & Testing 💻

HMU provides a rich terminal interface via `health_monitor_utility.py` (powered by Click's `NestedHelpGroup` in `cli/hmu_cli.py`). Every telemetry command supports `--grpc` to dump raw protobuf JSON for programmatic pipelines (`jq`).

### 11.1. CLI Command Reference

| Command | Arguments & Options | Description |
| :--- | :--- | :--- |
| `start-grpc-server` | None | Compiles protobuf stubs if needed, initializes database tables, and starts the HMU gRPC server on port `50101`. |
| `module info` | `--grpc` | Queries `SeraAppService.GetAppInfo` to display acronym, description, version, category, and downstream dependencies. |
| `module check-health` | `--grpc` | Queries `SeraAppService.GetDeepHealth` to display overall status and dependency health matrix. |
| `gpu-status` | `--grpc` | Displays real-time NVIDIA DCGM GPU utilization, Tensor core %, VRAM, temperature, power draw, NVLink, PCIe, and XID/Throttle flags across all clusters. |
| `node-status` | `<node_name> --loc <loc> [--grpc]` | Displays Host OS metrics (CPU %, RAM GB, Root Disk %, Disk R/W MB/s) and per-GPU telemetry for a specific worker node. |
| `vllm-status` | `[--history/-h <hours>] [--model/-m <str>] [--grpc]` | Displays real-time vLLM metrics (running/waiting reqs, Prompt/Gen TPS, KV Cache %, Hit %, Disk GiB, Preemptions) or historical trend tables. |
| `nim-status` | `--grpc` | Displays NVIDIA NIM pod status, pending requests, exec/s, Prompt/Gen TPS, and PVC Disk GiB. |
| `halo-status` | `--grpc` | Displays NVIDIA Triton (Halo) queue depth, batch throughput, error rates, segmented latencies (input/queue/compute/output/total ms), and Disk GiB. |
| `model-health` | `--grpc` | Runs active synthetic health checks across `KUBE`, `VM`, `F5`, and `INT-LB` layers, displaying `[Readiness|Identity|Inference]` matrices, TTFT, and degraded pods. |
| `cluster-map` | `--grpc` | Displays the physical node-to-GPU topology map, active models per GPU (with `[VRAM GB]`), and floating models. |
| `refresh-map` | None | Triggers a streaming global topology refresh (`RefreshClusterMap`), including remote `kubectl exec` VRAM mapping. |
| `model-config` | `[--model/-m <str>] [--engine/-e vllm\|nim\|triton\|morpheus] [--grpc]` | Displays untruncated container arguments, environment variables, resource limits, restart counts, and PVC mappings from the database. |
| `raw-pod` | `[search_string="nim"]` | Searches directly via `KubeManager` for the first running pod matching `search_string` and dumps 100% raw Kubernetes JSON. |
| `traffic-status` | None | Displays Kubernetes HAProxy Ingress frontend active sessions/conn-rate and backend queue depth, P99 latency, and 4xx/5xx error rates. |
| `vm-traffic` | `[location="all"] [--raw] [--grpc]` | Fetches external VM HAProxy statistics (`slo`, `prg`, `nyc`, or `all`) and displays curated AI Factory LLM backend routes or raw CSV JSON. |
| `valkey-cluster` | `--grpc` | Audits all 9 global Valkey nodes (`role`, `used_mem`, `max_mem`, `clients`, `ops/sec`) and tests TCP reachability on regional HAProxy VIPs (`:6379`). |
| `valkey-inspect` | `--grpc` | Displays occupied Valkey databases globally (`SLO/NYC/PRG` keys and connections), active masters, and detected `__GIA_DB_OWNER__` applications. |
| `redis-inspect` | `[--env="redis_dev"] [--grpc]` | Audits a 16-DB Redis instance, listing occupied DB owners/clients, free DB indexes, and recommending the next safe even-odd DB pair. |
| `prompt overview` | `[--hours/-h 24] [--model/-m <str>] [--grpc]` | Displays 24h summary metrics, token size quantile distributions (`P50`/`P90`/`P99`), active client volumes, active LoRAs, and active models. |
| `prompt list` | `[--hours/-h 24] [--model/-m] [--pod/-p] [--category/-c] [--lora] [--client/-cl] [--min-tokens/-t] [--sort time\|tokens] [--limit/-l 50] [--offset/-o 0] [--grpc]` | Lists captured prompt inferences with filtering, sorting, and offset pagination. |
| `prompt largest` | `[--hours/-h 1] [--model/-m] [--limit/-l 25] [--min-tokens/-t 16] [--grpc]` | Displays the largest token batch requests across pods over the specified window. |
| `prompt usage` | `[--days/-d 30] [--grpc]` | Displays daily token and inference billing rollups per model from `usage_model_stats`. |
| `prompt content` | `<request_id> [--grpc]` | Retrieves and formats the full raw prompt payload from disk alongside category, location, pod, and sampling parameters. |
| `prompt category` | `[--model/-m <str>] [--grpc]` | Displays learned prompt categories, active signature counts, regex patterns, classified payload counts, and average prompt/generation token sizes. |
| `usage-stats` | `[--days/-d 30] [--grpc]` | Root alias for viewing daily model token and inference statistics. |
| `payload-history` | `[--count/-c 50] [--filter/-f raw\|errors\|performance] [--model/-m <str>] [--grpc]` | Fetches recent pod log lines across AI engines filtered by error tracebacks, performance metrics, or raw output. |
| `payload-stream` | `[--limit/-l 1000] [--min-tokens/-t 2] [--grpc]` | Scans raw vLLM logs for `BatchDescriptor` debug lines (`num_tokens` and `cudagraph_mode`). |
| `cleanup-ghosts` | `[--dry-run]` | Scans `token_payloads` in batches of `1000` and purges orphaned database records whose `.txt` files no longer exist on disk. |
| `discover-metrics` | `[search_string="vllm"]` | Queries Prometheus across all clusters to list active metric names matching `search_string`. |
| `test-model` | `<url> [model] [--type/-t chat\|reasoning\|math\|embed\|rerank\|tool\|timeout\|vision\|vision-reasoning\|default] [--logprobs] [--lora <str>]` | Executes an isolated feature check against a model endpoint, auto-correcting partial model names and printing the raw JSON response and masked `curl` command. |
| `benchmark-model` | `<url> [model] [--mode/-m simple\|complex\|stress\|max] [--steps "auto"] [--thinking/--no-thinking] [--input-tokens 0] [--output-tokens 0] [--grpc]` | Runs a streamed concurrency benchmark (`2, 5, 10, 15, 20, 40...` workers), halting automatically on error cliffs (`>10%` failures at $\ge 10$ workers) and printing a final RPS/TPS/P95 summary matrix. |

### 11.2. CLI Quick Execution Examples

```bash
# Server Lifecycle & Standard Introspection
python3 health_monitor_utility.py start-grpc-server
python3 health_monitor_utility.py module info [--grpc]
python3 health_monitor_utility.py module check-health [--grpc]

# Hardware & Workload Telemetry
python3 health_monitor_utility.py gpu-status [--grpc]
python3 health_monitor_utility.py node-status <node_name> --loc <loc> [--grpc]
python3 health_monitor_utility.py vllm-status [--history 24] [--model <name>] [--grpc]
python3 health_monitor_utility.py nim-status [--grpc]
python3 health_monitor_utility.py halo-status [--history 24] [--grpc]

# Physical Topology & Remote VRAM Introspection
python3 health_monitor_utility.py cluster-map [--grpc]
python3 health_monitor_utility.py refresh-map

# Synthetic Health Checks & Feature Benchmarks
python3 health_monitor_utility.py model-health [--grpc]
python3 health_monitor_utility.py test-model <url> [model] --type chat
python3 health_monitor_utility.py benchmark-model <url> [model] --mode stress

# Cache & Traffic Auditing
python3 health_monitor_utility.py valkey-cluster [--grpc]
python3 health_monitor_utility.py valkey-inspect [--grpc]
python3 health_monitor_utility.py redis-inspect --env redis_dev [--grpc]
python3 health_monitor_utility.py traffic-status
python3 health_monitor_utility.py vm-traffic all [--grpc]

# Autonomous Prompt & Billing Analytics
python3 health_monitor_utility.py prompt overview [--hours 24] [--grpc]
python3 health_monitor_utility.py prompt list [--hours 24] [--limit 50] [--grpc]
python3 health_monitor_utility.py prompt largest [--hours 1] [--limit 25] [--grpc]
python3 health_monitor_utility.py prompt usage [--days 30] [--grpc]
python3 health_monitor_utility.py prompt category [--grpc]
python3 health_monitor_utility.py prompt content <request_id> [--grpc]
python3 health_monitor_utility.py usage-stats [--days 30] [--grpc]
python3 health_monitor_utility.py cleanup-ghosts [--dry-run]
```

---

## 12. Configuration, Security & Hardware Affinity 🔒

### 12.1. `config/settings.yaml` Reference

| Configuration Key | Purpose | Default / Configured Value |
| :--- | :--- | :--- |
| `module_info.acronym` | Ecosystem module identifier injected into `x-client-id` gRPC metadata | `"HMU"` |
| `module_info.version` | Semantic version fallback | `"0.4.2"` |
| `system_services.grpc.service_name` | Systemd unit name checked by `GetAppInfo` and `GetDeepHealth` | `"hmu-grpc.service"` |
| `system_services.grpc.host` / `port` | gRPC server bind address and port | `"0.0.0.0"` / `50101` |
| `system_services.grpc.max_workers` | ThreadPoolExecutor worker count for the gRPC server | `20` |
| `database.write_dsn` | PostgreSQL primary read/write connection string | `postgresql://sam_admin_prod_user:***@slo-aif-hadb.slo.gt-t.net:5432/sam_admin_prod_db` |
| `database.read_dsn` | PostgreSQL HAProxy read-only replica connection string | `postgresql://sam_admin_prod_user:***@slo-aif-ha.slo.gt-t.net:5433/sam_admin_prod_db` |
| `external_services.vault.url` | HashiCorp Vault endpoint | `"https://vault-aif.gt-t.net:8200"` |
| `external_services.vault.role_id` / `secret_id` | AppRole credentials used by `VaultClient` | Configured per environment |
| `rancher_clusters.{prg\|slo\|nyc}` | Multi-cluster Rancher proxy URLs and `cluster_id`s (`c-m-74sc9qpk`, `c-m-lh85lg66`, `c-m-vkz7jqjh`) | Configured per datacenter |
| `vault_paths.rancher_admin` | Vault KV v2 path template for Rancher bearer tokens | `"rancher/{loc}/system/gia_ai"` |
| `vault_paths.llm_api_key` | Vault KV v2 path for static global AI Factory API key (`sk_aif_...`) | `"haproxy/global/apps/gia-test"` |
| `vm_haproxy.{prg\|slo\|nyc}` | External VM HAProxy CSV endpoints (`http://10.250.{146\|133\|99}.50:9000/;csv`) and basic auth | Configured per datacenter |
| `cache_clusters.redis_dev.url` | Local Redis connection URL for `redis-inspect` | `"redis://:***@localhost:6379"` |
| `cache_clusters.valkey_prod.nodes` | 9 global Valkey cluster FQDNs (`{slo\|nyc\|prg}-aif-rds0{1..3}.*.gt-t.net`) | 9 nodes |
| `logging.directory` | Base directory for daily rolling application and error logs | `"/ai/assets/logs/hmu_utility"` |
| `transaction_logging.enabled` / `retain_count` | Enables JSON RPC transaction dumps and sets max retained files | `true` / `50` |
| `paths.generated_stubs_path` | Shared output directory for compiled `_pb2.py` and `_pb2_grpc.py` files | `"/ai/assets/protos/health_monitor_utility/generated"` |
| `paths.sera_proto_source_dir` | Directory containing `sera_standard.proto` | `"/ai/assets/protos/sera_standard"` |
| `llm_engines.primary` | Primary reasoning LLM (`google/gemma-4-31B-it`, `enable_thinking: "enable"`, `timeout: 300s`) | `"https://aif-llm-gemma4-31b.aif.gt-t.net"` |
| `llm_engines.secondary` | Secondary fast LLM for Auto-Classifier (`google/gemma-4-E4B-it`, `enable_thinking: "disable"`, `timeout: 180s`) | `"https://aif-llm-gemma4-e4b.aif.gt-t.net"` |

### 12.2. Security & Authentication Architecture
- **Zero Local Cluster Tokens:** Rancher bearer tokens are never stored in `settings.yaml`. They are fetched dynamically at boot and on `401`/`403` expiration from HashiCorp Vault (`secret/rancher/{loc}/system/gia_ai`) using `VaultClient`.
- **Static API Key Injection:** Keycloak JWT dependencies have been removed. `HealthManager`, `TestingManager`, and `vLLMClient` retrieve the static global API key (`sk_aif_...`) directly from Vault path `secret/haproxy/global/apps/gia-test`.
- **Valkey Credentials:** `CacheMonitor` retrieves Valkey `username` and `password` dynamically from Vault path `secret/valkey/system/gia_ai`.
- **DSN Sanitization:** `SeraStandardServiceImpl.GetDeepHealth` strips passwords from all PostgreSQL and Redis/Valkey URLs via `sanitize_dsn()` before exposing connection metadata over gRPC.

### 12.3. Hardware Affinity & Execution Boundaries
- HMU is a lightweight CPU-bound telemetry aggregator and diagnostic service running locally via `systemd` (`hmu-grpc.service`) on application nodes, requiring zero dedicated GPU hardware for its own execution.
- Its high-concurrency synthetic probing engine utilizes a pre-warmed 12-worker multiprocessing pool (`ProcessPoolExecutor`) to eliminate GIL contention during TLS handshakes and HTTP requests.
- HMU actively observes and profiles remote NVIDIA GPUs across Kubernetes clusters via DCGM Prometheus metrics and remote `kubectl exec` into `nvidia-driver-daemonset` pods, mapping VRAM usage without imposing GPU compute overhead.
- For autonomous prompt classification, HMU delegates inference workloads to the remote AI Factory Gemma-4 endpoint (`aif-llm-gemma4-e4b.aif.gt-t.net`) over HTTPS.

### 12.4. Cluster Instrumentation SOP (Kubernetes & Prometheus Setup Guide) 🛠️

To enable the HMU CLI tools (`gpu-status`, `vllm-status`, `nim-status`, `halo-status`, etc.), the target Kubernetes cluster must be properly instrumented so Rancher's Prometheus (`rancher-monitoring`) can discover and scrape the AI engines.

*Note: This guide is written for operators who may not be Kubernetes experts. Follow the verification steps carefully to ensure you do not cause an accidental outage.*

#### Step 12.4.1: Pre-Flight Verification (Look Before You Leap)

Before changing any configurations, inspect the current state of the cluster to see what needs to be configured.

**1. View current Services and their labels:**
This command lists all AI services, their current labels, and their exposed port names.
```bash
kubectl get svc -n sera-llm -o custom-columns='NAME:.metadata.name,LABELS:.metadata.labels,PORTS:.spec.ports[*].name'
```
*(Look for `gia-monitoring:enabled` in the labels. If it is missing, we need to add it in the steps below.)*

**2. View current NIM metrics configuration:**
NVIDIA NIMs hide their metrics by default. Check if they are currently enabled (`true`) or disabled (`<none>`).
```bash
kubectl get nimservice -n sera-llm -o custom-columns='NAME:.metadata.name,METRICS_ENABLED:.spec.metrics.enabled'
```

#### Step 12.4.2: Labeling & Patching Open-Source (vLLM) Deployments

vLLM pods expose Prometheus metrics automatically, but we must ensure their Service port is correctly named `vllm` and labeled so our `ServiceMonitor` can discover it.

**1. Standardize the Service Port Name (If necessary):**
If the `PORTS` column in Step 12.4.1 did not say `vllm`, patch it using this command (replace `<vllm-service-name>` with the actual service name):
```bash
kubectl patch svc <vllm-service-name> -n sera-llm --type='json' -p='[{"op": "replace", "path": "/spec/ports/0/name", "value":"vllm"}]'
```

**2. Label the Services:**
Apply the HMU opt-in label to all vLLM services.
```bash
kubectl label svc <vllm-service-1> <vllm-service-2> -n sera-llm gia-monitoring=enabled
```

#### Step 12.4.3: Instrumenting Proprietary NVIDIA NIM Deployments

**1. Enable Metrics in the NIM Blueprint:**
Patch the custom resources to instruct the Operator to turn metrics on.
```bash
kubectl patch nimservice <nim-name-1> <nim-name-2> -n sera-llm --type=merge -p '{"spec":{"metrics":{"enabled":true}}}'
```

**2. Force Operator Rollout (Restart):**
The Operator will not restart running pods automatically for this change. You must force a rolling restart.
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
Once healthy, apply the HMU opt-in label.
```bash
kubectl label svc <nim-name-1> <nim-name-2> -n sera-llm gia-monitoring=enabled
```

#### Step 12.4.4: Deploying Prometheus Discovery Monitors (`ServiceMonitor`)

Now that the services are labeled and outputting metrics, instruct Rancher Prometheus where to find them (`vLLM` at `/metrics` on port `vllm`, and `NIM` at `/v1/metrics` on port `api` every 15 seconds) using `ServiceMonitor` objects.

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

#### Step 12.4.5: Cluster Onboarding & Credential Registration (Vault & `settings.yaml`)

HMU requires a Rancher API key and the internal Cluster ID to successfully route queries through the Rancher Proxy.

**1. Retrieve the internal `cluster_id` via CLI:**
Rancher tags every namespace with a project ID that includes the cluster ID. Extract it via:
```bash
kubectl get namespace default -o jsonpath='{.metadata.annotations.field\.cattle\.io/projectId}' | cut -d: -f1 ; echo
```
*(Expected output: A string looking something like `c-m-v2sk8v8v`)*

**2. Generate and Store the Rancher Bearer Token in Vault:**
- Log into the Rancher Web UI.
- Click your profile picture/icon in the top right -> **Account & API Keys**.
- Click **Create API Key** (ensure it has no expiration or follows the standard rotation policy).
- Copy the generated `Bearer Token` (it will look like `token-xxxxx:yyyyyyyyyyy`).
- Store the token (`token`) and Rancher URL (`url`) in HashiCorp Vault at the cluster's designated path: `secret/rancher/<loc>/system/gia_ai` (where `<loc>` is `prg`, `slo`, or `nyc`).

**3. Register the Cluster in `config/settings.yaml`:**
Open your HMU `config/settings.yaml` file and add the new cluster location under the `rancher_clusters` block (note: bearer tokens are never stored in `settings.yaml`, as `VaultClient` hydrates them dynamically at runtime):
```yaml
rancher_clusters:
  slo:
    url: "https://<rancher-fqdn>"
    cluster_id: "<cluster_id_from_step_1>"
```

#### Step 12.4.6: Final Validation & Testing

Run these CLI commands from the HMU utility server to verify telemetry and topology data are flowing correctly.

**1. Refresh the Physical Topology:**
Forces the database to rebuild the K8s node, GPU, PVC, and remote VRAM mappings.
```bash
./health_monitor_utility.py refresh-map
```

**2. Verify vLLM Telemetry:**
```bash
./health_monitor_utility.py vllm-status
```
*(Expected: A table showing active/waiting requests, DISK GiB, KV cache %, cache hit %, and TPS for vLLM models in the new cluster).*

**3. Check Token Billing & Usage Rollups:**
```bash
./health_monitor_utility.py usage-stats
# Or via the prompt subcommand:
./health_monitor_utility.py prompt usage
```

---

## 13. Error Handling & Resilience 🛡️

- **Partial Cluster Failure Isolation:** Every multi-cluster monitor catches exceptions per datacenter (`PRG`, `SLO`, `NYC`) inside its worker future and appends a `ClusterError(cluster_location=loc, error_message=str(e))` to the response's `warnings` repeated field. A dead Rancher cluster or unreachable HAProxy VM never crashes the RPC or blocks healthy clusters from returning data.
- **Thundering Herd Vault Lock (`KubeManager`):** Concurrent threads encountering expired Rancher tokens (`401`/`403`) synchronize on `self._vault_lock`. The first thread refreshes the token from Vault and updates `self._last_vault_refresh`; subsequent waiting threads see `(now - self._last_vault_refresh) <= 60` and immediately reuse the refreshed session credentials.
- **Proxy Bypass & Socket Hygiene:** `VaultClient`, `health_manager.py` workers, and diagnostic scripts explicitly set `session.trust_env = False` or `session.proxies = {"http": None, "https": None}` to prevent corporate HTTP proxies from intercepting internal `.gt-t.net` traffic. In `TestingManager`, non-200 benchmark responses explicitly invoke `r.close()` and raise `RLIMIT_NOFILE` to the hard OS limit to prevent file descriptor exhaustion during high-concurrency stress runs.
- **NaN / Inf Quantile Sanitization:** `AiEngineMonitor._get_token_quantiles()` explicitly checks `math.isnan(val) or math.isinf(val)` on Prometheus `histogram_quantile` results and coerces them to `0.0` before protobuf serialization.
- **Self-Healing Process Pool:** `get_health_check_pool()` inspects `getattr(_HEALTH_CHECK_POOL, '_broken', False)`. If a worker process crashes or is killed by the OS, the broken pool is shut down (`cancel_futures=True`) and transparently replaced.

---

## 14. Future Enhancements & Roadmap 🚀

While the core HMU gRPC backend and autonomous classification daemons are active in production, the following enterprise capabilities remain on the development roadmap:

### 14.1. Long-Term Data Warehousing
HMU natively polls Prometheus for usage and billing rollups to daily intervals (via `_run_usage_stats_sync_daemon` into `usage_model_stats`) and retains raw prompt payloads for a 24-hour sliding window (`token_payloads`). However, broader enterprise metrics—such as long-term latency degradation patterns, historical GPU saturation curves, and multi-month departmental chargeback analytics—will eventually stream into the global Patroni PostgreSQL data warehouse to persist beyond Prometheus's standard retention window.

### 14.2. Asynchronous Kubernetes Event Caching
- **Kubernetes Event Capture:** Scrape the Kubernetes Events API (`/api/v1/namespaces/{ns}/events`) to catch infrastructure-level failures (e.g., `OOMKilled`, `NodeNotReady`, `FailedScheduling`, `FailedMount`) that cause a pod or container to vanish before its application logs can be flushed to disk or scraped by the Payload Scavenger daemon.

### 14.3. External HAProxy Config Inspector (Data Plane REST API)
Auditing the external VMs to ensure physical traffic routing aligns with internal Kubernetes state is critical. Currently, HMU inspects live traffic metrics via the external HAProxy `;csv` endpoint (`HAProxyVMClient`).
- **HAProxy Data Plane API:** The external HAProxy VMs (`PRG`, `SLO`, `NYC`) will be configured to expose the official HAProxy Data Plane REST API. HMU will query this API to retrieve the exact frontend/backend routing configuration and ACL rules as structured JSON, rather than relying on SSH or raw file parsing.
- **Firewall Prerequisites:** Network ACLs must be updated to allow traffic on the Data Plane API port from the HMU utility server to the external HAProxy VMs across all three data centers.

### 14.4. AI-Specific Service Level Indicators (SLIs) & Historical SLA Charting
While HMU already tracks real-time vLLM KV cache usage and cache hit rates via `GetVllmStatus`, future iterations will formally aggregate and persist core metrics required to chart business-level AI SLAs over time:
- **Historical Cache Hit Ratios:** Persisting prefix and KV cache hit rate trends over multi-week windows to validate the computational efficiency of prompt caching across specific business categories.
- **Category-Level TTFT SLAs:** Correlating segmented `inference_latency_ms` (Time-To-First-Token) directly with learned `PromptCategory` workloads to detect SLA breaches per consuming application.

### 14.5. Deprecate Cache Inspection (Valkey & Redis) to CMU
Because the **Configuration Management Utility (CMU)** governs the lifecycle of Valkey and Redis clusters (including ACL provisioning, DB index reservations, Vault credentials, and pre-flight DB checks), cache auditing logic currently hosted in HMU (`valkey-cluster`, `valkey-inspect`, `redis-inspect`) will be deprecated and transferred to CMU:
- **Consolidation into CMU:** Retire `CacheMonitor` and its associated RPCs in HMU, allowing CMU to serve as the single authoritative manager for both Valkey/Redis provisioning and cluster status auditing.
- **Focus on Core Telemetry:** Reinforce HMU's primary focus as "The Watcher" dedicated strictly to hardware DCGM metrics, AI engine performance (vLLM, NIM, Triton), tiered synthetic health probing, and autonomous prompt analytics.

### 14.6. Protocol Buffer & Database Namespace Alignment (`hmu_*`)
Because SAM administrative utilities share the `sam_admin_prod_db` PostgreSQL database, standardize HMU's SQLAlchemy tables with a strict `hmu_` prefix (`hmu_hardware_nodes`, `hmu_hardware_gpus`, `hmu_ai_deployments`, `hmu_prompt_categories`, `hmu_prompt_signatures`, `hmu_token_payloads`, `hmu_usage_model_stats`) to guarantee zero namespace collisions and safe schema maintenance across utilities.