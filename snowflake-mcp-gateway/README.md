# snowflake-mcp-gateway

A single, enforcing door to client data in Snowflake, exposed as an MCP server.

Every consumer (the agent's session manager, Claude Desktop for analysts, the alerts detector, a catalogue page) reaches Snowflake **only** through this gateway. The gateway decides who the caller is, which single child client the call is for, which Snowflake role runs it, what SQL is allowed, and what comes back. Snowflake grants and policies sit underneath as a backstop.

This file is written so a Claude Code instance can build the service step by step. Read the whole file before starting. Build the steps **in order**, and do not start a step until the previous step's acceptance checks pass.

---

## 1. Context

- **Domain:** payments. Clients form a hierarchy: a parent client with child clients, which can be in different regions (and so different Snowflake accounts).
- **Hard rule:** data from one child client must never reach a session, cache, log, or response for another. No data may leak through logs, traces, errors, or tool text.
- **Users:** about 100 people on one team. Each person supports a different set of clients.
- **Runtime:** AWS ECS Fargate, Red Hat UBI 9 base image, private subnets only, no internet egress.
- **LLM side:** Claude on Amazon Bedrock runs in a separate service (the session manager, built with LangChain Deep Agents). The gateway never calls an LLM.
- **Semantic layer:** today a YAML catalogue in Snowflake's semantic-model YAML shape, kept in Git. Later, Snowflake semantic views deployed from that YAML.
- **Sandbox:** the agent's code runs in a nono sandbox that has no network route to this gateway or to Snowflake. It only sees Parquet files the session manager puts in `/work/<session>`.

## 2. What the gateway is and is not

It is:
- An MCP server (Streamable HTTP transport) with a **fixed** set of read-only tools and resources.
- The only holder of Snowflake credentials.
- The single place where data guardrails are enforced.

It is not:
- An agent. No LLM calls, no sampling, no prompts.
- A write path. No tool writes to Snowflake, sends email, calls HTTP, or exports to anywhere except the per-client results bucket.

## 3. Architecture

```
 callers (each with its own registered identity)
 ┌──────────────────┐ ┌───────────────┐ ┌─────────────────┐ ┌──────────────┐
 │ Session manager  │ │ Claude Desktop│ │ Alerts detector │ │ Catalogue UI │
 │ (MCP client)     │ │ (via proxy)   │ │                 │ │              │
 └────────┬─────────┘ └──────┬────────┘ └───────┬─────────┘ └──────┬───────┘
          │ JWT: user + ONE child client + caller id                │
          ▼                                                         ▼
 ┌─────────────────────────── snowflake-mcp-gateway ───────────────────────────┐
 │ 1 auth ─▶ 2 entitlement ─▶ 3 tool registry ─▶ 4 SQL guard ─▶ 5 limits       │
 │                                                    │                         │
 │                                     backend (per child role session)         │
 │                                                    │                         │
 │ 6 output pipeline (preview, PII scan, size cap) ◀──┘   ─▶ Parquet to S3     │
 │ 7 resources (semantic catalogue)   8 audit log (no data)   9 kill switches   │
 └───────────────┬───────────────────────────────────────┬──────────────────────┘
                 │ PrivateLink                            │ S3 VPC endpoint, KMS per client
                 ▼                                        ▼
       Snowflake (per region)                    s3://<results-bucket>/<client_id>/...
       child read-only role, row access
       + masking policies, semantic views
```

Request lifecycle for a tool call:
1. Verify caller JWT. Extract `sub` (user), `client_id` (exactly one child client), `caller` (registered app id), `session_id`.
2. Check the user is entitled to `client_id`, and the caller is allowed to use this tool.
3. Check kill switches for the tool and the client.
4. Build or validate SQL (SQL guard).
5. Apply rate limits and the cost check. Large queries return `needs_approval` instead of running.
6. Run as the child client's read-only role, with a query tag.
7. Pass results through the output pipeline. Return a small preview plus a result handle.
8. Write one audit record.

## 4. Tech stack (use these unless there is a strong reason not to)

| Concern | Choice |
|---|---|
| Language | Python 3.12 |
| MCP | Official MCP Python SDK 2.x (`mcp>=2.3,<3`, server class `MCPServer`), Streamable HTTP transport. 2.x speaks the 2026-07-28 protocol revision and still serves 2025-era clients. Turn the SDK's built-in OpenTelemetry tracing off, or configure it so tool arguments and results are never recorded. |
| HTTP | Starlette/uvicorn (what the SDK uses) |
| Snowflake | `snowflake-connector-python` with key-pair auth |
| SQL parsing | `sqlglot` with the `snowflake` dialect |
| Config / models | `pydantic` v2, `pydantic-settings` |
| JWT | `pyjwt[crypto]` with JWKS fetch and cache |
| Results | `pyarrow` (Parquet), `boto3` (S3 with SSE-KMS) |
| Local test database | `duckdb` as a fake backend so unit tests need no Snowflake |
| Tooling | `uv`, `ruff`, `mypy --strict`, `pytest`, `pytest-asyncio` |
| Container | `registry.access.redhat.com/ubi9/python-312`, non-root user |

## 5. Repository layout

```
snowflake-mcp-gateway/
  README.md                    this file
  pyproject.toml
  Makefile                     make check = ruff + mypy + pytest (unit only)
  Dockerfile
  config/
    gateway.example.yaml       non-secret settings
    callers.example.yaml       registered callers and their allowed tools
    entitlements.example.yaml  v1 user -> child clients (later: a real store)
    killswitch.yaml            disabled tools/clients, hot-reloaded
  catalogue/                   semantic YAML (dev copy; prod is loaded from Git/S3)
  src/gateway/
    app.py                     builds the MCP server and HTTP app
    settings.py
    auth.py                    JWT verification, CallerContext
    entitlements.py
    registry.py                fixed tools, descriptions, annotations
    tools/
      search_semantic.py
      describe.py
      query_metric.py
      run_query.py
    sqlguard.py
    limits.py                  rate limits, cost check
    backends/
      base.py                  Backend protocol
      duckdb_fake.py
      snowflake_connector.py
      snowflake_managed_mcp.py (later, optional)
    catalogue/
      model.py                 pydantic models in Snowflake semantic-model shape
      yaml_source.py
      snowflake_source.py      (later) reads semantic views
      search.py
      compile.py               metric request -> SQL
      scrub.py                 removes sample values, verified queries
    resources.py               MCP resources + change notifications
    output.py                  preview, PII scan, other-client scan, size cap, Parquet to S3
    audit.py
    killswitch.py
    admin.py                   /admin/refresh-catalogue (CI only)
  tests/
    unit/ ...
    redteam/                   attack corpus, runs in CI
    integration/               marked, needs a dev Snowflake account
```

## 6. Tools (the complete, fixed list)

All tools are registered in `registry.py` with descriptions written in code. All have annotations `readOnlyHint: true`, `destructiveHint: false`, `idempotentHint: true`, `openWorldHint: false`. Never load tool names or descriptions from Snowflake or from config at runtime.

### `search_semantic`
- Input: `query: str` (max 200 chars), `kind: "metric" | "dimension" | "any" = "any"`, `limit: int = 10` (max 25).
- Output: list of `{name, kind, view, description, synonyms}` visible to the caller's client.
- Never touches client data. Reads only the catalogue.

### `describe`
- Input: `name: str` (a metric, dimension, or view name returned by search).
- Output: definition, grain, joins, allowed filters, unit, and the resource URI. Same text as the matching resource.

### `query_metric` (preferred way to get numbers)
- Input: `metrics: list[str]` (1 to 5), `dimensions: list[str]` (0 to 5), `filters: list[{dimension, op, value}]` with `op` in `= != > >= < <= in between`, `time_range: {start, end}` (required for time-grained metrics), `order_by`, `limit` (max 1000).
- The gateway compiles SQL itself from the catalogue (`catalogue/compile.py`), then sends it through the SQL guard anyway.
- Output: see "Result shape" below.

### `run_query` (fallback)
- Input: `sql: str` (max 8 KB), `purpose: str` (max 200 chars, logged).
- Must pass the SQL guard. Large or expensive queries return `needs_approval`.
- Output: see "Result shape".

### Result shape (both query tools)
```json
{
  "status": "ok | needs_approval | blocked",
  "reason": "string, only when not ok, never contains data",
  "columns": [{"name": "...", "type": "..."}],
  "row_count": 1234,
  "preview": [["..."], ["..."]],
  "preview_rows": 20,
  "result_handle": "res_<uuid>",
  "masked_columns": ["..."],
  "query_id": "snowflake query id"
}
```
- `preview` is at most 20 rows and 20 columns, cells truncated to 200 chars.
- The full result goes to `s3://<bucket>/<client_id>/<session_id>/<result_handle>.parquet` with SSE-KMS using that client's key. The session manager copies it into `/work/<session>`.

## 7. Resources (semantic catalogue)

- URI templates:
  - `semantic://{client_id}/view/{view_name}`
  - `semantic://{client_id}/metric/{metric_name}`
  - `semantic://{client_id}/dimension/{dimension_name}`
- `resources/list` returns only entries visible to the caller's `client_id` (shared definitions plus that child's overlay). A `client_id` in a URI that differs from the token's client is rejected.
- Content type `application/yaml`, in Snowflake semantic-model YAML shape.
- **Scrub before publishing:** remove `sample_values`, `verified_queries`, and any comment matching the PII scanner. CI should also reject these fields in shared views.
- Support `resources/subscribe`. When the catalogue refreshes, send `notifications/resources/updated` for changed URIs and `notifications/resources/list_changed` if the set changed.

