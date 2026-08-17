# Configuration Management Utility (CMU) - Design Document

**Version:** 0.2.0
**Date:** 2026-07-19
**Author:** Martin Kecka & Gemini AI
**Category:** (U) Utility / Administrative
**Acronym:** CMU

---

## 1. Introduction & Purpose 📜

The **Configuration Management Utility (CMU)** is a dedicated microservice in the SERA AI ecosystem designed to centralize and orchestrate logical configuration, identity, and infrastructure provisioning. 

CMU establishes a strict architectural boundary by isolating all REST/API-driven configuration lifecycle events into a single, cohesive control plane via a unified gRPC interface and CLI tool. This separation of concerns ensures that stateful, destructive node operations (handled by the Operations Management Utility - OMU) and application GitOps lifecycles (handled by the Application Management Utility - AMU) remain strictly decoupled from logical resource provisioning. By consolidating identity management, secret distribution, traffic routing, AI infrastructure orchestration, and native module `settings.yaml` configuration, CMU provides a unified interface for the Sera Admin Module (SAM) to manage ecosystem state securely and efficiently.

---

## 2. Core Architectural Principles (Architectural Boundaries) 🏗️

In the modern 4-pillar utility architecture, CMU's boundaries are strictly defined against its sibling services:

* **HMU (Watcher):** Read-only observability and telemetry across the ecosystem.
* **AMU (Deployer):** Application GitOps, portfolio management, and ServiceNow feedback integration.
* **OMU (Operator):** High-privilege, destructive OS-level patching, SSH execution, and Keepalived/VIP health tracking.
* **CMU (Configurator):** API-driven logical configuration and provisioning (Identity, Secrets, Databases, Caches, Traffic Routing, AI Infra weights, and Module `settings.yaml` state).

CMU is specifically designed to keep logical resource provisioning completely isolated from stateful node operations (OMU) and application payload deployments (AMU).

---

## 3. Database Schema & State Management 🗄️

CMU utilizes a PostgreSQL database to manage its structured audit trails and topology cache via robust `SQLReader` and `SQLWriter` abstractions.

### 3.1. Structured Audit Logging (The Ledger)
CMU implements a deterministic Audit Ledger (`cmu_audit_log`). By forcing deployments through CMU's gRPC endpoints, any cluster change not present in the CMU audit table can be immediately identified as a rogue intervention.
* **Schema:** `log_id`, `actor`, `domain`, `target_cluster`, `action`, `payload_reference`, `status`, `timestamp`.

### 3.2. AI Model & Topology Inventory (`AiModelInventory`)
To prevent the SAM UI from experiencing high latency during full Kubernetes API scans, CMU uses a local database mapping layer (`AiModelInventory`, `AiModelResource`). The `ModelDiscoveryManager` executes global sweeps across all clusters to persist the unified topology natively, dynamically deriving AI computational topologies linked to concrete Deployments, Services, Ingresses, and PVC references.

---

## 4. Core Sub-Systems & Responsibilities (Current Capabilities) ⚙️

CMU currently manages the following operational domains, exposing all functionality through the modular `CmuService` gRPC definitions and the internal Click CLI.

### 4.1. Stateful Data & Secrets Provisioning
CMU inherits and centralizes the technical capabilities previously managed by the legacy `llm-auth` utility, completely automating the lifecycle for ecosystem data stores.

* **Identity (Keycloak):** Automated generation of OIDC service clients with strictly scoped configurations (e.g., standard flow disabled, service accounts enabled) and custom protocol mappers (`oidc-audience-mapper` for `app-backend` audiences). CMU actively tests auth flows natively via the `client_credentials` grant type.
* **Secrets (Vault):** Uses AppRole authentication to connect to Vault KV v2 engines. Provisions cryptographically secure 32-character secrets dynamically (`get_or_create_client_secret`) and manages central storage of application credentials and DSNs under structured paths (e.g., `secret/keycloak/aif-llm`). Includes global depth-first search traversal to locate and manage secrets across the entire Vault tree.
* **Databases (PostgreSQL / Patroni):** * Connects to global Patroni endpoints to execute autocommit `CREATE ROLE` and `CREATE DATABASE` queries.
    * Enforces strict ecosystem naming conventions (`<module_acronym>_<category>_<env>_db`).
    * Automates recursive table ownership repair (`ALTER TABLE ... OWNER TO ...`) across public/custom schemas.
    * Executes and sanitizes raw SQL queries directly against target databases via gRPC.
