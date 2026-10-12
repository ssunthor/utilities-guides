# Data Inspector Utility (DIU) - Architecture Document

**Doc_Type:** Architecture
**Category:** (U) Utility
**Name:** Data Inspector Utility
**Acronym:** DIU
**Version:** 0.2.7
**Date:** 2026-10-12
**Author:** Martin Kecka & Gemini AI
**Status:** Active

---

## 1. Introduction & Purpose 📜

The **Data Inspector Utility (DIU)** is a dedicated administrative microservice and CLI inspection engine designed for developers, engineers, and administrators to query, inspect, and audit the contents of the SERA/GIA ecosystem's knowledge base. It provides a direct, low-level read interface to the relational and vector data stores populated by upstream ingestion pipelines such as the Confluence Ingestor (CFI).

Operating as an autonomous, fully functional application on port `50109`, DIU provides developers and operators with a rich Click CLI (enhanced with `rich` terminal rendering) for debugging, schema validation, and RAG pipeline quality assurance. Concurrently, it serves as the specialized knowledge-base inspection backend powering the **Sera Admin Module (SAM)** web interface. Within the ecosystem's administrative utility suite (AMU, CMU, DIU, HMU, OMU), DIU delivers direct visibility into ingested knowledge assets and AI enrichments without requiring direct database access.

DIU enables targeted lookups and hierarchical exploration of ingested Confluence pages, document chunks, structured tables, category directory trees, and AI-generated enrichments (summaries, HyDE questions, semantic analyses, multi-page synthesized Q&As, and vision metadata) across **PostgreSQL** (`knowledge_base_db`) and **ChromaDB**.

---

## 2. Architecture & Ecosystem Integration 🏗️

DIU operates on a client-server architecture, exposing dual gRPC servicers on port `50109` (`diu-grpc.service`) backed by a multi-worker thread pool, while providing operators with a standalone Click CLI and dynamic schema synchronization engines.

### 2.1. Dual Servicer & Core Architectural Principles
- **Dual Servicer Model:**
  1. **`DataInspectorUtility` (`diu.proto`):** Domain servicer providing targeted page and chunk lookups, structured table inspection, recursive category tree assembly, synthesized Q&A queries, and multi-dimensional database statistics.
  2. **`SeraAppService` (`sera_standard.proto`):** Implements the universal SERA ecosystem contract (`v1.2-universal`, maintained centrally by AMU) for self-reporting DIU's metadata, deep health matrix, documentation, and changelog.
- **Client-Server Decoupling:** The CLI never touches the database directly. All operator commands and frontend queries route through gRPC stubs with automatic `x-client-id` metadata injection (`[DIU]`, `[SAM]`).
- **Read-Only Aggregation:** Strictly an inspection engine; runtime operations never mutate underlying relational or vector knowledge stores, ensuring zero risk of accidental corruption during diagnostic exploration.
- **SAM Knowledge-Base Backend:** Powers the knowledge-base inspection widgets, table exploration panels, and RAG pipeline diagnostic views within the **Sera Admin Module (SAM)** UI.
- **Dynamic Schema Synchronization:** Uses automated AST/ORM introspection (`core/schema_generator.py`) against `confluence_ingestor.core.database.models` to generate strongly-typed Pydantic DTOs (`core/database/schemas.py`) and compiles Protobuf definitions on startup.
- **Dual-Database Coordination:** Synthesizes unified views by joining relational metadata from PostgreSQL (`knowledge_base_db`) with high-dimensional chunk metadata from ChromaDB (`confluence_store` and related collections).

### 2.2. Ecosystem Integration & Data Flow

```mermaid
graph TD
    subgraph Clients ["Clients, Operators & Control Plane"]
        SAM["Sera Admin Module (SAM UI)"]
        CLI["DIU CLI (Click + Rich)"]
        AMU["App Management Utility (AMU)"]
    end

    subgraph DIU_Core ["Data Inspector Utility (DIU :50109)"]
        Interceptor["MetadataClientInterceptor (x-client-id)"]
        GRPC_DIU["DataInspectorUtilityServicer"]
        GRPC_SERA["SeraStandardServiceImpl (v1.2)"]
        ServiceLayer["Core Services Layer (core/services.py)"]
        SQLReader["SQLReader (core/io_handlers/sql_reader.py)"]
        VDBReader["VDBReader (core/io_handlers/vdb_reader.py)"]
    end

    subgraph Storage ["Shared Knowledge Base Stores"]
        PG[("PostgreSQL (knowledge_base_db)")]
        Chroma[("ChromaDB (:8000)")]
    end

    subgraph External_Models ["Upstream Shared Code"]
        CFI["Confluence Ingestor ORM Models"]
    end

    CLI -->|gRPC Unary / Stream| Interceptor
    SAM -->|gRPC Unary / Stream| Interceptor
    AMU -->|GetAppInfo / GetDeepHealth| GRPC_SERA

    Interceptor --> GRPC_DIU
    Interceptor --> GRPC_SERA

    GRPC_DIU --> ServiceLayer
    ServiceLayer --> SQLReader
    ServiceLayer --> VDBReader

    SQLReader -->|SQLAlchemy QueuePool| PG
    VDBReader -->|ChromaVDB HTTP Client| Chroma
    CFI -.->|Dynamic Import & Schema Gen| SQLReader
```