### Catalogue sources and sync
- **v1 (`yaml_source.py`):** load YAML files from a path or S3 prefix at startup and on refresh.
- **Later (`snowflake_source.py`):** for each child role, run `SHOW SEMANTIC VIEWS` and `DESCRIBE SEMANTIC VIEW <name>` **as that child's role**, so Snowflake grants decide what each child can see. If available, use Snowflake's YAML export function (believed to be `SYSTEM$READ_YAML_FROM_SEMANTIC_VIEW`; verify before relying on it).
- **Refresh trigger:** CI calls `POST /admin/refresh-catalogue` after it deploys a catalogue change. This endpoint is authenticated with a separate CI credential and is not exposed through MCP.
- **Drift check:** an hourly job compares Snowflake's definitions with the loaded copy and emits a metric and alert on any difference.

## 8. Auth and role scoping

### Callers to gateway
- Every MCP request carries a bearer JWT issued by our IdP or by the session manager's token service. Verify signature (JWKS), `iss`, `aud = snowflake-mcp-gateway`, `exp` (max 15 minutes), and required claims: `sub`, `client_id`, `caller`, `session_id`.
- Reject a token that has no `client_id` or more than one.
- `config/callers.yaml` lists each caller id and the tools it may use. Example: the catalogue UI may only use `search_semantic`, `describe` and resources.

