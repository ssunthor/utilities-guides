# Operations Management Utility (OMU) - Design Document

**Version:** 0.1.0
**Date:** 2026-05-31
**Author:** Martin Kecka & Gemini AI
**Category:** (U) Utility / Administrative
**Acronym:** OMU (Formerly `patching` & `llm-auth`/IMU)

---

## 1. Introduction & Purpose 📜

The **Operations Management Utility (OMU)** is the dedicated "Stateful Mutator" and high-privilege operations microservice within the SERA AI Ecosystem. Born from the fusion of the fleet patching orchestrator and the Identity Manager Utility (IMU / `llm-auth`), OMU enforces a strict architectural boundary: **Stateful Resource Provisioning and Infrastructure Mutation.**

In the SERA ecosystem's triad of management:
* **HMU (The Watcher):** Read-only observability, telemetry, and diagnostics.
* **AMU (The Deployer):** GitOps, Helm, and application payload deployment.
* **OMU (The Builder & Operator):** High-privilege execution. Creates databases, mints Keycloak clients, writes Vault secrets, drains Kubernetes nodes, and orchestrates zero-downtime OS patching across stateful clusters (Patroni, Kafka, etcd, Valkey).

Eventually, OMU will be integrated into the headless gRPC aggregator model, allowing the Sera Admin Module (SAM) to trigger complex infrastructure operations autonomously via UI or CLI. Under this paradigm, SAM acts as the orchestration "brain" (holding the database, policies, and schedules), while OMU acts as the stateless "muscle" executing the mutations.

---

## 2. Core Architectural Principles 🏗️

1. **The Privilege Boundary:** OMU is the *only* utility permitted to hold root-level SSH access (`syseng_ansible`), database administrative credentials, and Vault master tokens. Keeping this isolated from the read-heavy HMU drastically reduces the ecosystem's security blast radius.
2. **Autonomous Introspection ("Check Before Mutate"):** OMU retains localized, synchronous introspection logic (e.g., querying Patroni REST APIs, Kafka KRaft quorums, or Redis Keyspaces). It does **not** rely on HMU's gRPC endpoints for pre-flight checks. This prevents cyclic dependencies during cold-starts, observability outages, or rolling network partitions.
3. **Idempotent Provisioning:** All creation scripts (Keycloak, Vault, PostgreSQL, Valkey) must gracefully handle pre-existing resources, either skipping safely or failing cleanly without overwriting production state.
4. **Zero-Downtime Orchestration:** Destructive actions (like OS patching) must strictly follow distributed consensus rules. Leaders are failed over, Kubernetes workloads are safely drained (`cordon`/`drain`), and nodes are validated post-reboot before the orchestrator touches the next node in the topology.
5. **Zero-Tolerance Failure Handling:** If a single node fails during a rolling patching execution (e.g., fails to return via SSH, or critical services fail to start), OMU instantly halts the entire run and raises a critical alarm to SAM to prevent cascading cluster degradation.

---

## 3. SAM Orchestration & Shared Database Schema 🗄️

To support autonomous, scheduled fleet patching based on repository updates, OMU relies on a shared database managed by the Sera Admin Module (SAM). This removes reliance on local JSON inventory files and enables historical auditing.

### 3.1. Unified Target Inventory (`ManagedNodes`)
Defines the fleet and dictates exactly *how* OMU should handle each node.

| Column | Type | Description |
| :--- | :--- | :--- |
| `hostname` | `VARCHAR` | Primary Key (e.g., `slo-aif-dev01`). |
| `netbox_id` | `INTEGER` | Used by OMU to push post-patch kernels back to NetBox. |
| `is_vm` | `BOOLEAN` | Determines virtualization vs. DCIM API endpoints. |
| `patching_group` | `VARCHAR` | Logical grouping (e.g., `aif_psql`, `aif_dev`). |
| `strategy` | `VARCHAR` | The specific OMU orchestrator to invoke: `psql`, `kafka`, `valkey`, `dcs`, `ha`, `qdrant`, or `generic`. |
| `is_excluded` | `BOOLEAN` | UI toggle for SAM to temporarily exclude a problematic node. |

### 3.2. Execution Auditing (`PatchingRuns` & `NodePatchLogs`)
Provides a highly structured, relational audit trail of infrastructure mutations.

**`PatchingRuns` (The Overview)**
| Column | Type | Description |
| :--- | :--- | :--- |
| `run_id` | `UUID` | Primary Key. |
| `patching_group` | `VARCHAR` | The target group patched. |
| `trigger_source`| `VARCHAR` | e.g., `auto_kernel_sync`, `manual_override`. |
| `started_at` | `TIMESTAMP` | Execution start time. |
| `completed_at` | `TIMESTAMP` | Execution finish time. |
| `status` | `VARCHAR` | `SUCCESS`, `HALTED_ON_ERROR`, `FAILED`. |

