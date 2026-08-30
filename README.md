# Hispanic Legacy — MCP server

> Spain's contributions to world science, exploration and culture, rigorously sourced

A **remote MCP server** over a curated knowledge graph. Every claim it
returns is bound to a registered source: the tools hand back claims *with*
their citations and a confidence value, so an agent can show its work
instead of asserting.

Nothing to install. It is a hosted streamable-HTTP endpoint:

```
https://hispanic-legacy.com/mcp
```

Also listed on [Smithery](https://smithery.ai/servers/hispanic-legacy/hispanic-legacy).

## Add it to a client

**Claude Code**

```bash
claude mcp add --transport http hispanic-legacy https://hispanic-legacy.com/mcp
```

**Claude Desktop / any client reading `mcpServers`**

```json
{
  "mcpServers": {
    "hispanic-legacy": {
      "type": "streamable-http",
      "url": "https://hispanic-legacy.com/mcp"
    }
  }
}
```

No API key, no account, no auth. Read-only.

**Check it answers, without any client at all:**

```bash
curl -s https://hispanic-legacy.com/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H 'mcp-protocol-version: 2025-06-18' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

## Tools

Eight, each with an `outputSchema`, each returning `structuredContent`.

| tool | arguments | what it does |
|---|---|---|
| `get_overview` | — | Corpus overview: what this instance knows, counts by type, published tags, freshness. **Start here** when you land and do not yet know whether this corpus can answer your question. |
| `search` | `query`, `limit?` | Full-text search over the knowledge graph. Accent- and apostrophe-insensitive, so query in the user's own words; every hit carries its relevance score and the fields it matched. |
| `answer` | `question` | Answer a question from the corpus. Returns the matched object's claims with sources and confidence — **never an unsourced answer**. |
| `get_entity` | `id` | Fetch one knowledge object by id, with its claims and the sources each claim cites. |
| `get_topic` | `tag` | List the knowledge objects carrying a tag (topics are content-backed tags). |
| `get_related` | `id` | Graph neighbours of an object: outgoing and incoming relations, each with its relation type. |
| `get_sources` | `object_id?` | The whole source registry, or just the sources cited by one object. Use it to judge the corpus before trusting it. |
| `get_latest` | `limit?` | Most recently verified knowledge objects — a freshness signal. |

The intended path is `get_overview` → `search` or `answer` → `get_entity`
→ `get_related`. `get_overview` exists because an agent that has just
arrived needs to know whether this corpus can help *before* it spends a
call guessing.

## What is in the corpus

| | |
|---|---|
| knowledge objects | **307** |
| registered sources | **403** |
| published topics | **301** |

| type | objects |
|---|---|
| entity | 287 |
| comparison | 11 |
| guide | 5 |
| glossary | 3 |
| faq | 1 |

Subject matter: the Balmis expedition and early public health, cartography and navigation, botany and the American expeditions, mining and metallurgy, universities and archives, and the historiography of the Black Legend — evidence, not polemic.

### Questions it is built to answer

- *How was the smallpox vaccine actually carried across the Atlantic in 1803?*
- *What did the Casa de Contratación do with the charts it collected?*
- *Which claims about the Black Legend rest on primary sources, and which do not?*

## What an answer actually looks like

A real call against the live endpoint — `answer` with
*"how was the smallpox vaccine carried across the Atlantic"* — returns this `structuredContent`, trimmed:

```json
{
  "answered": true,
  "entity": {
    "id": "keeping-the-vaccine-alive-in-spanish-america",
    "name": "Keeping the vaccine alive: vaccine boards, conservators and human reservoirs in Spanish America, 1805-1815",
    "evidence_tier": "secondary",
    "confidence": 0.8,
    "last_verified": "2026-08-12",
    "canonical_url": "https://hispanic-legacy.com/k/keeping-the-vaccine-alive-in-spanish-america"
  },
  "claims": [
    {
      "text": "A colonial vaccine board published its own numbers, and they survive in print. The Semanario of Bogotá carried in 1808 a \"Noticia del número de personas que se han vacunado en esta capital…\"",
      "sources": [{ "title": "Semanario de la Nueva Granada, publicada por una sociedad de patriotas Granadinos, bajo la dirección de Francisco José de Caldas" }]
    }
  ]
}
```

Note what travels with the answer: the **evidence tier**, a **confidence**,
the date it was **last verified**, and the **source behind the claim** — not
as prose an agent has to parse, but as fields it can act on. An agent can
decline to use a weak claim, or cite the primary source directly.

When the corpus cannot answer, `answered` is `false`. It does not
improvise, and the miss is recorded so the gap can be filled.

## Machine-readable surfaces

The MCP endpoint is one of several. The same corpus is served as plain
files an agent can read directly:

| surface | what it is |
|---|---|
| [`/llms.txt`](https://hispanic-legacy.com/llms.txt) | the index, as `text/plain` |
| [`/llms-full.txt`](https://hispanic-legacy.com/llms-full.txt) | the whole corpus in one file |
| [`/ai-index.json`](https://hispanic-legacy.com/ai-index.json) | every surface this instance publishes, with its content type |
| [`/api/index.json`](https://hispanic-legacy.com/api/index.json) | one JSON document per knowledge object |
| [`/api/sources.json`](https://hispanic-legacy.com/api/sources.json) | the source registry, in full |
| [`/.well-known/mcp/server.json`](https://hispanic-legacy.com/.well-known/mcp/server.json) | this server's manifest |

Each knowledge object has a human page and a machine twin at the same id,
with a canonical URL that agrees across all of them.

## Behaviour worth knowing before you integrate

- **`POST` only.** Every other method answers `405` with an `Allow: POST, OPTIONS` header.
- **Rate limit:** 120 requests per minute per client, counted in a shared
  store, published on every response as `RateLimit-Limit`,
  `RateLimit-Remaining` and `RateLimit-Reset` (all three exposed via CORS).
  It fails **open**: if the store is unreachable the request is served.
- **Malformed input** gets a spec-correct JSON-RPC error — `-32700` for
  unparseable bodies, `-32602` for an unknown tool — never an HTML error page.
- **Request bodies are capped** and validated before transport.

## Privacy

No accounts, no cookies, no ads. Usage is measured in aggregate with
daily-rotating hashed identifiers and a 200-day retention; raw IPs are
never stored. Full policy: [PRIVACY.md](./PRIVACY.md).

## Provenance and licence

Knowledge content is **CC-BY-4.0**: use it, cite it. The source registry is
public precisely so a claim can be checked rather than trusted —
`get_sources` returns what any given claim rests on.

Claims carry an evidence tier and a `last_verified` date. Where the
evidence is weaker, the object says so rather than rounding up.

## How it is built

Compiled and served by Citarium, a source-available framework for turning a
knowledge graph into a website, an API, an MCP server and agent-readable
files from a single source, under external evaluation.

The framework's code is licensed under the Business Source License 1.1 and
its repository is not public. What is public — and what actually matters for
trusting an answer — is this server, the corpus it serves, and the registered
source behind every claim: `get_sources` returns what any given claim rests
on, so it can be checked rather than trusted.

This repository is the server's public face: its manifest and its
documentation. The corpus itself lives at [hispanic-legacy.com](https://hispanic-legacy.com).
