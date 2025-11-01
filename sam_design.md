# Sera Admin Module (SAM) - Design Document

**Version:** 0.1
**Date:** 2025-11-01
**Author:** Martin Kecka & Gemini AI
**Category:** (U) Utility / Administrative

## 1. Introduction & Purpose

The **Sera Admin Module (SAM)** is the centralized, modular platform for monitoring, debugging, managing, and operating the entire SERA AI ecosystem.

The primary goal of SAM is to replace all disparate administrative tools (e.g., `history_lookup.py`, `systemd` log checks, direct file edits) with a single, unified, and secure interface. It provides a "single pane of glass" for developers, operators, and administrators to observe system health, debug user interactions, inspect data, and manage platform configurations.

This document outlines the "Sera Admin Suite" architecture, which consists of:

1.  **The `SAM` (Sera Admin Module) Project:** This is the central user-facing project (e.g., `/ai/sera_admin_module/`) that follows the standard SERA Module Blueprint. It contains three key components:
    * **`SAM-WebUI` (React):** The graphical frontend for administrators.
    * **`SAM-Backend` (Aggregator):** The "Backend for Frontend" (BFF) that serves the `SAM-WebUI`.
    * **`SAM-CLI` (Python):** The command-line interface for developers and "power users".

2.  **"Addon" Utility Modules:** A collection of *separate, independent, headless* gRPC utility modules (`DIU`, `HIU`, `CU`, `HMU`, `MTU`) that provide the actual backend functionality.

## 2. Core Architectural Principles

SAS is built on a "headless backend" and "aggregator" microservice model.

1.  **Central "Admin" Module:** `SAM` is the single project that provides the user-facing interfaces (`SAM-WebUI` and `SAM-CLI`) and the UI's backend.

