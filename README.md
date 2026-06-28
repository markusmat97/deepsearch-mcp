# deepsearch-mcp

Claude was eating my token budget in 3-4 prompts during research sessions. The culprit was simple — I was dumping entire web pages into context. This MCP server fetches broadly, ranks locally using two small transformer models, and returns only the passages that matter. No full pages, no external ranking APIs.

Works with Claude Desktop, Cursor, and Windsurf. Single-user, runs on your machine. I haven't tested it in multi-tenant or hostile network environments and wouldn't recommend it there.
## What happens when you run a query

- Hits a local SearXNG instance and paginates until it has links from enough distinct domains. If SearXNG is unreachable, it falls back to DuckDuckGo's HTML endpoint — unofficial, occasionally breaks, use it as a last resort.
- Each page goes through a fetcher that pins IPs, checks for SSRF risks, and cuts off downloads at 5 MB. Avoids memory blowouts with 8 workers running at once.
- Article text gets pulled from HTML, chopped into overlapping 3-sentence windows. Anything that looks like SEO filler gets dropped before ranking.
- Two-stage local ranking: bi-encoder does a fast pre-filter, cross-encoder scores what survives. Runs on your CPU or GPU, no API calls.
- Near-duplicate passages are removed via Jaccard comparison. Results are capped per domain and sorted newest-first where dates exist.
- You get excerpts with source URLs and dates. Synthesis is your LLM's job.
## What's under the hood

- **Two-stage reranking** — bi-encoder (all-MiniLM-L6-v2) narrows candidates fast, cross-encoder (ms-marco-MiniLM-L-6-v2) scores the survivors. No ranking API, no per-query cost.
- **SSRF-hardened fetcher** — hostnames are resolved once, IPs checked against private/loopback/reserved/multicast/CGNAT ranges before any connection opens. Pinned to that validated IP, TLS still runs against the real hostname. Redirects are followed manually, re-validated at each hop, hard hop limit enforced.
- **5 MB streaming ceiling** — download aborts the moment it crosses MAX_PAGE_SIZE_BYTES.
- **Jaccard deduplication** — overlapping passages compared before output. Near-duplicates dropped.
- **Token report** — each response includes output token count vs raw extracted tokens. Local heuristic, rough estimate.
- **Dual-pass recency** — first pass uses a time filter, second pass runs without it to catch evergreen sources that don't surface in recency-filtered results.

## Setup

Python 3.10 or newer. You'll need a local SearXNG instance with JSON output enabled — the server expects it at http://localhost:8080/search by default.
```bash
pip install -r requirements.txt
```
First run downloads the two ranking models from Hugging Face (~180 MB combined) and caches them locally.


## Adding it to Claude Desktop

Open `claude_desktop_config.json`  via Settings → Developer → Edit Config. Add this block with your actual absolute paths:
```json
{
  "mcpServers": {
    "deep-search": {
      "command": "C:\\path\\to\\venv\\Scripts\\python.exe",
      "args": ["C:\\path\\to\\deepsearch-mcp.py"]
    }
  }
}
```
Fully quit Claude Desktop and reopen it. The `deep_search tool` should appear.


## Tweaking the defaults

These constants are at the top of `deepsearch-mcp.py`. If you change a value, update the comment next to it too.


| Constant | Meaning |
| --- | --- |
| `SEARXNG_URL` | Local SearXNG instance lives |
| `MAX_PAGE_SIZE_BYTES` | Per-page download ceiling |
| `TARGET_DISTINCT_DOMAINS` | Unique domains to collect before stopping pagination |
| `MAX_PAGES_TO_ACCUMULATE` | Hard cap on pagination depth |
| `SEARXNG_TIME_RANGE` | Recency window on the first pass |
| `DUAL_PASS_RECENCY` | Toggle the second unfiltered pass |
| `MAX_CANDIDATES_POOL` | Passages that survive the bi-encoder pre-filter |
| `MAX_RESULTS` | Max excerpts in final output |
| `MAX_PER_DOMAIN` | Max excerpts from any single domain |
| `MAX_FETCH_WORKERS` | Concurrent fetch workers |


## Limitations & scope

- **Single-user only.** The fetch path has SSRF protections but has never been audited for hostile multi-tenant use. Don't expose it to untrusted traffic.
- **DuckDuckGo fallback is fragile**. The HTML scraping endpoint can block requests or change structure without warning.
- **Slow on big result sets**. Eight workers run concurrently but with 90 target domains and real network variance, some clients will hit tool-call timeouts on slower connections.
- **Large pages get cut off**. Anything over 5 MB is dropped mid-download. Most articles are fine, but long Wikipedia pages and legal documents sometimes don't make it through. I ran into this a few times during testing and just accepted it as a tradeoff.


## License

[MIT ]
