# A2A Protocol Bridge — Dependency Report

**Product:** sn_a2a_bridge  
**Date:** 2026-05-24  
**Author:** Vladimir Kapustin

## Python Dependencies

| Package | Version | Purpose | Type | Risk |
|---------|---------|---------|------|------|
| `requests` | >=2.28 | HTTP client for ServiceNow REST API calls | Runtime | Low — mature, widely used |
| `dataclasses` | stdlib | Immutable data containers (ExternalAgent, A2AConfig, BridgeReport) | Runtime | None — standard library |
| `json` | stdlib | Report serialization (ensure_ascii=False for UTF-8) | Runtime | None — standard library |
| `datetime` | stdlib | UTC timestamp generation for reports | Runtime | None — standard library |
| `pathlib` | stdlib | Report file path management | Runtime | None — standard library |
| `os` | stdlib | Environment variable access for credentials | Runtime | None — standard library |
| `unittest` | stdlib | Test framework | Dev | None — standard library |
| `pytest` | >=7.0 | Test runner with verbose output | Dev | None — optional |

### Dependency Installation

```bash
pip install requests pytest
```

`requests` is the only non-stdlib runtime dependency. `pytest` is optional (tests run with `unittest` also).

## ServiceNow Platform Dependencies

| Component | Type | Required | Notes |
|-----------|------|----------|-------|
| REST API (Table API) | Platform Feature | Yes | `GET /api/now/table/sn_agent` for agent discovery |
| `sn_agent` table | Platform Table | Yes | Source of truth for external agent records |
| `sysparm_query` support | API Feature | Yes | Filtering agents by `type=external` |
| `sysparm_fields` support | API Feature | Yes | Selecting specific fields to minimize payload |
| `sysparm_limit` support | API Feature | Optional | Pagination control (default 500) |
| Basic Auth | Auth Method | Yes | HTTP Basic Authentication over HTTPS |
| ServiceNow Instance (any release) | Platform | Yes | Compatible with Utah, Vancouver, Washington DC, Zurich, Australia |

### Plugin Dependencies

None. The bridge does not require any ServiceNow plugins to be activated. It operates purely through the standard REST Table API, which is available on all ServiceNow instances with REST access enabled.

## External Service Dependencies

None. The bridge is entirely self-contained:
- No external API calls beyond the target ServiceNow instance
- No cloud services required
- No database servers needed
- No message brokers or queues

## Development Dependencies

| Tool | Version | Purpose |
|------|---------|---------|
| Python | >=3.10 | Runtime |
| Git | Any | Version control |
| pytest | >=7.0 | Test runner |
| pip | Any | Package management |

## Dependency Risk Assessment

| Risk | Level | Mitigation |
|------|-------|------------|
| `requests` vulnerable to CVE | Low | Pin version; requests is mature and actively maintained; CVEs are patched quickly |
| ServiceNow REST API deprecated | Low | Table API is a core platform feature; no deprecation announced through Australia release |
| `sn_agent` table schema change | Medium | Schema changes rare on platform tables; bridge uses only standard fields (sys_id, name, integration_type) |
| Python version EOL | Low | Python 3.10 supported through Oct 2026; upgrade path to 3.11/3.12 is trivial |
| API rate limiting | Low | bridge makes a single GET per scan; well under any rate limit threshold |

## Compatibility Matrix

| ServiceNow Release | REST API | sn_agent Table | Status |
|-------------------|----------|----------------|--------|
| Utah | ✅ | ✅ | Compatible |
| Vancouver | ✅ | ✅ | Compatible |
| Washington DC | ✅ | ✅ | Compatible |
| Zurich | ✅ | ✅ | Compatible |
| Australia | ✅ | ✅ | Compatible (primary target) |

## Dependency Tree

```
sn_a2a_bridge
├── requests (runtime)
│   ├── urllib3
│   ├── certifi
│   └── charset-normalizer
├── dataclasses (stdlib)
├── json (stdlib)
├── datetime (stdlib)
├── pathlib (stdlib)
└── os (stdlib)
```