---

## 3. Module File Map 📂

```plaintext
/ai/utilities/data_inspector_utility
├── bootstrap_helper.py                  # Virtual environment bootstrapper and dependency verifier
├── data_inspector_utility.py            # Main application entrypoint; loads YAML config, sets up logging, invokes Click CLI
├── requirements.txt                     # Python package dependencies (grpcio, sqlalchemy, chromadb-client, rich, click, etc.)
├── README.md                            # Repository overview and links to central SERA guides
├── output.txt                           # Scratch/debug output file
├── cli/
│   ├── __init__.py                      # CLI package marker
│   ├── diu_cli.py                       # Root Click group (NestedHelpGroup), stub injection, and `start-grpc-server` command
│   ├── utils.py                         # gRPC channel builders, `MetadataClientInterceptor`, JSON printer, and `@handle_grpc_errors`
│   ├── commands/
│   │   ├── __init__.py                  # Commands package marker
│   │   ├── debug_cmds.py                # `debug store`: Dumps full page & chunk details (limit 10,000) to JSON file
│   │   ├── module_cmds.py               # `module info` & `module check-health`: Queries SERA Standard RPCs
│   │   ├── pages_cmds.py                # `pages list`, `pages get`, `pages search`: Page/chunk inspection and text filtering
│   │   ├── spaces_cmds.py               # `spaces list`, `spaces hierarchy`, `spaces directory`: Space & tree lookups
│   │   ├── sqna_cmds.py                 # `sqna search`: Multi-page Synthesized Q&A lookups
│   │   ├── stats_cmds.py                # `stats show`: Global and per-space ingestion statistics table
│   │   └── tables_cmds.py               # `tables show`: Direct structured table lookup by Table ID
│   └── formatters/
│       └── display.py                   # Rich terminal rendering logic (tables, panels, syntax-highlighted JSON)
├── config/
│   ├── __init__.py                      # Config package marker
│   └── settings.yaml                    # Master runtime configuration (ports, DSNs, VDB collections, paths, logging)
├── core/
│   ├── __init__.py                      # Core package marker
│   ├── schema_generator.py              # Introspects CFI SQLAlchemy models to auto-generate `core/database/schemas.py`
│   ├── services.py                      # Business logic layer; orchestrates SQL + VDB readers and populates Pydantic DTOs
│   ├── test_diu.py                      # Standalone async diagnostic script for testing table/asset payloads
│   ├── database/
│   │   ├── database.py                  # SQLAlchemy read/write `QueuePool` engine initialization and `get_db_session()` context manager
│   │   └── schemas.py                   # Auto-generated & static Pydantic V2 DTOs (`from_attributes=True`)
│   └── io_handlers/
│       ├── __init__.py                  # IO handlers package marker
│       ├── sql_reader.py                # `SQLReader` class containing all SQLAlchemy 2.0 queries against PostgreSQL
│       ├── sql_writer.py                # `SQLWriter` placeholder class for SERA ecosystem structural compliance
│       └── vdb_reader.py                # ChromaDB client loader and batch chunk metadata fetcher
├── database/
│   └── __init__.py                      # Legacy package marker
├── docs/
│   ├── CHANGELOG.yaml                   # Structured release history parsed by AMU
│   └── diu_design.md                    # Symlink -> /ai/assets/guides/utilities_guides/diu_design.md
├── grpc_api/
│   ├── __init__.py                      # gRPC API package marker
│   ├── diu_service.py                   # `DataInspectorUtilityServicer` implementation and Protobuf/RFC-3339 converters
│   ├── generate_stubs.py                # Compiles `diu.proto` and `sera_standard.proto` into shared generated stubs directory
│   ├── sera_standard_service.py         # `SeraStandardServiceImpl` (v1.2-universal) for AppInfo, DeepHealth, Docs, Changelog
│   ├── services.py                      # gRPC server bootstrapper, thread pool setup, health servicer, and server reflection
│   └── protos/
│       └── diu.proto                    # Protobuf service and message definitions for `diu` package
├── logs/                                # Local rotating log storage directory
└── utils/
    ├── __init__.py                      # Utils package marker
    └── logging_setup.py                 # `MultiProcessSafeDailyLogger` and mode-aware console/file logging setup
```

