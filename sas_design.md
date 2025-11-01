# Sera Admin Suite (SAS) - Design Document

**Version:** 1.4
**Date:** 2025-11-01
**Author:** Martin Kecka & Gemini AI
**Category:** (U) Utility / Administrative

## 1. Introduction & Purpose

The **Sera Admin Suite (SAS)** is the centralized, modular platform for monitoring, debugging, managing, and operating the entire SERA AI ecosystem.

The primary goal of SAS is to replace all disparate administrative tools (e.g., `history_lookup.py`, `systemd` log checks, direct file edits) with a single, unified, and secure interface. It provides a "single pane of glass" for developers, operators, and administrators.

SAS is composed of two primary clients and a collection of new and existing backend utility services:

1.  **Clients (The "Face"):**
    * **`SAS-WebUI`:** A modern React-based web application (SPA) providing a rich, graphical interface.
    * **`SAS-CLI`:** A powerful Python-based Command Line Interface (CLI) for "power users" and scripting. Both clients are part of the SAS.

2.  **Backends (The "Brains"):**
    * A set of specialized, single-responsibility gRPC utility services that provide data to the clients.

## 2. Core Architectural Principles

SAS is built on a "headless backend" and "aggregator" microservice model.

1.  **Aggregator Pattern:** The `SAS-WebUI` (React) is a "dumb" client. It communicates with **only one** backend service: the new **`ADS (Admin Service)`**. This `ADS` service acts as an *aggregator*, fanning out gRPC requests to all other specialized utility backends (like `DIU`, `HIU`, `CU`), simplifying the frontend.

2.  **Specialized Utility Backends:** All administrative functionality is encapsulated in discrete, single-responsibility gRPC services (e.g., `DIU` for data, `HIU` for history, `CU` for configuration).

3.  **Decoupled Clients:** The `SAS-WebUI` and `SAS-CLI` are both clients of the same gRPC services. Any feature available in the UI can also be accessed via the CLI.

4.  **Observability via `trace_id`:** The entire ecosystem will be built around a **Distributed Tracing** model. Every user request generates a `trace_id` which is propagated through all gRPC calls. This allows SAS to provide a complete, end-to-end log of a single user turn.

5.  **Centralized Logging (Loki):** All modules will stop logging to local files (`/ai/assets/logs/`). They will be re-configured to stream logs to a central **Loki** instance, making them queryable by `trace_id`.

6.  **Configuration: Decentralized Files, Centralized Management:**
    * **Source of Truth:** Each module's local `config/settings.yaml` file is the *absolute source of truth* for that module.
    * **No Startup Dependency:** Modules start on their own by reading their local file. They have **zero** startup dependency on the `CU`.
    * **Centralized Operator:** The `CU (Configuration Utility)` is a powerful *operator tool*. It has the permissions to **read** all `settings.yaml` files for display in the UI, **write** changes back to those files, and **push** a "reload" command to the relevant module.

## 3. System Components

### 3.1. `SAS-WebUI` (The Frontend) - \[NEW MODULE\]

* **Technology:** React (Vite, shadcn/ui or Material-UI), served by the Sera AI Gateway.
* **Backend:** Communicates *exclusively* with the `ADS (Admin Service)` via gRPC-Web.

#### Key UI Pages:

1.  **🌍 Ecosystem Health Dashboard:** The landing page. Displays the real-time status of all modules and external dependencies (LLM endpoints, HAProxy, databases) by consuming data from the `HMU`.
2.  **🔍 Centralized Log Viewer:** An embedded **Grafana** dashboard configured to query the **Loki** database. The primary feature is a search bar for `trace_id`.
3.  **💬 Conversation History Inspector:** A UI for the `HIU`. Lists all user sessions and turns. Includes a "Debug this turn" button that opens the Log Viewer with the turn's `trace_id` pre-filled.
4.  **📚 Knowledge Base Inspector:** A UI for the `DIU`. Allows browsing Confluence spaces, pages, and chunks, and inspecting their metadata.
5.  **📥 Centralized Feedback Inbox:** A single inbox that aggregates feedback items from *all* modules (CAP, CAA, etc.).
6.  **🛠️ Configuration Manager:** A "self-service" UI for managing system configurations (consuming the `CU`), split into:
    * **Module Settings Inspector:** A UI to view and edit the `settings.yaml` file for any registered module. **Crucial Security:** Any field containing `dsn`, `password`, `key`, or `token` will be **masked (`[********]`) by default**. An admin-only "Reveal" button will be required to see or edit these values.
    * **Config Push Button:** A "Save & Reload Module" button to push changes via the `CU`.
    * **Prompt Editor:** A user-friendly view for editing just the "persona" sections of config files.
7.  **🚀 Module Template Builder:** A wizard-style UI that calls the `MTU` to generate a downloadable `.zip` file with the skeleton code for a new SERA module.

### 3.2. `ADS (Admin Service)` (The Backend Aggregator) - \[NEW MODULE\]