### Entitlements
- Before any tool runs, check `(sub, client_id)` against the entitlement store. v1 is `entitlements.yaml`; keep it behind an interface so it can move to DynamoDB or the IdP later.
- The gateway re-checks even if the caller already checked.

### Gateway to Snowflake
- One service user per Snowflake account, key-pair auth, private key from AWS Secrets Manager (read at startup, never logged).
- The service user is granted each child client's read-only role, and nothing else. No admin roles.
- Map `client_id` to `(account, role, warehouse)` in config.
- **One connection pool per child role.** A connection is opened with that role and `USE SECONDARY ROLES NONE`, and is never reused for another role. Assert the current role before every query (`SELECT CURRENT_ROLE()` on checkout, cached per connection).
- Set on every query: `QUERY_TAG` as JSON `{client_id, user, session_id, tool, caller}`, `STATEMENT_TIMEOUT_IN_SECONDS`, the client's warehouse.
- Snowflake side (document in `docs/snowflake-setup.sql`, applied by a human): child read-only roles, row access policies on the client hierarchy, masking policies on PII columns, a dedicated warehouse with a resource monitor, network policy allowing only the PrivateLink endpoint.

## 9. Guardrails (all must exist before go-live)

1. **Who is calling:** valid JWT, one client, registered caller, per-caller tool allowlist.
2. **Client scoping:** child read-only role per call, secondary roles off, per-role connection pools, per-client result storage and KMS key, no cross-client caching.
3. **Tool surface:** fixed read-only tools, descriptions in code, no sampling, no prompts, no elicitation, no roots. The server never declares those capabilities.
4. **SQL guard (`sqlguard.py`):** parse with sqlglot (never regex). Allow exactly one `SELECT` (CTEs allowed). Reject DML, DDL, `CALL`, `EXECUTE`, `COPY`, `PUT`, `GET`, `USE`, `SET`, `ALTER SESSION`, multiple statements, and comments that hide statements. Every table reference must be in the client's allowlisted databases/schemas/views. Deny functions that reach outside or reveal metadata (external functions, `SYSTEM$*`, `GET_PRESIGNED_URL`, `INFORMATION_SCHEMA` and `ACCOUNT_USAGE` outside an allowlist). Deny `SELECT *` on tables tagged wide. Inject or tighten `LIMIT`.
5. **Execution limits:** statement timeout, row cap, warehouse resource monitor, rate limits per user and per client, cost check using `EXPLAIN` (bytes or partitions scanned). Over threshold returns `needs_approval`.
6. **Output pipeline (`output.py`):** preview only; full result to S3 Parquet. Scan the preview and column names for card numbers (Luhn check), IBANs, emails, phone numbers, national ids, and names of other clients (from the client registry). Masked or blocked results say so in `masked_columns` or `reason`. Hard size cap on every response.
7. **Resources:** filtered per client by grants, scrubbed of sample values and verified queries.
8. **Network:** private subnets, internal load balancer only, egress only to Snowflake PrivateLink, S3 and Secrets Manager VPC endpoints, and the IdP JWKS endpoint. The nono sandbox security group cannot reach the gateway.
9. **Audit (`audit.py`):** one JSON record per call: time, caller, user, client_id, session_id, tool, SQL hash (SHA-256), normalized SQL template without literals, row count, bytes, duration, status, guard decisions, Snowflake query id. **Never** result values, literals from filters, or previews. Audit goes to its own log group, not the app log.
10. **Kill switches and tests:** `killswitch.yaml` can disable any tool or client without a deploy (hot reload). The red-team suite in `tests/redteam/` runs in CI on every change.