---

## 4. Core Processing Lifecycle ⚙️

### 4.1. Boot & Server Initialization Sequence (`start-grpc-server`)
1. **Environment Bootstrap:** `data_inspector_utility.py` inserts the local script directory and `/ai` root into `sys.path`, then invokes `bootstrap_helper.bootstrap_and_reexecute_if_needed()` to ensure execution within the module's virtual environment.
2. **Configuration & Logging:** Loads `config/settings.yaml` and initializes `setup_logging(config)`. Because `start-grpc-server` is present in `sys.argv`, console logging is set to the configured level (`INFO`), and daily rotating file handlers (`diu.log.YYYY-MM-DD`, `diu_err.log.YYYY-MM-DD`) are attached.
3. **Dynamic Schema Generation:** `diu_cli.py` executes `core/schema_generator.py` via `subprocess.run()`. The generator imports `SourceMetadata` and `SourceChunk` from `confluence_ingestor.core.database.models`, inspects their SQLAlchemy columns, combines them with static relationship schemas, and writes `core/database/schemas.py`. If generation fails, startup aborts with exit code `1`.
4. **gRPC Stub Compilation & Process Re-exec:** Calls `check_and_generate_stubs(config)` in `grpc_api/generate_stubs.py`. It compares the modification timestamps (`mtime`) of `diu.proto` and `sera_standard.proto` against the compiled `_pb2.py` files in `/ai/assets/protos/data_inspector_utility/generated`.
   - If missing or outdated, `grpc_tools.protoc` compiles both `.proto` files and creates `__init__.py` markers across the target path.
   - If stubs were regenerated (`regenerated == True`), the process immediately replaces itself via `os.execv(sys.executable, [sys.executable, "-u"] + sys.argv)` to guarantee fresh module imports.
5. **Database Pool Initialization:** Calls `initialize_database(config)` in `core/database/database.py`, creating separate `_read_engine` and `_write_engine` instances using SQLAlchemy `QueuePool` (`pool_size=5`, `max_overflow=10`, `pool_timeout=30`, `pool_recycle=1800`, `pool_pre_ping=True`). Executes a validation `SELECT 1` inside a `get_db_session()` context manager.
6. **gRPC Server Binding:** `grpc_api.services.serve(config)` initializes a `grpc.server` with a `ThreadPoolExecutor(max_workers=10)`, registers `DataInspectorUtilityServicer`, `SeraStandardServiceImpl`, standard gRPC `HealthServicer`, and `reflection`, binds to `0.0.0.0:50109`, and blocks on `server.wait_for_termination()`.

### 4.2. Runtime Request Lifecycle (CLI / Client -> Server)
1. **CLI Invocation:** When a user runs a command (e.g., `pages get 12345`), `setup_logging` sets console output to `WARNING` to keep terminal output clean. `diu_cli.cli` injects `stub` (`DataInspectorUtilityStub`) and `sera_stub` (`SeraAppServiceStub`) into `ctx.obj`.
2. **Metadata Interception:** Outbound unary-unary and unary-stream calls pass through `MetadataClientInterceptor`, which appends `('x-client-id', 'DIU')` to the gRPC invocation metadata.
3. **Servicer Execution & Session Management:** `DataInspectorUtilityServicer` extracts the caller ID via `_get_caller(context)` for audit logging, opens a read-only database session (`with get_db_session() as db:`), and invokes the corresponding function in `core/services.py`.
4. **Data Aggregation & DTO Assembly:** `core/services.py` queries PostgreSQL via `SQLReader(db)` and ChromaDB via `vdb_reader.get_metadata_for_chunks()`, assembling the results into validated Pydantic models (`schemas.FullPageResult`, `schemas.DatabaseStatistics`, etc.).
5. **Protobuf Serialization:** `_prepare_dict_for_proto()` recursively traverses the dumped Pydantic dictionary, converting naive or aware `datetime.datetime` objects into UTC RFC 3339 strings with millisecond precision (`%Y-%m-%dT%H:%M:%S.%f` truncated to milliseconds + `'Z'`), then populates the Protobuf message via `google.protobuf.json_format.ParseDict`.
6. **Client Rendering:** The CLI receives the Protobuf response. If `--grpc` is passed, `print_grpc_json()` dumps raw JSON using `MessageToJson(preserving_proto_field_name=True)`. Otherwise, the response is converted to a Python dict via `MessageToDict` and rendered in the terminal using `rich` panels and tables (`cli/formatters/display.py`).

