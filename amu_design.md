# Application Management Utility (AMU) - Design Document

**Version:** 0.2.7
**Date:** 2026-07-14
**Author:** Martin Kecka & Gemini AI
**Category:** (U) Utility / Administrative
**Acronym:** AMU

---

## 1. Introduction & Purpose 📜

The **Application Management Utility (AMU)** is a headless, centralized gRPC microservice designed to handle **Application Portfolio Management (APM)** and **ITSM Integration** for the SERA AI ecosystem.

While the Health Monitor Utility (HMU) strictly monitors physical infrastructure (GPUs, pods, hardware utilization), AMU monitors the **logical reality** of the applications. It acts as the central broker for application registries, distributed documentation, centralized user feedback, and logical dependency graphing.

AMU serves as the primary data aggregator for the Sera Admin Module (SAM) UI regarding application metadata and user interaction states.

---

## 2. Core Architectural Principles 🏗️

1. **Logical Abstraction:** AMU does not care about pods, nodes, or physical hardware. It cares about Application IDs, versions, user feedback tickets, and logical dependencies.
2. **Standardized Ecosystem Contracts:** AMU enforces ecosystem compliance. To be managed by AMU, an application must implement a standardized set of gRPC endpoints via the `sera_standard.proto`.
3. **Network Boundary & Topology Mapping:** AMU classifies application dependencies into four distinct architectural boundaries (`LOCAL`, `AI_FACTORY`, `CORPORATE`, `EXTERNAL`) and explicit topology groups (`infra`, `connector`, `internal`, `external`).
4. **Database-Backed Registry:** AMU uses its own PostgreSQL database to maintain the global registry of applications and state logic, decoupled into robust Reader/Writer I/O patterns, with support for High Availability (HA) routing via `write_dsn` and `read_dsn`.
5. **Continuous Discovery:** AMU employs a background daemon thread that continuously polls and refreshes metadata for actively registered endpoints to ensure logical reality matches the registry.
6. **Shared File-System Access:** For MVP log inspection and documentation aggregation, AMU relies on direct read access to standardized shared volumes (e.g., `/ai/assets/logs/` and `/ai/assets/docs/`).

---

## 3. Database Schema & State Management 🗄️

AMU maintains the global list of active microservices and logical dependencies within its own structured PostgreSQL schema. This strict separation ensures AMU is the sole source of truth for application portfolio state.

### 3.1. Core Application Registry (`models.py`)
* **Applications:** Tracks metadata such as `acronym`, `name`, `category`, and `version`.
* **Dependencies:** Uses dynamic data from `GetDeepHealth()` to construct a Directed Acyclic Graph (DAG) in the database, tracking topological `dependency_group` context alongside network zones.
* **Service Matrix:** Tracks dynamic multi-service arrays and `service_health_matrix` per application, rather than single hardcoded endpoints, verifying deep health reachability via active TCP socket pinging.
* **External Agents:** Unmanaged ecosystem agents are tracked natively via isolated `ExternalAgent` infrastructure with `ecosystem` metadata support.

---

## 4. Core Sub-Systems & Responsibilities ⚙️

### 4.1. Application Registry & Dependency DAG
AMU maintains the global list of active microservices in PostgreSQL. 
* It uses the data from `GetDeepHealth()` to build a **Directed Acyclic Graph (DAG)** of dependencies, dynamically classifying them into environment targets (`dev`, `prod`), network zones, and explicit dependency groups (`internal`, `external`).
* It natively tracks dynamic multi-service arrays and `service_health_matrix` per application, rather than single hardcoded endpoints.
* Tracks unmanaged ecosystem agents natively via the isolated `ExternalAgent` infrastructure and persists manual AI Agent properties via the `inventory_metadata` store.
* The Port Manager exposes structured ecosystem port utilization to prevent overlap during onboarding.

