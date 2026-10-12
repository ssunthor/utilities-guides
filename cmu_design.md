# Configuration Management Utility (CMU) - Architecture Document

**Doc_Type:** Architecture
**Category:** (U) Utility
**Name:** Configuration Management Utility
**Acronym:** CMU
**Version:** 0.3.5
**Date:** 2026-10-12
**Author:** Martin Kecka & Gemini AI
**Status:** Active

---

## 1. Introduction & Purpose 📜

The **Configuration Management Utility (CMU)** is a dedicated administrative microservice and CLI control plane within the SERA/GIA AI ecosystem engineered to centralize and orchestrate logical configuration, identity lifecycle management, infrastructure resource provisioning, and AI model deployment across multi-region data centers (`slo`, `prg`, `nyc`).

Operating as an autonomous, fully functional application on port `50105`, CMU provides engineers and operators with an interactive Click CLI for infrastructure manipulation and validation. Concurrently, it serves as the core infrastructure and configuration backend engine powering the **Sera Admin Module (SAM)** web interface. Within the ecosystem's administrative utility suite (AMU, CMU, DIU, HMU, OMU), CMU isolates API-driven and SSH/SFTP-orchestrated control-plane lifecycles away from application runtimes into a unified administrative gateway.

CMU governs end-to-end resource provisioning—spanning Keycloak OIDC identities, HAProxy native API keys (`sk_aif_`) and load balancer configurations, HashiCorp Vault KV v2 secrets and IAM policies, Patroni PostgreSQL databases, Valkey/Redis cache clusters, Active Directory GSS-TSIG DNS records, Kubernetes PVC storage management, and vLLM/NIM AI model & LoRA weight rollouts.

---

## 2. Architecture & Ecosystem Integration 🏗️

CMU operates on a client-server architecture, exposing dual gRPC servicers on port `50105` (`cmu-grpc.service`) backed by a multi-worker thread pool, while providing operators with a standalone Click CLI and autonomous background audit monitors.

### 2.1. Dual Servicer & Core Architectural Principles
- **Dual Servicer Model:**
  1. **`CmuService` (`cmu.proto`):** Domain control-plane router delegating execution across specialized backend managers for Vault, Keycloak, HAProxy, PostgreSQL, Valkey, Active Directory DNS, Rancher/Kubernetes, and vLLM/NIM model deployment.
  2. **`SeraAppService` (`sera_standard.proto`):** Implements the universal SERA ecosystem contract (`v1.2-universal`, maintained centrally by AMU) for self-reporting CMU's metadata, deep health matrix, documentation, and changelog.
- **Headless Control-Plane Gateway:** Establishes a strict architectural boundary by isolating high-privilege infrastructure provisioning away from user applications and operator workstations into an audited, centralized service.
- **Autonomous Standalone Operation:** Functions as a complete standalone system with direct SSH, API, and database orchestration, providing an interactive Click CLI equipped with safety confirmation prompts (`click.confirm`) for destructive actions.
- **SAM Infrastructure Backend:** Powers the infrastructure provisioning, credential inspection, and GPU model rollout workflows within the **Sera Admin Module (SAM)** UI.
- **Manager-Delegated Architecture:** The gRPC servicer layer acts as a lightweight router that initializes singleton domain managers at startup, maintaining cached client pools for SSH (Paramiko), Kubernetes, and HashiCorp Vault.
- **Dynamic Call Tracing & Background Auditing:** Automatically intercepts and logs `x-client-id` metadata headers (`[SAM]`, `[CLI]`, `[ORCH]`) across all RPCs while running hourly background daemons (`VaultAuditMonitor`, `HAProxyAuditMonitor`) that aggregate usage metrics into PostgreSQL.

### 2.2. Ecosystem Integration & Data Flow

```mermaid
graph TD
    subgraph Clients ["Clients, Operators & Control Plane"]
        SAM["Sera Admin Module (SAM UI)"]
        CLI["CMU Click CLI (cmu_cli.py)"]
        ORCH["GIA Orchestrator / Peers"]
    end

    subgraph CMU_Core ["Configuration Management Utility (CMU :50105)"]
        GRPC["gRPC Server (10 Workers)"]
        CMU_SVC["CmuServiceServicer"]
        SERA_SVC["SeraStandardServiceImpl (v1.2)"]
        MON_V["VaultAuditMonitor (60m Daemon)"]
        MON_H["HAProxyAuditMonitor (60m Daemon)"]
        
        subgraph Managers ["Domain Execution Managers"]
            VM["VaultManager / VaultClient"]
            KM["KeycloakManager"]
            HM["HAProxyManager"]
            PM["PostgreSQLManager"]
            VKM["ValkeyManager / RedisManager"]
            DM["DNSManager"]
            MM["ModelsManager / LoraManager"]
            KBM["KubeManager / PrometheusManager"]
            GM["GitlabManager / DiscoveryManager"]
            SM["SSHManager (Paramiko)"]
        end
    end

    subgraph Infra ["Multi-Region Infrastructure (SLO / PRG / NYC)"]
        VAULT[("HashiCorp Vault<br/>(KV v2 & AppRole)")]
        KC["Keycloak<br/>(Realm: aif-llm)"]
        HAP["HAProxy Fleet<br/>(6 Nodes: slo/prg/nyc)"]
        PG[("PostgreSQL / Patroni<br/>(CMU DB & App DBs)")]
        VK[("Valkey Global (9 Nodes)<br/>& Local Dev Redis")]
        AD["Active Directory DNS<br/>(GICORP.MYPOP2POP.COM)"]
        RANCHER["Rancher / Kubernetes<br/>(App & GPU Clusters)"]
        GITLAB["GitLab<br/>(aif_models & aif_common)"]
    end

    CLI -->|gRPC + x-client-id| GRPC
    SAM -->|gRPC + x-client-id| GRPC
    ORCH -->|gRPC + x-client-id| GRPC

    GRPC --> CMU_SVC
    GRPC --> SERA_SVC
    CMU_SVC --> Managers

    MON_V -->|Tail Audit Logs| SM
    MON_V -->|Upsert Metrics| PG
    MON_H -->|Tail Routing Logs| SM
    MON_H -->|Upsert Metrics| PG

    VM --> VAULT
    KM --> KC
    HM --> SM
    HM --> VAULT
    SM -->|SSH / SFTP| HAP
    SM -->|SSH| VAULT
    PM --> PG
    VKM --> VK
    DM -->|AXFR / nsupdate GSS-TSIG| AD
    MM --> GM
    MM --> KBM
    GM --> GITLAB
    KBM --> RANCHER
```

---

## 3. Module File Map 📂

