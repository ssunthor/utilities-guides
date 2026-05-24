# Identity Manager Utility (IMU) - Design Document

**Version:** 0.2.0
**Date:** 2026-05-24
**Author:** Martin Kecka & Gemini AI
**Category:** (U) Utility / Administrative
**Acronym:** IMU (Formerly `llm-auth`)

---

## 1. Introduction & Purpose 📜

The **Identity Manager Utility (IMU)** is the dedicated provisioning, authentication, and security management microservice within the SERA AI Ecosystem. Born from the legacy `llm-auth` CLI tool, IMU enforces a strict boundary of **Stateful Resource Provisioning and Mutation**.

While the Health Monitor Utility (HMU) acts as the read-only observability platform ("Single Pane of Glass"), IMU is strictly designed to interact with Vault, Keycloak, PostgreSQL, and Valkey to mint credentials, generate Access Control Lists (ACLs), and reserve hardware databases. Eventually, IMU will be integrated into the headless gRPC aggregator model, allowing the Sera Admin Module (SAM) to trigger provisioning tasks autonomously via UI or CLI.

---

## 2. Core Architectural Principles 🏗️

1. **Stateful Provisioning Boundary:** IMU is the *only* utility permitted to create, mutate, or delete credentials and database reserves. 
2. **No Read-Only Telemetry:** IMU will not maintain extensive dashboards or charts regarding connection counts, active memory usage, or active queries. This is strictly offloaded to HMU.
3. **"Check Before Create" Introspection:** IMU will retain minimal, localized introspection logic (e.g., "Is DB 4 free?") required exclusively to ensure safe provisioning. It does **not** rely on HMU's gRPC endpoints for this introspection to prevent cyclic dependencies during a cold-start or observability outage.
4. **Idempotent Operations:** All creation and provisioning scripts must gracefully handle pre-existing resources, either skipping safely or failing cleanly without accidentally overwriting production credentials.
5. **Secure Auditing:** Every provisioning action will be tied to a `trace_id` to ensure absolute auditability of credential generation and distribution.

---

## 3. Core Sub-Systems & Responsibilities ⚙️

IMU's mandate covers the creation, rotation, and deletion of the following stateful infrastructure elements:

### 3.1. Authentication & SSO (Keycloak)
* Automated generation of OpenID Connect (OIDC) clients for new microservices.
* Role and group generation for user and service accounts.
* Client-secret extraction and secure storage payload formulation.

### 3.2. Secret Management (HashiCorp Vault)
* Centralized writing, updating, and auditing of application credentials.
* Secure storage for LLM inference keys (e.g., OpenAI, Anthropic, HuggingFace).
* Secure storage for Database DSNs generated during the PostgreSQL and Valkey provisioning phases.

### 3.3. Database Provisioning (PostgreSQL)
* Programmatic creation of dedicated physical databases within the global Patroni cluster.
* Generation of tightly-scoped, application-specific roles and complex passwords.
* Automatic injection of the finalized DSN string into HashiCorp Vault.

### 3.4. Global Cache Provisioning (Valkey & Redis)
* Reservation of memory blocks (DB indexes) globally across the 9-node Patroni/Valkey topology.
* Generation of ACL users with precise `+@read +@write` cache rules, limiting applications to their reserved databases.
* Syncing of Redis/Valkey DSNs into the application's Vault payload.

---

## 4. Developer Guide: CLI Usage 🛠️

IMU acts as the administrative tool to generate prerequisites for any new SERA module.

### 4.1. Bootstrapping a New Service
When a new service (e.g., `SNC Connector`) is created, IMU is used to generate its credentials:

[bash]
# Example: Generate a Keycloak Client and Vault space for SNC
python3 imu_cli.py keycloak create-client --name snc-connector --type confidential
python3 imu_cli.py vault init-path --app snc-connector
[/bash]

### 4.2. Reserving Database Space
Before deploying the service, reserve PostgreSQL and Cache boundaries:

[bash]
# Example: Provision a Patroni Database
python3 imu_cli.py psql provision --app snc-connector --db snc_db

# Example: Reserve Valkey global cache
python3 imu_cli.py cache provision --app snc-connector --nodes 9
[/bash]

---

## 5. Roadmap & Future Enhancements 🚀

* **gRPC Headless Service:** Migrate the local Python/Click tools into a headless `IMU` gRPC service that exposes `rpc ProvisionService(ProvisionRequest)` so that SAM can offer a "1-Click Service Bootstrap" UI.
* **Transition to Click Framework:** Complete the internal refactoring of all legacy arg-parse/sys.argv scripts (`llm-auth`) into the centralized `click` command-group framework for standardized CLI operations.
* **Vector Database Provisioning:** Expand provisioning workflows to support automated collection creation and ACL generation for Qdrant (Production) and Chroma (Local).
* **Automated Credential Rotation:** Implement a CRON-compatible or gRPC-triggered rotation mode that safely generates new passwords, patches Vault, and orchestrates rolling restarts of dependent Kubernetes deployments via the AMU GitOps tags.
