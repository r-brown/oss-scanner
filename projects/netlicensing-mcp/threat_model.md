# Threat model: NetLicensing MCP server

## What this project does and where untrusted input enters

`netlicensing-mcp` is a Model Context Protocol (MCP) server, written with FastMCP, that exposes the Labs64
NetLicensing REST API (products, modules, license templates, licensees, licenses, bundles, tokens,
transactions, payment methods) as MCP tools and prompts. An AI agent calls the tools on a user's behalf, and
the server turns each call into an authenticated HTTPS request to NetLicensing.

It runs in two modes:

- **stdio** (default): started by a desktop MCP client such as Claude Desktop or an IDE. One user, one API key
  from `NETLICENSING_API_KEY`.
- **HTTP** (`python -m netlicensing_mcp.server http`, or `MCP_TRANSPORT=http`): a streamable-HTTP server that
  can be shared by many tenants. The API key is read per request from the `X-NetLicensing-API-Key` header or
  `Authorization: Bearer ...` (`ApiKeyMiddleware`), held in a `contextvars.ContextVar`, and the server runs
  stateless. `/health` is open. It binds `127.0.0.1:8000` by default; `deploy/aws/` deploys it behind a load
  balancer.

Untrusted input reaches the code through:

- **Tool arguments chosen by an LLM.** The LLM may itself be steered by prompt injection from data it has read
  (licensee names, product descriptions and other free text stored in a NetLicensing account). Treat every
  tool argument as attacker-influenced: identifiers, numbers, names, free text, dates, booleans, and the
  confirmation token.
- **HTTP requests** in HTTP mode: headers (API key), the MCP JSON-RPC body, the path and the `Origin`/`Host`
  headers.
- **Responses from the NetLicensing API**, which are parsed, redacted, wrapped into the response envelope
  and returned to the LLM. Their text fields are attacker-influenced too (stored by other parties).
- **Environment variables** (operator-controlled).

## Trust model

- **The caller owns their API key.** A tool call may do anything that key permits in NetLicensing. That is
  the product. What matters is that the server does not let a caller do more than their key allows, see
  another caller's key or data, or be tricked into acting outside the intended endpoint.
- **In HTTP mode tenants are separate.** One caller must never be able to use, observe, consume or invalidate
  another caller's API key, confirmation token, cached client state or response data, including under
  concurrent requests. Per-request key handling must not leak across requests (`api_key_ctx`).
- **The LLM is not a trusted principal.** Destructive operations (delete, `force_cascade`, token revocation,
  transfers, updates to payment methods) are gated by the confirmation-token flow in `safety.py`: a preview
  returns a short-lived, single-use token bound to the operation, the target and the caller; the second call
  must present it. Bypassing, forging, reusing or cross-caller use of a token **is** a finding.
- **The operator is trusted**: environment variables, the deployment, and `NETLICENSING_BASE_URL`.
- **NetLicensing is trusted as the upstream**, but its payload is data, never instructions or paths.

## Components that matter most / least

Most important:

1. `client.py`: request construction. `_validated_path` rejects identifiers that would change the REST path
   (`../`, encoded variants); every identifier interpolated into a path must go through it (see
   GHSA-hxpf-9xvq-wph8 and `tests/test_path_traversal.py`). Also Basic-auth header construction, the shared
   `httpx.AsyncClient`, error unwrapping and what is logged.
2. `server.py` `ApiKeyMiddleware` and API-key resolution (`api_key_ctx`): header precedence, no query-string
   form, stateless operation, missing-key handling (503), the `NETLICENSING_ALLOW_DEMO` demo fallback, `/health`,
   and bind defaults.
