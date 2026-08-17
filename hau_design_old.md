# HALO Admin Utility (HAU) - Design Document

**Version:** 0.0.8
**Date:** 2026-07-30
**Author:** Martin Kecka & Gemini AI
**Category:** (U) Utility / Administration & Operations Portal
**Acronym:** HAU

---

## 1. Introduction & Purpose 📜

The **HALO Admin Utility (HAU)** serves as the "Workstream 2: Administration & Operations Portal." It provides GTT with the primary operational control plane to provision, govern, monitor, and support all customer tenants within the HALO ecosystem.

**Business Outcome:** GTT obtains a single administration portal to onboard customers, provision tenants, deploy collectors, manage releases, monitor platform health, control feature availability, support customers, and manage the entire customer lifecycle through to billing.

Unlike the internal SERA utilities (AMU, OMU, CMU), HAU manages external, customer-facing deployments, collectors, and AI inference pipelines (like Morpheus and Triton). Because HALO operates outside the standard SERA ecosystem, it implements its own independent monitoring, UI, and deployment abstractions.

---

## 2. Core Architectural Principles 🏗️

1. **Reused Utility Patterns:** HAU heavily borrows architectural concepts from the SERA ecosystem:
    * **AMU Pattern:** GitOps-style configuration pushes to deployed environments and release management.
    * **CMU Pattern:** Logical configuration management, managing tenant feature flags, and product configurations.
    * **HMU Pattern:** Heavy infrastructure telemetry aggregation (tracking Kubernetes, AI pipelines, and LLM model performance).
2. **Decoupled Execution & UI:** HAU serves as the headless execution engine (structured exactly like standard SERA utilities with a Click CLI and gRPC server). The web interface is completely offloaded to a dedicated **HALO Admin Module (HAM)** (FastAPI + React frontend), mirroring the SAM paradigm.
3. **CMD as Source of Truth:** Core customer, tenant, and billing lifecycle data is authoritative in CMD. HAU integrates directly with CMD to drive downstream provisioning.
4. **Independent Deep Health Diagnostics:** HAU uses a bespoke asynchronous `ClusterHealthChecker` to evaluate backend dependencies (Redis and Netbox inside the `halo` namespace, vLLM) and fetches Kafka status remotely via OMU.

---

## 3. Database Schema & State Management 🗄️

HAU will manage state for provisioning and inventory that complements CMD.
* **Tenant State:** Tracking the lifecycle from Onboarding -> AI Burn-in -> Go-Live Approval -> Billing.
* **Collector Inventory:** Maintaining a live inventory mapping collectors to customer regions and HA pair configurations.
* **Feature Management:** Storing specific Product Configurations and Tenant Feature toggles per TAD (Tenant Architecture Document).
* **Git Inventory Cache:** Relational mapping of GitLab projects, branches, and container registry tags mapped natively against active Kubernetes workloads.
* **Release & Audit Logging:** Immutable `GitReleaseAuditLog` tracking end-to-end pipeline execution, LLM changelog generation methods, container footprint scaling, and job telemetry.

---

## 4. Core Sub-Systems & Responsibilities ⚙️

### 4.1. Tenant Lifecycle & Billing (CMD Integration)
* **Tenant Management:** View all customer tenants with drill-down details sourced from CMD.
* **Auto Tenant Provisioning:** Automatically provision new tenants from approved TADs.
* **Go-Live Governance:** Manage the onboarding completion checklist and go-live approvals.
* **Billing Readiness:** Strict state boundary to prevent billing from starting until onboarding and handover checks are fully complete.

### 4.2. Infrastructure Provisioning & Inventory
* **Collector Provisioning & Inventory:** Automatically provision collectors in the correct customer region and maintain a live inventory of all deployed locations.
* **HA & Connectivity Management:** Configure and display HA collector pairs. Deploy virtual firewalls where collectors communicate over the internet, and configure/manage IPsec connectivity where required.
* **Deployment Management:** Record all customer deployments, regions, and collector locations.

### 4.3. Configuration & Feature Management (CMU/AMU Patterns)
HAU acts as the central orchestration "brain" that governs deployment state and configuration. HAU generates standard Kubernetes manifests and commits them to a dedicated control directory within the `halo-repository`. This incorporates a Geolocation Routing Layer to manage isolated customer tenants across global sites:

