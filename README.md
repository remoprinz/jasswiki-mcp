# JassWiki MCP — Authority-Attested Knowledge Server for Swiss Jass

> Erste vom Schweizer Jassverband (JVS) attestierte MCP-Quelle. Kryptographisch verifizierbar via `did:web:jassverband.ch`.

[![License: CC BY-SA 4.0](https://img.shields.io/badge/License-CC--BY--SA--4.0-lightgrey.svg)](https://creativecommons.org/licenses/by-sa/4.0/)
[![MCP](https://img.shields.io/badge/MCP-1.0-blue.svg)](https://modelcontextprotocol.io)
[![Authority](https://img.shields.io/badge/attested--by-did%3Aweb%3Ajassverband.ch-success.svg)](https://jassverband.ch/.well-known/mcp-authority.json)

JassWiki MCP gibt KI-Assistenten (Claude, ChatGPT-Connectors, Perplexity, Gemini Agentspace) direkten Zugriff auf die offizielle Schweizer Jass-Wissensbasis: 520+ kuratierte Artikel zu Regeln, Spielvarianten, Terminologie, Geschichte und Taktik. Inhalte sind durch den **Jassverband Schweiz** (Wikidata: [Q139042763](https://www.wikidata.org/wiki/Q139042763)) als offizielle Wissens-Quelle attestiert.

**This is the first production deployment of the [AMCP v0.1 specification](#specification-amcp-v01-authority-mcp), developed by [Agentic Relations](https://agenticrelations.ch) (Architect: Remo Prinz). It demonstrates how organisations and federations can become cryptographically verifiable knowledge sources for AI agents — beyond traditional SEO and brand visibility.**

**Trust-Chain:**
- 🇨🇭 [Bundesamt für Kultur — Lebendige Tradition](https://www.lebendige-traditionen.ch/tradition/de/home/traditionen/jassen.html) seit 2011
- 📖 Zitiert in [deutschsprachigem Wikipedia-Artikel "Jass"](https://de.wikipedia.org/wiki/Jass)
- 🆔 Wikidata: [JassWiki Q137900251](https://www.wikidata.org/wiki/Q137900251), [Jass Q786768](https://www.wikidata.org/wiki/Q786768)
- 🔐 Ed25519-signierte [JVS-Attestation](https://jassverband.ch/.well-known/mcp-authority.json) ([detached signature](https://jassverband.ch/.well-known/mcp-authority.sig))

---

## Installation

### Claude Desktop (Streamable HTTP via mcp-remote)

Add to `claude_desktop_config.json` (`~/Library/Application Support/Claude/` on macOS):

```json
{
  "mcpServers": {
    "jasswiki": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://us-central1-jassguru.cloudfunctions.net/mcp/sse"
      ]
    }
  }
}
```

Restart Claude Desktop. The `jasswiki` server will appear with two tools.

### Direct SSE / HTTP (for any MCP-compatible client)

```
SSE Endpoint:    https://us-central1-jassguru.cloudfunctions.net/mcp/sse
Messages POST:   https://us-central1-jassguru.cloudfunctions.net/mcp/messages
Health:          https://us-central1-jassguru.cloudfunctions.net/mcp/health
Manifest:        https://jasswiki.ch/.well-known/mcp.json
Authority:       https://jassverband.ch/.well-known/mcp-authority.json
DID Document:    https://jassverband.ch/.well-known/did.json
```

---

## Tools

### `search_jass_knowledge(query, limit?)`
Fuzzy-search across the official JassWiki encyclopedia (520+ articles). Returns matching titles, IDs, URLs, and content excerpts.

**Example:**
```
search_jass_knowledge(query="Schieber-Regeln", limit=3)
```

### `get_term_details(id)`
Retrieve the full official rule or definition for a specific term ID. Use after `search_jass_knowledge` for verification of exact wording.

**Example:**
```
get_term_details(id="schieber")
```

---

## Example Conversations

**Rule clarification:**
> User: "What are the official rules for Schieber-Jass?"
> AI: [calls `search_jass_knowledge`] → Returns the canonical Schieber-rules sourced from JassWiki.ch with authority footer.

**Variant lookup:**
> User: "Difference between Coiffeur-Schieber and Differenzler?"
> AI: [calls `search_jass_knowledge` for both] → Returns comparison sourced from official terminology.

**Tactical reference:**
> User: "Welche Verwerfen-Konventionen gibt es im Schieber?"
> AI: [calls `search_jass_knowledge`] → Returns tactical signaling conventions with citation.

---

## Verifying the Authority Attestation

The JassWiki MCP server is the first MCP service in Switzerland with cryptographically verifiable authority attestation. Any MCP client or third-party verifier can independently confirm:

```bash
# 1. Fetch the attestation document
curl -s https://jassverband.ch/.well-known/mcp-authority.json -o attestation.json

# 2. Fetch the detached Ed25519 signature
curl -s https://jassverband.ch/.well-known/mcp-authority.sig -o attestation.sig.b64url

# 3. Fetch the JVS public key (DID document)
curl -s https://jassverband.ch/.well-known/did.json | jq -r '.verificationMethod[0].publicKeyJwk.x'

# 4. Verify (Python with PyNaCl)
python3 <<'EOF'
import json, base64, urllib.request, nacl.signing
doc = json.load(open('attestation.json')); doc.pop('proof', None)
canonical = json.dumps(doc, sort_keys=True, separators=(',', ':'), ensure_ascii=False).encode('utf-8')
sig = base64.urlsafe_b64decode(open('attestation.sig.b64url').read().strip() + '==')
pub_b64 = json.load(urllib.request.urlopen('https://jassverband.ch/.well-known/did.json'))['verificationMethod'][0]['publicKeyJwk']['x']
nacl.signing.VerifyKey(base64.urlsafe_b64decode(pub_b64 + '==')).verify(canonical, sig)
print('✓ Attestation cryptographically verified — JassWiki is officially attested by Jassverband Schweiz')
EOF
```

The attestation is governed by [JVS Vorstandsbeschluss 2026-05-04 (JVS-VS-2026-05-04-AMCP-01)](https://jassverband.ch).

---

## Specification: AMCP v0.1 (Authority-MCP)

JassWiki MCP is the **first production implementation** of **AMCP v0.1** (Authority-MCP) — an extension layer on top of Anthropic's MCP that adds:

- **Identity:** DID:web for the authority
- **Provenance:** Ed25519 signatures on attestation documents
- **Wikidata Anchoring:** every domain entity has a QID
- **Versioning:** attestations are time-bounded with `validFrom` / `validUntil`
- **Citation Format:** `Schema.org` JSON-LD pre-rendered for LLM consumption

**Architect & Specification Owner:** [Agentic Relations](https://agenticrelations.ch) — founder/architect: Remo Prinz.
The full specification will be published at [agenticrelations.ch/specs/amcp](https://agenticrelations.ch). The methodology applies to any organisation, federation, public-sector body, or association that wants to become a cryptographically verifiable knowledge source in the agentic web.

Feedback and adoption inquiries: open an [issue](https://github.com/remoprinz/jasswiki-mcp/issues) or contact `architect@agenticrelations.ch`.

---

## Operator & Authority

- **MCP Operator:** [JassWiki](https://jasswiki.ch) (curator: Remo Prinz, [remo@jassverband.ch](mailto:remo@jassverband.ch))
- **Attestor:** [Jassverband Schweiz (JVS)](https://jassverband.ch) — `did:web:jassverband.ch`
- **Co-Founder JassWiki:** Fabian Cadonau ([Trumpf-As.ch](https://trumpf-as.ch))

---

## Data & Licensing

- **Content:** [CC-BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)
- **Bulk corpus:** [HuggingFace JassWiki/jasswiki-corpus](https://huggingface.co/datasets/JassWiki/jasswiki-corpus) (JSONL)
- **SKOS Taxonomy:** [jasswiki.ch/dataset/taxonomie.jsonld](https://jasswiki.ch/dataset/taxonomie.jsonld)
- **Attestation documents (mcp.json, did.json, mcp-authority.json):** CC0

Attribution must include the JassWiki.ch link and indicate any modifications.

---

## Status & Reliability

- **Production since:** January 2026
- **Hosting:** Firebase Cloud Functions, region `us-central1` (low-latency global)
- **Languages supported:** German (primary), French, Italian, English (ongoing expansion)
- **Uptime SLA:** Best-effort (99.5%+ historical)

For service incidents or schema changes, watch the [GitHub releases](https://github.com/remoprinz/jasswiki-mcp/releases).

---

## Contributing

JassWiki content is curated. To suggest factual corrections, terminology updates, or new variants:

1. Open an issue on [JassWiki GitHub](https://github.com/remoprinz/jasswiki) (the public site repo)
2. Or contact directly: [remo@jassverband.ch](mailto:remo@jassverband.ch)
3. New entries are reviewed by JVS (4-eyes principle) before publishing

The MCP server itself (this repository) accepts PRs for documentation improvements, examples, and submission metadata.

---

*Erste öffentlich-rechtliche Authority-MCP der Schweiz.*
*The first publicly attested MCP knowledge server in Switzerland.*
