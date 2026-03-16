# Fabric Data Agent Auto-Factory

Automatically scans a Microsoft Fabric tenant, profiles all data items, and creates thematically clustered Data Agents — complete with system instructions, data instructions, and example queries.

## What it does

| Phase | Description |
|-------|-------------|
| **1. Discovery** | Enumerates all workspaces and lists Lakehouses, Warehouses, Semantic Models, KQL Databases |
| **2. Profiling** | Reads schema (tables + columns) and samples N rows per table via Spark SQL / sempy / Kusto |
| **3. Clustering** | Sends profiles to Azure OpenAI (GPT-4o) which groups related data into thematic agents |
| **4. Creation** | Uses `fabric-data-agent-sdk` to create agents, add data sources, set instructions, add example queries, and publish |
| **5. Catalog** | Writes an agent inventory to a Delta table for the Agent Garden |

## Prerequisites

- **Microsoft Fabric** with an F2+ capacity
- A **Fabric Notebook** (the SDK only runs inside Fabric notebooks)
- **Data Agent tenant settings** enabled
- **Azure OpenAI** deployment (GPT-4o recommended) with the notebook identity having access
- **Workspace access** (Viewer+ role) to the workspaces you want to scan

## Quick Start

1. Upload `data_agent_auto_factory.ipynb` to a Fabric workspace
2. Open the notebook and edit **Cell 0 (Configuration)**:
   - Set `AOAI_ENDPOINT` and `AOAI_DEPLOYMENT` to your Azure OpenAI resource
   - Optionally set `WORKSPACE_FILTER` to limit scanning to specific workspaces
   - Leave `DRY_RUN = True` for the first run
3. **Run All** cells
4. Review the proposed agents in the **Human Review Checkpoint** section
5. Set `DRY_RUN = False` and re-run from Phase 4 to create and publish the agents

## Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `AOAI_ENDPOINT` | *(required)* | Azure OpenAI endpoint URL |
| `AOAI_DEPLOYMENT` | `gpt-4o` | Model deployment name |
| `SAMPLE_ROWS` | `5` | Rows sampled per table for profiling |
| `MAX_TABLES_PER_ITEM` | `50` | Skip items with more tables |
| `AGENT_NAME_PREFIX` | `Auto` | Prefix for created agent names |
| `WORKSPACE_FILTER` | `None` | List of workspace names to scan (None = all) |
| `DRY_RUN` | `True` | Preview only — set to `False` to create agents |
| `CATALOG_LAKEHOUSE` | `None` | Lakehouse name for the catalog Delta table |

## Architecture

```
┌──────────────────────────────────────────────────────┐
│                   Fabric Notebook                     │
│                                                      │
│  ┌─────────┐   ┌──────────┐   ┌────────────────┐   │
│  │Discovery │──>│Profiling  │──>│ Azure OpenAI   │   │
│  │(REST API)│   │(Spark SQL)│   │ (Clustering)   │   │
│  └─────────┘   └──────────┘   └──────┬─────────┘   │
│                                       │              │
│                              ┌────────▼─────────┐   │
│                              │  Human Review     │   │
│                              │  Checkpoint       │   │
│                              └────────┬─────────┘   │
│                                       │              │
│  ┌──────────────┐   ┌────────────────▼──────────┐  │
│  │ Agent Garden  │<──│  Data Agent SDK           │  │
│  │ Catalog       │   │  (create, configure,      │  │
│  │ (Delta table) │   │   publish agents)         │  │
│  └──────────────┘   └──────────────────────────┘   │
└──────────────────────────────────────────────────────┘
```

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
| Azure OpenAI errors | Verify endpoint/deployment and token access |
| Agent has wrong tables | Edit `agent_plan` in the manual edit cell before Phase 4 |