```plaintext
/ai/utilities/configuration_management_utility
├── bootstrap_helper.py                     # Virtual environment and dependency bootstrapper
├── configuration_management_utility.py     # Main executable entry point; loads config and invokes Click CLI
├── requirements.txt                        # Python package dependencies (grpcio, hvac, kubernetes, paramiko, etc.)
├── README.md                               # Repository overview and links to central documentation
├── .gitignore                              # Git ignore rules
├── config/
│   ├── dns_gia.keytab                      # Dynamically written Kerberos keytab (0600) for AD DNS GSS-TSIG updates
│   └── settings.yaml                       # Master configuration file for CMU services, clusters, and Vault paths
├── docs/
│   ├── CHANGELOG.yaml                      # Structured release history parsed by AMU/SAM
│   └── cmu_design.md                       # Symlink -> /ai/assets/guides/utilities_guides/cmu_design.md
├── utils/
│   └── logging_setup.py                    # MultiProcessSafeDailyLogger with 14-day retention and console/file routing
├── cli/
│   ├── __init__.py                         # CLI package initializer
│   ├── cmu_cli.py                          # Root Click group (NestedHelpGroup), start-grpc-server, and init-db commands
│   ├── utils.py                            # gRPC channel factories, MetadataClientInterceptor, and JSON output helpers
│   └── commands/
│       ├── __init__.py                     # Commands package initializer
│       ├── app_cmds.py                     # CLI: `app` (deploy, verify)
│       ├── cache_cmds.py                   # CLI: `cache` (create, inspect, update-perms, delete, redis-inspect, redis-free)
│       ├── db_cmds.py                      # CLI: `db` (list, create, inspect, query, fix-ownership, delete)
│       ├── dns_cmds.py                     # CLI: `dns` (servers, list, create, delete)
│       ├── git_cmds.py                     # CLI: `git` (sync, push, tree)
│       ├── haproxy_cmds.py                 # CLI: `haproxy` (sync, apply)
│       ├── identity_cmds.py                # CLI: `identity` (create, delete, audit, list, get-config, set-desc, test, get-secret)
│       ├── lora_cmds.py                    # CLI: `lora` (upload, deploy, disable, list)
│       ├── models_cmds.py                  # CLI: `models` (deploy, verify, sync, inventory, list)
│       ├── module_cmds.py                  # CLI: `module` (info, check-health)
│       ├── rancher_cmds.py                 # CLI: `rancher` (rotate-tokens)
│       ├── secret_cmds.py                  # CLI: `secret` (list, get, write, move, delete, audit, users list/get)
│       └── volumes_cmds.py                 # CLI: `volumes` (search, expand, scan)
├── core/
│   ├── __init__.py                         # Core package initializer
│   ├── clients/
│   │   ├── __init__.py                     # Clients package initializer
│   │   └── vault_client.py                 # VaultClient wrapper around hvac with @reauthenticate_on_failure decorator
│   ├── database/
│   │   ├── __init__.py                     # Database package initializer
│   │   ├── database.py                     # SQLAlchemy read/write engine initialization and get_db_session context manager
│   │   └── models.py                       # ORM models: AiModelInventory, AiModelResource, VaultAuditMetric, LLMClientUsageMetric
│   ├── io_handlers/
│   │   ├── __init__.py                     # IO handlers package initializer
│   │   ├── sql_reader.py                   # Read queries for model inventory, LoRA PVCs, and Vault/LLM audit metrics
│   │   └── sql_writer.py                   # Transactional inventory replacement and PostgreSQL ON CONFLICT DO UPDATE upserts
│   ├── managers/
│   │   ├── __init__.py                     # Managers package initializer
│   │   ├── constants.py                    # Ecosystem validation constants (VALID_CATEGORIES, VALID_ENVIRONMENTS)
│   │   ├── discovery_manager.py            # ModelDiscoveryManager: K8s AI workload discovery & Git YAML reverse-indexing
│   │   ├── dns_manager.py                  # DNSManager: AD DC discovery, AXFR zone transfers, and kinit/nsupdate GSS-TSIG
│   │   ├── gitlab_manager.py               # GitlabManager: GitLab API file/tree fetching, shallow clones, and Kustomize overlay resolution
│   │   ├── haproxy_manager.py              # HAProxyManager: Config sync/apply, sk_aif_ key provisioning, live auth tests, log scraping
│   │   ├── keycloak_manager.py             # KeycloakManager: OIDC client provisioning, secret syncing, auth testing, and event audits
│   │   ├── kube_manager.py                 # KubeManager: Rancher K8s API client, Kustomize build, drift detection, scale-to-0 rollouts, PVC ops
│   │   ├── lora_manager.py                 # LoraManager: YAML LoRA injection/removal, ephemeral transfer pod orchestration, PVC stats
│   │   ├── models_manager.py               # ModelsManager: Sequential multi-tier model deployment, drift verification, and DB sync
│   │   ├── prometheus_manager.py           # PrometheusManager: Rancher proxy PromQL queries for PVC byte usage
│   │   ├── psql_manager.py                 # PostgreSQLManager: Patroni role/DB provisioning, ownership repair, safe deletion, raw SQL
│   │   ├── rancher_auth_manager.py         # RancherAuthManager: Local provider login, token rotation, and cluster ID mapping to Vault
│   │   ├── redis_manager.py                # RedisManager: Local uncontrolled Dev Redis auditing (DB 0-15) and flushing
│   │   ├── ssh_manager.py                  # SSHManager: Cached Paramiko SSH/SFTP client with sudo -S execution
│   │   ├── valkey_manager.py               # ValkeyManager: 9-node global cache/sentinel ACL provisioning, DB reservation, and inspection
│   │   └── vault_manager.py                # Vault KV v2 CRUD helpers, recursive tree/key search, AppRole/Policy inspection, leader log tailing
│   └── monitors/
│       ├── haproxy_audit_monitor.py        # Daemon thread: Hourly HAProxy log scraping and LLMClientUsageMetric DB upsert
│       └── vault_audit_monitor.py          # Daemon thread: Hourly Vault leader log scraping and VaultAuditMetric DB upsert
└── grpc_api/
    ├── __init__.py                         # gRPC API package initializer
    ├── protos/
    │   └── cmu.proto                       # Protocol Buffer definitions for CmuService
    ├── generate_stubs.py                   # Automated protoc stub generator with mtime staleness detection
    ├── cmu_service.py                      # CmuServiceServicer implementation with @handle_rpc logging/exception decorator
    ├── sera_standard_service.py            # SeraStandardServiceImpl (GetAppInfo, GetDeepHealth, GetFeedback, GetChangelog, GetDocs)
    └── services.py                         # ThreadPool gRPC server bootstrapper with Health and Server Reflection enabled
```

---

## 4. Core Processing Lifecycle ⚙️

### 4.1. Boot & Server Initialization Sequence
When invoked via `python3 configuration_management_utility.py start-grpc-server`:
1. **Environment Bootstrap:** `bootstrap_helper.bootstrap_and_reexecute_if_needed()` verifies the Python virtual environment and dependencies.
2. **Configuration & Logging:** `load_config()` parses `config/settings.yaml`. `setup_logging()` initializes `MultiProcessSafeDailyLogger` (writing `DEBUG` logs to `cmu.log.YYYY-MM-DD` and `WARNING+` to `cmu_err.log.YYYY-MM-DD` with a 14-day backup rotation) and sets stdout console logging to `INFO` (for server mode) or `WARNING` (for CLI mode).
3. **Dynamic Stub Verification:** `check_and_generate_stubs()` compares the modification timestamps (`mtime`) of `cmu.proto` and `sera_standard.proto` against the compiled `*_pb2.py` files in `/ai/assets/protos/configuration_management_utility/generated`. If missing or outdated, it invokes `grpc_tools.protoc` and executes `os.execv()` to restart the Python process cleanly with the newly generated stubs.
4. **Database Engine Setup:** `initialize_database()` creates separate SQLAlchemy connection pools for `write_dsn` (port `5432`) and `read_dsn` (port `5433`) with `pool_size=10`, `max_overflow=20`, and `pool_pre_ping=True`. `init_db()` ensures all declarative tables exist.
5. **Servicer & Manager Hydration (`CmuServiceServicer.__init__`):**
   - Authenticates `VaultClient` against `https://vault-aif.gt-t.net:8200` using AppRole (`role_id` / `secret_id`).
   - Loops through `rancher_clusters` (`prg`, `slo`, `nyc`) and reads `secret/rancher/{loc}/system/gia_ai` from Vault, injecting the live Rancher API `token` and `clusters` dictionary directly into the in-memory `config`.
   - Initializes singleton instances of `HAProxyManager`, `ValkeyManager` (using `valkey/global/system/gia_ai` credentials), `PostgreSQLManager` (using `postgres/global/system/gia_ai` credentials), `KeycloakManager` (authenticating against `master` realm `admin-cli` using `keycloak/global` credentials), `GitlabManager`, `ModelsManager`, and `DNSManager` (which decodes `keytab_b64` from `active_directory/global/shared/dns_gia` to `config/dns_gia.keytab` with `0600` permissions).
6. **Background Monitor Activation:** Spawns daemon threads for `VaultAuditMonitor` and `HAProxyAuditMonitor`.
7. **gRPC Server Binding:** `services.serve()` starts a `grpc.server` with `ThreadPoolExecutor(max_workers=10)` on `0.0.0.0:50105`, registering `CmuService`, `SeraAppService`, standard gRPC Health Checking (`grpc.health.v1.Health`), and Server Reflection.