Errors: return generic messages with an error id. Never echo SQL, values, or Snowflake error text that might contain data back to the caller; log the detail to the audit store only.

## 10. Build plan

Each step ends with a commit. Keep each step small. Run `make check` before every commit. If a step needs a real Snowflake account, AWS resources, or a decision not covered here, stop and ask the human.

### Step 0: Scaffold
- Create the layout above, `pyproject.toml` with the stack, `Makefile` (`check`, `test`, `lint`, `typecheck`, `run`), `.gitignore`, pre-commit with ruff.
- **Accept:** `make check` passes on an empty test suite; `uv run python -c "import gateway"` works.

### Step 1: Settings and config models
- `settings.py` with pydantic-settings (env prefix `GATEWAY_`). Models for callers, entitlements, client map (`client_id -> account, role, warehouse, allowed_schemas, kms_key_arn`), limits, killswitch.
- Example config files in `config/`. No secrets in any file.
- **Accept:** unit tests load the example configs; invalid config (unknown field, missing role) fails at startup with a clear message.

### Step 2: MCP server skeleton
- `app.py` builds the server with the MCP SDK (`MCPServer`), Streamable HTTP at `/mcp`, stateless so any task behind the load balancer can serve any request, plus `/healthz`. Register the four tools with stub handlers that return `status: blocked, reason: not implemented`. Set annotations. Declare only the `tools` and `resources` capabilities.
- **Accept:** a test using `langchain.mcp.MCPAdapter` as the client (the same client the session manager uses) lists exactly four tools with the expected names, descriptions and annotations; `prompts/list` and sampling are not offered; `/healthz` returns 200.