* **Category:** (U) Utility
* **Purpose:** The single, unified gRPC backend for the `SAS-WebUI`.
* **Responsibilities:**
    * **Health Aggregation:** Exposes a `GetSystemHealth()` gRPC method that calls the `HMU`.
    * **Feedback Aggregation:** Exposes `GetAggregatedFeedback()` and `UpdateFeedback()` methods.
        * `GetAggregatedFeedback()` calls `CAP.GetFeedback()`, `CAA.GetFeedback()`, etc., in parallel and merges the results.
        * `UpdateFeedback()` routes the gRPC call to the correct module (e.g., `CAP.UpdateFeedback()`).
    * **Proxy/Wrapper:** Provides simple gRPC proxy methods for `DIU`, `HIU`, `CU`, and `MTU`.

### 3.3. `HMU (Health Monitor Utility)` - \[NEW MODULE\]

* **Category:** (U) Utility
* **Purpose:** Provides real-time, centralized health status for the entire ecosystem.
* **Responsibilities:**
    * Runs as a persistent service.
    * Continuously pings the standard gRPC health check endpoint of *every* module.
    * Continuously monitors external dependencies (AI Factory LLM endpoints, HAProxy, PostgreSQL, etc.).
    * Exposes a single `GetSystemStatus()` gRPC endpoint.

### 3.4. `HIU (History Inspector Utility)` - \[NEW MODULE\]

* **Category:** (U) Utility
* **Purpose:** A robust gRPC service refactor of the `history_lookup.py` script.
* **Responsibilities:**
    * Provides a secure, read-only gRPC API to query the Orchestrator's PostgreSQL database.
    * Exposes endpoints like `ListSessions()`, `GetSessionDetails(session_id)`.
    * **CRITICAL API Feature:** The `GetTurnDetails(turn_id)` method **must** return the `trace_id` associated with that turn.

### 3.5. `CU (Configuration Utility)` - \[NEW MODULE\]

* **Category:** (U) Utility
* **Purpose:** The secure backend for *all* ecosystem configuration management.
* **Responsibilities:**
    * **Module Registry:** Manages a central manifest (`module_registry.yaml`) that maps module names to their on-disk file paths (e.g., `cap: /ai/cv_analysis_processor/config/settings.yaml`).
    * **Config Reader:** Exposes `GetModuleConfig(module_name)`. This method reads the specified `settings.yaml` file from the server's filesystem, tags all secret fields (keys with `dsn`, `key`, `token`, etc.), and returns the structured data as JSON.
    * **Config Writer:** Exposes `UpdateModuleConfig(module_name, config_json)`. This method:
        1.  Securely reads the target `settings.yaml` file.
        2.  Merges the changes from the `config_json` (preserving comments and structure).
        3.  Writes the file back to disk.
        4.  Calls the `ReloadConfig()` gRPC endpoint on the target module (e.g., `CAP.ReloadConfig()`).
    * **Config Reloader:** Exposes `ReloadModuleConfig(module_name)` to just trigger the reload call.

### 3.6. `SAS-CLI` (The Command-Line Client) - \[NEW MODULE\]

* **Technology:** Python (`click` or `argparse`).
* **Purpose:** A comprehensive CLI tool for developers and power users.
* **Architecture:** A client that consumes the gRPC backends directly.
* **Usage:**
    ```
    # It will consume the new utility services directly
    sas-cli health                               # Calls HMU.GetSystemStatus
    sas-cli history list-sessions                # Calls HIU.ListSessions
    sas-cli history get-turn <turn_id> --logs    # Calls HIU.GetTurnDetails, gets trace_id, then queries Grafana/Loki
    sas-cli kb find-page "Netbox"                # Calls DIU.FindPageByTitle
    sas-cli feedback list --source cap           # Calls ADS.GetAggregatedFeedback
    
    # New Config Commands
    sas-cli config list                          # Calls CU.ListRegisteredModules
    sas-cli config get cap                       # Calls CU.GetModuleConfig("cap")
    sas-cli config set cap llm_interface.model "gemma4:latest" # Calls CU.UpdateModuleConfig(...)
    sas-cli config reload cap                    # Calls CU.ReloadModuleConfig("cap")

    sas-cli module new <module-name>             # Calls MTU.GenerateModule
    ```

### 3.7. `MTU (Module Template Utility)` - \[NEW MODULE\]

* **Category:** (U) Utility
* **Purpose:** The secure backend for the "Module Template Builder."
* **Responsibilities:**
    * Provides a gRPC API for generating the skeleton code for a new SERA module.
    * `rpc GenerateModule(GenerateModuleRequest) returns (GenerateModuleResponse);`
    * The request will include parameters like `module_name` ("MyNewTool"), `category` ("Connector"), and `acronym` ("MNT").
    * Returns a base64-encoded `.zip` file with the complete module template.

## 4. Core Infrastructure & Module Requirements