### 4.2. Request Interception & Execution Lifecycle
1. **CLI Stub Initialization:** Whenever a CLI command runs, `MetadataClientInterceptor` intercepts unary-unary and unary-stream calls to inject `('x-client-id', 'CMU')` into the gRPC invocation metadata.
2. **Server-Side RPC Wrapper (`@handle_rpc`):**
   - Extracts `x-client-id` from context metadata.
   - Inspects request attributes (`path`, `client_id`, `model`, `cluster`, `username`, `dbname`, `zone`, `role_name`) and logs the invocation.
   - Wraps both standard return values and generator streams (`inspect.isgenerator`), catching any unhandled exceptions and returning a graceful protobuf response with `success=False, message=str(e)`.

---

## 5. Detailed Business Logic, Rules & HITL 🧠

CMU is a headless infrastructure control plane; **Human-in-the-Loop (HITL) learning workflows are not applicable for this module**, though interactive confirmation prompts (`click.confirm` / `click.prompt`) guard destructive CLI actions unless bypassed via `--yes` or `--force`.

### 5.1. Naming & Validation Rules
All database (`PostgreSQLManager`) and cache (`ValkeyManager`) provisioning strictly enforces `VALID_CATEGORIES` and `VALID_ENVIRONMENTS` defined in `core/managers/constants.py`:
- **Valid Categories:** `assistant`, `connector`, `framework`, `ingestor`, `processor`, `retriever`, `utility`, `gateway`, `analyzer`, `admin`
- **Valid Environments:** `dev`, `stg`, `prod`

### 5.2. Domain-Specific Provisioning & Operational Logic

#### A. Identity Provisioning & Auditing (HAProxy vs. Keycloak)
CMU supports two distinct identity architectures controlled via the `--keycloak` flag (`is_keycloak` boolean in gRPC):
1. **HAProxy Native API Keys (Default):**
   - **Creation (`provision_haproxy_identity_stream`):** Generates a random token prefixed with `sk_aif_` + 32 hex characters (`secrets.token_hex(16)`), computes its SHA-256 hex digest, stores metadata and the raw key in Vault at `secret/haproxy/global/apps/<client_id>`, appends `<sha256_hash>  <client_id>` to `cache/haproxy/llm_keys.map`, uploads the map via Paramiko SFTP to `/tmp/llm_keys.map` across all 6 HAProxy nodes, copies it to `/etc/haproxy/llm_keys.map`, and runs `systemctl reload haproxy`.
   - **Live Testing (`test_haproxy_identities`):** Reads client secrets from `secret/haproxy/global/apps/*` and fires live HTTP `GET` requests with `Authorization: Bearer <secret>` against `https://aif-llm-gemma4-e4b.aif.gt-t.net/v1/models`, verifying `200 OK` vs `401 Unauthorized`.
2. **Keycloak OIDC Clients (`--keycloak`):**
   - **Creation (`provision_full_identity_ecosystem`):** Generates a 32-character alphanumeric secret, stores it in Vault at `secret/keycloak/aif-llm/apps/<client_id>`, and creates a confidential OpenID Connect client in the `aif-llm` realm (`serviceAccountsEnabled=True`, `standardFlowEnabled=False`) with an `oidc-audience-mapper` injecting `app-backend` into access and introspection tokens. Includes automatic recovery if Keycloak returns a known HTTP `500` error despite creating the client.
   - **Testing (`IdentityTest`):** Compares the Vault secret (merging legacy `secret/keycloak/aif-llm` keys and `secret/keycloak/aif-llm/apps/*` documents) against the live Keycloak client configuration (`OK`, `WARNING`, or `FAILED` sync status) and tests OAuth2 `client_credentials` token generation.

#### B. HAProxy Configuration Management
- **Sync (`sync_configs_stream`):** SSHs into all 6 nodes in `haproxy_cluster`, reads `/etc/haproxy/haproxy.cfg` and `/etc/haproxy/llm_keys.map`, and writes them to `cache/haproxy/<node_fqdn>/`.
- **Apply (`apply_configs_stream`):** Targets a specific region (`slo`, `nyc`, `prg`) or `all`. For each node:
  1. Backs up the live remote `/etc/haproxy/haproxy.cfg` to `cache/haproxy/<node>/haproxy.cfg.<YYYYMMDD_HHMMSS>.bak`.
  2. Uploads the local cached `haproxy.cfg` via SFTP to `/tmp/haproxy.cfg.new`.
  3. Executes pre-flight syntax verification: `haproxy -c -f /tmp/haproxy.cfg.new`. If exit status != `0`, deletes the temp file and aborts for that node.
  4. Copies `/tmp/haproxy.cfg.new` to `/etc/haproxy/haproxy.cfg` and runs `systemctl reload haproxy`.

#### C. PostgreSQL / Patroni Provisioning
- **Naming Convention:** Database: `<module>_<category>_<env>_db`; Role: `<module>_<category>_<env>_user`.
- **Provisioning (`provision_full_db_ecosystem`):** Generates a 24-character alphanumeric password, executes `CREATE ROLE <role> WITH LOGIN PASSWORD ... ROLE <admin_user>` and `CREATE DATABASE <db> OWNER <role>` with `ISOLATION_LEVEL_AUTOCOMMIT`. Saves 4 DSNs (`dsn_k8s_write` on `db-write:5432`, `dsn_k8s_read` on `db-read:5433`, `dsn_ext_write` on `slo-aif-hadb.slo.gt-t.net:5432`, `dsn_ext_read` on `slo-aif-ha.slo.gt-t.net:5433`) to `secret/postgres/global/apps/<db_name>`.
- **Safe Teardown (`delete_full_db_ecosystem`):** Checks `pg_tables` for non-system tables. Refuses deletion if tables exist unless `force=True`. Terminates active backends via `pg_terminate_backend(pid)`, drops the database, executes `DROP OWNED BY <role> CASCADE`, drops the role, and deletes the Vault secret.

#### D. Valkey Global Cache & Dev Redis Management
- **Global Reservation (`provision_full_cache_ecosystem`):** Queries all 9 global nodes across `slo`, `nyc`, and `prg` to identify regional masters and occupied DB indices (DB `0` and `1` are reserved for shared/system use). Finds the first contiguous pair of free DB indices `(app_db_1, app_db_2)` starting from `max(start_db, 2)`. Writes `__GIA_DB_OWNER__ = "Reserved: <MODULE> (<CATEGORY>) - Setup Pending"` to both DBs on all regional masters.
- **ACL Application (`provision_app_user`):** Concurrently configures all 9 nodes (`ThreadPoolExecutor(max_workers=9)`) on both Cache (port `6379`) and Sentinel (port `26379`):
  - **Cache ACL (`6379`):** `on ><password> sanitize-payload ~* &* +@read +@write +@connection +@pubsub +@transaction +@scripting -@admin`
  - **Sentinel ACL (`26379`):** `on ><password> sanitize-payload resetchannels -@all +auth +ping +hello +role +sentinel|get-master-addr-by-name +sentinel|master +sentinel|masters +sentinel|replicas +sentinel|sentinels`
- **Vault Storage:** Saves credentials, DB indices, and 6 DSNs (`dsn_k8s_cache`, `dsn_ext_cache`, `dsn_k8s_cache_2`, `dsn_ext_cache_2`, `dsn_celery_broker`, `dsn_celery_result`) to `secret/valkey/global/apps/<username>`.

#### E. Active Directory DNS Management
- **DC Discovery & Filtering (`_get_dns_server`):** Resolves `GICORP.MYPOP2POP.COM` and filters results to healthy regional Domain Controller subnets (`10.250.15.`, `10.250.31.`, `10.250.47.`, `10.250.84.`) where the final octet is `5` or `6`.
- **Zone Resolution & AXFR (`list_records`):** Walks up the domain tree (`_get_primary_zone`) via `SOA` queries to find the root zone (e.g., `gt-t.net` for `aif.gt-t.net`), performs an AXFR zone transfer (`timeout=5.0s`) with automatic failover across DCs, and filters records matching the requested sub-zone.
- **Dynamic Updates (`_run_nsupdate`):** Acquires a Kerberos ticket via `kinit -V -k -t config/dns_gia.keytab <username>@GICORP.MYPOP2POP.COM` and pipes `server`, `zone`, and `update add/delete` commands into `nsupdate -g` (GSS-TSIG).