### 4.2. GitOps Orchestration Engine
AMU features an integrated GitOps Engine allowing it to manage the physical files of registered applications:
* Safely execute repository commits (`git sync`), rollbacks (`git rollback`), and history retrieval directly via gRPC.
* Tracks `physical_path` and `is_managed` states in the registry, enabling centralized automation for local deployments.
* Contains a dedicated `deploy push-std` CLI command to orchestrate the rollout of updates (like `sera_standard_service.py`) across all managed modules, complete with automated service restarts and deep-diff validation.

### 4.3. Centralized Feedback Broker (SNC Connector)
AMU acts as the middleman between user-facing applications (like Cash Application Assistant) and IT Service Management (ServiceNow).
1. AMU aggregates feedback from registered apps.
2. AMU normalizes this feedback and stores it in its PostgreSQL database.
3. AMU utilizes the **SNCConnector** (`core/connectors/snc_connector.py`) to fire direct gRPC payloads to the ServiceNow Connector, generating official Incidents or Service Requests dynamically.

### 4.4. Log Inspection Engine
AMU provides administrators with a direct view into application logs without requiring `kubectl` access.
* **Current State:** AMU directly parses log files written to the standardized shared volume: `/ai/assets/logs/{app_acronym}/`.
* **Future State:** AMU will transition to querying a centralized logging database (VictoriaLogs) using `trace_id` for distributed log correlation.

### 4.5. Documentation Aggregator
AMU replaces localized script-based documentation fetchers. It proxy-calls the `GetChangelog` and `GetDocs` endpoints of registered apps, parses the raw strings into structured JSON payloads, and serves them directly to the SAM WebUI.

### 4.6. Background Rediscovery Daemon
AMU maintains a background daemon thread (`grpc_api/services.py`) that continuously polls and refreshes metadata for actively registered endpoints. This auto-rediscovery behavior is controlled natively by the `discovery` block in `settings.yaml`, eliminating the need for manual CLI registry syncs or external systemd timers.

---

## 5. Module Structure 📂

The AMU codebase is separated into logical boundaries to ensure clean domain responsibilities:

* `cli/`: Command-line interfaces for manual registration, info, and deployments (`amu_cli.py` and commands).
* `config/`: Centralized settings (`settings.yaml`).
* `core/`: The core business logic including GitOps (`git_manager.py`), dependency resolution (`registry_manager.py`), ports (`port_manager.py`), and IO (`io_handlers/`).
* `database/`: Database models and connection logic.
* `grpc_api/`: The `amu_service.py` entrypoint and standard ecosystem protobuf definitions.
* `scripts/`: Dumping ground for one-time migrations and testing scripts.

---

## 6. Developer Guide: How to Register a New App 🛠️

For AMU to successfully track, monitor, and route tickets for your application, it must comply with the ecosystem standards.

> [!TIP]
> **LLM Automation:** When asking an AI Agent to onboard a new application, point it directly to the implementation guide: `docs/amu_llm_guide.md`.

### 6.1. Implement the Universal SERA Standard Protobuf
Your application's gRPC server MUST expose the `SeraAppService` defined in `/ai/assets/protos/sera_standard/sera_standard.proto`. This is now a **`v1-universal`** contract enforced across the ecosystem.

Developers have two options to implement this contract:
1. **Option A: The Universal Service (Recommended):** Standardize your `settings.yaml` structure and seamlessly inject the pre-built `sera_standard_service.py` handler. This natively resolves `GetAppInfo`, `GetDeepHealth`, and dynamic dependency topology without custom code. Refer to the configuration guide: `docs/conf_concept.md`.
2. **Option B: Custom Implementation:** Build a custom gRPC service that manually satisfies the expected structure of the following RPC methods:
   * **`GetAppInfo()`**: Returns static metadata (Acronym, Name, Category, Version) and a flat list of downstream dependencies.
   * **`GetDeepHealth()`**: Evaluates the app's ability to reach its dependencies (e.g., Databases, APIs). The AMU heuristic engine parses the `connection_metadata` (like `host` or `instance_url`) to map the dependency into the correct network boundary (`LOCAL`, `AI_FACTORY`, `CORPORATE`, `EXTERNAL`).
   * **`GetChangelog()`**: Returns raw YAML from your `docs/CHANGELOG.yaml`.
   * **`GetDocs()`**: Returns raw Markdown from your design documentation.
   * **`GetFeedback()`**: (If applicable) Exposes UI feedback loops.