---

## 5. Detailed Business Logic, Rules & HITL 🧠

DIU is a headless, read-only inspection microservice and CLI tool; **Human-in-the-Loop (HITL) workflows are not applicable for this module.** Its core engine consists of deterministic data aggregation, hierarchical tree reconstruction, and recursive chunk expansion rules.

### 5.1. Hierarchical Sub-Chunk Expansion (`--expand`)
When querying chunks via `SQLReader.get_chunks_for_source_id(source_id, limit, search_text, expand)`:
- **Standard Filtering (`expand=False`):** Executes a SQL `ILIKE` filter (`%search_text%`) across `header_text`, `raw_text_content`, and `CAST(structured_tables_json AS VARCHAR)`, ordered by `chunk_sequence_number` and capped at `limit`.
- **Expanded Filtering (`expand=True`):**
  1. Fetches **all** chunks for `source_id` ordered by `chunk_sequence_number`.
  2. Evaluates each chunk in memory for a case-insensitive substring match against `header_text`, `raw_text_content`, or stringified `structured_tables_json`.
  3. When chunk `i` matches, it is added to `result_chunks` (tracked via `included_ids` set to prevent duplicates).
  4. If chunk `i` has a non-null `header_level` (`parent_level`), an inner pointer `j = i + 1` iterates forward through subsequent chunks.
  5. Every subsequent chunk `j` is included as a child sub-chunk until a chunk is encountered where `sub.header_level is not None` and `sub.header_level <= parent_level`, which terminates the current branch expansion.
  6. The final deduplicated list is sorted by `chunk_sequence_number` and sliced to `[:limit]`.

### 5.2. In-Memory Confluence Directory Tree Reconstruction (`get_directory_tree`)
`core/services.py:get_directory_tree(db_session, space_key)` reconstructs a clean navigational hierarchy from flat database records:
1. Fetches all `ConfluenceCategoryHierarchy` records for `space_key` (ordered by `depth`, `category_path`) and up to `10,000` `SourceMetadata` page records.
2. Builds two lookup maps: `source_to_page` (`page.id -> SourceMetadata`) and `title_to_page` (`page.title -> SourceMetadata`).
3. Iterates over every category record `cat`:
   - Matches `cat.category_name` against `title_to_page`.
   - Instantiates a `DirectoryNode` with `type="directory_page"` if a backing page exists (populating `source_id` and `page_id`), or `type="directory"` if it is a pure container.
   - Extracts child category names from `cat.direct_child_category_paths_json` (splitting on `" > "` and taking the last segment).
   - Iterates over `cat.direct_child_page_ids_json`: any child page whose title is **not** in `child_cat_names` is appended as a leaf `DirectoryNode` with `type="page"`.
4. Links category nodes to their parents via `path_to_node[cat.parent_path].children.append(node)`.
5. **Root Filtering Rule:** Only category nodes with `cat.depth == 1` are appended to `root_nodes`. This intentionally skips the space's singular home page (`depth == 0`) and any orphaned `depth == 0` archive pages, ensuring a clean top-level directory structure for the SAM UI.
6. Sorts `root_nodes` alphabetically by `title`.

### 5.3. Structured Table Resolution (`lookup_table`)
When looking up a specific `table_id`:
1. `SQLReader.get_chunks_by_table_id` executes a raw PostgreSQL text cast filter: `structured_tables_json::text LIKE '%"<table_id>"%'`.
2. `services.lookup_table` iterates over candidate chunks, parsing `structured_tables_json` (handling both `dict` keyed by `table_id` and `list` of dicts containing `'table_id'`).
3. Queries `LLMSummary` where `summary_type == 'table'` and `source_identifier LIKE '%#table_<table_id>'` ordered by `id DESC` to attach the latest `TableSummary`.