The complete folder structure and operational patterns for the repository are detailed within the Halo repository's concept documentation (`deployment/docs/ecosystem/repo_concept.md`).

**Deployment Flow (GitLab CI pipeline trigger):**
For the initial phase, to bypass strict ArgoCD linting rules on legacy monolithic manifests, deployments are executed via GitLab CI:
1. **Manifest Generation:** HAU dynamically generates Kustomize overlays, feature flags, and Kubernetes secrets based on the TAD (Tenant Architecture Document) and customer state in CMD.
2. **State Commit:** HAU commits these Kubernetes manifests to the specific tenant folder within `/halo-repository/deploy/clusters/`.
3. **Manifest Introspection (Routing):** Before triggering deployment, HAU parses the manifests to read the `halo/target-cluster` annotation (e.g., `slo-hal-clu01` or `slo-aif-nvclu01`). Because the exact cluster name intrinsically includes the location (`slo`, `prg`, `nyc`), this single self-describing annotation naturally solves both workload type targeting and geolocation routing.
4. **Pipeline Trigger:** When an operator clicks "Deploy" in the HAU dashboard, HAU triggers a specific GitLab CI deployment pipeline via REST API, passing the target tenant ID and the explicitly determined target cluster name as a variable.
5. **Runner Execution:** The GitLab Runner, serving as the execution "muscle", picks up the job, authenticates directly to the target Kubernetes cluster provided in the variable, and executes `kubectl apply -k` (or equivalent) to deploy the manifests.
6. **Status Polling:** HAU polls the GitLab API to display real-time pipeline execution status back to the operator in the UI.

**ArgoCD and strict automated GitOps remain the future target state once the application stack matures and manifests pass strict validation.**

* **Configuration Management:** Push configuration changes to deployed environments.
* **Tenant Feature Management:** Enable/disable features per tenant based on TAD. Support different product configurations per customer.
* **Release & Upgrade Management:** Controlled rollout of new functionality to selected customers. Schedule and execute tenant upgrades, tracking HALO release versions fleet-wide.

### 4.4. AI & Monitoring Operations (HMU Pattern)
* **AI Training Management:** Manage the AI burn-in and learning period before customer handover.
* **Tenant Operations Dashboard:** Show onboarding status, collector health, tenant health, and deployment status.
* **Monitoring & Telemetry:** Monitor pipelines (e.g., Netflow, Syslog DFP), integrations, and overall platform health (Kafka via OMU, vLLM, local Netbox/Redis).
* **Monitoring Integration:** Forward actionable monitoring events (like UI crashes or backend degraded states) into external operational platforms (e.g., ME3 monitoring platform) via standard stateless webhooks/snmp/api, ensuring HAU remains decoupled from third-party API changes.

### 4.5. User Administration & Access Control (RBAC)
* **Audit Logging:** All administrative actions (triggering deployments, toggling features, editing billing states) are logged against the authenticated user ID for security and compliance.

### 4.6. AI-Driven Release Orchestration & Automated Changelogs
HAU natively integrates `vLLMClient` and deep git introspection to automate the entire GitLab release lifecycle. The previously planned "Changelog Fragment" pattern has been completely superseded by a vastly superior **Dynamic Map-Reduce Generation** strategy, allowing changelogs to be generated dynamically from the git commit tree itself.
* **Dynamic Changelog Generation:** HAU reads the exact code changes directly from Git (`git diff HEAD^1 HEAD`). For massive codebase changes (e.g., initial commits > 20k tokens), HAU automatically falls back to a **Map-Reduce** pipeline. It chunks the file diffs into 500-line blocks, uses the LLM to extract features (Map), and merges them into a clean semantic YAML document (Reduce). This provides an ultimate "safety net"—even if developers write zero changelogs or vague commit messages, the exact semantic intent of the code is flawlessly documented.
* **End-to-End Orchestration:** HAU automates the transition from feature branches to target branches. It dynamically creates GitLab Merge Requests, actively streams CI/CD pipeline job traces directly to the CLI, and handles automated retries.
* **Artifact & Release Management:** Upon successful pipeline completion, HAU utilizes the Docker V2 API to instantly retag built images to semantic versions without a rebuild. It establishes an immutable Git Tag and pushes the AI-generated semantic YAML into a formatted, native GitLab Release payload.

