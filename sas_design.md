# Sera Admin Suite (SAS) - Design Document

**Version:** 1.1
**Date:** 2025-11-01
**Author:** Martin Kecka & Gemini AI
**Category:** (U) Utility / Administrative

## 1. Introduction & Purpose

The **Sera Admin Suite (SAS)** is the centralized, modular platform for monitoring, debugging, managing, and operating the entire SERA AI ecosystem.

The primary goal of SAS is to replace all disparate administrative tools (e.g., `history_lookup.py`, `systemd` log checks, direct database queries) with a single, unified, and secure interface. It provides a "single pane of glass" for developers, operators, and administrators to observe system health, debug user interactions, inspect data, and manage platform configurations.

SAS is composed of two primary clients and a collection of new and existing backend utility services:

1.  **Clients (The "Face"):**
    * **`SAS-WebUI`:** A modern React-based web application (SPA) providing a rich, graphical interface.
    * **`SAS-CLI`:** A powerful Python-based Command Line Interface (CLI) for "power users" and scripting. Both clients are part of the SAS.

2.  **Backends (The "Brains"):**
    * A set of specialized, single-responsibility gRPC utility services that provide data to the clients.

## 2. Core Architectural Principles

SAS is built on a "headless backend" and "aggregator" microservice model.

1.  **Aggregator Pattern:** The `SAS-WebUI` (React) is a "dumb" client. It communicates with **only one** backend service: the new **`ADS (Admin Service)`**. This `ADS` service acts as an *aggregator*, fanning out gRPC requests to all other specialized utility backends (like `DIU`, `HIU`, etc.) and collecting the results. This simplifies the frontend, centralizes UI-facing logic, and reduces client-side complexity.

2.  **Specialized Utility Backends:** All administrative functionality is encapsulated in discrete, single-responsibility gRPC services (e.g., `DIU` for data, `HIU` for history). These are the *only* components with direct access to databases or log streams.

3.  **Decoupled Clients:** The `SAS-WebUI` and `SAS-CLI` are both clients of the same gRPC services. Any feature available in the UI can also be accessed via the CLI, ensuring parity and scriptability.

4.  **Observability via `trace_id`:** The entire ecosystem will be built around a **Distributed Tracing** model. Every user request generates a `trace_id` which is propagated through all gRPC calls. This allows SAS to provide a complete, end-to-end log of a single user turn, from the Orchestrator to the RAA to the NBC and back.

5.  **Centralized Logging (Loki):** All modules will stop logging to local files (`/ai/assets/logs/`). They will be re-configured to stream logs to a central **Loki** instance, making them queryable by `trace_id`.

## 3. System Components

### 3.1. `SAS-WebUI` (The Frontend) - \[NEW MODULE\]

* **Technology:** React (Vite, shadcn/ui or Material-UI), served by the Sera AI Gateway.
* **Authentication:** Relies on HTTP headers (`X-Forwarded-User`, `X-Forwarded-Groups`) injected by the Gateway.
* **Backend:** Communicates *exclusively* with the `ADS (Admin Service)` via gRPC-Web.

#### Key UI Pages:

1.  **🌍 Ecosystem Health Dashboard:** The landing page. Displays the real-time status of all modules and external dependencies (LLM endpoints, HAProxy, databases) by consuming data from the `HMU`.
2.  **🔍 Centralized Log Viewer:** An embedded **Grafana** dashboard configured to query the **Loki** database. The primary feature is a search bar for `trace_id`.
3.  **💬 Conversation History Inspector:** A UI for the `HIU`. Lists all user sessions and turns. Includes a "Debug this turn" button that opens the Log Viewer with the turn's `trace_id` pre-filled.
4.  **📚 Knowledge Base Inspector:** A UI for the `DIU`. Allows browsing Confluence spaces, pages, and chunks, and inspecting their metadata and generated content (summaries, HyDE questions, etc.).
5.  **📥 Centralized Feedback Inbox:** A single inbox that aggregates feedback items from *all* modules (CAP, CAA, etc.). Allows admins to review, update status, and assign feedback.
6.  **🛠️ Prompt & Config Editor:** A "self-service" UI for managing system configurations (e.g., editing persona prompts, updating tool definitions) by consuming the `CU`.
7.  **🚀 Module Template Builder:** A wizard-style UI that calls the `MTU` to generate a downloadable `.zip` file with the skeleton code for a new SERA module (gRPC server, Dockerfile, etc.).

### 3.2. `ADS (Admin Service)` (The Backend Aggregator) - \[NEW MODULE\]