* **Caches (Valkey / Redis):** * Audits a 9-node global topology (SLO, NYC, PRG clusters) asynchronously.
    * Provisions ACL user rules to both cache nodes (port 6379) and Sentinels (port 26379), stripping admin rights while allowing `sanitize-payload` and `resetchannels`.
    * Reserves logical database indices globally by injecting `__GIA_DB_OWNER__` metadata tags into the keyspace to prevent index collisions across regions.
    * Actively audits and frees databases within the localized, uncontrolled Dev Redis instances.

### 4.2. Deployment & GitOps Orchestration
CMU handles the remote execution and drift detection for Kubernetes manifests and AI model weights across on-premise hardware via the GitLab and Kubernetes APIs.

* **LoRA & AI Model Remote Orchestration Pipeline (Kube):**
    Managed natively by the `LoraManager`, CMU handles the automated pipeline (`models deploy`, `lora upload`, `lora deploy`, `lora disable`) to distribute fine-tuned weights (like Gemma 4 e4b LoRAs) across globally distributed clusters. Because CMU in SLO lacks direct storage network access to NetApp instances in NYC and PRG, the pipeline supports two storage strategies depending on Trident provisioning limits:
    * **Option A (Shared PVC):** Stream weights into a dedicated `adapters` subfolder within the *existing* model cache PVC (e.g., the 100Gi `sera-ai-vllm-gemma-4-31b` volume). This avoids wasting NetApp block storage if Trident enforces large minimum volumes (e.g., 25GB+).
    * **Option B (Dedicated PVC):** Programmatically provision a brand new, dedicated K8s PVC (e.g., `insight-lora-pvc`) for maximum blast-radius isolation, assuming storage constraints permit.

    Regardless of the chosen storage option, the orchestration logic operates sequentially:
    1.  **Mount:** Deploy a lightweight, ephemeral "transfer pod" to the remote cluster that mounts the chosen PVC.
    2.  **Stream:** Securely stream the LoRA weight files over the frontend network directly into the transfer pod's PVC mount using dynamic Kubernetes native API tarball streams.
    3.  **Deploy:** Terminate the transfer pod, dynamically inject `--lora-modules` and volume arguments into the GitLab YAML, and trigger a graceful restart. CMU intercepts live Kubernetes rollout statuses to track exact replica update statuses and gracefully handle partial pod crashes during deployment.

* **Cluster Drift Detection:**
    * Parses live Kubernetes cluster states (Deployments, Services, PVCs, ConfigMaps, and CustomResourceDefinitions like `NIMService`) and recursively compares them against declared GitLab YAML manifests to detect state mismatches.
    * Engineered smart drift detection rules to gracefully handle Kubernetes default omissible fields (e.g., `initialDelaySeconds: 0`), preventing false-positive alerts by rendering them as warning hints.
    * Enables real-time bulk drift verification grouped by `--logical-model` utilizing the internal database natively.

* **Git Integration:**
    * Implements shallow clone (`--depth 1`) strategies for local Git cache synchronizations, drastically reducing local disk footprints.
    * Interacts directly with the GitLab API to fetch raw YAML files and recursive repository trees.

### 4.3. Kubernetes Volumes & Capacity Auditing
CMU natively exposes global Kubernetes storage lifecycle management decoupled through `PrometheusManager`:
* **Global PVC Search & Inspection:** Instantly locate active Kubernetes volumes with deep CSI `volumeHandle` mappings, providing explicit linkage between PVCs and underlying physical NetApp volumes globally.
* **Real-time Capacity Sweeps:** The `volumes scan` pipeline leverages bulk PromQL metrics to audit global ecosystem volume capacities, identifying starvation through dynamic `--usage-threshold` logic.
* **Dynamic Expansion:** The `volumes expand` capability resizes Kubernetes PVCs natively on the fly, with intelligent `--size auto` parameters calculating proportional 10% expansions or flat 5Gi minimums without manual calculation.

---

## 5. Module Structure 📂

CMU encompasses a comprehensive codebase dedicated to configuration logic:

* `cli/`: Commands spanning multiple domains (`app_cmds`, `cache_cmds`, `db_cmds`, `git_cmds`, `identity_cmds`, `lora_cmds`, `models_cmds`, `secret_cmds`, `volumes_cmds`).
* `config/`: The `settings.yaml` definitions.
* `core/`:
  * `managers/`: The core execution layer interfacing with external systems (`gitlab_manager`, `keycloak_manager`, `kube_manager`, `lora_manager`, `psql_manager`, `vault_manager`, `prometheus_manager`).
  * `database/`: Database abstractions (`sql_reader.py`, `sql_writer.py`) and models for the audit log and inventory cache.
* `grpc_api/`: The `cmu_service.py` to route SAM API calls into the underlying managers.
* `scripts/`: Diagnostic utilities for topology and GPU discovery testing.

---

## 6. Developer Guide & CLI Usage 🛠️