#### F. Kubernetes Volume Capacity & Auto-Expansion
- **Volume Scanning (`scan_volumes`):** Queries Rancher's Prometheus proxy (`kubelet_volume_stats_used_bytes`) in bulk, cross-referencing Kubernetes PVC capacities and active mounting pods to flag volumes exceeding `usage_threshold_pct` (default `90.0%`).
- **Proportional Auto-Expansion (`expand_volume`):** When `--size auto` is passed:
  - If current capacity is in `Gi` and `< 100Gi`: adds a flat `+5Gi`.
  - If current capacity is in `Gi` (`>= 100Gi`) or `Ti`: increases capacity by `+10%` (`int(val * 1.1)`).

---

## 6. AI, LLM & Machine Learning Execution 🤖

While CMU does not perform LLM inference itself, it is the primary orchestrator for deploying and managing **vLLM** and **NVIDIA NIM (`NIMService`)** inference engines and **LoRA (Low-Rank Adaptation)** weights across the GPU Kubernetes clusters (`*-aif-nvclu01`).

### 6.1. Sequential Multi-Tier Model Deployment (`ModelsManager.deploy_model`)
To prevent total capacity blackouts and GPU Out-Of-Memory (OOM) deadlocks during model updates:
1. **Manifest Resolution:** Resolves the model's Kustomize overlay (`clusters/<cluster>`) or raw YAML manifest via the database inventory or Git cache.
2. **Phase 1 — Base Infrastructure:** Filters and applies all non-compute resources first (`Service`, `PersistentVolumeClaim`, `ConfigMap`, `Ingress`).
3. **Phase 2 — Sequential Compute Rollout:** Filters `Deployment` and `NIMService` manifests (optionally restricted to a single GPU tier via `--tier`, e.g., `gpu2` or `gpu3`). For each deployment sequentially:
   - Applies the updated manifest to namespace `sera-llm`.
   - **Synchronous GPU Memory Teardown:**
     - Standard `Deployment` (`force_delete_deployment_pods`): Patches `spec.replicas = 0`, polls every `5s` (up to `180s`) until all matching pods physically terminate and release 100% of GPU VRAM, then patches `spec.replicas` back to the original count.
     - `NIMService` (`force_delete_nim_pods`): Because the NIM Operator enforces `replicas >= 1`, CMU deletes existing pods directly via `delete_namespaced_pod` and waits up to `180s` for termination before monitoring the new rollout.
   - **Dynamic Milestone-Aware Rollout Monitoring (`wait_for_deployment_rollout`):** Polls deployment status and tails the last `1000` lines of running vLLM container logs to stream real-time startup milestones (`Loading model from scratch`, `Prefetching checkpoint files`, `Loading weights took`, `Model loading took`, `Available KV cache memory`, `Graph capturing finished`, `JIT kernel warmup`, `Running FlashInfer autotune`, `torch.compile took`, `init engine ... took`, `Loaded new LoRA adapter`, `Starting vLLM server on http`, `Application startup complete`).
   - **Dynamic Idle Timeout Reset:** Uses a `900s` base timeout that automatically resets (`start_time = time.time()`) whenever a new vLLM log milestone is detected, preventing premature timeouts on large models or heavily loaded GPU nodes. If all desired replicas enter `CrashLoopBackOff`, `Error`, or non-zero exit states, CMU immediately dumps the last 50 error/traceback lines and aborts the rollout.
4. **Phase 3 — Post-Deploy Topology Sync:** Automatically triggers `sync_models('all')` to update the PostgreSQL inventory.

### 6.2. LoRA Adapter Lifecycle (`LoraManager`)
- **Tarball Streaming via Ephemeral Transfer Pod (`upload_lora` / `deploy_lora`):**
  1. Applies `pvc-lora.yaml` and spawns an ephemeral pod `lora-transfer-<model>` (`alpine:latest`, `sleep 3600`) mounting the LoRA PVC at `/data/lora` in namespace `sera-llm`.
  2. Executes `rm -rf /data/lora/<lora_name> && mkdir -p /data/lora/<lora_name>` inside the transfer pod.
  3. Compresses the local LoRA directory into a temporary `.tar.gz` archive and streams it in `128KB` chunks over the Kubernetes WebSocket exec API (`tar xzf - -C /data/lora/<lora_name>`), reporting transfer speed every `50MB`.
  4. Deletes the transfer pod.
- **GitOps YAML Injection (`inject_lora_to_yaml`):** Automatically patches the base `deployment.yaml` with `--enable-lora`, `--max-lora-rank 64`, `--lora-modules <lora_name>=/data/lora/<lora_name>`, and the `lora-volume` mount/claim, commits the change, pushes to GitLab (`aif_models`), and delegates the rollout to `ModelsManager.deploy_model`.

---

## 7. Data Models, State, Storage & Auditing 🗄️

CMU persists state in PostgreSQL (`sam_admin_prod_db`) using SQLAlchemy ORM (`core/database/models.py`), separating read-only queries (`read_dsn` on port `5433`) from mutating transactions (`write_dsn` on port `5432`).

### 7.1. PostgreSQL Schema Definitions

| Table Name | ORM Class | Primary Key / Constraints | Columns & Purpose |
| :--- | :--- | :--- | :--- |
| `ai_model_inventory` | `AiModelInventory` | `id` (UUID String) | `logical_name` (String), `model_name` (String, nullable), `engine_type` (String: `VLLM`, `NIM`, `unknown`), `cluster_location` (String: `slo`, `prg`, `nyc`), `last_synced` (DateTime UTC). Stores discovered K8s workloads. |
| `ai_model_resources` | `AiModelResource` | `id` (UUID String), FK `inventory_id` -> `ai_model_inventory.id` (Cascade Delete) | `kind` (`Deployments`, `Services`, `Endpoints`, `PVCs`, `Ingresses`, `ConfigMaps`, `Active Pods`), `namespace` (String), `name` (String), `details_json` (Text), `source_file` (String, GitLab repo path). |
| `vault_audit_metrics` | `VaultAuditMetric` | `id` (UUID String), Unique: `(date, user_identity, secret_path, operation)` | `date` (Date), `user_identity` (String), `secret_path` (String), `operation` (String), `access_count` (Integer), `last_accessed_at` (DateTime). Aggregated daily Vault KV access metrics. |
| `llm_client_usage_metrics` | `LLMClientUsageMetric` | `id` (UUID String), Unique: `(date, client_id, auth_type)` | `date` (Date), `client_id` (String), `auth_type` (`HAProxy` or `Keycloak`), `ping_count` (Integer, HTTP `GET`), `usage_count` (Integer, HTTP `POST`/other), `last_ping_at` (DateTime), `last_usage_at` (DateTime). |

### 7.2. Local Filesystem Cache State
- **Git Repositories:** Shallow clones (`--depth 1`) stored in `cache/gtt-ai-factory-models` (`aif_models`) and `cache/ai-infrastructure` (`aif_common`). Synchronized via `git fetch origin main && git reset --hard origin/main && git clean -fd`.
- **HAProxy Configurations:** Node-specific configs and timestamped backups stored in `cache/haproxy/<node_fqdn>/haproxy.cfg` and `llm_keys.map`, plus the central `cache/haproxy/llm_keys.map`.

---

## 8. APIs, Observability & Communication Contracts 📡

CMU exposes two gRPC services on port `50105`, along with standard gRPC Health Checking (`cmu.CmuService` and `sera.standard.v1.SeraAppService`) and Server Reflection.

### 8.1. `CmuService` RPC Contract (`cmu.proto`)