### 5.4. Multi-Dimension Database Statistics Aggregation (`get_database_statistics`)
Executes 11 distinct aggregation queries grouped by `space_key` (defaulting null keys to `"N/A"`) and accumulates both per-space metrics and a global `TOTALS` vector:
1. **Pages:** `COUNT(SourceMetadata.id)`
2. **Chunks:** `COUNT(SourceChunk.id)`
3. **Total_Words:** `SUM(SourceChunk.word_count)`
4. **Analyses:** `COUNT(SourceChunk.id)` where `llm_analysis IS NOT NULL`
5. **Page_Summaries:** `COUNT(LLMSummary.id)` where `summary_type == 'page'` joined on `CAST(SourceMetadata.id AS VARCHAR) == LLMSummary.source_identifier`
6. **Chunk_Summaries:** `COUNT(LLMSummary.id)` where `summary_type == 'chunk'` joined on `SourceChunk.vdb_chunk_id == LLMSummary.source_identifier`
7. **Table_Summaries:** `COUNT(LLMSummary.id)` where `summary_type == 'table'` joined on `LLMSummary.source_identifier LIKE SourceChunk.vdb_chunk_id || '#%'`
8. **Generated_Qs:** Coalesces `space_key` across direct page questions (`source_metadata_id`) and chunk questions (`source_chunk_id -> SourceChunk.source_metadata_id`) using outer joins.
9. **Synthesized_QnA:** `COUNT(DISTINCT SynthesizedQnA.id)` joined through `SynthesizedQnA.source_chunks -> SourceMetadata`.
10. **Vision_Records:** Unnests `SourceChunk.image_details_json` via PostgreSQL `json_array_elements(...)->>'relative_path'` and counts distinct matching `ImageVisionMetadata.id` records.
11. **Last Run Timestamp (`last_run_at`):** Queries `MAX(IngestionRunStatistic.created_at)` where `run_type == 'sync'`, formatted as `%Y-%m-%d %H:%M:%S`.

---

## 6. AI, LLM & Machine Learning Execution 🤖

*Not applicable for this module.*

DIU does not execute LLM inference, embedding generation, or machine learning training at runtime. Instead, it inspects, aggregates, and validates the AI artifacts produced by upstream modules (such as CFI), including:
- Page, chunk, and table summaries (`LLMSummary` records, prompt identifiers, and model names).
- HyDE generated questions (`GeneratedQuestion` records, user intents, and embedding model identifiers).
- Multi-page synthesized Q&As (`SynthesizedQnA`).
- Chunk semantic analyses (`llm_analysis` JSON) and multi-modal vision descriptions (`ImageVisionMetadata`).

---

## 7. Data Models, State, Storage & Auditing 🗄️

### 7.1. Relational Database Access (PostgreSQL)
DIU connects to `knowledge_base_db` using two SQLAlchemy `QueuePool` engines (`_read_engine` and `_write_engine`) configured in `core/database/database.py`:
- **Pool Parameters:** `pool_size=5`, `max_overflow=10`, `pool_timeout=30`, `pool_recycle=1800`, `pool_pre_ping=True`.
- **Session Context Manager:** `get_db_session(read_only: bool = True)` defaults to `_ReadSessionFactory` and closes the session in a `finally` block without committing.
- **External ORM Models:** Queries are constructed against `confluence_ingestor.core.database.models` (`SourceMetadata`, `SourceChunk`, `LLMSummary`, `ConfluenceCategoryHierarchy`, `GeneratedQuestion`, `ImageVisionMetadata`, `SynthesizedQnA`, `IngestionRunStatistic`).

### 7.2. Vector Database Access (ChromaDB)
Managed via `core/io_handlers/vdb_reader.py` using the ecosystem `ChromaVDB` wrapper (`vdb.chroma_client`):
- **Target Collections:** Dynamically collects all keys in `settings.yaml` under `vector_database` ending in `collection_name` or named `collections` (`confluence_store`, `confluence_page_summaries`, `confluence_question_embeddings`, `confluence_synthesized_qna`).
- **Batch Metadata Lookup:** `get_metadata_for_chunks(vdb_chunk_ids)` queries each configured collection via `vdb_client.get_documents(collection_name=c_name, ids=vdb_chunk_ids, include=["metadatas"])` and maps `vdb_chunk_id -> metadata dict`.

### 7.3. Dynamic & Static Pydantic Schemas (`core/database/schemas.py`)
Generated by `core/schema_generator.py` using `ConfigDict(from_attributes=True)`:
- **Auto-Generated ORM Models:** `PageMetadata` (from `SourceMetadata` plus `generated_questions`, `page_summaries_json`, `synthesized_questions`) and `ChunkDetail` (from `SourceChunk` plus `generated_questions`, `chunk_summary_json`, `table_summaries_json`, `vdb_metadata`, `vision_metadata_json`).
- **Static Domain DTOs:** `GeneratedQuestion`, `Summary`, `TableSummary`, `CategoryNode`, `SQNASourcePage`, `DirectoryNode`, `SynthesizedQnADetail`, `TableDetailResult`, `FullPageResult`, `PageSummary`, `PaginatedPagesResponse`, `SpaceStats`, `DatabaseStatistics`.

