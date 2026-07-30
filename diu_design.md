# Data Inspector Utility (DIU) - Design Document

**Version:** 0.2.6
**Date:** 2026-07-12
**Author:** Martin Kecka & Gemini AI
**Category:** (U) Utility / Data Inspection
**Acronym:** DIU

---

## 1. Introduction & Purpose 📜

The **Data Inspector Utility (DIU)** is a command-line tool and gRPC microservice designed for developers and administrators to query and inspect the contents of the SERA ecosystem's knowledge base. It provides a direct, low-level interface to the data stores populated by ingestors like the Confluence Ingestor (CFI).

The DIU allows for targeted lookups of pages, chunks, and their associated AI-generated enrichments (summaries, questions, analyses) directly from the **PostgreSQL** and **ChromaDB** databases. Its primary purpose is to serve as a crucial tool for *debugging*, *validation*, and *quality assurance* during the development and operation of data ingestion and retrieval pipelines.

This utility is a foundational component of the future **Sera Admin Module (SAM)**, a planned UI-driven module that will provide a centralized interface for monitoring and managing the health and content of all ecosystem modules.

---

## 2. Core Architectural Principles 🏗️

* **Client-Server Architecture:** Operates as a lightweight CLI client that communicates with a persistent gRPC server, ensuring a clean separation of concerns.
* **Read-Only Aggregation:** DIU never writes to the knowledge base. It is strictly an inspection utility designed to join relational metadata with vector embeddings without mutating the underlying data.
* **Strongly-Typed Schemas:** Uses automated script generation to convert SQLAlchemy ORM models into Pydantic DTOs and Protobuf messages, guaranteeing data structure consistency.
* **Dual-Database Coordination:** Synthesizes results by querying structured metadata from PostgreSQL and high-dimensional semantic data from ChromaDB/Qdrant in parallel.
* **Ecosystem Compliance:** DIU fully implements `sera_standard_service.py` to seamlessly participate in AMU GitOps tracking, DeepHealth discovery, and multi-service topology routing.

---

## 3. Database Schema & State Management 🗄️

DIU does not possess its own mutable database; rather, it attaches to the centralized ecosystem knowledge stores to provide unified views.

### 3.1. Relational Metadata (PostgreSQL)
DIU leverages `sql_reader.py` and `database/schemas.py` to query structured ingestion metadata natively from the ecosystem `_utility_db`:
* **Confluence Spaces & Pages:** Source IDs, Titles, and Category Hierarchies.
* **Chunk Metadata:** Page associations, tokens, and chunk indexes.
* **AI Enrichments:** LLM-generated summaries, table structures, and synthesized questions (HyDE data).

### 3.2. Vector Storage (ChromaDB / Qdrant)
DIU maps relational records to their mathematical embeddings via `vdb_reader.py` connected to the `_utility_vdb`:
* Performs raw similarity searches directly against the vector spaces to validate ingestion quality, allowing administrators to visually inspect the exact payload returned by standard vector lookups before RAG reranking algorithms apply.

---

## 4. Core Sub-Systems & Responsibilities (Core Features) ⚙️

* **Multiple Lookup Methods:**
    * Find Confluence pages by `Page ID`, partial `Title` (with global `search_text` chunk filtering), or internal PostgreSQL `Source ID`.
    * Query and display the Confluence Category Hierarchy, or recursively assemble the entire Confluence knowledge tree in memory via `get_directory_tree`.
    * Look up detailed structured table JSON content, metadata, and LLM table summaries directly by Table ID via the `LookupTable` RPC.
    * Look up Synthesized Q&A records by ID or text query, or embed Multi-Page Synthesized Q&As natively into page metadata payloads.

* **UI Discovery Endpoints:** Provides efficient, paginated endpoints to:
    * List all available **Space Keys**.
    * List all pages within a specific space, with support for server-side filtering.
    * Generate a full nested tree directory representation of a Confluence Space.

* **Comprehensive Data Display:** Renders a rich, formatted view of all related data for a given page, including:
    * Core page metadata from **PostgreSQL**.
    * All associated page-level LLM summaries and artifacts.
    * All page-level generated questions (HyDE) and embedded `SynthesizedQnA` records.
    * A detailed, paginated list of all associated chunks.
    * **Dynamic Sub-Chunk Expansion:** Supports recursive `--expand` chunk filtering: intelligently matches target chunks and extracts all associated sub-chunks based on heading hierarchy.