* **Category:** (U) Utility
* **Purpose:** The single, unified gRPC backend for the `SAS-WebUI`.
* **Responsibilities:**
    * **Health Aggregation:** Exposes a `GetSystemHealth()` gRPC method that calls the `HMU` and formats the response for the UI.
    * **Feedback Aggregation:** Exposes `GetAggregatedFeedback()` and `UpdateFeedback()` methods.
        * `GetAggregatedFeedback()` calls `CAP.GetFeedback()`, `CAA.GetFeedback()`, etc., in parallel and merges the results into a single list.
        * `UpdateFeedback()` inspects the request (e.g., `feedback_id: "cap-123"`), determines the source module, and routes the gRPC call to the correct module (e.g., `CAP.UpdateFeedback()`).
    * **Proxy/Wrapper:** Provides simple gRPC proxy methods for `DIU`, `HIU`, `CU`, and `MTU`. This simplifies the `SAS-WebUI`'s logic, as it only needs to manage one gRPC client connection (`ADS`).

### 3.3. `HMU (Health Monitor Utility)` - \[NEW MODULE\]

* **Category:** (U) Utility
* **Purpose:** Provides real-time, centralized health status for the entire ecosystem.
* **Responsibilities:**
    * Runs as a persistent service.
    * Continuously pings the standard gRPC health check endpoint of *every* module (Orchestrator, CFR, NBC, RAA, CAP, CAA, etc.).
    * Continuously monitors external dependencies (AI Factory LLM endpoints, HAProxy, PostgreSQL, ChromaDB, Redis).
    * Caches all statuses in memory.
    * Exposes a single `GetSystemStatus()` gRPC endpoint for the `ADS` to consume.

### 3.4. `HIU (History Inspector Utility)` - \[NEW MODULE\]

* **Category:** (U) Utility
* **Purpose:** A robust gRPC service refactor of the `history_lookup.py` script.
* **Responsibilities:**
    * Provides a secure, read-only gRPC API to query the Orchestrator's PostgreSQL database (`conversation_sessions`, `history_turns`, `history_turn_summaries`).
    * Exposes endpoints like `ListSessions()`, `GetSessionDetails(session_id)`.
    * **CRITICAL API Feature:** The `GetTurnDetails(turn_id)` method **must** return the `trace_id` associated with that turn.

### 3.5. `CU (Configuration Utility)` - \[NEW MODULE\]

* **Category:** (U) Utility
* **Purpose:** The secure backend for the "Self-Service Prompt Editor."
* **Responsibilities:**
    * Provides a gRPC API for CRUD (Create, Read, Update, Delete) operations on system configurations.
    * This includes: Persona prompts, tool definitions (`tools.d/`), and other key settings from `settings.yaml` files.
    * **Note:** This module will interact with a new centralized configuration store (e.g., a dedicated PostgreSQL table or a Git repository) rather than editing live `.yaml` files on the filesystem, which is a significant security risk.

### 3.6. `SAS-CLI` (The Command-Line Client) - \[NEW MODULE\]

* **Technology:** Python (`click` or `argparse`).
* **Purpose:** A comprehensive CLI tool for developers and power users, acting as the second client for the SAS.
* **Architecture:** It is a *client*, just like the React UI. It will *not* contain any business logic. It will consume the gRPC backends (`ADS`, `HMU`, `HIU`, `DIU`, `CU`, `MTU`) directly.
* **Usage:**
    ```
    # It will consume the new utility services directly
    sas-cli health                               # Calls HMU.GetSystemStatus
    sas-cli history list-sessions                # Calls HIU.ListSessions
    sas-cli history get-turn <turn_id> --logs    # Calls HIU.GetTurnDetails, gets trace_id, then queries Grafana/Loki
    sas-cli kb find-page "Netbox"                # Calls DIU.FindPageByTitle
    sas-cli feedback list --source cap           # Calls ADS.GetAggDregatedFeedback
    sas-cli prompt get general_assistant         # Calls CU.GetPromptTemplate
    sas-cli module new <module-name>             # Calls MTU.GenerateModule
    ```

### 3.7. `MTU (Module Template Utility)` - \[NEW MODULE\]

* **Category:** (U) Utility
* **Purpose:** The secure backend for the "Module Template Builder."
* **Responsibilities:**
    * Provides a gRPC API for generating the skeleton code for a new SERA module.
    * `rpc GenerateModule(GenerateModuleRequest) returns (GenerateModuleResponse);`
    * The request will include parameters like `module_name` ("MyNewTool"), `category` ("Connector"), and `acronym` ("MNT").
    * The service will generate a complete, best-practice module template (including `Dockerfile`, `requirements.txt`, `config/settings.yaml`, gRPC server template, `README.md`, etc.) and return it as a base64-encoded `.zip` file in the response.

## 4. Core Infrastructure & Module Requirements

To make SAS functional, the following changes **must** be implemented across the entire ecosystem.

### 4.1. Distributed Tracing (`trace_id`)