3. `safety.py`: confirmation tokens (entropy, TTL, single use, owner binding, store growth).
4. `redaction.py` and `responses.py`: default redaction of `apiKey`, `licenseeSecret`, `nodeSecret`,
   `password`, `secret`; masking of APIKEY token numbers and SHOP URLs on reads; one-time display tagging for
   newly created tokens; `console_url` generation (an APIKEY token's `number` must never appear in a URL).
   A secret that reaches tool output or a log unmasked is a finding.
5. `tools/*.py`: per-entity payload building, boolean/empty-string "leave unchanged" semantics, anything that
   turns a caller value into a different upstream field or endpoint (parameter injection into the
   form-encoded body or query string).
6. `workflows/*` (`get_customer_health`, `explain_validation`): they are documented as **read-only**. A path
   by which they consume a floating session, quota or node-lock slot, or modify data, is a finding.

Less important: `prompts/audit.py` (static prompt text), `deploy/` scripts, `healthcheck.py`, `glama.json` and
`server.json` (registry metadata).

Out of scope: vulnerabilities in the NetLicensing service itself, in FastMCP/`mcp`, httpx and other
dependencies, and anything that needs a malicious operator.

## How to exercise it

The image is built with all dependencies installed in `/src/.venv` (already on `PATH`). The checkout is at
`/src`. There is no network, so the NetLicensing API is mocked.

- Tests: `cd /src && pytest -q --no-cov`. They mock the upstream with `httpx` fixtures (see
  `tests/test_tools.py`, `tests/test_auth_middleware.py`, `tests/test_stateless_http_key.py`).
- Run the server: `python -m netlicensing_mcp.server http` (set `NETLICENSING_API_KEY=test` or
  `NETLICENSING_ALLOW_DEMO=true`), then talk to it on `127.0.0.1:8000` with `curl` using MCP JSON-RPC, or
  start a local fake NetLicensing (a few lines of Python) and point `NETLICENSING_BASE_URL` at it to observe
  exactly what the server sends.
- Lint and types: `ruff check .`, `mypy src/`.

A good reproducer is a pytest test or a short `curl` against a locally started server and a local fake
upstream, showing the outgoing request or the response that crosses the boundary.

## How you rate severity

- **Critical**: reading, using or replacing another tenant's API key or data in HTTP mode; a way to make the
  server send authenticated requests to a different endpoint or host than intended (path traversal, SSRF,
  `NETLICENSING_BASE_URL` override from a request); remote code execution; a confirmation-token bypass that
  lets a destructive operation run without the preview step, or with another caller's token.
- **High**: an API key, licensee secret or node secret returned in tool output or written to a log in clear
  text where redaction should have masked it; the demo fallback activating without the explicit opt-in;
  parameter injection that changes the meaning of an upstream call (for example turning a read into a write,
  or adding `forceCascade`); a "read-only" workflow that mutates state; an authentication bypass on the HTTP
  transport.
- **Medium**: remote denial of service from one small request (unbounded memory in the token store, large
  response handling, regex backtracking); information disclosure in error messages; missing validation that
  lets an LLM cause an unintended, but caller-permitted, change; DNS rebinding or cross-origin access to a
  locally bound HTTP server.
- **Low**: hardening gaps with no demonstrated impact; verbose logging without secrets.
- Memory-safety does not apply (Python). A caller using their own key to do what that key allows is not a
  vulnerability. A claim without a working reproducer is at most Medium.

## Anything to leave alone

- Anything a caller can do with their own valid API key within what NetLicensing allows.
- Prompt injection that merely makes an LLM client *ask* for something, when the server's confirmation flow
  still gates the destructive step. The finding is a gap in the server's own gates.
- The documented demo mode (`NETLICENSING_ALLOW_DEMO=true`) when explicitly enabled, and the `demo:demo`
  sandbox credentials.
- Unauthenticated `/health`, and the default `127.0.0.1` bind.
- Operator-controlled configuration (`NETLICENSING_BASE_URL`, `MCP_REDACT_FIELDS`, `MCP_VERBOSE` output).
- The `deploy/` scripts, `assets/`, `tests/`, and dependency CVEs with no reachable path from this code
  (CI already runs `pip-audit` with documented ignores).

## Reports and patches

One finding per report, with file and line, the trust boundary crossed, and a reproducer that runs inside the
image without network access. A minimal patch in the style of the surrounding code is preferred, with a test in
`tests/` next to the existing ones. Keep tool names, argument names and the response envelope stable.
