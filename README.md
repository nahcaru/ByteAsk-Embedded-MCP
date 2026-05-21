# ByteAsk Embedded MCP

The open-source **MCP server** behind [ByteAsk Embedded Docs](https://docs.byteask.ai/embedded):
a **source-grounded, page-cited evidence-retrieval** server for coding agents
(Claude Code, Codex, Cursor) that write firmware / driver / protocol code and
need *exact* facts — SunSpec points, register offsets, Modbus function codes,
trip thresholds, SCPI commands, API symbols.

It returns **verbatim snippets with page citations** — never an authored answer —
and when nothing is relevant enough it says **no match** rather than fabricate.
**Every document is treated equally** (no authority/trust layer, no filters).

> **What's in this repo:** the MCP *server* — tools, transports (stdio +
> Streamable HTTP), bearer auth, DNS-rebinding protection, and result rendering —
> plus a small, pluggable retrieval interface.
>
> **What's *not* in this repo:** the retrieval engine and the document corpus.
> How documents are parsed, chunked, embedded, and ranked, and the licensed
> source material itself, sit behind the [`SearchBackend`](src/byteask_embedded_mcp/backend.py)
> seam and power the hosted endpoint at `https://mcp.byteask.ai/mcp`. This repo
> ships an in-memory **`SampleBackend`** (a few illustrative, public-knowledge
> records) so the server runs out of the box.

---

## The tools

Two retrieval tools plus a request tool. Input is natural language (or an exact
identifier). Output is compact markdown.

- **`search_docs(query, limit=8)`** — search the corpus; return ranked evidence.
  Each hit: document title, a section + page citation, the verbatim snippet, and
  a `result_id`. A "no confident match" response means *not found — do not
  fabricate*.
- **`get_context(result_id)`** — expand a hit to its full source section.
- **`request_document(request)`** — ask for a missing document to be added
  (logged server-side).

Example output:

```markdown
## Results for "what Modbus function code writes multiple registers"

### Sample — Modbus Application Protocol (illustrative) — §6.12, p.30
> Function code 16 (0x10), Write Multiple Registers, writes a block of contiguous
> holding registers (1 to 123 registers) in a remote device. ...
_ref: sample:modbus-fc16_
```

---

## Quickstart (local)

Requires Python ≥ 3.10 and [`uv`](https://docs.astral.sh/uv/).

```bash
uv sync
uv run byteask-embedded-mcp            # run as an MCP server (stdio)
```

That's it — the bundled `SampleBackend` serves a couple of illustrative records,
so `search_docs` works immediately. `uv run pytest` runs the offline tests.

---

## Connect a client

### Hosted endpoint (no install)

The hosted server speaks **Streamable HTTP** at `https://mcp.byteask.ai/mcp`:

```bash
claude mcp add --transport http byteask-embedded-docs https://mcp.byteask.ai/mcp
```

Or via the `mcp-remote` bridge (Codex, Cursor, …):

```json
{ "mcpServers": { "byteask-embedded-docs": {
  "command": "npx", "args": ["-y", "mcp-remote", "https://mcp.byteask.ai/mcp"] } } }
```

### Local (this repo)

The project-scoped [`.mcp.json`](.mcp.json) registers the stdio server for
clients that read it. Manually, for Claude Code:

```bash
claude mcp add byteask-embedded-docs -- uv run byteask-embedded-mcp
```

---

## Plug in your own retrieval

The server depends only on a two-method interface
([`backend.py`](src/byteask_embedded_mcp/backend.py)):

```python
class SearchBackend(Protocol):
    def search(self, query, limit=8, effort=None) -> dict: ...
    def get_context(self, result_id, effort=None) -> dict: ...
```

Implement it, expose a factory `make_backend(config) -> SearchBackend`, and point
the server at it:

```bash
BYTEASK_BACKEND="my_pkg.my_module:make_backend"
```

The exact return-value contracts are documented at the top of `backend.py`.

---

## Running over HTTP

```bash
MCP_TRANSPORT=http MCP_HTTP_AUTH_TOKEN=$(openssl rand -hex 32) \
  uv run byteask-embedded-mcp --host 0.0.0.0 --port 8000
```

Clients then send `Authorization: Bearer <token>`. The bundled bearer check is a
shared-secret **stub** — replace it with real auth (OAuth 2.1 resource server,
mTLS, or a trusted reverse proxy) before exposing publicly. DNS-rebinding
protection stays on independently via `MCP_ALLOWED_HOSTS`.

---

## Configuration (`.env`, all env-overridable)

| Var | Default | Notes |
|---|---|---|
| `BYTEASK_BACKEND` | — | `module:callable` returning a `SearchBackend`; empty → `SampleBackend` |
| `BYTEASK_LOGS` | `logs` | where query / request JSONL logs go |
| `MCP_TRANSPORT` | `stdio` | `stdio` (local agents) or `http` |
| `MCP_HTTP_HOST` / `MCP_HTTP_PORT` | `127.0.0.1` / `8000` | HTTP bind address |
| `MCP_HTTP_AUTH_TOKEN` | — | bearer token for HTTP (empty = unauthenticated, dev only) |
| `MCP_ALLOWED_HOSTS` | — | comma-separated hosts allowed in the `Host` header (`*` disables) |
| `LOG_LEVEL` | `INFO` | stderr log verbosity |

See [`.env.example`](.env.example).

---

## Project layout

```
src/byteask_embedded_mcp/
  server.py     # FastMCP app + 3 tools (search_docs, get_context, request_document)
  backend.py    # SearchBackend protocol + in-memory SampleBackend (swap for real retrieval)
  render.py     # structured result -> compact markdown
  http_auth.py  # Streamable HTTP entrypoint + stub bearer-token guard
  config.py     # server config (transport, logging, backend selection)
  schemas.py    # Hit / Section result types
  obs.py        # per-call JSONL logging
tests/          # offline unit tests (renderer, backend, server tools)
```

## Security notes

- **stdout stays clean** in stdio mode (it is the JSON-RPC channel); all logs go
  to stderr / `logs/*.jsonl`.
- The HTTP **bearer check is a stub** — unauthenticated if no token is set, and a
  shared secret at best. Harden it before exposing widely.
- **DNS-rebinding protection** is on by default for the HTTP transport.

## License

MIT — see [LICENSE](LICENSE).
