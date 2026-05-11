# JassWiki MCP — Authority-Attested Knowledge Server for Swiss Jass

> Erste vom Schweizer Jassverband (JVS) attestierte MCP-Quelle. Kryptographisch verifizierbar via `did:web:jassverband.ch`.

[![smithery badge](https://smithery.ai/badge/remoprinz/jass)](https://smithery.ai/servers/remoprinz/jass)
[![Official MCP Registry](https://img.shields.io/badge/MCP%20Registry-ch.jassverband%2Fjasswiki-blue.svg)](https://registry.modelcontextprotocol.io/v0/servers?search=jasswiki)
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

## Verifying the Authority Attestation (W3C Verifiable Credential)

The JassWiki MCP server is the first MCP service in Switzerland with cryptographically verifiable authority attestation, expressed as a **W3C Verifiable Credential** (Data Model 2.0) signed with the **`eddsa-jcs-2022`** cryptosuite. Any standard W3C VC verification library can confirm it.

### Quick verify with any W3C VC library

The attestation document is a self-contained Verifiable Credential — fetch and verify in one step:

```bash
curl -s https://jassverband.ch/.well-known/mcp-authority.json
```

The `proof.proofValue` field contains the signature in multibase base58btc format. Use any of:
- **Node.js:** [`@digitalbazaar/vc`](https://github.com/digitalbazaar/vc) + [`@digitalbazaar/eddsa-jcs-2022-cryptosuite`](https://github.com/digitalbazaar/eddsa-jcs-2022-cryptosuite)
- **TypeScript / Browser:** [`@veramo/credential-w3c`](https://veramo.io/) with `eddsa-jcs-2022` plugin
- **Python:** [`pyld`](https://github.com/digitalbazaar/pyld) + [`PyNaCl`](https://pynacl.readthedocs.io/) (manual JCS implementation, see below)
- **Rust:** [`ssi`](https://github.com/spruceid/ssi) crate

### Manual verification (Python, no W3C VC library required)

```bash
pip install pynacl base58
```

```python
import json, hashlib, urllib.request
import nacl.signing, base58

# 1. Fetch the VC + DID document
vc = json.load(urllib.request.urlopen('https://jassverband.ch/.well-known/mcp-authority.json'))
did = json.load(urllib.request.urlopen('https://jassverband.ch/.well-known/did.json'))

# 2. Reconstruct the signed hash data per eddsa-jcs-2022
def jcs(obj):
    return json.dumps(obj, sort_keys=True, separators=(',', ':'), ensure_ascii=False).encode('utf-8')

unsigned_doc = {k: v for k, v in vc.items() if k != 'proof'}
proof_config = {k: v for k, v in vc['proof'].items() if k != 'proofValue'}

doc_hash = hashlib.sha256(jcs(unsigned_doc)).digest()
proof_hash = hashlib.sha256(jcs(proof_config)).digest()
hash_data = proof_hash + doc_hash  # W3C VC Data Integrity ordering

# 3. Decode signature from multibase base58btc
proof_value = vc['proof']['proofValue']
assert proof_value.startswith('z')  # 'z' = base58btc multibase prefix
signature = base58.b58decode(proof_value[1:])

# 4. Decode public key from DID document
import base64
pub_jwk_x = did['verificationMethod'][0]['publicKeyJwk']['x']
public_key = base64.urlsafe_b64decode(pub_jwk_x + '==')

# 5. Verify
nacl.signing.VerifyKey(public_key).verify(hash_data, signature)
print('✓ W3C VC verified — JassWiki MCP is officially attested by Jassverband Schweiz')
```

The attestation is governed by [JVS Vorstandsbeschluss 2026-05-04 (JVS-VS-2026-05-04-AMCP-01)](https://jassverband.ch).

---

## Specification: AMCP v0.1 (Authority-MCP)

JassWiki MCP is the **first production implementation** of **AMCP v0.1** — *An Applied Profile of W3C Verifiable Credentials and DID:web for MCP Server Authority Attestation*.

AMCP is **not a new standard** — it is a thin profile that combines:

| Layer | Existing Standard |
|---|---|
| Identity of the authority | [W3C DID Core](https://www.w3.org/TR/did-1.0/) (`did:web` method) |
| Cryptographic envelope | [W3C Verifiable Credentials Data Model 2.0](https://www.w3.org/TR/vc-data-model-2.0/) |
| Signature suite | [`eddsa-jcs-2022`](https://w3c.github.io/vc-di-eddsa/) (Ed25519 + JCS canonicalization) |
| Entity descriptions | [Schema.org](https://schema.org) + JSON-LD |
| Canonical entity IDs | [Wikidata QIDs](https://www.wikidata.org/) |
| Server protocol | [Anthropic MCP](https://modelcontextprotocol.io) |
| Discovery | [HTTP Link header (RFC 8288)](https://datatracker.ietf.org/doc/html/rfc8288) + `.well-known/` |

The contribution of AMCP is the *combination* — specifying how an organisation declares its official MCP server with cryptographic proof, Wikidata-anchored entity references, and a verifiable trust chain.

**Spec owner & architect:** [Agentic Relations](https://agenticrelations.ch) — founder: Remo Prinz.

📄 **Read the full specification:** [**AMCP v0.1 Working Draft**](./docs/amcp-v0.1-spec.md) (3,600 words; canonical URL `https://agenticrelations.ch/specs/amcp/v0.1`)

The methodology applies to any organisation, federation, public-sector body, or association that wants to become a cryptographically verifiable knowledge source in the agentic web.

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