---

## 5. Module Structure 📂

To maintain ecosystem consistency, the administration plane is decoupled into two distinct components, mirroring the relationship between the standard SERA utilities and the SERA Admin Module (SAM):

### 1. HALO Admin Utility (HAU)
The headless, backend Python microservice living in `/ai/utilities/halo_admin_utility`. It acts as the execution engine for GitOps commits, state introspection, and background monitoring.
* `core/introspectors/`: Houses the `ClusterHealthChecker` logic. Kafka status is obtained via gRPC call to OMU. Netbox and Redis are checked directly within the `halo` namespace using internal logic.
* `core/io_handlers/`: Centralized database access enforcing a strict Repository pattern via `SQLReader` and `SQLWriter` to decouple state from business logic.
* `core/orchestrators/`: Domain-specific business logic executors (`ChangelogOrchestrator`, `ReleaseOrchestrator`, `InventoryOrchestrator`).
* `core/managers/`: Decoupled client wrappers for external integrations (`GitlabManager`, `GitPipelineManager`, `ContainerRegistryManager`, `VaultClient`, `PrometheusManager`).
* `grpc_api/`: The gRPC interfaces. The servicer (`hau_service.py`) acts strictly as a thin traffic cop, receiving payloads and routing them to the isolated domain orchestrators rather than executing direct business logic.
* `cli/hau_cli.py`: Standardized Click-based command-line interface.

### 2. HALO Admin Module (HAM)
The dedicated user interface living in a separate `/ai/halo_admin_module` directory, structured identically to the highly mature SERA Admin Module (SAM).
* `rest_api/`: The FastAPI layer exposing REST endpoints to the UI and acting as the aggregator.
* `core/connectors/`: Dedicated gRPC client wrappers. Beyond `hau_connector.py` for HALO-specific provisioning, HAM reuses standard SERA ecosystem utilities by integrating `hmu_connector.py` (vLLM/Triton AI engine monitoring), `cmu_connector.py` (global configuration), and `omu_connector.py` (Kafka/Patroni stateful status) to prevent rebuilding existing logic.
* `ui/`: The Vite + React dashboard containing modular features where GTT operators interact with the Tenant Operations Dashboard, Go-Live checklists, and collector inventory maps.

---

## 6. Independent Diagnostics Engine (Monitoring) 🔍

To monitor the external HALO dependencies, HAU utilizes a bespoke asynchronous HTTP/RPC probing architecture (`ClusterHealthChecker`):
* **Kafka Telemetry:** Fetched remotely via OMU gRPC connector.
* **Redis Analytics:** Asynchronously executes `INFO memory` and `INFO clients` to parse peak memory usage and client saturation.
* **vLLM & Local Data Services (Netbox/Redis):** Internal probes within the `halo` namespace to verify data-plane responsiveness.
* **UI Crash Reporting:** A dedicated endpoint (`/api/diagnostics/ui-crash`) allowing the React frontend to report unhandled exceptions directly to the backend telemetry state.

---

## 7. Future Enhancements & Roadmap 🚀

### 7.1. Component-Level GitOps Release Orchestration
To enable seamless and granular upgrades across multiple tenants, HAU will act as a true GitOps Release Orchestrator leveraging the GitLab Container Registry. Instead of a monolithic rollout, each application component (`backend`, `dashboard`, `syslog-dfp`, etc.) will have its independent container versioning lifecycle (`dev`, `main`, tags).

**Implementation Strategy:**
1. **Container Release Tracking:** HAU will monitor the GitLab Container Registry (via API or webhooks) to track the availability of new container tags for each individual component. This state will be stored in the HAU database to construct a "Release Matrix."
2. **Dynamic Kustomize Overrides:** When an operator decides to bump a tenant's component version (e.g., upgrading `tenant-acme`'s dashboard to `v1.2.0`), HAU will modify the tenant's specific `kustomization.yaml` file (rewriting the `images:` block to point to the new registry tag).
3. **Phased / Canary Rollouts:** By updating only a specific tenant's GitOps state, HAU can safely deploy a new container version to a `halo-dev` tenant, monitor its stability, and then gradually roll out the patch across the rest of the fleet without risking global disruption.

