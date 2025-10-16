# /ai/assets/docs/tool_guides/diu_design.md
# Data Inspector Utility (DIU) - Design Document

* **Version:** 1.0
* **Date:** 2025-09-08
* **Author:** Martin Kecka, Gemini AI
* **Category:** Utility (U)
* **Acronym:** DIU (Data Inspector Utility)

---

## 1. Overview

The **Data Inspector Utility (DIU)** is a command-line tool and gRPC service designed for developers and administrators to query and inspect the contents of the SERA ecosystem's knowledge base. It provides a direct, low-level interface to the data stores populated by ingestors like the Confluence Ingestor (CFI).

The DIU allows for targeted lookups of pages, chunks, and their associated AI-generated enrichments (summaries, questions, analyses) directly from the **PostgreSQL** and **ChromaDB** databases. Its primary purpose is to serve as a crucial tool for *debugging*, *validation*, and *quality assurance* during the development and operation of data ingestion and retrieval pipelines.

This utility is a foundational component of the future **SERA Admin Suite**, a planned UI-driven module that will provide a centralized interface for monitoring and managing the health and content of all ecosystem modules.

---

## 2. Features

* **Client-Server Architecture:** Operates as a lightweight CLI client that communicates with a persistent gRPC server, ensuring a clean separation of concerns.

* **Multiple Lookup Methods:**
    * Find Confluence pages by `Page ID`, partial `Title`, or internal PostgreSQL `Source ID`.
    * Query and display the Confluence Category Hierarchy.
    * Look up Synthesized Q&A records by ID or a text query.

* **UI Discovery Endpoints:** Provides efficient, paginated endpoints to:
    * List all available **Space Keys**.
    * List all pages within a specific space, with support for server-side filtering.

* **Comprehensive Data Display:** Renders a rich, formatted view of all related data for a given page, including:
    * Core page metadata from **PostgreSQL**.
    * All associated page-level LLM summaries and artifacts.
    * All page-level generated questions (HyDE).
    * A detailed, paginated list of all associated chunks.

* **Detailed Chunk Inspection:** For each chunk, it displays:
    * Core chunk metadata from **PostgreSQL**.
    * LLM-generated chunk summaries.
    * LLM-generated table summaries.
    * Formatted semantic analysis (`llm_analysis`).
    * A list of associated images, links, and structured tables.
    * Metadata from the Vector DB (**ChromaDB**).
    * The raw, unprocessed text content.

* **Database Statistics:** Provides a global, space-by-space statistical summary of all content in the knowledge base, including counts of pages, chunks, words, and all AI enrichment types.

* **Automated Schema Generation:** Uses a central script to auto-generate Pydantic DTOs and Protobuf API contracts from the SQLAlchemy ORM models, ensuring data consistency across the application stack.

---

## 3. Architecture & Core Components

### Licensed by Google

The DIU is architected as a two-part application to ensure modularity and reusability.

* **gRPC Server (`data_inspector_utility.py serve`):** This is the core backend component. It runs as a persistent service and is solely responsible for:
    * Connecting to the shared **PostgreSQL** and **ChromaDB** instances.
    * Executing queries against the data stores.
    * Assembling the complete data payloads.
    * Exposing all functionality via a strongly-typed gRPC API defined in `diu.proto`.

* **CLI Client (`data_inspector_utility.py cli`):** This is the user-facing tool. It is a lightweight wrapper that:
    * Parses user commands and arguments.
    * Makes gRPC requests to the DIU server.
    * Receives data from the server.
    * Passes the data to the display module for rendering in the terminal.

* **Core Logic Modules (`core/`):**
    * `services.py`: The business logic layer. It orchestrates calls to the data readers and assembles the final data structures using Pydantic schemas.
    * `io_handlers/sql_reader.py`: Contains all SQLAlchemy logic for querying the **PostgreSQL** database.
    * `io_handlers/vdb_reader.py`: Contains all logic for querying the **ChromaDB** vector database.