### Step 3: Caller authentication
- `auth.py`: verify JWT (JWKS with caching; a static test key in tests), build `CallerContext(user, client_id, caller, session_id)`. Enforce the per-caller tool allowlist.
- **Accept:** tests for missing token, expired token, wrong audience, no `client_id`, two client ids, unknown caller, caller calling a tool it is not allowed: all rejected with 401/403 and no tool code runs.

### Step 4: Entitlements and kill switches
- `entitlements.py` (YAML-backed, interface for later stores) and `killswitch.py` (file watched, reloaded without restart).
- **Accept:** user not entitled to the client is rejected; toggling a tool or client in `killswitch.yaml` takes effect within 5 seconds in a test.

### Step 5: Backend interface and DuckDB fake
- `backends/base.py`: `Backend.run(sql, ctx, limits) -> ResultSet` and `Backend.explain(sql, ctx) -> Cost`. `duckdb_fake.py` loads two synthetic child clients (`client_a`, `client_b`) with payments-like tables and fake card numbers.
- **Accept:** unit tests run queries per client against the fake; a query for `client_a` can never see `client_b` tables (enforced by per-client schema mapping).

### Step 6: SQL guard
- `sqlguard.py` as specified in guardrail 4. Return a structured decision `{allowed, reasons, rewritten_sql}`.
- Build `tests/redteam/sql_attacks.yaml`: stacked statements, comment tricks, `CALL`, `COPY INTO @stage`, external functions, `SYSTEM$` calls, other client's schema, fully qualified cross-database names, `SELECT *` on wide tables, Unicode lookalikes, giant `IN` lists, recursive CTE bombs.
- **Accept:** every attack is blocked; a set of legitimate queries passes; `LIMIT` is injected or tightened.

### Step 7: Output pipeline
- `output.py`: preview limits, PII scanner (Luhn for card numbers, IBAN checksum, email, phone, national id patterns), other-client name scan using the client registry, size cap, Parquet writer to S3 (use `moto` in tests) at the per-client prefix with SSE-KMS.
- **Accept:** fake card numbers in results are masked in the preview; a result mentioning another client's name is blocked; the Parquet object lands only under the caller's client prefix; the response JSON never exceeds the cap.

### Step 8: Wire `run_query`
- Full pipeline: auth, entitlement, killswitch, SQL guard, limits, backend, output, audit.
- **Accept:** end-to-end test with the SDK client against the DuckDB fake returns a preview and handle; blocked SQL returns `status: blocked` with a generic reason; an audit record is written and contains no result values.

### Step 9: Semantic catalogue (YAML source)
- `catalogue/model.py` mirrors Snowflake's semantic-model YAML (tables, dimensions, time_dimensions, facts, metrics, relationships, synonyms, descriptions). `yaml_source.py` loads shared files plus per-child overlays. `scrub.py` strips `sample_values` and `verified_queries`. `search.py` does keyword and synonym search (no embeddings in v1).
- Implement `search_semantic` and `describe`.
- **Accept:** search finds metrics by name and synonym; `client_a` never sees `client_b`'s overlay; scrubbed fields never appear in any output.

### Step 10: `query_metric`
- `compile.py` turns a metric request into SQL using the catalogue's joins and expressions. Output goes through the SQL guard like any other SQL.
- **Accept:** golden-SQL tests for a set of requests; unknown metric or dimension returns a clear error listing close matches; filters only on dimensions the catalogue allows.

### Step 11: Resources
- `resources.py`: templates, list, read, subscribe, notifications. `admin.py`: `POST /admin/refresh-catalogue` with a separate CI credential, reloads the catalogue, diffs, and notifies subscribers.
- **Accept:** resource list is filtered per client; reading another client's URI is rejected; after a refresh, a subscribed test client receives `resources/updated` for exactly the changed URIs.