| Domain | RPC Method | Request Type | Response Type | Streaming | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Cache** | `CreateCache` | `CreateCacheRequest` | `CreateCacheResponse` | Unary | Provisions Valkey user, reserves 2 DB indices globally, saves DSNs to Vault. |
| **Cache** | `InspectCache` | `InspectCacheRequest` | `InspectCacheResponse` | Unary | Audits active masters, key counts, connections, and `__GIA_DB_OWNER__` tags across 9 nodes. |
| **Cache** | `InspectDevRedis` | `InspectDevRedisRequest` | `InspectDevRedisResponse` | Unary | Audits local Dev Redis (DB 0-15) for keys, clients, and free DB pairs. |
| **Cache** | `FreeDevRedis` | `FreeDevRedisRequest` | `FreeDevRedisResponse` | Unary | Executes `FLUSHDB` on a specific local Dev Redis DB index. |
| **Cache** | `UpdateCachePerms` | `UpdateCachePermsRequest` | `UpdateCachePermsResponse` | Unary | Re-applies Valkey & Sentinel ACL rules using existing password from Vault. |
| **Cache** | `DeleteCache` | `DeleteCacheRequest` | `DeleteCacheResponse` | Unary | Deletes Valkey ACL user across all 9 nodes, flushes assigned DBs, removes Vault secret. |
| **Vault** | `VaultList` | `VaultListRequest` | `VaultListResponse` | Unary | Lists Vault paths/keys flatly or recursively (`recursive=True`) as a JSON tree. |
| **Vault** | `VaultGet` | `VaultGetRequest` | `VaultGetResponse` | Unary | Reads a document, a specific key, or searches globally across Vault if `path` is empty. |
| **Vault** | `VaultWrite` | `VaultWriteRequest` | `VaultWriteResponse` | Unary | Merges and writes `kv_pairs` to `secret/<path>`. |
| **Vault** | `VaultMove` | `VaultMoveRequest` | `VaultMoveResponse` | Unary | Copies a key from `src_path`/`src_key` to `dest_path`/`dest_key` and deletes source key. |
| **Vault** | `VaultDelete` | `VaultDeleteRequest` | `VaultDeleteResponse` | Unary | Deletes a single key or recursively deletes an entire Vault path and all versions. |
| **Vault** | `VaultAudit` | `VaultAuditRequest` | `VaultAuditResponse` | Unary | Returns aggregated DB metrics (`raw=False`) or live SSH-tailed leader logs (`raw=True`). |
| **Vault IAM** | `VaultUserList` | `VaultUserListRequest` | `VaultUserListResponse` | Unary | Lists all AppRole roles in Vault. |
| **Vault IAM** | `VaultUserGet` | `VaultUserGetRequest` | `VaultUserGetResponse` | Unary | Reads AppRole configuration details. |
| **Vault IAM** | `VaultPolicyList` | `VaultPolicyListRequest` | `VaultPolicyListResponse` | Unary | Lists all ACL policies in Vault. |
| **Vault IAM** | `VaultPolicyGet` | `VaultPolicyGetRequest` | `VaultPolicyGetResponse` | Unary | Reads raw HCL policy rules for a given policy name. |
| **Database** | `DbList` | `DbListRequest` | `DbListResponse` | Unary | Lists PostgreSQL databases and/or roles matching `filter_pattern`. |
| **Database** | `DbCreate` | `DbCreateRequest` | `DbCreateResponse` | Unary | Provisions Patroni role and database, stores 4 DSNs in Vault. |
| **Database** | `DbInspect` | `DbInspectRequest` | `DbInspectResponse` | Unary | Lists all non-system tables and their owners in `dbname`. |
| **Database** | `DbQuery` | `DbQueryRequest` | `DbQueryResponse` | Unary | Executes raw SQL with autocommit against `dbname` and returns columns/rows. |
| **Database** | `DbDelete` | `DbDeleteRequest` | `DbDeleteResponse` | Unary | Safely drops database, role, and Vault secret (enforces empty-table check unless `force=True`). |
| **Identity** | `IdentityList` | `IdentityListRequest` | `IdentityListResponse` | Unary | Lists HAProxy clients from Vault or Keycloak clients from realm `aif-llm`. |
| **Identity** | `IdentityGetConfig` | `IdentityGetConfigRequest` | `IdentityGetConfigResponse` | Unary | Downloads full Keycloak client JSON representation. |
| **Identity** | `IdentitySetDesc` | `IdentitySetDescRequest` | `IdentitySetDescResponse` | Unary | Updates description for a Keycloak client. |
| **Identity** | `IdentityTest` | `IdentityTestRequest` | `IdentityTestResponse` | Unary | Tests HAProxy keys via live gateway HTTP requests or Keycloak clients via OAuth2 grant. |
| **Identity** | `IdentityGetSecret` | `IdentityGetSecretRequest` | `IdentityGetSecretResponse` | Unary | Retrieves raw client secret from Vault. |
| **Identity** | `IdentityCreate` | `IdentityCreateRequest` | `IdentityCreateResponse` | **Server Stream** | Provisions HAProxy API key (and pushes to fleet) or Keycloak client + Vault secret. |
| **Identity** | `IdentityDelete` | `IdentityDeleteRequest` | `IdentityDeleteResponse` | **Server Stream** | Revokes HAProxy API key across fleet/Vault or deletes Keycloak client + Vault secret. |
| **Identity** | `IdentityAudit` | `IdentityAuditRequest` | `IdentityAuditResponse` | Unary | Queries `LLMClientUsageMetric` table for ping/usage counts and timestamps. |
| **Deployment** | `GitListTree` | `GitListTreeRequest` | `GitListTreeResponse` | Unary | Fetches recursive repository tree from GitLab (`aif_models` or `aif_common`). |
| **Deployment** | `GitGetFile` | `GitGetFileRequest` | `GitGetFileResponse` | Unary | Fetches raw file content from GitLab API. |
| **Deployment** | `GitSync` | `GitSyncRequest` | `GitSyncResponse` | Unary | Syncs and hard-resets local Git cache repository. |
| **Deployment** | `GitPush` | `GitPushRequest` | `GitPushResponse` | Unary | Commits and pushes local cache modifications to GitLab `main`. |
| **Deployment** | `DeployModel` | `DeployModelRequest` | `DeployModelResponse` | **Server Stream** | Sequentially deploys model manifests/Kustomize overlays with GPU teardown & log tracking. |
| **Deployment** | `DeployApp` | `DeployAppRequest` | `DeployAppResponse` | Unary | Deploys an application YAML manifest from `aif_common` to a target cluster. |
| **Deployment** | `VerifyModel` | `VerifyModelRequest` | `VerifyModelResponse` | Unary | Detects drift between GitLab manifests/Kustomize and live GPU clusters. |
| **Deployment** | `VerifyApp` | `VerifyAppRequest` | `VerifyAppResponse` | Unary | Detects drift between `aif_common` manifests and live App clusters. |
| **Deployment** | `ModelsSync` | `ModelsSyncRequest` | `ModelsSyncResponse` | Unary | Runs global K8s discovery and updates `AiModelInventory` in PostgreSQL. |
| **Deployment** | `ModelsInventory` | `ModelsInventoryRequest` | `ModelsInventoryResponse` | Unary | Queries `AiModelInventory` and `AiModelResource` topology from PostgreSQL. |
| **Deployment** | `ModelsList` | `ModelsListRequest` | `ModelsListResponse` | Unary | Returns partitioned lists of `models` vs `other` infrastructure workloads. |
| **Volumes** | `VolumesSearch` | `VolumesSearchRequest` | `VolumesSearchResponse` | Unary | Searches PVCs/PVs across all Rancher clusters with Prometheus usage stats. |
| **Volumes** | `VolumesExpand` | `VolumesExpandRequest` | `VolumesExpandResponse` | Unary | Resizes a PVC (`new_size` or `auto` for +5Gi/+10%). |
| **Volumes** | `VolumesScan` | `VolumesScanRequest` | `VolumesScanResponse` | Unary | Scans all clusters for PVCs exceeding `usage_threshold` percentage. |
| **LoRA** | `LoraUpload` | `LoraUploadRequest` | `LoraUploadResponse` | **Server Stream** | Streams LoRA weights to PVC via transfer pod without restarting the model. |
| **LoRA** | `LoraDeploy` | `LoraDeployRequest` | `LoraDeployResponse` | **Server Stream** | Injects LoRA into Git YAML, streams weights to PVC, and rolls out deployment. |
| **LoRA** | `LoraDisable` | `LoraDisableRequest` | `LoraDisableResponse` | **Server Stream** | Strips specific or `--all` LoRAs from Git YAML and rolls out deployment. |
| **LoRA** | `LoraList` | `LoraListRequest` | `LoraListResponse` | Unary | Lists LoRAs on PVCs (`du -sh`, `ls -ld`) and checks if applied in `deployment.yaml`. |
| **DNS** | `DnsList` | `DnsListRequest` | `DnsListResponse` | Unary | Performs AXFR zone transfer and returns filtered DNS records. |
| **DNS** | `DnsCreate` | `DnsCreateRequest` | `DnsCreateResponse` | Unary | Creates a DNS record via GSS-TSIG `nsupdate`. |
| **DNS** | `DnsDelete` | `DnsDeleteRequest` | `DnsDeleteResponse` | Unary | Deletes a DNS record via GSS-TSIG `nsupdate`. |
| **DNS** | `DnsServers` | `DnsServersRequest` | `DnsServersResponse` | Unary | Resolves AD domain controllers and performs reverse PTR lookups. |
| **HAProxy** | `SyncHAProxyConfigs` | `SyncHAProxyConfigsRequest` | `SyncHAProxyConfigsResponse` | **Server Stream** | Pulls `haproxy.cfg` and `llm_keys.map` from all 6 HAProxy nodes into local cache. |
| **HAProxy** | `ApplyHAProxyConfigs` | `ApplyHAProxyConfigsRequest` | `ApplyHAProxyConfigsResponse` | **Server Stream** | Backs up, SFTP uploads, validates (`haproxy -c`), and reloads `haproxy.cfg` on target nodes. |