### 7.2. Intelligent LLM Codebase Validation (Code Review)
Since HAU natively possesses `vLLMClient` integrations and advanced codebase chunking/diffing mechanics, we can expand its role beyond just changelog generation into **Intelligent Code Validation**.

While standard GitLab CI/CD pipelines are excellent for static linting and unit testing, they lack deep contextual awareness. HAU could eventually act as an automated "Senior Developer" during the merge lifecycle:
* **Semantic Code Review:** When a developer proposes a merge, HAU's LLM pipeline can review the `git diff` for logical flaws, security vulnerabilities, or anti-patterns that static analysis tools miss.
* **Contextual Quality Gates:** HAU can evaluate whether the code changes actually fulfill the intent described in the commit messages or linked tickets.
* **Automated Feedback:** Providing direct, actionable feedback to the developer before the merge is approved, ensuring only high-quality code enters the `dev` and `main` branches.

### 7.3. Human-in-the-Loop (HITL) Release Validation & Merge Control
To ensure absolute accuracy in customer-facing release notes (preventing LLM hallucinations) and to integrate release generation seamlessly into the developer lifecycle without friction, HAU will split the release workflow into a two-phase execution model when the UI (HAM) is fully implemented:

1. **Phase 1: Merge Interception (Draft & Preview):** When a developer initiates a merge from `feature` to `dev`, HAU intercepts the merge. It syncs the repository, generates the semantic YAML changelog via the LLM against the feature branch's diff, and returns the proposed payload to the frontend *without* making any git commits.
2. **Phase 2: Validation (Execute & Orchestrate):** The generated semantic YAML is presented to the developer (or operator) in the UI for validation or manual tweaking. Upon clicking "Approve & Release", the UI sends the finalized YAML back to HAU, which automatically prepends the validated YAML to the repository's master `docs/CHANGELOG.yaml`, and executes the `git commit`, `git push`, and GitLab MR creation in one atomic transaction.

This pattern shifts HAU from a "zero-touch automation" tool to an "automation-assisted" control plane, ensuring perfect changelog hygiene with absolutely zero merge conflicts and zero developer overhead.

### 7.4. LLM-Driven Incremental Documentation Auto-Updates
To keep ecosystem design documentation (`_design.md`) perfectly synchronized with codebase evolution, HAU will automatically propose documentation updates during the Git Release lifecycle.

Rather than forcing the LLM to rewrite entire `.md` files—which is token-heavy and risks formatting regressions—HAU will leverage a **Semantic Section Replacement** architecture. 

Because all HALO documentation strictly adheres to the 13-point mandatory structure defined in `docs_concept.md`, the LLM interaction becomes highly precise and deterministic:
1. **Decision Phase:** The LLM reads the freshly generated `CHANGELOG.yaml` and the existing `_design.md` document, determining *if* an update is warranted, and explicitly identifying *which* of the 13 strict sections (e.g., `5. Detailed Business Logic & Rules`) require modifications.
2. **Focused Rewrite:** If an update is required, the LLM reads the raw Git commit diffs for deep technical context, and generates a completely rewritten payload for **only** that specific section.
3. **Deterministic Splice:** A Python parser within `release_orchestrator.py` dynamically splices the LLM's rewritten section back into the Markdown file using regex header matching, leaving the rest of the document completely untouched.

This architecture guarantees 0% hallucination for unaffected sections, perfectly preserves mandatory document structures, and leverages local high-compute (24x H100 GPUs) specifically for extreme precision in targeted contextual rewriting.

### 7.5. Tenant Policy & Customization Architecture (Migration Plan)

To transition HALO from a purely discovery-based inventory system to a robust, GitOps-driven control plane, HAU separates **Discovered State** (live Kubernetes telemetry) from **Desired Configuration** (governance and policy rules). 

