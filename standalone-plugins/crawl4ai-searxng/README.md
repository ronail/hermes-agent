# crawl4ai-searxng

Two local web backends for [Hermes](https://hermes-agent.nousresearch.com) in
one plugin: **crawl4ai** for browser-backed page extraction and **SearXNG** for
on-demand local search. No API keys, no hosted service, no per-request billing —
both run entirely on your machine.

| Provider | Capability | What it does |
|---|---|---|
| `crawl4ai` | extract | Renders JS-heavy pages in headless Chromium and returns clean markdown. A zero-cost local stand-in for Firecrawl/Tavily. |
| `searxng-local` | search | Starts your SearXNG checkout on the first search, keeps it warm across calls, stops it when idle. |

They are ordinary web-provider plugins, so you configure them exactly like the
bundled backends via `web.search_backend` / `web.extract_backend`.

## Install

```bash
# From a checkout of this repo
pip install ./standalone-plugins/crawl4ai-searxng
```

Or copy the plugin directory into your Hermes home and enable it:

```bash
cp -r standalone-plugins/crawl4ai-searxng/crawl4ai_searxng \
      ~/.hermes/plugins/web/crawl4ai-searxng
hermes plugins enable crawl4ai-searxng
```

The pip route declares `crawl4ai` as a dependency, so Hermes' package manager
installs it for you. Provision the Chromium binary once afterwards:

```bash
crawl4ai install
```

That step is separate from the Python install and is not automated — crawl4ai
downloads a browser on first use.

## Configure

```yaml
# ~/.hermes/config.yaml
web:
  search_backend: "searxng-local"
  extract_backend: "crawl4ai"
```

With no `web.backend` set, Hermes auto-detects: whichever provider is available
for the requested capability. Both can coexist — search with SearXNG, extract
with crawl4ai.

Pick the backends any other way you like:

```bash
hermes tools        # interactive picker
```

### SearXNG

The manager looks for a checkout at `$SEARXNG_DIR`, falling back to
`~/Projects/searxng`. It expects `searx/settings_hermes.yml` inside that
checkout and uses its `venv/` interpreter when present (POSIX and Windows
layouts both work), otherwise whatever `python3` is on `PATH`.

```bash
export SEARXNG_DIR=~/src/searxng     # if not at ~/Projects/searxng
```

The process is started on the first search, reused for subsequent searches in
the same session, and stopped after 60s idle or on Hermes exit.

## Behaviour worth knowing

- **`is_available()` never installs and never touches the network.** It runs on
  every `hermes tools` paint, so it only checks whether the package imports
  (crawl4ai) or a settings file exists (SearXNG).
- **Both providers register even when unavailable**, so `hermes tools` can list
  them and offer the install. `is_available()` gates dispatch, not registration.
- **A missing dependency degrades per-URL, not per-call.** crawl4ai returns one
  error entry per requested URL rather than discarding the batch, so one bad
  page never costs you the others.
- **No credentials.** `SEARXNG_DIR` is optional and only relocates a checkout.

## Tests

```bash
pip install pytest
python -m pytest tests/
```

The suite runs without crawl4ai or a SearXNG checkout installed — it stubs the
HTTP and lifecycle boundaries and asserts the contracts (capability flags,
response shape, no-network availability probe, per-URL error isolation).

To validate the plugin the way Hermes does:

```bash
hermes plugins validate standalone-plugins/crawl4ai-searxng/crawl4ai_searxng
```

## Layout

```
crawl4ai-searxng/
├── pyproject.toml                    # deps + hermes_agent.plugins entry point
├── crawl4ai_searxng/           # the installable plugin directory
│   ├── __init__.py                   # register(ctx) — registers both providers
│   ├── plugin.yaml                   # manifest: kind: backend
│   ├── crawl4ai_provider.py          # browser-backed extraction
│   ├── searxng_provider.py           # on-demand SearXNG search
│   └── searxng_lifecycle.py          # subprocess lifecycle (start/idle-stop)
└── tests/test_providers.py
```

`crawl4ai_searxng/` is both the pip package and a drop-in directory plugin —
`plugin.yaml` and `__init__.py` sit side by side, which is what
`hermes plugins validate` requires.

## License

MIT