To make SAS functional, the following changes **must** be implemented across the entire ecosystem.

### 4.1. Distributed Tracing (`trace_id`)

* **Priority:** **CRITICAL**
* **Requirement:** A unique `trace_id` must be generated for every user interaction and propagated to all downstream services for end-to-end logging.
* **Module Changes:**
    * **Sera Orchestrator:** Must be updated to *generate* a `trace_id` at the beginning of `process_query`.
    * **All Modules:** Must be refactored to:
        1.  Accept a `trace_id` via gRPC metadata in all incoming calls.
        2.  Include this `trace_id` in *every single log message* they generate.
        3.  Pass this `trace_id` along in any *outgoing* gRPC calls they make.

### 4.2. Centralized Logging (Loki Stack)

* **Priority:** **HIGH**
* **Requirement:** All logs must be streamed to a central, queryable database instead of being written to local files.
* **Module Changes:**
    * **All Modules:** Must be re-configured to stop logging to files (e.g., `/ai/assets/logs/`) and log *only* to `stdout` (standard output).
* **New Infrastructure:**
    * **Loki, Promtail, Grafana:** This stack must be deployed to act as the central log database and viewer.

### 4.3. Standardized Health Checks

* **Priority:** **HIGH**
* **Requirement:** The `HMU` needs a reliable, standard way to ping every service.
* **Module Changes:**
    * **All gRPC Modules:** Must implement the standard [gRPC Health Checking Protocol](https://github.com/grpc/grpc/blob/master/doc/health-checking.md).

### 4.4. Module-Level Feedback Endpoints

* **Priority:** **MEDIUM**
* **Requirement:** The `ADS` needs a standard way to aggregate feedback.
* **Module Changes:**
    * **CAP, CAA (and future UI modules):** Must be refactored to expose new gRPC endpoints on their *existing* services:
        * `rpc GetFeedback(GetFeedbackRequest) returns (FeedbackListResponse);`
        * `rpc UpdateFeedback(UpdateFeedbackRequest) returns (FeedbackResponse);`

### 4.5. **\[NEW\]** Standardized Config Reload Endpoint

* **Priority:** **MEDIUM**
* **Requirement:** The `CU` needs a way to notify modules that their `settings.yaml` file has changed.
* **Module Changes:**
    * **All Modules:** Must expose a new, standard gRPC endpoint (e.g., `rpc ReloadConfig(ReloadConfigRequest) returns (ReloadConfigResponse);`).
    * When called, this endpoint will trigger the module to re-load its `settings.yaml` file from disk into its running config object.

## 5. Phased Implementation Plan

### Phase 1: Foundation & Read-Only Inspection (Developer Value)
* **Goal:** Give developers immediate, powerful tools for debugging and monitoring.
* **Tasks:**
    1.  Build `HIU` (gRPC service) by refactoring `history_lookup.py`.
    2.  Build `HMU` (gRPC service) to monitor existing health endpoints.
    3.  Build `ADS` (gRPC service) with endpoints to call `DIU`, `HIU`, and `HMU`.
    4.  Build the `SAS-WebUI` (React) with the first 3 pages: **Health**, **KB Inspector**, and **History Inspector**.
    5.  Begin work on the `SAS-CLI`, targeting the same backends.
* **Value:** Replaces `history_lookup.py` and provides a single dashboard for system status.

### Phase 2: End-to-End Observability (Operator Value)
* **Goal:** Enable "magic" debugging with `trace_id`.
* **Tasks:**
    1.  **(The Hard Part)** Implement `trace_id` propagation across all critical modules.
    2.  Deploy the Loki/Grafana stack.
    3.  Re-configure all modules to log to `stdout` and stream to Loki.
    4.  Add the **Log Viewer** (embedded Grafana) page to `SAS-WebUI`.
    5.  Enhance the **History Inspector** page: the `HIU` must return the `trace_id`, and the UI must add the "Debug this turn" button that opens Grafana pre-filtered to that `trace_id`.
* **Value:** Slashes debugging time from hours to seconds.

### Phase 3: Interactive Management (Administrator Value)
* **Goal:** Enable management and administration from the UI.
* **Tasks:**
    1.  Refactor `CAP` and `CAA` to add the `Get/UpdateFeedback` gRPC endpoints.
    2.  Add the **Feedback Inbox** page to `SAS-WebUI`.
    3.  Implement the **`ReloadConfig()`** gRPC endpoint in all modules.
    4.  Design and build the `CU` (gRPC service) with its `Get/Update/Reload` logic.
    5.  Add the **Configuration Manager** page to `SAS-WebUI`, ensuring secret-masking is implemented.
    6.  Build the `MTU` (gRPC service) for generating new module templates.
    7.  Add the **Module Builder** page to `SAS-WebUI`.
    8.  Expand `SAS-CLI` to cover all new management functions.
* **Value:** Fulfills the "self-service" goal, centralizes all administrative tasks, and accelerates new module development.