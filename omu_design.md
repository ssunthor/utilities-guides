# Operations Management Utility (OMU) - Design Document

**Version:** 0.3.6
**Date:** 2026-07-12
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

OMU is integrated into the headless gRPC aggregator model, where the Sera Admin Module (SAM) acts as a centralized UI gateway. Under this paradigm, OMU autonomously handles its own scheduling, policies, and database tables for execution, while SAM queries OMU to display results and coordinate alerts.

---

## 2. Core Architectural Principles 🏗️

1. **The Privilege Boundary:** OMU is the *only* utility permitted to hold root-level SSH access (`syseng_ansible`), database administrative credentials, and Vault master tokens. Keeping this isolated from the read-heavy HMU drastically reduces the ecosystem's security blast radius.
2. **Autonomous Introspection ("Check Before Mutate"):** OMU retains localized, synchronous introspection logic (e.g., querying Patroni REST APIs, Kafka KRaft quorums, or Redis Keyspaces). It natively tracks a `health_mode` (API vs SSH) to intelligently optimize polling without relying on HMU's gRPC endpoints, preventing cyclic dependencies during cold-starts or network partitions.
3. **Idempotent Provisioning:** All creation scripts (Keycloak, Vault, PostgreSQL, Valkey) must gracefully handle pre-existing resources, either skipping safely or failing cleanly without overwriting production state.
4. **Zero-Downtime Orchestration:** Destructive actions (like OS patching) must strictly follow distributed consensus rules. Leaders are failed over, Kubernetes workloads are safely drained (`cordon`/`drain`), and nodes are validated post-reboot before the orchestrator touches the next node in the topology.
5. **Zero-Tolerance Failure Handling:** If a single node fails during a rolling patching execution, OMU instantly halts the entire run and raises a critical alarm. It implements cross-cluster safety boundaries (e.g., querying Keycloak health via gRPC before patching the next geographical site) to spare the fleet from cascaded outages.
6. **Ecosystem Compliance:** OMU fully integrates the `sera_standard_service.py` architecture for unified multi-service tracking, generating proper DeepHealth `dependency_group` tags natively.

---

## 3. Database Schema & State Management 🗄️

While the primary PostgreSQL database cluster is hosted by the Sera Admin Module (SAM), **each utility strictly owns and manages its own tables within that database**. OMU natively supports HA Routing (`write_dsn`/`read_dsn`) and is fully autonomous in managing its fleet state, patching history, and scheduling.

### 3.1. Unified Target Inventory (`ManagedNodes`)
Defines the fleet and dictates exactly *how* OMU should handle each node. Replaces legacy local JSON files with a persistent PostgreSQL backend. Note that `KUBE` device types are automatically excluded from global patch completion metrics.
* **Schema Highlights:** Tracks physical mapping (`site`), grouping (`group`), classification (`cluster_type`, `device_type`), execution logic (`strategy`), and state timelines (`last_patched`, `last_inspected`). It natively tracks the target `health_mode` (API vs SSH) to optimize polling.

### 3.2. Execution Auditing (`PatchingRuns` & `NodePatchLogs`)
Provides a highly structured, relational audit trail of infrastructure mutations.
* **PatchingRuns:** The high-level execution wrapper tracking start/end times, trigger sources, and overall terminal status (Success/Halted/Failed).
* **NodePatchLogs:** The granular node-level data linked via cascaded foreign keys. It records pre/post kernel versions, upgraded package counts, and dumps the full raw terminal `stdout/stderr` directly into the database for transparent execution auditing.

---

## 4. Core Sub-Systems & Responsibilities ⚙️

OMU's mandate covers both Day-1 Provisioning and Day-2 Operations for the infrastructure layer.

### 4.1. Fleet Orchestration & Patching (Day 2)
* **Unified gRPC Backend:** Centralized `ExecuteRollingPatch` flow executing patching sequentially for Patroni, HAProxy, DCS, Vault, Kafka, Valkey, Qdrant, Generic, Rancher, and K8s components.
* **Persistent Session Caching:** `SSHManager` natively pools paramiko transport sessions per host. It intelligently reuses active connections across sequential pre-flight, OS patching, and post-flight validation commands, eliminating extreme TLS handshake overhead.
* **Dynamic Topologies:** Discovers cluster topologies (Leader/Follower, Master/Slave) in real-time. For Patroni, it recursively queries standby leaders to aggregate streaming replication `SYNC STATE` and `LAG BYTES` globally.
* **Stateful Failovers:** Programmatically triggers Patroni switchovers, etcd leader migrations, Vault leader handoffs (via API `_step_down_leader`), and Valkey sentinel failovers prior to rebooting nodes.
* **Global Rancher Orchestration:** Natively manages zero-downtime rolling OS upgrades for standalone Rancher nodes. Includes configuration drift safeguards (extracting TLS cert `notAfter` via OpenSSL) and embedded K3s ETCD safety hooks to flush state to disk before reboot.
* **Kubernetes Eviction:** Safely cordons and drains Kubernetes master and worker nodes. It enforces ETCD quorum health checks and `Running` validation for critical `kube-system` pods before allowing node draining.
* **Automated Resilience Hooks:** Pre-flight checks natively automate LVM root partition resizing (expanding by 4GB if >85% capacity), lazy unmounting of NFS backups, and automated pre-patch GPG key renewals for third-party repositories.
* **Unstructured Node Management:** Full lifecycle management for generic nodes via the `GenericPatchingOrchestrator`, which automatically identifies Docker containers, systemd services, and supervisord tasks, tearing them down gracefully prior to reboot.
* **Validation Hooks:** Executes pre-flight (disk space, apt locks) and post-flight (watchdog verification, Raft consensus recovery, cluster health) checks.

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