### 6.2. Minimum Verification Rules
When a new application is registered with AMU, AMU will perform a strict validation check before adding it to the database:
1. **Metadata Existence:** Verifies the presence of a properly formatted `settings.yaml` with a `module_info` block.
2. **Documentation Structure:** Verifies the existence of a `docs/` folder, a `CHANGELOG.yaml`, and a main design document.
3. **API Compliance:** Attempts to ping the application's `/info` and `/health` endpoints to ensure the gRPC contract is respected.

*(Note: Verification rules will evolve and become stricter as the ecosystem matures.)*

### 6.3. Ensure Proper Configuration
AMU relies on your `settings.yaml` to extract the port if you are using path-based registration. Your configuration must group API and gRPC endpoints under `system_services`. Ensure your config has:

```yaml
system_services:
  grpc:
    port: 50xxx
  api:
    port: 78xxx
```

Also, ensure dependencies are explicitly structured in an `external_services` YAML block with `type` and `dependency_name`.

### 6.4. Register via the CLI
Use the AMU CLI to onboard the application. You can do this by exact endpoint, by directory path, or via an automated port sweep.

[bash]
# Option A: Register by explicit Endpoint
python3 application_management_utility.py registry register --endpoint "localhost:50091"

# Option B: Register by Directory Path
python3 application_management_utility.py registry register --path /ai/snow_connector

# Option C: Auto-Discover (Sweeps standard port ranges)
python3 application_management_utility.py registry register --auto
[/bash]

### 6.5. Verify Your Registration
Once registered, verify that AMU successfully parsed your app and mapped its dependencies correctly:

[bash]
# Check the registry
python3 application_management_utility.py registry list

# Verify the network boundary mappings
python3 application_management_utility.py registry deps
[/bash]

---

## 7. Roadmap & Future Enhancements 🚀

* **API Payload Refactoring (AppInfo vs DeepHealth):** Once all ecosystem applications have adopted the `sera_standard_service.py`, refactor `sera_standard.proto` to strictly separate static vs. dynamic dependency data. Currently, `GetDeepHealth` builds heavy static metadata (like `connection_metadata`, `dependency_group`, and `dependency_type`) alongside dynamic health status. This static topology data logically belongs in `GetAppInfo` to reduce the payload overhead of constant health polling. The AMU Registry and SAM UI (`AmuTopologyWidget.jsx`) are already architected to gracefully merge static AMU registry data with lightweight dynamic health statuses.
* **Gateway Integration:** In future iterations of the Sera AI Gateway architecture, AMU will act as the control plane to dynamically "deploy" or "deregister" applications in the Gateway's proxy configuration.
* **Standardization & Compliance Matrix:** Build an automated scoring system within the dashboard that grades registered applications on ecosystem compliance (e.g., verifying `CHANGELOG.yaml`, `docs/`, `system_services`, and LLM configs are properly populated).
* **Remote Action Execution:** Expand the `sera_standard.proto` to allow AMU to trigger lifecycle events on remote microservices (e.g., `RESTART_SERVICES`, `REBUILD_UI`). *Note: While currently this would target `systemctl` or `npm`, the abstraction ensures seamless transition when services migrate to Kubernetes/GitLab orchestration in the future.*
* **VictoriaLogs Integration:** Transition the Log Inspection Engine from parsing flat files in `/ai/assets/logs/` to querying VictoriaLogs via PromQL/MetricsQL.
* **Automated Changelog Blasts:** Allow AMU to automatically parse new `CHANGELOG.yaml` entries upon application registration and broadcast release notes to users via email or UI notifications.
* **Deep Feedback Analytics:** Pass raw user feedback strings through local LLMs to automatically categorize issues as "Bugs" vs "Feature Requests" before handing them off to the SNC Connector.