### 7.4. Auditing
Every incoming gRPC request extracts the `x-client-id` metadata header (`[DIU]`, `[SAM]`, or `[UNKNOWN]`) and logs the caller identity alongside the RPC name and query parameters to `diu.log.YYYY-MM-DD`.

---

## 8. APIs, Observability & Communication Contracts 📡

DIU exposes two gRPC services on port `50109` with gRPC Server Reflection and Standard Health Checking (`grpc.health.v1.Health`) enabled for both `diu.DataInspectorUtility` and `sera.standard.v1.SeraAppService`.

### 8.1. `diu.DataInspectorUtility` RPC Contract (`grpc_api/protos/diu.proto`)

| RPC Method | Request Message | Response Message | Pattern | Description |
| :--- | :--- | :--- | :--- | :--- |
| `GetDatabaseStatistics` | `GetDatabaseStatisticsRequest` | `DatabaseStatsResponse` | Unary | Returns per-space and total counts across 10 content dimensions plus `last_run_at`. |
| `LookupBySourceId` | `LookupBySourceIdRequest` | `PageDetailResponse` | Unary | Fetches full page & chunk details by internal PostgreSQL `source_id`. Sets `NOT_FOUND` if missing. |
| `LookupByPageId` | `LookupByPageIdRequest` | `PageDetailResponse` | Unary | Fetches full page & chunk details by Confluence `page_id` (`source_native_id`). Sets `NOT_FOUND` if missing. |
| `GetPagesByTitle` | `GetPagesByTitleRequest` | `stream PageDetailResponse` | Server Stream | Searches pages by `ILIKE` title and optional `search_text` / `expand` chunk filtering. |
| `GetCategoryHierarchy` | `GetCategoryHierarchyRequest` | `stream CategoryNodeResponse` | Server Stream | Streams category hierarchy nodes for a `space_key` and optional `category_path`. |
| `LookupSynthesizedQnA` | `LookupSynthesizedQnARequest` | `stream SynthesizedQnADetailResponse` | Server Stream | Streams multi-page Synthesized Q&As filtered by numeric ID, question substring, or `space_key`. |
| `ListSpaces` | `ListSpacesRequest` | `ListSpacesResponse` | Unary | Returns sorted list of all distinct ingested `space_keys`. |
| `ListPagesInSpace` | `ListPagesInSpaceRequest` | `ListPagesInSpaceResponse` | Unary | Returns paginated `PageSummary` records (`page_size`, `offset`, `title_filter`) and `total_pages`. |
| `LookupTable` | `LookupTableRequest` | `TableDetailResponse` | Unary | Returns structured table JSON, parent page/chunk metadata, and LLM `TableSummary` by `table_id`. |
| `GetDirectoryTree` | `GetDirectoryTreeRequest` | `GetDirectoryTreeResponse` | Unary | Returns recursively assembled `DirectoryNodeResponse` tree starting at `depth == 1`. |

### 8.2. `sera.standard.v1.SeraAppService` Contract (`grpc_api/sera_standard_service.py`)
DIU implements the universal SERA standard contract (`sera.standard.v1.SeraAppService` - `v1.2-universal`), centrally maintained and deployed by the Application Management Utility (AMU). It provides standardized endpoints (`GetAppInfo`, `GetDeepHealth`, `GetFeedback`, `GetChangelog`, `GetDocs`) enabling AMU and the SAM UI to self-discover DIU's metadata, deep TCP socket dependency health matrix, changelog history, and design documentation without requiring custom integration logic.

---

## 9. User Interface (UI) & Visualizations 🖥️

*Not applicable for this module.*

DIU does not host a web frontend. It provides rich terminal visualizations via the `rich` library (`cli/formatters/display.py`) for CLI users and serves raw structured gRPC/JSON payloads to the **Sera Admin Module (SAM)** frontend.

---

## 10. Background Services & Workers 🛠️

*Not applicable for this module.*

Aside from the main persistent gRPC server thread pool (`diu-grpc.service` with `max_workers=10`), DIU does not run asynchronous background pollers, task queues, or scheduled cron workers.

---

## 11. CLI, Tooling & Testing 💻

The DIU CLI (`cli/diu_cli.py`) uses a custom `NestedHelpGroup` to display all top-level groups and nested subcommands in a unified help table. Every inspection command supports the `--grpc` flag to bypass `rich` formatting and output raw Protobuf-to-JSON payloads.

### 11.1. CLI Command Reference