The legacy `is_internal` flags and restrictive `TenantFeatureFlag` tables are deprecated in favor of a centralized `TenantCustomization` architecture. This establishes HAU as the absolute Source of Truth for tenant behavior, enabling the "Rubber Band" sandbox model, accurate UI categorization, and automated lifecycle management.

#### 1. Database Schema Evolution
The database schema introduces a 1:1 relational model tying the discovered `HaloTenant` to a strict configuration definition.

*   **Deprecated Entities:** `TenantFeatureFlag` table and the `is_internal` / `is_internal_overridden` columns on `HaloTenant`.
*   **New Entity:** `TenantCustomization`

| Column | Type | Purpose / Allowable Values |
| :--- | :--- | :--- |
| `tenant_id` | `String (FK)` | One-to-One mapping to `HaloTenant`. |
| `tenant_category` | `String` | Visually groups the tenant in the UI (`INTERNAL`, `CUSTOMER`). |
| `tenant_tier` | `String` | Defines the operational blast radius (`SANDBOX`, `STAGING`, `PRODUCTION`, `SYSTEM_CORE`). |
| `update_policy` | `String` | Deployment governance (`AUTO_LATEST`, `GATED_RELEASE`, `MANUAL_ONLY`). |
| `owner` | `String` | The designated GitLab user or SysEng owner responsible for the tenant. |
| `purpose` | `String` | Brief description of the tenant's existence (solves sandbox garbage collection). |
| `maintenance_window` | `String` | CRON or schedule string for acceptable disruption periods. |
| `configuration_payload`| `JSON` | A dynamic blob holding all non-discoverable configurations. |
| `component_versions` | `JSON` | Stores explicit component container version pins for tenant environments. |

#### 2. The `configuration_payload` JSON Structure
To prevent endless database migrations for new features, all tenant-specific overrides and environment variables are stored in the JSON payload. This payload hydrates the Kustomize manifests during deployment generation.
*   **Ingress & Routing:** Custom vanity domains (e.g., `corp.defensehalo.ai`), SSL certificates.
*   **Infrastructure Quotas:** `max_eps`, `data_retention_days`, requested GPU allocations.
*   **External Integrations:** Hardcoded `.env` values, unique Keycloak Realm IDs, external syslog forwarding paths.
*   **Application Toggles:** Replacements for feature flags (e.g., `enable_advanced_ml_models: true`).

#### 3. Strict Synchronization Boundaries & Category Inference
The `InventoryOrchestrator` handles background synchronization with Kubernetes clusters but is strictly barred from modifying deployment policies or categories.
*   **Best-Effort Inference:** When the orchestrator discovers a brand-new namespace, it runs a keyword string-match on the namespace name. If it detects keywords like `corp`, `core`, or `dev`, it sets the initial `tenant_category` to `INTERNAL`. Otherwise, it defaults to `CUSTOMER`. It sets fail-safe defaults for the rest (`tenant_tier = UNKNOWN`, `update_policy = MANUAL_ONLY`).
*   **Immutability & Permanent Overrides:** The sync logic *only* updates live telemetry (active pods, image tags, cluster IP). It never overwrites the `tenant_category`, `update_policy`, `tenant_tier`, or `owner`. If the UI is used to correct a miscategorized tenant (e.g., changing `halo-democorp` to `INTERNAL`), that UI action becomes the permanent Source of Truth, completely eliminating the need for complex `is_overridden` flags.

#### 4. The "Rubber Band" Sandbox Workflow
This architecture natively supports flexible, self-cleaning developer sandboxes by combining the `TenantCustomization` state with the `ReleaseOrchestrator`.

1.  **Baseline Execution:** A developer sandbox is configured with `update_policy = AUTO_LATEST` tracking the `dev` branch.
2.  **Manual Override (Stretch):** The developer uses the UI to deploy a specific feature branch container to their namespace. The orchestrator rewrites the Kustomize overlay to pin the feature image. The sandbox diverges from the baseline.
4.  **Auto Snap-Back (Release):** The `ReleaseOrchestrator` concludes the CI/CD pipeline, retrieves all tenants where `update_policy == AUTO_LATEST` (and matching the target environment), rewrites their Kustomize manifests to the new merged `dev` tag, and deploys. The manual override is wiped out, and the environment snaps back to the standardized baseline.