CMU provides an extensive CLI to automate logical provisioning and infrastructure deployment dynamically.

```bash
# Identity and Secrets
python3 configuration_management_utility.py identity get-secret --client-id gia-test
python3 configuration_management_utility.py secret delete --path keycloak/aif-llm --key test_token

# Model, Cache & Volume Management
python3 configuration_management_utility.py lora deploy --target insight
python3 configuration_management_utility.py volumes expand --size auto --target-pvc ...
python3 configuration_management_utility.py volumes scan --usage-threshold 90

# Drift Detection & Kube Verification
python3 configuration_management_utility.py models verify --logical-model gemma-4-31b --show-live
```

---

## 7. Roadmap & Planned Capabilities 🚀

The following capabilities define the future architectural state of CMU, extending its role as the definitive control plane for ecosystem state.

### 7.1. Structured Audit Logging (The Ledger)
To eliminate the accountability gap within Rancher and Kubernetes, CMU will implement a deterministic Audit Ledger in PostgreSQL. Because Kubernetes audit logs are noisy and fail to capture logical business intent, CMU will act as a strict tollbooth for deployments. By forcing deployments through CMU's gRPC endpoints, any cluster change not present in the CMU audit table can be immediately identified as a rogue, manual intervention (e.g., a master node root override).

This will be implemented manually within the gRPC endpoint handlers for mutating calls (e.g., `DeployModel`, `DbCreate`, `VaultWrite`, `GitPush`). Read-only operations will bypass the audit logger to prevent database bloat.

**Table Schema (`cmu_audit_log`):**
* **`log_id`** (UUID, Primary Key)
* **`actor`** (String): Extracted automatically via `x-client-id` gRPC metadata to identify the specific service account or user triggering the action.
* **`domain`** (String): The targeted infrastructure domain (e.g., `Models`, `Apps`, `Cache`, `Database`, `Vault`).
* **`target_cluster`** (String): The destination environment (e.g., `slo`, `prg`, `nyc`, `global`).
* **`action`** (String): The explicit operation performed (e.g., `DEPLOY_LORA`, `CREATE_DB`, `SYNC_GIT`).
* **`payload_reference`** (String): Contextual metadata regarding the payload (e.g., `/models/k8s/vllm/gemma-4-31b/deployment.yaml` or a specific Vault path).
* **`status`** (String): Operational result (`SUCCESS` or `FAILED`).
* **`timestamp`** (DateTime): The exact UTC execution time.

### 7.2. Traffic Routing & Load Balancing (HAProxy)
* **Data Plane API Integration:** CMU replaces legacy SSH file-parsing by natively communicating with the external HAProxy Data Plane REST API.
* **Dynamic Configuration:** Handles transactional updates to frontends, backends, ACLs, and routing rules as structured JSON. This removes the need for brittle `haproxy.cfg` file edits over SSH, although OMU retains SSH access for actual node-level failovers and Keepalived VIP management.

### 7.3. Native Module Configuration (`settings.yaml` Management)
Taking inspiration from the originally planned SAM configuration backend, CMU acts as the secure operator for all ecosystem application configurations:
* **Config Reader/Writer:** Exposes gRPC endpoints (`GetModuleConfig`, `UpdateModuleConfig`) to securely read target `settings.yaml` files. It securely parses and tags secret fields (keys containing `dsn`, `key`, `token`) for display, merges changes preserving comments and structure, and writes them back to the filesystem.
* **Config Reloader:** Exposes `ReloadModuleConfig` which triggers standard `ReloadConfig(ReloadConfigRequest)` gRPC calls on downstream modules to apply changes dynamically without requiring a full systemd/Kube pod restart.

### 7.4. Autonomous Background Volume Expansion Worker
While CMU already natively implements intelligent on-demand volume capacity calculation (`--size auto`), it will eventually run an autonomous worker to completely automate expansion.
* **The Logic:** Scans ecosystem PVCs on a cron schedule via the existing `VolumesScan` PromQL-powered pipeline. 
* **The Audit Threshold:** Integrates with the `cmu_audit_log`. If a specific PVC triggers the auto-expansion rule more than 3 times consecutively, CMU will halt auto-expansion for that volume and trigger an email alert via OMU to prevent uncontrolled storage inflation.

### 7.5. Keycloak Client Audit Tracking
To provide long-term historical tracking of Keycloak client usage (e.g., active sessions and last login timestamps over 1 year ago), CMU will introduce a PostgreSQL table to track usage.
* **The Logic:** A daily cron-sync will fetch active usage data from Keycloak's Admin API and persist it locally in CMU.
* **Data Retention:** The database will employ a 6-month auto-purge policy to maintain storage efficiency while still providing deep usage history unavailable in native Keycloak.