| Command Group & Subcommand | Arguments & Options | Description |
| :--- | :--- | :--- |
| `start-grpc-server` | *None* | Runs schema generator, compiles protos, initializes DB pools, and starts the gRPC server. |
| `module info` | `[--grpc]` | Retrieves App Info via SERA Standard gRPC. |
| `module check-health` | `[--grpc]` | Retrieves Deep Health matrix via SERA Standard gRPC. |
| `stats show` | `[--grpc]` | Displays the global and per-space Database Content Statistics table with totals footer. |
| `spaces list` | `[--grpc]` | Lists all unique ingested Confluence space keys. |
| `spaces hierarchy` | `<space_key> [path] [--grpc]` | Displays full category hierarchy table for a space, or detailed panel if a single node matches. |
| `spaces directory` | `<space_key> [--grpc]` | Outputs the nested directory tree for a space as formatted JSON. |
| `pages list` | `<space_key> [--title] [--limit 50] [--offset 0] [--grpc]` | Displays a paginated Rich table of pages within a space. |
| `pages get` | `<page_id> [--limit-chunks 10] [--grpc]` | Displays full page metadata, summaries, questions, and chunk panels by Confluence Page ID. |
| `pages search` | `--title <query> [--search-text] [--include-qa] [--limit-pages 5] [--limit-chunks 10] [--expand] [--grpc]` | Searches pages by title and optional chunk text with hierarchical sub-chunk expansion. |
| `tables show` | `<table_id> [--grpc]` | Displays structured table content as a Rich table alongside parent page metadata and LLM summary. |
| `sqna search` | `[query='all'] [--space] [--grpc]` | Searches multi-page Synthesized Q&A records by ID, question text, or space key. |
| `debug store` | `<page_id> [--output 'page_dump.json']` | Fetches all details for a page (`limit_chunks=10000`) and writes formatted JSON to disk. |

### 11.2. Standalone Tooling
- **`core/schema_generator.py`:** Can be executed directly (`python3 core/schema_generator.py`) to regenerate `core/database/schemas.py` from `confluence_ingestor` ORM models.
- **`core/test_diu.py`:** Standalone async diagnostic script utilizing `DIUClient` to inspect structured table keys and markdown samples.

---

## 12. Configuration, Security & Hardware Affinity 🔒

### 12.1. Configuration Reference (`config/settings.yaml`)

| YAML Key Path | Purpose | Default / Configured Value |
| :--- | :--- | :--- |
| `module_info.name` | Human-readable module name | `"Data Inspector Utility"` |
| `module_info.version` | Current semantic version | `"0.2.7"` |
| `module_info.acronym` | Ecosystem module identifier | `"DIU"` |
| `module_info.category` | Ecosystem module category | `"Utility"` |
| `system_services.grpc.service_name` | Systemd unit name checked by SERA health probes | `"diu-grpc.service"` |
| `system_services.grpc.host` | gRPC server bind address | `"0.0.0.0"` |
| `system_services.grpc.port` | gRPC server listen port | `50109` |
| `system_services.grpc.max_workers` | ThreadPoolExecutor worker count | `10` |
| `database.write_dsn` | PostgreSQL write connection string | `"postgresql://ai_assistant_user:***@localhost:5432/knowledge_base_db"` |
| `database.read_dsn` | PostgreSQL read connection string | `"postgresql://ai_assistant_user:***@localhost:5432/knowledge_base_db"` |
| `vector_database.host` | ChromaDB server hostname (overridable via `VDB_HOST` env) | `"localhost"` |
| `vector_database.port` | ChromaDB server port (overridable via `VDB_PORT` env) | `8000` |
| `vector_database.collection_name` | Primary chunk embeddings collection | `"confluence_store"` |
| `vector_database.page_summary_collection_name` | Page summaries vector collection | `"confluence_page_summaries"` |
| `vector_database.qa_vdb_collection_name` | Generated questions (HyDE) vector collection | `"confluence_question_embeddings"` |
| `vector_database.synthesized_qna_collection_name` | Multi-page synthesized Q&A vector collection | `"confluence_synthesized_qna"` |
| `paths.proto_source_dir` | Relative path to DIU proto definitions | `"grpc_api/protos"` |
| `paths.sera_proto_source_dir` | Global path to `sera_standard.proto` | `"/ai/assets/protos/sera_standard"` |
| `paths.generated_stubs_path` | Shared output directory for compiled Python gRPC stubs | `"/ai/assets/protos/data_inspector_utility/generated"` |
| `schema_generator.source_model_paths` | Reference model path for schema generation | `["confluence_ingestor/database/models.py"]` |
| `logging.level` | Default server console log level | `"INFO"` |
| `logging.log_dir` | Centralized log output directory | `"/ai/assets/logs/diu_utility"` |
| `logging.app_log_filename` | Base filename for daily application logs | `"diu.log"` |
| `logging.error_log_filename` | Base filename for daily warning/error logs | `"diu_err.log"` |
| `logging.enable_file_logging` | Enables `MultiProcessSafeDailyLogger` file output | `true` |

