# Application Management Utility (AMU) - Design Document

**Version:** 0.1
**Date:** 2026-05-18
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

1. **Logical Abstraction:** AMU does not care about pods, nodes, or physical hardware. It cares about Application IDs, versions, user feedback tickets, and logical dependencies (e.g., "Cash Application Assistant depends on the SNOW Connector").
2. **Standardized Ecosystem Contracts:** AMU enforces ecosystem compliance. To be managed by AMU, an application must implement a standardized set of gRPC endpoints (`/info`, `/health`, `/feedback`).
3. **Database-Backed Registry:** AMU uses its own PostgreSQL database to maintain the global registry of applications, tracking their metadata and the state of their feedback tickets.
4. **Shared File-System Access:** For MVP log inspection and documentation aggregation, AMU relies on direct read access to standardized shared volumes (e.g., `/ai/assets/logs/` and `/ai/assets/docs/`).

---

## 3. The "SERA Standard" Application Contract 🤝

For an application to be successfully registered and managed by AMU, it must implement a standardized gRPC interface. This allows AMU to query any app generically without needing custom, app-specific logic.

### 3.1. Standardized Endpoints
Every compliant microservice must expose the following gRPC methods:

* **`GetAppInfo()`**: Returns static metadata about the application.
  * **Payload:** Version, Acronym, Category, Description, and a list of downstream **Logical Dependencies** (e.g., `["psql", "redis", "snow_connector", "vllm-gemma4-31b"]`).
* **`GetDeepHealth()`**: Returns the *logical* health of the application.
  * **Payload:** A matrix evaluating the application's ability to reach its declared dependencies (e.g., "PostgreSQL Connection: OK", "SNOW Connector: FAILED").
* **`GetFeedback() / SubmitFeedback()`**: Exposes the application's internal user feedback loops to the central AMU broker.

### 3.2. Minimum Verification Rules
When a new application is registered with AMU (e.g., via CLI `amu-cli register --app cap`), AMU will perform a basic validation check before adding it to the database:
1. **Metadata Existence:** Verifies the presence of a properly formatted `settings.yaml` with a `module_info` block.
2. **Documentation Structure:** Verifies the existence of a `docs/` folder, a `CHANGELOG.yaml`, and a main design document.
3. **API Compliance:** Attempts to ping the application's `/info` and `/health` endpoints to ensure the gRPC contract is respected.

*(Note: Verification rules will evolve and become stricter as the ecosystem matures.)*

---

## 4. AMU Sub-Systems & Responsibilities ⚙️

### 4.1. Application Registry & Dependency DAG
AMU maintains the global list of active microservices in PostgreSQL. 
* Because microservices share resources (like the ServiceNow connector), AMU uses the data from `GetAppInfo()` to build a **Directed Acyclic Graph (DAG)** of dependencies. 
* This allows the SAM UI to render accurate dependency maps and trace logical cascading failures (e.g., if the SNOW Connector dies, AMU knows exactly which frontend applications are affected).

### 4.2. Centralized Feedback Broker
AMU acts as the middleman between user-facing applications (like Cash Application Assistant) and IT Service Management (ServiceNow).
1. AMU continuously polls (or receives pushed) feedback from registered apps via the `/feedback` gRPC contract.
2. AMU normalizes this feedback and stores it in its PostgreSQL database.
3. AMU routes actionable feedback through the `snow_connector` to generate official INC/RITM tickets.
4. AMU tracks the ticket state and serves the unified "Feedback Inbox" directly to the SAM UI for administrators.

### 4.3. Log Inspection Engine
AMU provides administrators with a direct view into application logs without requiring `kubectl` access.
* **Current State:** AMU directly parses log files written to the standardized shared volume: `/ai/assets/logs/{app_acronym}/`.
* **Future State:** AMU will transition to querying a centralized logging database (VictoriaLogs) using `trace_id` for distributed log correlation.

### 4.4. Documentation Aggregator
AMU replaces localized script-based documentation fetchers. It scans the registered applications, parses their `docs/` directories, and serves aggregated Markdown and Changelog JSON payloads directly to the SAM WebUI.

---

## 5. High-Level Database Schema (PostgreSQL) 🗄️

AMU will utilize a structured relational database to maintain ecosystem state.

* **`applications` table:**
  * `id` (PK)
  * `acronym` (Unique, e.g., 'CAA', 'HMU')
  * `name`, `version`, `category`, `description`
  * `grpc_endpoint` (How AMU contacts it)
  * `registered_at`, `last_verified_at`
* **`app_dependencies` table (The DAG):**
  * `parent_app_id` (FK)
  * `child_dependency_name` (e.g., 'snow_connector')
* **`feedback_tickets` table:**
  * `id` (PK)
  * `app_id` (FK - Where the feedback originated)
  * `user_id`, `timestamp`
  * `raw_content`, `ai_analysis_summary`
  * `snow_ticket_number` (Nullable, injected once synced with ServiceNow)
  * `status` (New, In Progress, Resolved)

---

## 6. Roadmap & Future Enhancements 🚀

* **Gateway Integration:** In future iterations of the Sera AI Gateway architecture, AMU will act as the control plane. AMU will possess the ability to dynamically "deploy" or "deregister" applications in the Gateway's proxy configuration, moving away from hardcoded `config.py` files.
* **VictoriaLogs Integration:** Transition the Log Inspection Engine from parsing flat files in `/ai/assets/logs/` to querying VictoriaLogs via PromQL/MetricsQL.
* **Automated Changelog Blasts:** Allow AMU to automatically parse new `CHANGELOG.yaml` entries upon application registration and broadcast release notes to users via email or UI notifications.