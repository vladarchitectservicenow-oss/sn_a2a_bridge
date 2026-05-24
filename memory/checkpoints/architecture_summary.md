# A2A Protocol Bridge — Architecture Summary

**Product:** sn_a2a_bridge  
**Scope:** x_sn_a2a_bridge  
**Release:** Australia (May 2026)  
**Author:** Vladimir Kapustin  
**License:** AGPL-3.0-only

## Executive Summary

The A2A Protocol Bridge is a Python-based scoped application that runs as a command-line tool and optionally as a ServiceNow scoped application. It discovers, classifies, migrates, and validates external AI agents connected to a ServiceNow instance, producing structured reports in HTML and JSON formats. The bridge addresses the multi-agent integration crisis by providing a canonical A2A configuration registry, automated legacy-to-A2A migration, manifest validation, and rich observability reporting.

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    A2A Protocol Bridge                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐ │
│  │  Agent   │  │  Config  │  │  Config  │  │ Report  │ │
│  │ Scanner  │──│Generator │──│Validator │──│ Engine  │ │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬────┘ │
│       │             │             │             │        │
│       ▼             ▼             ▼             ▼        │
│  ┌──────────────────────────────────────────────────┐   │
│  │              ServiceNow REST API                   │   │
│  │  GET /api/now/table/sn_agent?type=external        │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

## Component Details

### 1. Agent Scanner (`list_external_agents`)
- **Purpose:** Discover all external agents registered in the ServiceNow instance
- **Data Source:** `sn_agent` table via REST API
- **Filter:** `type=external` with fields: `sys_id, name, integration_type, endpoint_url, auth_type, a2a_enabled`
- **Output:** List of `ExternalAgent` dataclass instances
- **Classification Logic:** Agents with integration_type in (`manual`, `webhook`, `legacy`) are marked deprecated

### 2. Config Generator (`generate_a2a_config`)
- **Purpose:** Generate canonical A2A protocol configuration from a deprecated agent record
- **Input:** `ExternalAgent` instance
- **Output:** `A2AConfig` dataclass with protocol_version, endpoint_url, capabilities, auth
- **Capability Inference:** Webhook agents receive `["text", "actions"]`; others receive `["text"]`
- **Auth Mapping:** Agent's `auth_type` field is preserved in the A2A config

### 3. Config Validator (`validate_a2a`)
- **Purpose:** Validate that an A2A config meets protocol requirements
- **Required Fields:** endpoint_url, capabilities (non-empty), name (non-empty)
- **Output:** List of issue strings (empty = valid)
- **Failure Mode:** Returns descriptive issue messages for each missing field

### 4. Report Engine (`save_report`, `_render_html`)
- **Purpose:** Produce human- and machine-readable scan reports
- **HTML Output:** Styled dashboard with summary statistics, migrated agent table, issues table
- **JSON Output:** Complete `BridgeReport` serialized via `dataclasses.asdict()` with `ensure_ascii=False`
- **File Naming:** `a2a_bridge_{date}_{host}.html` / `.json`

## Data Model

### ExternalAgent
```
sys_id: str           — ServiceNow sys_id
name: str             — Agent display name
integration_type: str — webhook | manual | legacy | a2a | oauth2
endpoint_url: str     — Agent REST endpoint
auth_type: str        — basic | oauth2 | api_key
a2a_enabled: bool     — Whether A2A protocol is already enabled
deprecated: bool      — Computed: True if integration_type in (manual, webhook, legacy)
```

### A2AConfig
```
source_agent_id: str   — Reference to ExternalAgent.sys_id
name: str              — Agent name
protocol_version: str  — Default "1.0"
endpoint_url: str      — Agent endpoint URL
capabilities: List[str]— [text] or [text, actions]
auth: dict             — {type, scope}
```

### BridgeReport
```
instance: str               — ServiceNow instance URL
timestamp: str              — ISO 8601 UTC timestamp
total_external_agents: int  — Total agents discovered
deprecated_count: int       — Agents needing migration
a2a_compliant_count: int    — Already A2A-compliant agents
migrated: List[A2AConfig]   — Successfully generated configs
issues: List[str]           — Validation failures
```

## Runtime Characteristics

- **Language:** Python 3.10+
- **Dependencies:** `requests` (HTTP), `dataclasses` (stdlib), `datetime` (stdlib), `json` (stdlib), `pathlib` (stdlib)
- **Execution Model:** Single-threaded synchronous CLI tool
- **Stateless:** No local database; reports are filesystem-based
- **Read-Only:** Never modifies ServiceNow data; GET requests only
- **Performance:** ~5 seconds for 500-agent scan over local network

## Deployment Topology

```
┌──────────────┐     HTTPS GET      ┌──────────────────┐
│  A2A Bridge  │───────────────────▶│   ServiceNow     │
│  (CLI Tool)  │                    │   Instance       │
│              │◀───────────────────│   (REST API)     │
└──────┬───────┘     JSON/HTML      └──────────────────┘
       │
       ▼
┌──────────────┐
│   Reports/   │
│   Directory  │
└──────────────┘
```

## Security Architecture

- All communication over HTTPS with TLS certificate validation
- Credentials via environment variables (never in source code)
- Read-only API access (GET requests only)
- No persistent credential storage on disk
- No outbound connections beyond the ServiceNow REST API