### Step 12: Limits and approval
- `limits.py`: token-bucket rate limits per user and per client (in memory in v1, interface for Redis or DynamoDB later), cost check via `backend.explain`, `needs_approval` result with a cost summary and an approval token the caller can resubmit with (approval itself happens in the caller's UI, signed by the session manager).
- **Accept:** burst tests trip the limit; an expensive query returns `needs_approval`; resubmitting with a valid approval token runs it; a forged token is rejected.

### Step 13: Snowflake connector backend
- `snowflake_connector.py`: key-pair auth from Secrets Manager, per-role pools, `USE SECONDARY ROLES NONE`, current-role assertion, query tag, timeout, warehouse. `EXPLAIN` for cost. Write `docs/snowflake-setup.sql` for the human to apply.
- **Stop and ask** for a dev Snowflake account, the service user, and two synthetic child roles before running integration tests.
- **Accept (integration, marked):** queries run as the right role (checked via `CURRENT_ROLE()` and query history); `QUERY_TAG` appears in query history; a query against the other child's table fails at Snowflake even if the SQL guard is bypassed in the test.

### Step 14: Audit and observability
- Finalize `audit.py` (separate sink), metrics (calls, blocks by reason, latency, rows), and alarms for repeated blocks per user.
- **Accept:** a test scans every log line produced during the full test suite for the synthetic sensitive values (fake card numbers, client names in results) and finds none.

### Step 15: Container and deployment notes
- `Dockerfile` on UBI 9 Python 3.12, non-root, read-only root filesystem friendly. `docs/deploy.md` covering ECS Fargate service, internal load balancer, security groups (session manager and approved callers in, nono sandbox out), VPC endpoints, PrivateLink, Secrets Manager, per-client KMS keys.
- **Accept:** image builds; container starts with example config and serves `/healthz`; `docs/deploy.md` lists every network path and nothing else.

### Step 16: Red-team suite in CI
- Collect all attack tests into `tests/redteam/` and add: prompt-injection text inside data values (must come back as data, never change behavior), tool description drift, cross-client resource URIs, oversized outputs, token replay with a different `client_id`.
- **Accept:** CI fails if any red-team test passes an attack.

### Step 17 (later): Snowflake semantic views source
- `snowflake_source.py` reading semantic views per child role, scrub, hourly drift check. Switch by config.
- **Accept:** same tests as step 9 and 11 pass with the Snowflake source in the integration environment.

### Step 18 (optional, later): Snowflake managed MCP server as a backend
- `snowflake_managed_mcp.py` implementing `Backend` by calling Snowflake's managed MCP server (read-only SQL tool, and Cortex Analyst only if approved) with an OAuth token scoped to the child role (`session:role:<ROLE>`). Bulk extracts stay on the connector backend. Pin our own tool descriptions; ignore server-supplied text.
- **Check first:** whether the managed server accepts service authentication other than interactive OAuth, whether `QUERY_TAG` is set through its SQL tool, and that Cortex cross-region inference is off.

## 11. How the session manager uses the gateway

- The manager connects with `langchain.mcp.MCPAdapter` (`langchain[mcp]>=1.4`, built on FastMCP 4, which uses MCP SDK 2.x), passing a `fastmcp.Client` with a per-session bearer JWT for the session's child client. This namespace is in beta, so pin exact versions. The older `langchain-mcp-adapters` package still requires SDK 1.x and is not used.
- Do not register an elicitation handler on the client; approvals go through our own UI.
- `langchain.mcp` wraps tools only. Code that needs the catalogue resources reads them with the FastMCP client directly (`client.read_resource(...)`).
- It wraps the four tools with its own fixed descriptions and does not accept `tools/list_changed` updates mid-session.
- It copies Parquet results from S3 into `/work/<session>` for the nono sandbox. The sandbox never calls the gateway.

## 12. Rules for whoever builds this

- Never add a tool that writes, sends, or exports. Never declare sampling, prompts, elicitation, or roots.
- Never log result values, filter literals, previews, or raw Snowflake error text outside the audit store.
- Never reuse a Snowflake connection across child roles.
- Never read tool names or descriptions from Snowflake or config at runtime.
- Tests use synthetic clients only. No real client data in the repo, fixtures, or catalogue examples.
- Stop and ask before: creating cloud resources, touching a real Snowflake account, changing guardrail behavior, or adding a dependency outside section 4.