* **Priority:** **CRITICAL**
* **Requirement:** A unique `trace_id` must be generated for every user interaction and propagated to all downstream services for end-to-end logging.
* **Module Changes:**
    * **Sera Orchestrator:** Must be updated to *generate* a `trace_id` at the beginning of `process_query`.
    * **All Modules (Orchestrator, RAA, NBC, CFR, etc.):** Must be refactored to:
        1.  Accept a `trace_id` via gRPC metadata in all incoming calls.
        2.  Include this `trace_id` in *every single log message* they generate.
        3.  Pass this `trace_id` along in any *outgoing* gRPC calls they make to other services.

### 4.2. Centralized Logging (Loki Stack)

* **Priority:** **HIGH**
* **Requirement:** All logs must be streamed to a central, queryable database instead of being written to local files.
* **Module Changes:**
    * **All Modules:** Must be re-configured to stop logging to files (e.g., `/ai/assets/logs/`) and log *only* to `stdout` (standard output).
* **New Infrastructure:**
    * **Loki:** A Loki instance must be deployed to act as the central log database.
    * **Promtail (or similar):** A log-shipping agent must be deployed to scrape `stdout` from all running module containers and ship the logs to Loki.
    * **Grafana:** A Grafana instance must be deployed and configured with Loki as a data source. This is what the `SAS-WebUI` will embed.

### 4.3. Standardized Health Checks

* **Priority:** **HIGH**
* **Requirement:** The `HMU` needs a reliable, standard way to ping every service.
* **Module Changes:**
    * **All gRPC Modules:** Must implement the standard [gRPC Health Checking Protocol](https://github.com/grpc/grpc/blob/master/doc/health-checking.md). This is a trivial, built-in gRPC feature to add.
    * **External Dependencies:** `HMU` will be configured to ping HTTP endpoints for HAProxy, LLM services, etc.

### 4.4. Module-Level Feedback Endpoints

* **Priority:** **MEDIUM**
* **Requirement:** The `ADS` needs a standard way to aggregate feedback from all modules that have a UI.
* **Module Changes:**
    * **CAP, CAA (and future UI modules):** Must be refactored to expose new gRPC endpoints on their *existing* services:
        * `rpc GetFeedback(GetFeedbackRequest) returns (FeedbackListResponse);`
        * `rpc UpdateFeedback(UpdateFeedbackRequest) returns (FeedbackResponse);`

## 5. Phased Implementation Plan

SAS is a major initiative that should be delivered in logical, value-adding phases.

### Phase 1: Foundation & Read-Only Inspection (Developer Value)

* **Goal:** Give developers immediate, powerful tools for debugging and monitoring.
* **Tasks:**
    1.  Build `HIU` (gRPC service) by refactoring `history_lookup.py`.
    2.  Build `HMU` (gRPC service) to monitor existing health endpoints.
    3.  Build `ADS` (gRPC service) with endpoints to call `DIU`, `HIU`, and `HMU`.
    4.  Build the `SAS-WebUI` (React) with the first 3 pages: **Health**, **KB Inspector** (using `DIU`), and **History Inspector** (using `HIU`).
    5.  Begin work on the `SAS-CLI`, targeting the same backends.
* **Value:** Replaces `history_lookup.py` and provides a single dashboard for system status.

### Phase 2: End-to-End Observability (Operator Value)

* **Goal:** Enable "magic" debugging with `trace_id`.
* **Tasks:**
    1.  **(The Hard Part)** Implement `trace_id` propagation across all critical modules (Orchestrator, RAA, NBC, CFR).
    2.  Deploy the Loki/Grafana stack.
    3.  Re-configure all modules to log to `stdout` and stream to Loki.
    4.  Add the **Log Viewer** (embedded Grafana) page to `SAS-WebUI`.
    5.  Enhance the **History Inspector** page: the `HIU` must return the `trace_id`, and the UI must add the "Debug this turn" button that opens Grafana pre-filtered to that `trace_id`.
* **Value:** Slashes debugging time from hours to seconds.

### Phase 3: Interactive Management (Administrator Value)

* **Goal:** Enable management and administration from the UI.
* **Tasks:**
    1.  Refactor `CAP` and `CAA` to add the `Get/UpdateFeedback` gRPC endpoints.
    2.  Add the **Feedback Inbox** page to `SAS-WebUI` (consuming `ADS`).
    3.  Design and build the `CU` (gRPC service) and its new database schema for storing configs.
    4.  Add the **Prompt & Config Editor** page to `SAS-WebUI`.
    5.  Build the `MTU` (gRPC service) for generating new module templates.
    6.  Add the **Module Builder** page to `SAS-WebUI`.
    7.  Expand `SAS-CLI` to cover all new management functions.
* **Value:** Fulfills the "self-service" goal, centralizes all administrative tasks, and accelerates new module development.