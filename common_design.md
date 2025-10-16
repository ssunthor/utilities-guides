# SERA Common Library - Design Document

**Version:** 1.0
**Date:** 2025-10-16
**Author:** Martin Kecka, Gemini AI
**Status:** Draft

## 1. Overview & Rationale

As the SERA ecosystem expands, multiple modules (`cash_application_assistant`, `data_inspector_utility`, etc.) have begun to duplicate a significant amount of boilerplate and utility code. Core components such as logging setup, configuration loaders, LLM interfaces, and bootstrapping logic are being copied between projects. This violates the "Don't Repeat Yourself" (DRY) principle, increases the maintenance burden, and creates inconsistencies.

To address this, we will create a new, centralized Python package named `sera-common`. This project will serve as a shared library, providing a single, version-controlled source of truth for all cross-cutting concerns within the SERA ecosystem. Its adoption will simplify module development, enforce consistency, and make the entire platform more robust and maintainable.

## 2. Architectural Role & Scope

The `sera-common` project is strictly an installable library, not a runnable service. Its purpose is to be imported by other SERA modules.

### 2.1 What Belongs in sera-common

* **Shared Utilities:** Generic, reusable code that is not specific to any single module's business logic.
* **Common gRPC Contracts:** The `.proto` files for services that can be implemented by multiple modules (e.g., a common `FeedbackService`).
* **Generated gRPC Stubs:** The compiled Python stubs (`*_pb2.py`, `*_pb2_grpc.py`) for those common services. This is critical for providing a clean, importable API definition.
* **Abstract Base Classes:** Common interfaces or base classes that enforce a standard pattern across different modules.

### 2.2 What Does NOT Belong in sera-common

* **Application-Specific Logic:** Any business logic that is specific to a single module (e.g., `core/services.py` from CAA).
* **Runnable Services:** The library will not contain any entry points for running a web server, gRPC server, or UI (`rest_api/`, `grpc_api/service.py`, `ui/`).
* **Module-Specific Configuration:** The library will not have its own `config/settings.yaml`. It will be configured by the application that consumes it.
* **Deployment Scripts:** No `deploy.sh`, `Dockerfile`, or `systemd` files. The library is published as a package, not deployed as a service.
* **Module-Specific Prompts:** Prompt templates are business logic and belong within the modules that use them.

## 3. Proposed Directory Structure

The `sera-common` project will be structured as a standard, modern Python package to ensure it is easily installable via pip.

[bash]
/ai/sera-common/
├── .gitignore
├── pyproject.toml # Defines the installable 'sera_common' package and its dependencies.
├── README.md      # Documentation on how to install and use the library.
├── requirements.txt # Dependencies needed by the library itself (e.g., grpcio, sqlalchemy, PyYAML).
│
├── scripts/
│   └── generate_stubs.sh # Script to compile ALL common .proto files into the src/ directory.
│
└── src/ # Standard directory for source code in a Python package.
    └── sera_common/
        ├── __init__.py # Makes 'sera_common' a package.
        │
        ├── bootstrap_helper.py # The one, shared bootstrap helper.
        │
        ├── protos/ # The SOURCE .proto contracts for shared services.
        │   ├── __init__.py
        │   └── common/
        │       └── feedback_service.proto
        │
        ├── generated/ # The COMPILED Python stubs for shared services.
        │   ├── __init__.py
        │   └── common/
        │       ├── __init__.py
        │       ├── feedback_service_pb2.py
        │       └── feedback_service_pb2_grpc.py
        │
        └── utils/ # Collection of shared utility modules.
            ├── __init__.py
            ├── config_loader.py
            ├── logging_setup.py
            ├── transaction_logger.py
            └── llm_interface.py
[/bash]

**Key Architectural Decision:** The generated gRPC stubs are included inside the installable package. This is a deliberate choice to replace the fragile dependency on the shared `/ai/assets/protos` filesystem path with a robust, version-controlled Python import mechanism.

## 4. Integration & Refactoring Plan

### 4.1 Integration Workflow for Consuming Modules

Modules like `cash_application_assistant` will integrate the new library as follows:

**Installation (Local Development):**

The library will be installed in "editable" mode into each module's virtual environment. This creates a symlink, so changes to `sera-common` are immediately reflected.

[bash]
# From within a module's directory (e.g., /ai/cash_application_assistant/)
# Activate the virtual environment first
pip install -e ../sera-common
[/bash]

**Update requirements.txt:**

An entry will be added to each module's `requirements.txt` to make the dependency explicit.

[bash]
# /ai/cash_application_assistant/requirements.txt
...
-e ../sera-common
...
[/bash]

**Refactor Imports:**

All imports pointing to local utility files or the old shared assets path will be updated to import from the `sera_common` package.

**Old Way (Fragile):**
[python]
# In caa_main.py, this entire block is removed:
# stubs_root_path = os.path.abspath(os.path.join(generated_stubs_path, '..', '..', '..'))
# if stubs_root_path not in sys.path:
# sys.path.insert(0, stubs_root_path)

from utils.logging_setup import setup_logging
from protos.remittance_ingestor.generated import rmi_pb2 # This stays, it's module-specific
[/python]

**New Way (Robust):**
[python]
from sera_common.utils.logging_setup import setup_logging
from sera_common.generated.common import feedback_service_pb2_grpc # For a shared service
from protos.remittance_ingestor.generated import rmi_pb2 # For the module-specific service
[/python]

### 4.2 Phase 1 Refactoring Plan

The following files will be reviewed, generalized, and moved from their respective modules into the `sera-common` library. The original files in the modules will then be deleted and replaced with imports from `sera_common`.

* `bootstrap_helper.py`: A single, canonical version will be created in `sera_common`.
* `utils/logging_setup.py`: Centralize the multi-file logging standard (`app.log`, `app_err.log`).
* `utils/transaction_logger.py`: Centralize the `PipelineTransactionLogger` class to provide a consistent, structured, file-based logging mechanism for multi-step operations (`transaction.log`). This will standardize how modules audit complex workflows.
* `utils/config_loader.py`: Centralize the standard `settings.yaml` loader.
* `core/llm_interface.py`: Centralize the `OllamaClient` to ensure all modules interact with LLMs consistently. This file will be moved to `sera_common/utils/llm_interface.py`.

The developer responsible for this task will go through each module, perform the refactoring, and update the `requirements.txt` file accordingly.

## 5. Versioning & Deployment Strategy

This initiative will pave the way for a more robust production deployment workflow.

* **Development:** In the local/dev environment, changes to `sera-common` are made available instantly via the editable install. Consuming services simply need to be restarted (e.g., `sudo systemctl restart <service-name>`) to pick up the changes loaded into memory.
* **Production (Future State):**
    * The `sera-common` library will be strictly versioned (e.g., `1.0.0`, `1.1.0`) in its `pyproject.toml`.
    * Upon a push to the main branch, a CI/CD pipeline will test the library and publish a versioned package (`.whl` file) to a private artifact repository (e.g., GitLab Package Registry).
    * Consuming modules (like CAA) will pin their dependency to a specific, stable version in their `requirements.txt` (e.g., `sera-common==1.1.0`).
    * To update, a developer for CAA will consciously update this version number, triggering CAA's own CI/CD pipeline, which will build a new container with the updated dependency and deploy it.

This two-stage approach allows for rapid iteration during development while ensuring stability and predictability in production.