*Note: `DbFixOwnershipRequest` and `DbFixOwnershipResponse` messages are defined in `cmu.proto` and implemented in `CmuServiceServicer.DbFixOwnership`, though the `rpc DbFixOwnership` line is currently omitted from the `service CmuService` block in `cmu.proto`.*

### 8.2. `SeraAppService` Standard Contract (`sera_standard_service.py`)
CMU implements the universal SERA standard contract (`sera.standard.v1.SeraAppService` - `v1.2-universal`), centrally maintained and deployed by the Application Management Utility (AMU). It provides standardized endpoints (`GetAppInfo`, `GetDeepHealth`, `GetFeedback`, `GetChangelog`, `GetDocs`) enabling AMU and the SAM UI to self-discover CMU's metadata, deep TCP socket dependency health matrix, changelog history, and design documentation without requiring custom integration logic.

---

## 9. User Interface (UI) & Visualizations 🖥️

*Not applicable for this module.* CMU is a headless backend utility that exposes rich terminal output via its Click CLI and serves data to the Sera Admin Module (SAM) UI over gRPC.

---

## 10. Background Services & Workers 🛠️

CMU spawns two autonomous daemon threads upon gRPC server initialization (`CmuServiceServicer.__init__`):

### 10.1. `VaultAuditMonitor` (`core/monitors/vault_audit_monitor.py`)
- **Trigger & Cadence:** Runs in a background daemon thread (`_run_audit_sync_daemon`) immediately on startup and sleeps for `3600` seconds (60 minutes) between cycles.
- **Execution Logic:**
  1. Queries `SQLReader.get_max_vault_audit_cursor()` to retrieve the latest `last_accessed_at` timestamp from `vault_audit_metrics`.
  2. Calls `audit_vault_logs()` with `lines=100000`: queries `https://<fqdn>:8200/v1/sys/leader` across `vault_cluster` nodes to find the active leader (`is_self == True`), SSHs into the leader using `syseng_ansible` credentials, and runs `cat /var/log/vault_audit.log /opt/vault/vault_audit.log 2>/dev/null | grep 'secret/data' | tail -n 100000`.
  3. Parses JSON audit lines, truncates nanosecond ISO8601 timestamps to microseconds, filters out records `<= cursor_time`, and aggregates hit counts and high-watermark timestamps by `(date, display_name, path, operation)`.
  4. Executes PostgreSQL `ON CONFLICT (date, user_identity, secret_path, operation) DO UPDATE` via `SQLWriter.upsert_vault_audit_metrics()`.

### 10.2. `HAProxyAuditMonitor` (`core/monitors/haproxy_audit_monitor.py`)
- **Trigger & Cadence:** Runs in a background daemon thread (`_run_audit_sync_daemon`) immediately on startup and sleeps for `3600` seconds (60 minutes) between cycles.
- **Execution Logic:**
  1. Queries `SQLReader.get_max_llm_client_usage_cursor()` (`MAX(GREATEST(last_ping_at, last_usage_at))`).
  2. Connects via SSH to all 6 nodes in `haproxy_cluster` and runs:
     `cat /var/log/haproxy.log 2>/dev/null | grep 'internal_llm_router' | grep '{' | tail -n 100000`
  3. Parses lines matching regex `^([^ ]+).*internal_llm_router.*\{([^\|]+)\|([^\}]+)\}\s+"([A-Z]+)\s` to extract `(timestamp_str, client_id, auth_type, http_method)`.
  4. Filters out records `<= cursor_time` and partitions traffic by HTTP method:
     - `GET` requests increment `ping_count` and update `last_ping_at` (e.g., `/v1/models` health checks).
     - All other methods (e.g., `POST /v1/chat/completions`) increment `usage_count` and update `last_usage_at`.
  5. Upserts aggregated metrics into `llm_client_usage_metrics` using `func.coalesce` to preserve existing non-null timestamps.

---

## 11. CLI, Tooling & Testing 💻

All CLI commands support `--grpc` (where applicable) to output raw protobuf responses formatted as JSON via `print_grpc_json()`.