* **Detailed Chunk Inspection:** For each chunk, it displays:
    * Core chunk metadata from **PostgreSQL**.
    * LLM-generated chunk summaries and table summaries.
    * Formatted semantic analysis (`llm_analysis`).
    * A list of associated images, links, and structured tables.
    * Metadata from the Vector DB (**ChromaDB**).
    * The raw, unprocessed text content.

* **Database Statistics:** Provides a global, space-by-space statistical summary of all content in the knowledge base, including counts of pages, chunks, words, AI enrichment types, and the latest synchronization timestamp (`last_run_at`).

---

## 5. Module Structure & Core Components 📂

The DIU is architected as a two-part application to ensure modularity and reusability, closely mirroring ecosystem standards.

* **gRPC Server (`data_inspector_utility.py start-grpc-server`):** This is the core backend component. It runs as a persistent service (`diu-grpc.service`) and is solely responsible for:
    * Connecting to the shared **PostgreSQL** and **ChromaDB** instances.
    * Executing complex SQLAlchemy queries and vector searches.
    * Assembling the complete data payloads.
    * Exposing all functionality via a strongly-typed gRPC API defined in `grpc_api/protos/diu.proto` alongside the `sera_standard.proto`.

* **CLI Client (`data_inspector_utility.py <domain_command>`):** This is the user-facing tool. It is a lightweight wrapper that:
    * Parses user commands using the `Click` framework, separated by domain (`spaces`, `pages`, `stats`, `sqna`, `tables`).
    * Makes gRPC requests to the DIU server via `core/diu_client.py` and natively injects stubs via ecosystem decorators.
    * Passes the data to the display module (`cli/formatters/display.py`) for rendering in the terminal using `rich`.

* **Core Logic Modules (`core/`):**
    * `services.py`: The business logic layer. It orchestrates calls to the data readers and assembles the final data structures using Pydantic schemas.
    * `io_handlers/sql_reader.py`: Contains all SQLAlchemy logic for querying the **PostgreSQL** database.
    * `io_handlers/vdb_reader.py`: Contains all logic for querying the **ChromaDB** vector database.

* **Configuration (`config/`):**
    * Relies purely on a standardized `settings.yaml` structure, separating API/gRPC configs into `system_services` and external database integrations into `external_services` for standard deep health tracking.

---

## 6. Command-Line Interface (CLI) Usage 💻

The DIU CLI has been modularized. Run `python3 data_inspector_utility.py [COMMAND] --help` for full details.

```bash
# Run the gRPC server (usually runs in background as diu-grpc.service)
python3 data_inspector_utility.py start-grpc-server

# Get a global summary of all ingested content statistics (with Last Run timestamp)
python3 data_inspector_utility.py stats show

# List all ingested spaces
python3 data_inspector_utility.py spaces list

# Display the full nested Confluence knowledge tree for a space
python3 data_inspector_utility.py spaces directory "SE"

# Search for pages and dynamically expand associated sub-chunks via heading hierarchy
python3 data_inspector_utility.py pages search --title "SD-WAN" --search-text "FortiManager" --expand

# Output the raw unformatted JSON response directly from the gRPC layer
python3 data_inspector_utility.py pages search --title "SD-WAN" --grpc

# Lookup structured table JSON content by Table ID
python3 data_inspector_utility.py tables show 1059

# List all synthesized Q&A records
python3 data_inspector_utility.py sqna search
```

---

## 7. Roadmap & Future Enhancements 🚀

* **SERA Admin Module (SAM) Integration:** Complete the frontend dashboard to natively consume DIU's endpoints, transforming raw database records into administrative visual tools.
    * **Knowledge Tree Navigation (UI):** The backend `get_directory_tree` RPC is now fully implemented to assemble nested hierarchies in memory. The SAM UI must now be updated to consume this endpoint to build a visual directory browser, allowing administrators to inspect chunk boundaries without loading full documents.
    * **A/B Retrieval Diagnostics:** Expose a "Raw Vector Query" endpoint to execute pure semantic searches directly against ChromaDB. This will power a side-by-side diagnostic UI in SAM, comparing raw ingestion quality (DIU) against the filtered, reranked output of the production RAG pipeline (CFR).
* **History Inspector Utility (HIU) Consolidation:** Refactor the standalone `/ai/tools/history_lookup/history_lookup.py` script into a dedicated HIU microservice, mirroring the architecture and Click-based CLI of the DIU.