### 12.2. Security & Identity Propagation
- **DSN Sanitization:** `SeraStandardServiceImpl.GetDeepHealth` strips passwords from all PostgreSQL and cache connection strings via `sanitize_dsn()` before serializing `connection_metadata` over gRPC.
- **Client Identity Tracking:** `MetadataClientInterceptor` injects `x-client-id: DIU` on all CLI requests, allowing the server to log caller provenance.
- **Network Scope:** Uses `grpc.insecure_channel` and `add_insecure_port` bound to internal data center networks; intended strictly for internal microservice and operator access.

### 12.3. Hardware Affinity
DIU is a lightweight CPU-bound I/O aggregator. It requires no GPU allocation and runs on standard application nodes alongside the PostgreSQL and ChromaDB instances.

---

## 13. Error Handling & Resilience 🛡️

1. **CLI gRPC Error Decorator (`@handle_grpc_errors`):**
   * Wraps all Click CLI commands in `cli/utils.py`. Catches `grpc.RpcError`, prints a formatted error message (`gRPC Error [<StatusCode>]: <details>`) in red to `stderr`, and exits cleanly with status code `1`.
2. **Graceful VDB Degradation:**
   * In `core/io_handlers/vdb_reader.py`, if ChromaDB is unreachable, unconfigured, or a collection raises `chromadb.errors.NotFoundError`, the error is logged and `get_metadata_for_chunks` returns an empty dictionary (`{}`) for the affected collection. Page and chunk inspection continues uninterrupted using PostgreSQL relational data.
3. **Database Connection Resilience:**
   * SQLAlchemy engines use `pool_pre_ping=True` and `pool_recycle=1800` to automatically detect and replace stale PostgreSQL connections before executing queries. Server startup validates connectivity with `SELECT 1` and aborts immediately if the database is unreachable.
4. **Startup Self-Healing Stubs:**
   * If `.proto` definitions are updated, `start-grpc-server` automatically recompiles the stubs and uses `os.execv` to restart the Python process cleanly, preventing stale protobuf descriptor crashes.
5. **DeepHealth Status Escalation:**
   * `GetDeepHealth` performs 1.0-second TCP socket checks (`check_tcp_ping`). If any service is not `"running"` or any dependency is not `"OK"`, `overall_status` degrades to `"DEGRADED"`. If any PostgreSQL dependency is `"DOWN"`, `overall_status` escalates to `"DOWN"`.
6. **Log Rotation Resilience:**
   * `MultiProcessSafeDailyLogger` automatically rolls over log files at midnight (`YYYY-MM-DD` suffix) and prunes log files older than `backup_count=14` days, suppressing `OSError` exceptions during concurrent cleanup.

---

## 14. Future Enhancements & Roadmap 🚀

The following initiatives define the planned enhancements and roadmap for DIU, expanding inspection capabilities across the CLI and frontend while maintaining architectural parity with ecosystem standards:

### 14.1. CLI Parity for Source ID Lookups (`LookupBySourceId`)
The `LookupBySourceId` RPC is fully implemented in `DataInspectorUtilityServicer` and `core/services.py`, but `cli/commands/pages_cmds.py` currently only exposes `pages get <page_id>`. Adding a `--source-id` flag or dedicated `pages get-source <source_id>` command will complete CLI parity with the underlying gRPC servicer.

### 14.2. Vision Metadata & Rich Directory Tree Terminal Rendering
`core/services.py` and `diu.proto` (`Chunk.vision_metadata_json`) already fetch and serialize `ImageVisionMetadata` (`vision_keywords`, `ai_description`), and `cli/commands/spaces_cmds.py` dumps `spaces directory` as raw JSON. Updating `cli/formatters/display.py` to render `vision_metadata_json` inside chunk panels and a visual `rich.tree.Tree` for `spaces directory` will complete terminal visualization support.

### 14.3. A/B Raw Vector Query Diagnostics Endpoint
Expose a dedicated gRPC endpoint in `diu.proto` to execute raw semantic similarity searches directly against ChromaDB collections (`confluence_store`, `confluence_question_embeddings`), enabling side-by-side RAG diagnostic comparisons in the Sera Admin Module (SAM) UI before reranking algorithms are applied.

### 14.4. Schema Generator Configuration & Path Alignment
Update `schema_generator.source_model_paths` in `config/settings.yaml` and the header comment in `diu.proto` to reflect the current `confluence_ingestor/core/database/models.py` path and automated startup execution.