| Command Group | Subcommands & Key Options | Operational Description |
| :--- | :--- | :--- |
| **Root (`cli`)** | `start-grpc-server`<br/>`init-db [--force]` | Starts the CMU gRPC server on port `50105`.<br/>Initializes (or wipes and recreates with `--force`) PostgreSQL tables. |
| **`module`** | `info [--grpc]`<br/>`check-health [--grpc]` | Calls `SeraAppService.GetAppInfo` and `GetDeepHealth`. |
| **`secret`** | `list [-p PATH] [-r]`<br/>`get [KEY] [-p PATH]`<br/>`write <PATH> <K=V>...`<br/>`move <SRC_P> <SRC_K> <DST_P> <DST_K>`<br/>`delete <PATH> [-k KEY] [-y]`<br/>`audit [SEARCH] [-n 100] [--raw]`<br/>`users list` / `users get <ROLE>` | Full Vault KV v2 management, global key search, recursive tree rendering, aggregated or raw SSH audit log inspection, and AppRole/ACL policy inspection. |
| **`identity`** | `create <ID> --env --requestor --description [--keycloak]`<br/>`delete <ID> [--keycloak]`<br/>`audit [ID] [--keycloak]`<br/>`list`<br/>`get-config <ID>`<br/>`set-desc <ID> <DESC>`<br/>`test [ID] [--keycloak]`<br/>`get-secret <ID>` | Manages HAProxy `sk_aif_` API keys (default) or Keycloak OIDC clients (`--keycloak`). Tests live gateway routing or OAuth2 grants and queries LLM usage metrics. |
| **`haproxy`** | `sync [--grpc]`<br/>`apply --location [slo\|nyc\|prg\|all]` | Syncs `/etc/haproxy/haproxy.cfg` and `llm_keys.map` from all 6 HAProxy nodes or validates and applies cached configs to target regions. |
| **`db`** | `list [-t dbs\|roles\|all] [-f FILTER]`<br/>`create <MOD> <CAT> <ENV>`<br/>`inspect <DBNAME>`<br/>`query <DBNAME> <SQL> [-y]`<br/>`fix-ownership <DBNAME> <OWNER> [-y]`<br/>`delete <DBNAME> [--force] [-y]` | Manages Patroni PostgreSQL databases and roles, repairs table ownership, executes raw SQL (with destructive keyword prompts), and safely tears down DBs. |
| **`cache`** | `create <MOD> <CAT> <ENV> [--start-db 1]`<br/>`inspect`<br/>`update-perms <USER>`<br/>`delete <USER> [--yes]`<br/>`redis-free <DB_INDEX> [--yes]` | Manages global 9-node Valkey ACLs and DB reservations, plus local Dev Redis inspection and DB flushing. |
| **`dns`** | `servers`<br/>`list <ZONE>`<br/>`create <ZONE> <NAME> <TYPE> <VAL> [--ttl 3600]`<br/>`delete <ZONE> <NAME> <TYPE>` | Lists AD DNS servers, executes AXFR zone transfers, and performs secure GSS-TSIG dynamic DNS updates. |
| **`git`** | `sync [--domain aif_models\|aif_common]`<br/>`push --domain ... -m <MSG>`<br/>`tree [--domain ...] [--path ...]` | Syncs, commits/pushes, or lists repository trees for GitLab infrastructure repos. |
| **`models`** | `deploy --model <NAME> [--cluster all] [--tier gpu2\|gpu3] [--verify]`<br/>`verify [--model NAME \| --file-path PATH] [--cluster all] [--show-live]`<br/>`sync [--cluster NAME]`<br/>`inventory [--model] [--location] [--namespace]`<br/>`list [--location]` | Orchestrates sequential vLLM/NIM model rollouts, multi-cluster Kustomize drift detection, global K8s topology discovery, and database inventory queries. |
| **`lora`** | `upload --model <M> --path <P> --location <L> --lora-name <N>`<br/>`deploy --model <M> --path <P> --location <L> --lora-name <N> [--tier T]`<br/>`disable --model <M> --location <L> [--lora-name N \| --all]`<br/>`list [--model M] [--location all]` | Uploads, enables, disables, or lists LoRA adapters across GPU clusters. |
| **`app`** | `deploy --file-path <P> [--cluster slo] [--namespace default]`<br/>`verify --file-path <P> [--cluster all] [--namespace] [--show-live]` | Deploys or verifies standard application manifests from `aif_common`. |
| **`volumes`** | `search <VOL_NAME> [--cluster]`<br/>`expand --cluster <C> --namespace <NS> --pvc-name <P> --size <S\|auto>`<br/>`scan [--cluster all] [--usage-threshold 90.0]` | Searches PVCs/PVs globally, expands PVC storage dynamically, and scans for high-utilization volumes via Prometheus. |
| **`rancher`** | `rotate-tokens` | Rotates Rancher API tokens via local provider login across all regions and updates `secret/rancher/{loc}/system/gia_ai` in Vault. |

---

## 12. Configuration, Security & Hardware Affinity 🔒

### 12.1. Configuration Parameters (`config/settings.yaml`)

| Configuration Key | Default / Configured Value | Purpose |
| :--- | :--- | :--- |
| `module_info.acronym` | `"CMU"` | Module identifier injected into `x-client-id` gRPC metadata. |
| `module_info.version` | `"0.3.5"` | Static config version (runtime `GetAppInfo` reads `docs/CHANGELOG.yaml`). |
| `system_services.grpc.service_name` | `"cmu-grpc.service"` | Systemd unit name checked during health probes. |
| `system_services.grpc.host` / `port` | `"0.0.0.0"` / `50105` | Bind address and port for the CMU gRPC server. |
| `system_services.grpc.max_workers` | `10` | ThreadPoolExecutor worker count for gRPC requests. |
| `logging.level` / `directory` | `"INFO"` / `"/ai/assets/logs/cmu_utility"` | Log verbosity and target directory for daily rotating log files. |
| `paths.proto_source_dir` | `"grpc_api/protos"` | Relative directory containing `cmu.proto`. |
| `paths.sera_proto_source_dir` | `"/ai/assets/protos/sera_standard"` | Shared directory containing `sera_standard.proto`. |
| `paths.generated_stubs_path` | `"/ai/assets/protos/configuration_management_utility/generated"` | Output path for compiled `*_pb2.py` and `*_pb2_grpc.py` stubs. |
| `database.write_dsn` | `postgresql://sam_admin_prod_user:***@slo-aif-hadb.slo.gt-t.net:5432/sam_admin_prod_db` | Primary read/write Patroni PostgreSQL connection string. |
| `database.read_dsn` | `postgresql://sam_admin_prod_user:***@slo-aif-ha.slo.gt-t.net:5433/sam_admin_prod_db` | Read-replica Patroni PostgreSQL connection string. |
| `external_services.active_directory` | `domain: GICORP.MYPOP2POP.COM`, `vault_secret_path: active_directory/global/shared/dns_gia` | AD DNS domain and Vault path for Kerberos keytab credentials. |
| `external_services.vault.url` | `"https://vault-aif.gt-t.net:8200"` | HashiCorp Vault API endpoint (AppRole authenticated). |
| `external_services.segitlab.groups` | `aif_models` (`SysEng/gtt-ai-factory-models`), `aif_common` (`AI/common`) | GitLab project paths, OAuth tokens, and local cache clone directories. |
| `external_services.keycloak` | `url: https://keycloak-aif.gt-t.net`, `realm: aif-llm` | Keycloak base URL and target OIDC realm. |
| `rancher_clusters` | `prg`, `slo`, `nyc` (`*-aif-clu01` & `*-aif-nvclu01`) | Regional Rancher URLs and target App/GPU cluster names. |
| `postgres_cluster` | `host: slo-aif-hadb.slo.gt-t.net`, `port: 5432`, `cluster_name: slo-aif-pgcluster` | Target PostgreSQL cluster for application DB provisioning. |
| `cache_clusters.redis_dev.url` | `"redis://:***@localhost:6379"` | Local development Redis instance URL. |
| `cache_clusters.valkey_global_nodes` | 9 nodes (`{slo,nyc,prg}-aif-rds0{1..3}.*.gt-t.net`) | Global Valkey and Sentinel cluster nodes. |
| `vault_paths` | `rancher_admin`, `valkey_admin`, `psql_admin`, `syseng_ansible`, `keycloak_admin` | Canonical Vault KV v2 paths for administrative credentials. |
| `vault_cluster` | `nyc-aif-vlt01`, `prg-aif-vlt01`, `slo-aif-vlt01` | Vault cluster FQDNs polled for leader discovery during log audits. |
| `haproxy_cluster` | 6 nodes (`{slo,prg,nyc}-aif-ha0{1,2}.*.gt-t.net`) | Global HAProxy load balancer fleet nodes. |

### 12.2. Security & Privilege Isolation
- **Vault Session Auto-Renewal:** `VaultClient` disables proxy environment interference (`session.proxies = {"http": None, "https": None}`) and decorates credential-fetching methods with `@reauthenticate_on_failure`, automatically re-running AppRole login if a token expires or returns `403 Forbidden` / `permission denied`.
- **Keycloak Token Refresh:** `KeycloakManager` tracks `token_expires_at` (`expires_in - 5s`) and transparently re-authenticates with the `master` realm `admin-cli` before issuing admin REST calls.
- **In-Memory SSH Keys & Sudo Sanitization:** `SSHManager` loads the `syseng_ansible` OpenSSH private key (supporting RSA and Ed25519) directly from memory (`io.StringIO`) without writing private keys to disk, executes root commands via `sudo -S -p '' bash -c ...` with `get_pty=False` (preventing wtmp log spam), and strips `sudo_password` strings from stdout streams.
- **Kerberos Keytab Permissions:** `DNSManager` writes `config/dns_gia.keytab` with strict `0600` file permissions prior to invoking `kinit`.

### 12.3. Hardware Affinity & Execution Boundaries
- CMU itself is lightweight and CPU-bound, running on standard application nodes.
- All AI model and LoRA operations (`DeployModel`, `VerifyModel`, `LoraUpload`, `LoraDeploy`, `LoraDisable`, `LoraList`, `ModelDiscoveryManager`) explicitly target Rancher GPU clusters (`is_gpu=True` -> `gpu_cluster_name`: `prg-aif-nvclu01`, `slo-aif-nvclu01`, `nyc-aif-nvclu01`) and support multi-pool GPU hardware tiers (`--tier gpu2`, `--tier gpu3`).