**`NodePatchLogs` (The Granular Data)**
| Column | Type | Description |
| :--- | :--- | :--- |
| `log_id` | `UUID` | Primary Key. |
| `run_id` | `UUID` | Foreign Key to `PatchingRuns`. |
| `hostname` | `VARCHAR` | The specific node patched. |
| `pre_kernel` | `VARCHAR` | Kernel version before `apt-get upgrade`. |
| `post_kernel` | `VARCHAR` | Kernel version post-reboot. |
| `packages_upgraded`| `INTEGER` | Count parsed from APT output. |
| `skipped_reboot`| `BOOLEAN` | Flagged if uptime < 24h and no reboot required. |
| `raw_execution_log`| `TEXT` | Full stdout/stderr dumped from OMU execution. |

---

## 4. Core Sub-Systems & Responsibilities ⚙️

OMU's mandate covers both Day-1 Provisioning and Day-2 Operations for the infrastructure layer.

### 4.1. Fleet Orchestration & Patching (Day 2)
* **Dynamic Topologies:** Discovers cluster topologies (Leader/Follower, Master/Slave) in real-time to determine safe rolling restart orders.
* **Stateful Failovers:** Programmatically triggers Patroni switchovers, etcd leader migrations, and Valkey sentinel failovers prior to rebooting nodes.
* **Kubernetes Eviction:** Safely cordons and drains Kubernetes master and worker nodes (`aif_app_k8master`, `aif_gpu_k8worker`) via the K8s API to reschedule workloads before OS patching.
* **Validation Hooks:** Executes pre-flight (disk space, apt locks) and post-flight (watchdog verification, Raft consensus recovery) checks.

### 4.2. Authentication & SSO (Day 1)
* Automated generation of OpenID Connect (OIDC) clients for new microservices via Keycloak.
* Role and group generation for user and service accounts.
* Client-secret extraction and secure storage payload formulation.

### 4.3. Secret Management (Day 1 & Day 2)
* Centralized writing, updating, and auditing of application credentials in HashiCorp Vault.
* Secure storage for LLM inference keys and auto-generated Database DSNs.

### 4.4. Database & Storage Provisioning (Day 1)
* **PostgreSQL:** Programmatic creation of dedicated physical databases and scoped roles within the global Patroni cluster.
* **Valkey/Redis:** Reservation of memory blocks (DB indexes) globally. Generation of ACL users with precise `+@read +@write` rules, limiting applications to their reserved databases.

---

## 5. Developer Guide: CLI Usage 🛠️

OMU centralizes all infrastructure mutation commands under a single `click` framework, categorizing them by operational domain.

### 5.1. Identity & Provisioning

[bash]
# Generate a Keycloak Client and initialize Vault space
python3 operation_management_utility.py identity create-client --name snc-connector --type confidential
python3 operation_management_utility.py vault init-path --app snc-connector

# Provision a Patroni Database & generate Vault DSN
python3 operation_management_utility.py storage provision-db --app snc-connector --db snc_db --engine postgres

# Reserve Valkey global cache space
python3 operation_management_utility.py storage provision-cache --app snc-connector --nodes 9
[/bash]

### 5.2. Fleet Operations & Patching

[bash]
# Audit the global HAProxy ingress fleet (Introspection)
python3 operation_management_utility.py fleet status --component haproxy

# Execute a rolling OS patch on the Kubernetes GPU worker nodes in Slough
python3 operation_management_utility.py fleet patch --group aif_gpu_k8worker --location slo

# Execute a zero-downtime rolling patch on the global Kafka KRaft cluster
python3 operation_management_utility.py fleet patch --component kafka
[/bash]

---

## 6. Roadmap & Future Enhancements 🚀

* **gRPC Headless Service:** Migrate the local Python/Click tools into a headless `OMU` gRPC service. This will expose production-ready RPCs like `rpc ProvisionService(ProvisionRequest)` and `rpc ExecuteRollingPatch(PatchRequest)` so that the SAM-WebUI can trigger operations autonomously.
* **SAM-Driven Policy Execution (The Brain vs. Muscle Split):** Formally align with the Sera Admin Module (SAM) orchestrator. Under this paradigm, OMU remains a stateless, high-privilege execution utility ("dumb executor"). SAM acts as the central orchestration "brain"—holding the evaluation policies, scheduling cron jobs, calculating maintenance windows, and handling Slack/PagerDuty alerts.
* **Automated Kernel & Repository Delta Engine:** Expand OMU's package management capabilities to act as an introspection worker for SAM. OMU will expose fast, parallelized inspection endpoints to query localized OS kernel versions from the fleet, alongside a unified repository metadata hook. In a single response, OMU will provide SAM with the combined state (Running Fleet Kernels vs. Latest Approved Mirror Packages). SAM will then evaluate patch drift and automatically trigger target patch execution if:
  * The running kernel version does not match the latest approved repository stable kernel provided by OMU.
  * The elapsed time since the node's last successful patch exceeds 30 days.
* **Credential Rotation Engine:** Implement a secure, automated rotation mode that generates new database/Keycloak passwords, safely writes them to HashiCorp Vault, and coordinates with AMU via GitOps tags to trigger rolling restarts of dependent Kubernetes pods.