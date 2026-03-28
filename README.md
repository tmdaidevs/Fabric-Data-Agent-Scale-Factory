# Fabric Data Agent Auto-Factory

Automatically scans a Microsoft Fabric tenant, profiles all data items, **understands the context of each dataset**, and uses AI to intelligently decide which Data Agents to create — grouping related data into coherent, domain-specific agents complete with system instructions, data instructions, and example queries.

## The Problem

Organizations using Microsoft Fabric accumulate dozens to hundreds of data sources across many workspaces. Manually creating a **Data Agent** (an AI chatbot that answers natural-language questions about your data) for each source — configuring data connections, writing system instructions, crafting example queries — is tedious and doesn't scale.

## How It Works

This notebook doesn't just blindly create one agent per data source. It uses a **2-stage intelligence pipeline**:

### Stage 1: Per-Table Context Extraction
For **each individual table**, the LLM analyzes the schema and sample data to produce:
- A semantic description ("NYC taxi trip records with pickup/dropoff locations, fares...")
- Domain tags (e.g., `["transportation", "taxi", "nyc", "realtime"]`)
- Table role classification (fact, dimension, event/streaming, reference)

Additionally, a **programmatic column-name similarity check** detects cross-item relationships — e.g., an Eventhouse table and a Lakehouse table that share `pickup_datetime`, `fare_amount`, `trip_distance` columns are flagged as related.

### Stage 2: Context-Aware Agent Clustering
The second LLM call receives:
- A **flat list of tables** (decoupled from their Fabric items) with semantic contexts and tags
- **Similarity hints** showing which tables across different items have overlapping columns

This enables the LLM to:
- Group tables by **content**, not by which Fabric item they live in
- Combine historical Lakehouse data with real-time Eventhouse data into one agent
- Split an Eventhouse with unrelated tables into different agents
- Assign the same table to **multiple agents** when relevant (e.g., Weather data in both Taxi and Supply Chain agents)

## Phases

| Phase | Description |
|-------|-------------|
| **1. Discovery** | Enumerates all workspaces and lists Lakehouses, Warehouses, Semantic Models, KQL Databases |
| **2. Profiling** | Reads schema (tables + columns) and samples rows per table to understand data context |
| **2.5. Context Extraction** | Per-table LLM context extraction + programmatic column similarity matrix |
| **3. Clustering** | Context-aware, table-level clustering via LLM using semantic tags and cross-item similarity hints |
| **4. Creation** | Uses `fabric-data-agent-sdk` to create agents, add data sources, set instructions, add example queries, and publish |
| **5. Catalog** | Writes an agent inventory to a Delta table for the Agent Garden |

## Prerequisites

- **Microsoft Fabric** with an F2+ capacity
- A **Fabric Notebook** (the SDK only runs inside Fabric notebooks)
- **Data Agent tenant settings** enabled
- **Azure OpenAI** deployment (e.g., `gpt-4.1-mini`) accessible via a Service Principal or Workspace Identity
- **Service Principal** (App Registration) with `Cognitive Services OpenAI User` role on the Azure OpenAI resource
- **Workspace access** (Viewer+ role) to the workspaces you want to scan

## Quick Start

1. Upload `data_agent_auto_factory.ipynb` to a Fabric workspace
2. Open the notebook and edit **Cell 0 (Configuration)**:
   - Set `resource_name` and `deployment_name` to your Azure OpenAI resource and model deployment
   - Set `tenant_id`, `client_id`, and `client_secret` for your Service Principal (or load the secret from Key Vault)
   - Optionally set `WORKSPACE_FILTER` to limit scanning to specific workspaces
   - Leave `DRY_RUN = True` for the first run
3. **Run All** cells
4. Review the proposed agents in the **Human Review Checkpoint** section
5. Set `DRY_RUN = False` and re-run from Phase 4 to create and publish the agents

## Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `resource_name` | `FoundryFabric` | Azure OpenAI resource name |
| `deployment_name` | `gpt-4.1-mini` | Model deployment name |
| `tenant_id` | *(required)* | Azure AD tenant ID for the Service Principal |
| `client_id` | *(required)* | Service Principal (App Registration) client ID |
| `client_secret` | *(required)* | Service Principal secret (prefer Key Vault) |
| `SAMPLE_ROWS` | `5` | Rows sampled per table for profiling |
| `MAX_TABLES_PER_ITEM` | `50` | Skip items with more tables |
| `MAX_COLUMNS_IN_SAMPLE` | `20` | Max columns sent to LLM per table |
| `AGENT_NAME_PREFIX` | `Auto` | Prefix for created agent names |
| `MAX_DATASOURCES_PER_AGENT` | `5` | Max data sources per agent (SDK limit) |
| `MAX_EXAMPLES_PER_DATASOURCE` | `20` | Example queries generated per data source |
| `WORKSPACE_FILTER` | `None` | List of workspace names to scan (None = all) |
| `ITEM_TYPES_TO_SCAN` | All 4 types | Which Fabric item types to include |
| `DRY_RUN` | `True` | Preview only — set to `False` to create agents |
| `CATALOG_LAKEHOUSE` | `None` | Lakehouse name for the catalog Delta table |

## Architecture

```
┌───────────────────────────────────────────────────────────────┐
│                      Fabric Notebook                           │
│                                                               │
│  ┌──────────┐   ┌───────────────┐   ┌──────────────────────┐ │
│  │ Discovery │──>│   Profiling    │──>│  Phase 2.5           │ │
│  │ (REST API)│   │ (Spark / DAX /│   │  Per-Table LLM       │ │
│  │           │   │  Kusto)       │   │  Context Extraction   │ │
│  └──────────┘   └───────────────┘   │  + Column Similarity  │ │
│                                      └──────────┬───────────┘ │
│                                                  │             │
│                     table_contexts + similarity_hints           │
│                                                  │             │
│                                      ┌───────────▼──────────┐ │
│                                      │  Phase 3: LLM        │ │
│                                      │  Context-Aware       │ │
│                                      │  Table-Level         │ │
│                                      │  Clustering          │ │
│                                      └───────────┬──────────┘ │
│                                                  │             │
│                                      ┌───────────▼──────────┐ │
│                                      │  Human Review        │ │
│                                      │  Checkpoint          │ │
│                                      └───────────┬──────────┘ │
│                                                  │             │
│  ┌───────────────┐   ┌───────────────────────────▼──────────┐ │
│  │ Agent Garden   │<──│  Data Agent SDK                      │ │
│  │ Catalog        │   │  (create, configure, publish)        │ │
│  └───────────────┘   └──────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────┘
```

## Authentication

The notebook authenticates to Azure OpenAI using a **Service Principal** (OAuth2 client credentials flow). The token is acquired at runtime via:

```
POST https://login.microsoftonline.com/{tenant_id}/oauth2/v2.0/token
scope=https://cognitiveservices.azure.com/.default
```

> **Security Note:** Avoid hardcoding `client_secret` in the notebook. Use Key Vault instead:
> ```python
> client_secret = mssparkutils.credentials.getSecret("<vault-url>", "<secret-name>")
> ```

## Key Constraints

- **Max 5 data sources per agent** (SDK limit) — auto-splits oversized agents
- **Semantic Models don't support example queries** — only instructions are set
- **SDK runs exclusively in Fabric Notebooks** — no local execution
- **15,000 character limit** for instructions — auto-truncated
- **100 example queries** per data source max

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "Cannot list items" | Ensure Viewer+ role on target workspaces |
| "Cannot profile table" | Spark context needs access — may need Lakehouse shortcuts |
| "Failed to add datasource" | Item must be accessible via OneLake catalog |
| Azure OpenAI 401/403 | Verify SPN has `Cognitive Services OpenAI User` role on the AOAI resource |
| Token acquisition fails | Check `tenant_id`, `client_id`, `client_secret` values |
| Agent has wrong tables | Edit `agent_plan` in the manual edit cell before Phase 4 |