## 5. Module Structure 📂

To ensure maintainability and separation of concerns, OMU's `core/` directory is split into distinct, logical submodules:

* `core/introspectors/`: Houses all introspection logic (e.g., `psql_introspector.py`, `vault_introspector.py`). Responsible for reading cluster state, verifying node health, and formulating `IntrospectResponse` payloads.
* `core/patching/`: Contains the orchestration engines (`KafkaPatchingOrchestrator`, `GenericPatchingOrchestrator`, `RancherPatchingOrchestrator`, etc.) that execute the stateful upgrades.
* `core/managers/`: Dedicated domain-specific client wrappers and state managers (e.g., `RancherManager`, `K8sManager`, `PostgreSQLManager`).
* `core/clients/`: Standardized REST and API clients for external ecosystem systems.
* `core/inventory_sync.py`: Cross-cutting utility for pulling external truth (NetBox, Kubernetes) into the persistent local database.
* `config/`: Securely groups API endpoints under `system_services` and descriptive integrations (Vault, NetBox, Rancher) under `external_services` for standard DeepHealth tracking.

---

## 6. Developer Guide & CLI Usage 🛠️

OMU centralizes infrastructure commands under a unified `click` framework (`cli/omu_cli.py`), categorizing them by component. Every command supports the `--grpc` flag to optionally emit structured, clean JSON payloads.

```bash
# Run sequential health and state audits across all managed components
python3 operation_management_utility.py inventory summary
python3 operation_management_utility.py status all

# Dynamically modify the health check mode (API vs SSH) for a specific node
python3 operation_management_utility.py inventory modify slo-aif-dev01 --health-mode API

# Audit the global HAProxy ingress fleet
python3 operation_management_utility.py ha status

# Inspect pre-flight checks for Valkey nodes (filtered by location)
python3 operation_management_utility.py valkey inspect --location SLO

# Execute a zero-downtime rolling patch on the global Kafka KRaft cluster
python3 operation_management_utility.py kafka patch

# View Patroni PostgreSQL streaming replication states and leader topology
python3 operation_management_utility.py patroni status

# Manage Kubernetes clusters natively
python3 operation_management_utility.py k8s status
python3 operation_management_utility.py k8s inspect
python3 operation_management_utility.py k8s patch

# Manage global Rancher orchestration
python3 operation_management_utility.py rancher status
python3 operation_management_utility.py rancher inspect
python3 operation_management_utility.py rancher patch

# Manage unstructured / generic nodes (all groups sequentially)
python3 operation_management_utility.py generic patch --all --force-reboot

# Execute an ad-hoc SSH command across a specific patching group safely
python3 operation_management_utility.py ssh execute --group aif_dev --cmd "uptime"

# Clear stuck or aborted patching runs from the database (cascades logs)
python3 operation_management_utility.py audit clear-stuck
```

---

## 7. Roadmap & Future Enhancements 🚀

* **Autonomous Scheduling & Orchestration:** Expand OMU's package management capabilities to act as a fully autonomous patching scheduler. While the `SchedulerManager` daemon currently sweeps the fleet daily for inspections, OMU will eventually continuously query localized OS kernel versions from the fleet, alongside unified repository metadata. It will evaluate patch drift and automatically schedule patching runs if:
  * The running kernel version does not match the latest approved repository stable kernel.
  * The elapsed time since the node's last successful patch exceeds 30 days.
  OMU will store these schedules in a new `PatchingSchedules` table and execute them autonomously over weekends. SAM will query OMU via gRPC to report these schedules and trigger email notifications via EMA.
* **Credential Rotation Engine:** Implement a secure, automated rotation mode that generates new database/Keycloak passwords, safely writes them to HashiCorp Vault, and coordinates with AMU via GitOps tags to trigger rolling restarts of dependent Kubernetes pods.