---

## 13. Error Handling & Resilience 🛡️

- **Unified gRPC Fault Barrier (`@handle_rpc`):** Every `CmuService` method (both unary and streaming generator RPCs) is wrapped by `@handle_rpc`, which catches any unhandled exception, logs the error alongside the caller's `x-client-id`, and returns a structured protobuf response (`success=False, message=str(e)`) rather than crashing the worker thread.
- **HAProxy Pre-Flight Validation & Rollback Protection:** `HAProxyManager.apply_configs_stream` backs up the live remote config before uploading changes to `/tmp/haproxy.cfg.new` and runs `haproxy -c -f /tmp/haproxy.cfg.new`. If syntax validation fails, the temporary file is removed and the live HAProxy process is left untouched.
- **vLLM OOM Prevention & Crash Diagnostics:** `KubeManager.force_delete_deployment_pods` scales deployments to `0` and waits for physical pod termination before scaling up, preventing GPU OOM deadlocks. During rollout, `wait_for_deployment_rollout` resets its `900s` timeout clock on every verified vLLM log milestone and automatically extracts container error/traceback logs if pods enter `CrashLoopBackOff`.
- **Smart Kubernetes Drift Filtering (`_find_drift`):** Matches lists of dictionaries by their `name` key (preventing false-positive drift alerts when Kustomize reorders environment variables or mounts), ignores empty dicts/lists, flags Kubernetes-dropped default values (`0`, `False`, `""`) with actionable hints, and truncates string mismatches over 50 characters.
- **DNS Multi-DC Failover:** `DNSManager.list_records` maintains an `exclude_ips` set during AXFR zone transfers, automatically failing over across regional Active Directory Domain Controllers if a server refuses transfer or times out.

---

## 14. Future Enhancements & Roadmap 🚀

The following capabilities define the future architectural state of CMU, extending its role as the definitive control plane for ecosystem infrastructure and addressing next-generation operational requirements.

### 14.1. Structured Mutation Audit Ledger (`cmu_audit_log`)
While CMU v0.3.5 already runs autonomous background monitors for Vault secret access (`vault_audit_metrics`) and HAProxy LLM routing (`llm_client_usage_metrics`), it still requires a deterministic Audit Ledger in PostgreSQL to track mutating control-plane actions. Because Kubernetes and Rancher audit logs are noisy and fail to capture logical business intent, CMU will act as a strict tollbooth for all infrastructure changes. By forcing deployments and provisioning through CMU's gRPC endpoints, any cluster or infrastructure modification not present in the CMU audit table can be immediately flagged as a rogue, manual intervention (e.g., a direct root override on a node).

This will be implemented directly within the `@handle_rpc` decorator and mutating gRPC endpoint handlers (e.g., `DeployModel`, `LoraDeploy`, `DbCreate`, `CreateCache`, `VaultWrite`, `DnsCreate`, `ApplyHAProxyConfigs`, `GitPush`). Read-only inspection and list operations will bypass the audit logger to prevent database bloat.

**Table Schema (`cmu_audit_log`):**
- **`log_id`** (UUID, Primary Key)
- **`actor`** (String): Extracted automatically via `x-client-id` gRPC metadata to identify the specific service account, CLI operator, or upstream module (e.g., `SAM`, `AMU`) triggering the action.
- **`domain`** (String): The targeted infrastructure domain (`Models`, `LoRA`, `Apps`, `Cache`, `Database`, `Vault`, `Identity`, `DNS`, `HAProxy`, `Volumes`).
- **`target_cluster`** (String): The destination environment or region (`slo`, `prg`, `nyc`, `global`).
- **`action`** (String): The explicit operation performed (e.g., `DEPLOY_MODEL`, `DEPLOY_LORA`, `CREATE_DB`, `APPLY_HAPROXY`, `EXPAND_VOLUME`).
- **`payload_reference`** (String): Contextual metadata regarding the target payload (e.g., `models/k8s/vllm/gemma-4-31b/base/deployment/deployment.yaml`, `secret/postgres/global/apps/...`, or a specific PVC/DNS record name).
- **`status`** (String): Operational result (`SUCCESS` or `FAILED`).
- **`timestamp`** (DateTime): The exact UTC execution time.

### 14.2. Autonomous Background Volume Expansion Worker
While CMU already implements on-demand Prometheus capacity scanning (`volumes scan`) and proportional auto-expansion (`volumes expand --size auto` adding `+5Gi` under `100Gi` or `+10%` at/above `100Gi`), it will be extended with an autonomous background daemon (similar to `VaultAuditMonitor` and `HAProxyAuditMonitor`) to automate storage scaling completely:
- **The Logic:** Periodically scans all regional App and GPU Kubernetes clusters via the `VolumesScan` PromQL pipeline (`kubelet_volume_stats_used_bytes`). Any PVC crossing the critical threshold (e.g., `90%` utilization) automatically triggers `KubeManager.expand_volume(..., new_size='auto')`.
- **The Audit Threshold (Circuit Breaker):** Integrates directly with `cmu_audit_log`. If a specific PVC triggers autonomous expansion more than 3 consecutive times within a defined window, CMU will halt auto-expansion for that volume and dispatch an alert via OMU to prevent runaway storage allocation caused by application memory/disk leaks.

### 14.3. HAProxy Runtime Socket & Data Plane API Integration
In v0.3.5, `HAProxyManager` orchestrates `haproxy.cfg` and `llm_keys.map` updates via Paramiko SFTP uploads, `haproxy -c` pre-flight validation, and `systemctl reload haproxy`. While reliable, full process reloads briefly spawn new worker processes across the fleet.
- **Runtime Socket / Data Plane API:** Future iterations will integrate with the HAProxy Runtime API (via Unix socket over SSH or the HTTP Data Plane API) to inject and revoke `sk_aif_` SHA-256 key hashes directly into in-memory map tables (`add map` / `del map` on `/etc/haproxy/llm_keys.map`) and dynamically drain or weight backend GPU cluster endpoints with zero process reloads.

### 14.4. Vector Database (Qdrant / ChromaDB) Provisioning Domain
While `settings.yaml` already defines `vault_paths.qdrant_admin` (`qdrant/global/system/admin`) and `SeraStandardServiceImpl` checks vector database health, CMU does not yet expose a dedicated CLI/gRPC domain for vector store lifecycle management:
- **The Logic:** Introduce a `QdrantManager` and `vdb` CLI/gRPC command group (`vdb create`, `vdb inspect`, `vdb delete`) to automate collection provisioning, API key/ACL generation, storage quota configuration, and automatic credential persistence to `secret/qdrant/global/apps/<module>_<category>_<env>_vdb`.

### 14.5. Protocol Buffer & Database Namespace Alignment
- **Database Table Prefixing (`cmu_*`):** Because all SAM administrative utilities (AMU, CMU, HMU, OMU) share the `sam_admin_prod_db` PostgreSQL database, future schema migrations will standardize table names with a strict `cmu_` prefix (`cmu_ai_model_inventory`, `cmu_ai_model_resources`, `cmu_vault_audit_metrics`, `cmu_llm_client_usage_metrics`, `cmu_audit_log`) to guarantee zero namespace collisions across utilities.
- **Expose `DbFixOwnership` in `cmu.proto`:** Add `rpc DbFixOwnership (DbFixOwnershipRequest) returns (DbFixOwnershipResponse);` to the `service CmuService` block in `grpc_api/protos/cmu.proto` to match the existing message definitions, CLI command (`db fix-ownership`), and `CmuServiceServicer.DbFixOwnership` implementation.
- **Automated Audit Table Retention Worker:** Implement a scheduled cleanup routine within `VaultAuditMonitor` and `HAProxyAuditMonitor` to prune or roll up `vault_audit_metrics` and `llm_client_usage_metrics` rows older than 6 months, ensuring bounded table growth and fast query performance.