* **Display Module (`cli/display.py`):** Contains all presentation logic. It uses the `rich` library to render data received from the CLI into formatted tables and panels. This module is completely decoupled from the data sources.

### Role in the SERA Ecosystem

The DIU is classified as a **Utility (U)** module. It is a self-contained service designed for a specific, focused capability—data inspection.

While its immediate interface is the CLI for developers, its primary long-term purpose is to serve as the dedicated backend for the future **SERA Admin Suite**. This suite will be a higher-level module with a web-based UI, acting as a centralized dashboard for operators. It will consume the DIU's gRPC API (including the `ListSpaces` and `ListPagesInSpace` endpoints) to build its user interface and will call other utility modules (like a log inspector or health check utility) to provide a holistic view of the entire SERA ecosystem's health and status.

---

## 4. Command-Line Interface (CLI) Usage

The DIU is run using `python3 data_inspector_utility.py cli [COMMAND] [ARGS]`.

```bash
# Run the gRPC server (usually as a background service)
python3 data_inspector_utility.py serve

# Get a global summary of all ingested content
python3 data_inspector_utility.py cli stats

# Look up a page by its Confluence Page ID
python3 data_inspector_utility.py cli page-id 1003159557

# Look up pages with "Netbox" in the title (max 5 pages, 10 chunks each)
python3 data_inspector_utility.py cli title "Netbox" --limit-pages 5 --limit-chunks 10

# Look up a page by its internal PostgreSQL source_metadata.id
python3 data_inspector_utility.py cli source-id 2015

# Display the full category hierarchy for the "SE" space
python3 data_inspector_utility.py cli category SE

# Display details for a specific category path
python3 data_inspector_utility.py cli category SE "Provisioning Guides > Automation"

# List all synthesized Q&A records
python3 data_inspector_utility.py cli sqna

# Look up a synthesized Q&A record by its ID
python3 data_inspector_utility.py cli sqna 440
```

### Available Commands

* `stats`: Display summary statistics of all ingested data.
* `page-id <ID>`: Look up a Confluence page by its ID.
* `title <QUERY>`: Look up pages with a title containing the query text.
* `source-id <ID>`: Look up a page by its internal PostgreSQL `source_metadata.id`.
* `category <SPACE_KEY> [PATH]`: Look up the category hierarchy for a space, or a specific category path.
* `sqna [ID | QUERY]`: Look up Synthesized Q&A. Provide an ID, a search term, or no argument to list all.

### Global Options

* `--limit-pages INT`: Max pages to return for a title search (default: 5).
* `--limit-chunks INT`: Max chunks to display per page (default: 10).

## 5. Future Enhancements

The DIU serves as the blueprint for other specialized administrative and debugging tools within the SERA ecosystem. The following modules are planned to be built using the same client-server architecture.

### History Inspector Utility (HIU)

* **Objective:** To refactor the standalone `/ai/tools/history_lookup/history_lookup.py` script into a dedicated, robust gRPC service and CLI client, mirroring the architecture of the DIU.
* **User Value 🌟:** Provides a standardized and secure way for developers and administrators to inspect the SERA Orchestrator's conversation history. This is critical for debugging agent behavior, auditing interactions, and analyzing user engagement patterns.
* **Implementation Strategy 🛠️:**
    1.  Create a new `history_inspector_utility` module.
    2.  Define a `hiu.proto` file with gRPC services for listing sessions, looking up specific sessions or turns, and retrieving history statistics.
    3.  Implement the gRPC server with `sql_reader` and `vdb_reader` modules that query the `conversation_sessions`, `history_turns`, and corresponding ChromaDB collections.
    4.  Build a CLI client (`hiu_cli.py`) that consumes the gRPC service to provide the same functionality as the original script.
    5.  This new utility will eventually be integrated into the **SERA Admin Suite** dashboard, allowing for UI-based exploration of user conversation histories.