2.  **Specialized "Utility" Modules:** All "addon" utilities (`DIU`, `HIU`, `CU`, `HMU`, `MTU`) are independent, headless gRPC services, each with a single responsibility. They follow the standard [SERA Module Blueprint](https://gitlab.gt-t.net/ai/sera/guides/-/blob/master/tool_guides/module_blueprint.md).

3.  **Aggregator Pattern:** The `SAM-Backend` (part of the `SAM` module) is the *only* thing the `SAM-WebUI` (React) talks to. This `SAM-Backend` acts as an *aggregator*, fanning out gRPC requests to all other specialized utility modules (`DIU`, `HIU`, etc.) and collecting the results. This simplifies the frontend, centralizes UI-facing logic, and reduces client-side complexity.

4.  **CLI Bypasses Aggregator:** The `SAM-CLI` is a "power tool" and does *not* talk to the `SAM-Backend`. It bypasses the aggregator and makes gRPC calls *directly* to the specialized utility modules (`DIU`, `HIU`, `CU`, `HMU`, `MTU`). This is more robust for scripting and debugging, as it has fewer dependencies.

5.  **Observability via `trace_id`:** The entire ecosystem will be built around a **Distributed Tracing** model. Every user request generates a `trace_id` which is propagated through all gRPC calls. This allows SAM to provide a complete, end-to-end log of a single user turn.

6.  **Centralized Logging (Loki):** All modules will stop logging to local files (`/ai/assets/logs/`). They will be re-configured to stream logs to a central **Loki** instance, making them queryable by `trace_id`.

7.  **Configuration: Decentralized Files, Centralized Management:**
    * **Source of Truth:** Each module's local `config/settings.yaml` file is the *absolute source of truth* for that module's configuration.
    * **No Startup Dependency:** Modules start on their own by reading their local file. They have **zero** startup dependency on the `CU`.
    * **Centralized Operator:** The `CU (Configuration Utility)` is a powerful *operator tool*. It has the permissions to **read** all `settings.yaml` files for display in the UI, **write** changes back to those files, and **push** a "reload" command to the relevant module.

## 3. System Components

### 3.1. `sera_admin_module` (SAM) - \[NEW MODULE\]

This is the main project, following the module blueprint structure (e.g., `/ai/sera_admin_module/`).

* `/ui/`: Contains the **`SAM-WebUI`** (React) code.
* `/cli/`: Contains the **`SAM-CLI`** (Python/Click) code.
* `/rest_api/` and `/grpc_api/`: Contains the **`SAM-Backend`** (FastAPI/gRPC) aggregator service.
* `/core/connectors/`: Contains the `SAM-Backend`'s gRPC clients for talking to `DIU`, `HIU`, `CU`, `HMU`, `MTU`.
* `config/settings.yaml`: Contains the gRPC endpoints for all the utility modules that the `SAM-Backend` and `SAM-CLI` need to call.

---
#### 3.1.1. `SAM-WebUI` (React Frontend)

* **Technology:** React (Vite, shadcn/ui), lives in `sera_admin_module/ui/`.
* **Authentication:** Relies on HTTP headers (`X-Forwarded-User`, `X-Forwarded-Groups`) injected by the Sera AI Gateway.
* **Backend:** Communicates *exclusively* with the `SAM-Backend` (FastAPI/gRPC-Web) living in the same project.

**Key UI Pages:**

1.  **🌍 Ecosystem Health Dashboard:** The landing page. Displays the real-time status of all modules and external dependencies (LLM endpoints, HAProxy, databases) by consuming data from the `SAM-Backend` (which calls `HMU`).
2.  **🔍 Centralized Log Viewer:** An embedded **Grafana** dashboard configured to query the **Loki** database. The primary feature is a search bar for `trace_id`.
3.  **💬 Conversation History Inspector:** A UI that calls the `SAM-Backend` (which calls `HIU`). Lists all user sessions and turns. Includes a "Debug this turn" button that opens the Log Viewer with the turn's `trace_id` pre-filled.
4.  **📚 Knowledge Base Inspector:** A UI that calls the `SAM-Backend` (which calls `DIU`). Allows browsing Confluence spaces, pages, and chunks, and inspecting their metadata and generated content.
5.  **📥 Centralized Feedback Inbox:** A single inbox that calls the `SAM-Backend` (which aggregates from `CAP`, `CAA`, etc.). Allows admins to review, update status, and assign feedback.
6.  **🛠️ Configuration Manager:** A "self-service" UI that calls the `SAM-Backend` (which calls `CU`).
    * **Module Settings Inspector:** A UI to view and edit `settings.yaml` files. **Crucial Security:** Any field containing `dsn`, `password`, `key`, or `token` will be **masked (`[********]`) by default**. An admin-only "Reveal" button will be required to see or edit these values.
    * **Config Push Button:** A "Save & Reload Module" button to push changes via the `CU`.
    * **Prompt Editor:** A user-friendly view for editing just the "persona" sections of config files.
7.  **🚀 Module Template Builder:** A wizard-style UI that calls the `SAM-Backend` (which calls `MTU`) to generate a downloadable `.zip` file with the skeleton code for a new SERA module.

---
#### 3.1.2. `SAM-CLI` (Command-Line Client)

* **Technology:** Python (`click`), lives in `sera_admin_module/cli/`.
* **Architecture:** A "power tool" client. It does *not* use the `SAM-Backend`. It bypasses the aggregator and makes gRPC calls *directly* to the specialized utility modules (`DIU`, `HIU`, `CU`, `HMU`, `MTU`).
* **Usage:**
    ```bash
    # It will consume the utility services directly
    sam-cli health                               # Calls HMU.GetSystemStatus
    sam-cli history list-sessions                # Calls HIU.ListSessions
    sam-cli history get-turn <turn_id> --logs    # Calls HIU.GetTurnDetails, gets trace_id, then queries Grafana/Loki
    sam-cli kb find-page "Netbox"                # Calls DIU.FindPageByTitle
    sam-cli feedback list --source cap           # Calls CAP.GetFeedback
    
    # New Config Commands
    sam-cli config list                          # Calls CU.ListRegisteredModules
    sam-cli config get cap                       # Calls CU.GetModuleConfig("cap")
    sam-cli config set cap llm_interface.model "gemma4:latest" # Calls CU.UpdateModuleConfig(...)
    sam-cli config reload cap                    # Calls CU.ReloadModuleConfig("cap")

    sam-cli module new <module-name>             # Calls MTU.GenerateModule
    ```

---

### 3.2. "Addon" Utility Modules (All \[NEW MODULES\])

These are all *separate, independent projects* that follow the SERA Module Blueprint (headless, no UI).

### 3.2.1. `HMU (Health Monitor Utility)` - \[NEW MODULE\]

* **Category:** (U) Utility
* **Purpose:** Provides real-time, centralized health status for the entire ecosystem.
* **Responsibilities:**
    * Runs as a persistent service.
    * Continuously pings the standard gRPC health check endpoint of *every* module.
    * Continuously monitors external dependencies (AI Factory LLM endpoints, HAProxy, PostgreSQL, etc.).
    * Exposes a single `GetSystemStatus()` gRPC endpoint.

### 3.2.2. `HIU (History Inspector Utility)` - \[NEW MODULE\]

* **Category:** (U) Utility
* **Purpose:** A robust gRPC service refactor of the `history_lookup.py` script.
* **Responsibilities:**
    * Provides a secure, read-only gRPC API to query the Orchestrator's PostgreSQL database.
    * Exposes endpoints like `ListSessions()`, `GetSessionDetails(session_id)`.
    * **CRITICAL API Feature:** The `GetTurnDetails(turn_id)` method **must** return the `trace_id` associated with that turn.

### 3.2.3. `CU (Configuration Utility)` - \[NEW MODULE\]

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

### 3.2.4. `MTU (Module Template Utility)` - \[NEW MODULE\]

* **Category:** (U) Utility
* **Purpose:** The secure backend for the "Module Template Builder."
* **Responsibilities:**
    * Provides a gRPC API for generating the skeleton code for a new SERA module.
    * `rpc GenerateModule(GenerateModuleRequest) returns (GenerateModuleResponse);`
    * The request will include parameters like `module_name` ("MyNewTool"), `category` ("Connector"), and `acronym` ("MNT").
    * Returns a base64-encoded `.zip` file with the complete module template.

## 4. Core Infrastructure & Module Requirements

To make SAM functional, the following changes **must** be implemented across the entire ecosystem.

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
    * **All gRPC Modules:** Must implement the standard [gRPC Health Checking Protocol](httpss://github.com/grpc/grpc/blob/master/doc/health-checking.md).

### 4.4. Module-Level Feedback Endpoints

* **Priority:** **MEDIUM**
* **Requirement:** The `SAM-Backend` needs a standard way to aggregate feedback.
* **Module Changes:**
    * **CAP, CAA (and future UI modules):** Must be refactored to expose new gRPC endpoints on their *existing* services:
        * `rpc GetFeedback(GetFeedbackRequest) returns (FeedbackListResponse);`
        * `rpc UpdateFeedback(UpdateFeedbackRequest) returns (FeedbackResponse);`

### 4.5. Standardized Config Reload Endpoint

* **Priority:** **MEDIUM**
* **Requirement:** The `CU` needs a way to notify modules that their `settings.yaml` file has changed.
* **Module Changes:**
    * **All Modules:** Must expose a new, standard gRPC endpoint (e.g., `rpc ReloadConfig(ReloadConfigRequest) returns (ReloadConfigResponse);`). When called, this triggers the module to re-load its `settings.yaml` file from disk into its running config object (without a full restart).

## 5. Phased Implementation Plan

### Phase 1: Foundation & Read-Only Inspection (Developer Value)

* **Goal:** Give developers immediate, powerful tools for debugging and monitoring.
* **Tasks:**
    1.  Build `HIU` (gRPC service) by refactoring `history_lookup.py`.
    2.  Build `HMU` (gRPC service) to monitor existing health endpoints.
    3.  Build the `SAM` module project, including:
        * `SAM-Backend` (gRPC service) with endpoints to call `DIU`, `HIU`, and `HMU`.
        * `SAM-WebUI` (React) with the first 3 pages: **Health**, **KB Inspector** (using `DIU`), and **History Inspector** (using `HIU`).
        * `SAM-CLI` (Python) with commands to call `DIU`, `HIU`, and `HMU` directly.
* **Value:** Replaces `history_lookup.py` and provides a single dashboard for system status.

### Phase 2: End-to-End Observability (Operator Value)

* **Goal:** Enable "magic" debugging with `trace_id`.
* **Tasks:**
    1.  **(The Hard Part)** Implement `trace_id` propagation across all critical modules (Orchestrator, RAA, NBC, CFR).
    2.  Deploy the Loki/Grafana stack.
    3.  Re-configure all modules to log to `stdout` and stream to Loki.
    4.  Add the **Log Viewer** (embedded Grafana) page to `SAM-WebUI`.
    5.  Enhance the **History Inspector** page: the `HIU` must return the `trace_id`, and the UI must add the "Debug this turn" button that opens Grafana pre-filtered to that `trace_id`.
* **Value:** Slashes debugging time from hours to seconds.

### Phase 3: Interactive Management (Administrator Value)

* **Goal:** Enable management and administration from the UI.
* **Tasks:**
    1.  Refactor `CAP` and `CAA` to add the `Get/UpdateFeedback` gRPC endpoints.
    2.  Implement the **`ReloadConfig()`** gRPC endpoint in all modules.
    3.  Build the `CU` (gRPC service) with its `Get/Update/Reload` logic.
    4.  Build the `MTU` (gRPC service) for generating new module templates.
    5.  Enhance `SAM-Backend` to call the new `CU`, `MTU`, and feedback endpoints.
    6.  Add the **Feedback Inbox**, **Configuration Manager**, and **Module Builder** pages to `SAM-WebUI`.
    7.  Expand `SAM-CLI` to cover all new management functions.
* **Value:** Fulfills the "self-service" goal, centralizes all administrative tasks, and accelerates new module development.