### 7.6. Dynamic Tenant Scaffolding (GitOps Automation)
Instead of operators manually copying Kustomize overlays and risking copy-paste errors, HAU will natively auto-generate the complete GitOps directory structure for new tenants upon receiving a gRPC provisioning request from HAM.

1. **Deterministic Structure Generation:** HAU will automatically scaffold the `/tenants/<region>/<tenant-name>/` directory, creating explicit `app/` (CPU workloads) and `gpu/` (AI pipelines) sub-directories to maintain strict node-pool separation.
2. **Unified Smart Patching:** To prevent Kustomize complexity bloat, HAU will write unified, multi-target patches (e.g., a single patch targeting `labelSelector: app in (netflow, syslog, forti)` to inject the `NVIDIA_VISIBLE_DEVICES` override), drastically reducing the file count.
3. **Automated Base Composition:** Based on the features enabled in HAM, HAU will dynamically inject the necessary bases into the tenant's `kustomization.yaml` (e.g., `base/vendor/kafka`, `base/halo/pipelines/storage`, `base/halo/pipelines/netflow-dfp`).
4. **Secretless GitOps:** HAU will NEVER write raw secrets to Kustomize patches. All environmental secrets will be injected at runtime via Vault, ensuring the GitOps repository remains 100% clean and auditable.

### 7.7. Feature Catalog & Auto-Sync Workflow (GitOps Manifesting)

To decouple the UI (HAM) from raw environment variables in the frontend codebase and provide operators with a highly descriptive feature selection pane, HAU will implement an automated Feature Catalog sync pattern.

#### 1. The `FeatureCatalog` Database Model
Instead of hardcoding frontend variables or naïvely parsing the React source code, HAU maintains a robust `FeatureCatalog` database table. This acts as the Source of Truth for HAM to dynamically render feature checkboxes during tenant provisioning. The table will contain rich metadata, such as:
* **Feature ID & Name:** e.g., `netflow_dna`, "Network Netflow DNA".
* **Description & Grouping:** Grouped by capability (e.g., "Network Analytics"), allowing logical UI separation.
* **Environment Variable:** The actual toggle consumed by the frontend (e.g., `VITE_SHOW_NETFLOW_DNA`).
* **Infrastructure Dependencies:** Array of GitOps bases required if this feature is toggled (e.g., `base/halo/pipelines/netflow-dfp`).
* **Pricing & Defaults:** Indicators for whether this is a paid add-on, and if it is enabled by default for new tenants.

#### 2. Developer-Enforced `feature-manifest.yaml`
To avoid manual data entry and keep the `FeatureCatalog` up-to-date, developers are required to define their toggles declaratively.
When a developer builds a new UI capability wrapped in an environment variable (like `import.meta.env.VITE_SHOW_NEW_THING`), they must also update a standard `feature-manifest.yaml` located in the frontend repository. This file serves as the strict schema mapping the frontend's capabilities.

#### 3. Zero-Touch Release Pipeline Sync
Because HAU inherently orchestrates the end-to-end Git Release workflow (Section 4.6), we embed an automated sync hook directly into the release logic:
* When a developer merges a feature branch into `dev` or `main` and HAU triggers the CI/CD pipeline, the `ReleaseOrchestrator` automatically fetches the newly merged `feature-manifest.yaml` from the GitLab repository.
* HAU parses the YAML and automatically upserts (inserts or updates) the definitions into the `FeatureCatalog` database table.
* **Result:** The moment a frontend release finishes, the new feature instantly populates in the HAM UI provisioning screen with its full description, required infrastructure dependencies, and pricing metadata—requiring zero administrative overhead or hardcoding.

This architecture ensures high-fidelity UX in the administration portal while preserving strict, machine-readable governance over what environment variables get injected during the GitOps Kustomize generation phase.

> **Note on Current State (Initial Discovery):** An initial discovery of the frontend's hardcoded feature toggles (e.g., `VITE_SHOW_GALAXY`, `VITE_SHOW_NETFLOW_DNA`) has been performed and manually documented in `docs/hau_features.md`. Once the automated `feature-manifest.yaml` sync pipeline is fully implemented, this static document will be deprecated in favor of the live `FeatureCatalog` database.