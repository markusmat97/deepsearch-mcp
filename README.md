# markitdownwebsearcher

A local-first **Model Context Protocol (MCP)** server that gives LLM clients
(Claude Desktop, Cursor, Windsurf) a low-token web search tool. Instead of
dumping whole pages into context, it scrapes results, reranks them locally with
two small transformer models, and returns only the most relevant, deduplicated
excerpts.

## Features

- **Local two-stage reranking** — a bi-encoder (`all-MiniLM-L6-v2`) pre-filter
  followed by a cross-encoder (`ms-marco-MiniLM-L-6-v2`) rerank with
  sigmoid-normalized scoring. No external ranking API.
- **SSRF-validated fetch** — each hostname is resolved once and the IP checked
  against private/loopback/link-local/reserved/multicast/CGNAT ranges before
  connecting; the connection is pinned to the validated IP with TLS hostname
  verification.
- **Streaming byte ceiling** — page downloads abort at 512 KB to bound memory.
- **Deduplication** — overlapping sentence-window chunks with Jaccard
  similarity filtering on longer blocks.
- **Token report** — reports retained vs. extracted token counts (a rough local
  heuristic, not a benchmark against any other tool).

## Requirements

Python 3.10+. Install dependencies:

```bash
pip install -r requirements.txt
```

First run downloads the two models (~180 MB) from Hugging Face and caches them.

## Use with Claude Desktop

Add to `claude_desktop_config.json` (Settings → Developer → Edit Config),
using absolute paths:

```json
{
  "mcpServers": {
    "deep-search": {
      "command": "C:\\path\\to\\venv\\Scripts\\python.exe",
      "args": ["C:\\path\\to\\markitdownwebsearcher.py"]
    }
  }
}
```

Fully quit and reopen Claude Desktop. The `deep_search` tool should appear.

## Limitations & scope

- **Single-user, local use.** The fetch path is SSRF-validated but not audited
  for hostile multi-tenant deployment.
- **Search depends on scraping the DuckDuckGo HTML endpoint** (`html.duckduckgo.com`).
  This is unofficial, may break without notice, and is subject to DuckDuckGo's
  terms of service. Treat search reliability as best-effort.
- Pages are fetched sequentially with no concurrency; large source counts
  increase latency, which can hit a client's tool-call timeout.
- Large pages (e.g. some Wikipedia articles) may be skipped by the 512 KB ceiling.

## License

[MIT ]
