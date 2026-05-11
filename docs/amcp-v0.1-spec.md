# AMCP v0.1 — Authority-MCP

## An Applied Profile of W3C Verifiable Credentials and DID:web for MCP Server Authority Attestation

| | |
|---|---|
| **Author** | Remo Prinz, Agentic Relations |
| **Status** | Working Draft |
| **Version** | 0.1 |
| **Date** | 2026-05-11 |
| **Canonical URL** | `https://agenticrelations.ch/specs/amcp/v0.1` |
| **Reference Implementation** | `did:web:jassverband.ch` attesting `https://us-central1-jassguru.cloudfunctions.net/mcp` |
| **License** | CC-BY 4.0 |

---

## Abstract

This document defines **AMCP v0.1 (Authority-MCP)** — an applied profile that combines existing W3C standards (Verifiable Credentials Data Model 2.0, DID Core, JCS canonicalization) with Anthropic's Model Context Protocol (MCP) to establish **cryptographically verifiable authority chains** between organisations and their official MCP knowledge servers. AMCP introduces no new cryptography and no new identity primitives; it specifies a normative pattern for how an authoritative organisation declares that a particular MCP server is its officially endorsed knowledge source, in a manner that any standard W3C VC verification library can validate.

The profile is deployed in production by the Jassverband Schweiz attesting the JassWiki MCP server, and is published under CC-BY 4.0 for adoption by other federations, public-sector bodies, professional associations, and standards organisations.

---

## 1. Introduction

### 1.1 Background

The Model Context Protocol (MCP), introduced by Anthropic in November 2024, has rapidly become the de facto standard for connecting AI assistants to external data sources, tools, and prompts. As of mid-2026, public MCP registries (Anthropic's Official MCP Registry, Smithery, Glama, PulseMCP, awesome-mcp-servers) list tens of thousands of community-contributed MCP servers. Each server may expose tools and resources of arbitrary domain authority — from individual hobbyists publishing personal projects to governmental institutions publishing regulatory data.

The MCP specification itself defines protocol mechanics (transport, message format, capabilities, tool discovery) but is silent on **provenance**, **authority**, and **trust**. There is no normative way for a consumer of an MCP server (whether human, LLM, or autonomous agent) to determine:

1. Whether a given MCP server is genuinely operated, endorsed, or controlled by the organisation it claims to represent;
2. Whether the content served reflects the authoritative position of that organisation;
3. Whether the server's authority claim can be verified independently and cryptographically.

This gap matters increasingly as MCP servers are integrated into agentic decision-making pipelines (procurement, regulatory inquiry, medical reference, legal research) where downstream actions depend on the trustworthiness of the source.

### 1.2 Motivation

Existing approaches in the MCP ecosystem rely on weak proxies for authority:

- **Domain ownership** signalled by URL (`api.suva.ch/mcp` is presumed to be Suva's MCP) — vulnerable to typosquatting, subdomain takeover, and provides no cryptographic guarantee;
- **GitHub repository ownership** signalled by namespace (`github.com/<org>/<server>`) — vulnerable to repository transfer and provides no operational binding;
- **Manual curation** in registries (verified badges, manual review) — does not scale and provides no machine-verifiable artifact.

AMCP addresses this gap by specifying how an organisation publishes a **cryptographically signed attestation** that an MCP server is its official knowledge source. The attestation is itself a W3C Verifiable Credential, signed by the organisation's DID:web identity, and discoverable through standard HTTP mechanisms.

### 1.3 Design Goals

AMCP v0.1 is governed by the following design principles:

| Principle | Rationale |
|---|---|
| **Standards-aligned** | All cryptographic primitives, document formats, and identity mechanisms MUST be drawn from existing W3C, IETF, or equivalent specifications. AMCP introduces no novel crypto. |
| **Pragmatic** | The profile MUST be implementable with existing libraries and infrastructure. A reference implementation MUST be live before specification publication. |
| **Cryptographically verifiable** | Any third-party verifier MUST be able to validate an attestation independently, with no trust in the registry or directory. |
| **Discoverable** | Attestations MUST be discoverable via standard HTTP mechanisms (`.well-known/`, `Link` headers) without requiring proprietary registries. |
| **Wikidata-anchored** | All canonical entities (organisations, domain concepts, geographic scope) SHOULD reference Wikidata QIDs for cross-platform identity resolution. |
| **Versionable** | Attestations MUST carry temporal validity bounds and support supersession by later attestations. |

### 1.4 Non-Goals

AMCP v0.1 explicitly does NOT specify:

- A revocation protocol (deferred to v0.2);
- A multi-attestor consensus mechanism (federation governance is out of scope);
- A presentation protocol for selective disclosure (OID4VP is the recommended companion);
- A trust assessment algorithm or score (left to the verifier or registry operator);
- A compliance mapping to specific regulatory regimes (eIDAS, EU AI Act, FADP — to be addressed in companion implementation guides).

---

## 2. Terminology

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174).

| Term | Definition |
|---|---|
| **Authority** | The organisation issuing the attestation. Identified by a DID (typically `did:web`). |
| **Attestation** | A signed claim by the Authority that a particular MCP server is its official knowledge source. Expressed as a W3C Verifiable Credential. |
| **MCP Server** | A service implementing the Model Context Protocol, identified by an HTTPS endpoint URL. |
| **Verifier** | Any party (human, agent, registry) that validates the attestation. |
| **Trust Chain** | External, human-readable evidence supporting the Authority's claim to operate in its domain (e.g., Wikipedia citation, governmental recognition, professional association registration). |
| **Wikidata Anchor** | A Wikidata QID associated with an entity in the attestation. |
| **AMCP Profile** | This specification. |

---

## 3. Architecture Overview

```
   ┌──────────────────────────┐
   │      Authority           │
   │  did:web:jassverband.ch  │
   │  (Organisation, DID)     │
   └──────────┬───────────────┘
              │ issues
              │ signs (Ed25519)
              ▼
   ┌──────────────────────────────────────┐
   │  Attestation Document                │
   │  W3C VerifiableCredential v2.0       │
   │  type: MCPAuthorityAttestation       │
   │  proof: DataIntegrityProof           │
   │    cryptosuite: eddsa-jcs-2022       │
   │    proofValue: z<base58btc>          │
   └──────────┬───────────────────────────┘
              │ attests
              ▼
   ┌──────────────────────────┐
   │      MCP Server          │
   │  jasswiki MCP at         │
   │  cloudfunctions.net/mcp  │
   └──────────────────────────┘

   Discovery:
     - /.well-known/did.json           (DID document with public key)
     - /.well-known/mcp-authority.json (the VC)
     - /.well-known/mcp.json           (server manifest)
     - HTTP Link rel="authority-attestation"
     - HTTP X-MCP-Authority: did:web:...

   Verification:
     1. Fetch did.json → extract publicKeyJwk
     2. Fetch mcp-authority.json (the VC)
     3. Reconstruct hashData per eddsa-jcs-2022
     4. Verify Ed25519 signature (proof.proofValue)
     5. ✓ Authority confirmed
```

---

## 4. Specification

### 4.1 Authority Identity

The Authority MUST be identified by a Decentralized Identifier conforming to [W3C DID Core](https://www.w3.org/TR/did-1.0/). AMCP v0.1 REQUIRES support for the `did:web` method as defined in [DID:web Method Spec](https://w3c-ccg.github.io/did-method-web/). Other DID methods (e.g., `did:key`, `did:ion`) MAY be supported by implementations but MUST be advertised in the AMCP context.

The DID document MUST be published at `https://<domain>/.well-known/did.json` for `did:web:<domain>` identifiers. It MUST contain at least one `verificationMethod` entry of type `JsonWebKey2020` or `Multikey` with an Ed25519 public key encoded as `publicKeyJwk` (OKP / Ed25519) or `publicKeyMultibase` respectively.

Example (excerpt):
```json
{
  "@context": ["https://www.w3.org/ns/did/v1", "https://w3id.org/security/suites/jws-2020/v1"],
  "id": "did:web:jassverband.ch",
  "verificationMethod": [{
    "id": "did:web:jassverband.ch#authority-key-1",
    "type": "JsonWebKey2020",
    "controller": "did:web:jassverband.ch",
    "publicKeyJwk": {
      "kty": "OKP", "crv": "Ed25519",
      "x": "mwq05fLHa9z9NEa4q7P6wWfLoLa1KrJnoFnHz8nuSbk",
      "use": "sig"
    }
  }],
  "assertionMethod": ["did:web:jassverband.ch#authority-key-1"]
}
```

### 4.2 Attestation Document

The attestation document MUST be a W3C Verifiable Credential conforming to [VC Data Model 2.0](https://www.w3.org/TR/vc-data-model-2.0/).

#### 4.2.1 Required Properties

| Property | Cardinality | Description |
|---|---|---|
| `@context` | 1..* | MUST include `https://www.w3.org/ns/credentials/v2` as first entry, `https://schema.org`, and the AMCP context URI |
| `id` | 1 | Stable URI identifying this attestation (typically its public URL) |
| `type` | 2..* | MUST include both `"VerifiableCredential"` and `"MCPAuthorityAttestation"` |
| `issuer` | 1 | Object describing the Authority. MUST include `id` (DID), `type`, `name`, `url` |
| `validFrom` | 1 | ISO 8601 timestamp when attestation begins to apply |
| `credentialSubject` | 1 | Object describing the attested MCP server (see 4.2.2) |
| `proof` | 1 | Data Integrity Proof per 4.3 |

#### 4.2.2 credentialSubject Structure

The `credentialSubject` describes the MCP server being attested:

| Property | Cardinality | Description |
|---|---|---|
| `id` | 1 | HTTPS URL of the MCP server endpoint |
| `type` | 1 | MUST be `"MCPServer"` |
| `name` | 1 | Human-readable name |
| `operator` | 0..1 | Operating entity (may differ from issuer) |
| `discoveryManifest` | 0..1 | URL of the server's `/.well-known/mcp.json` |
| `humanInterface` | 0..1 | URL of the corresponding human-facing website |
| `scope` | 0..* | Domains of authority covered |
| `wikidataAnchors` | 0..* | Array of `{qid, label}` pairs anchoring domain concepts |
| `attestationStatement` | 0..1 | Object with language-keyed natural-language attestation text |
| `license` | 0..1 | SPDX identifier for content license |
| `languages` | 0..* | BCP-47 language tags supported |

#### 4.2.3 Recommended Properties

| Property | Cardinality | Description |
|---|---|---|
| `validUntil` | 0..1 | ISO 8601 expiry timestamp |
| `termsOfUse` | 0..* | Resolution/board decision references |
| `evidence` | 0..* | Trust-chain evidence (Wikipedia citations, governmental recognition, etc.) |
| `relatedResource` | 0..* | References to the AMCP specification and other applied profiles |

#### 4.2.4 Wikidata Anchors

Wikidata anchors REQUIRED for the `issuer` and RECOMMENDED for major entities in `credentialSubject`. Each anchor consists of `{"qid": "Q<n>", "label": "<text>"}`. Wikidata QIDs provide stable, cross-platform identity resolution independent of the Authority's domain.

### 4.3 Cryptographic Proof

The attestation MUST be signed using a W3C Data Integrity Proof conforming to [Verifiable Credentials Data Integrity 1.0](https://www.w3.org/TR/vc-data-integrity/).

AMCP v0.1 REQUIRES support for the `eddsa-jcs-2022` cryptosuite as defined in [Data Integrity EdDSA Cryptosuites v1.0](https://w3c.github.io/vc-di-eddsa/). This cryptosuite combines:

1. **JCS (JSON Canonicalization Scheme)** per [RFC 8785](https://datatracker.ietf.org/doc/html/rfc8785) for deterministic JSON serialisation;
2. **SHA-256** hashing of canonicalized credential and proof configuration;
3. **Ed25519** signature ([RFC 8032](https://datatracker.ietf.org/doc/html/rfc8032)) over the concatenated hashes;
4. **Multibase base58btc** encoding ([draft-multiformats-multibase](https://datatracker.ietf.org/doc/html/draft-multiformats-multibase)) for the resulting signature in `proof.proofValue`.

The rationale for selecting `eddsa-jcs-2022` over `eddsa-rdfc-2022` is implementation simplicity: JCS canonicalisation requires no RDF parsing, broadens the implementer base, and produces deterministic output for arbitrary JSON-LD documents without requiring a complete RDF dataset processor.

#### 4.3.1 Signing Algorithm

```
Input:
  - credential: the unsigned VC (without proof)
  - proofOptions: proof object without proofValue
  - privateKey: Ed25519 secret key

1. canonicalCredential = JCS(credential)
2. canonicalProofOptions = JCS(proofOptions)
3. credentialHash = SHA-256(canonicalCredential)
4. proofOptionsHash = SHA-256(canonicalProofOptions)
5. hashData = proofOptionsHash || credentialHash
6. signature = Ed25519.sign(privateKey, hashData)
7. proofValue = "z" + base58btc(signature)
8. proof = { ...proofOptions, proofValue }
9. signedCredential = { ...credential, proof }

Return: signedCredential
```

#### 4.3.2 Proof Object Structure

```json
{
  "type": "DataIntegrityProof",
  "cryptosuite": "eddsa-jcs-2022",
  "created": "2026-05-05T00:00:00Z",
  "verificationMethod": "did:web:jassverband.ch#authority-key-1",
  "proofPurpose": "assertionMethod",
  "proofValue": "z3PDMxe1RueGvYAJfvW5qxKyTM1W7NSs43jraigfMV2HHy1zbAziEMLbeZbFf3pP4ELvDwzVa2Eap94VKKTcKSY6i"
}
```

### 4.4 Discovery

#### 4.4.1 .well-known Locations

The Authority MUST publish:

| Path | Content | Content-Type |
|---|---|---|
| `https://<authority-domain>/.well-known/did.json` | DID document | `application/json` |
| `https://<authority-domain>/.well-known/mcp-authority.json` | Attestation VC | `application/json` |

The MCP server (which MAY be on a different domain) SHOULD publish:

| Path | Content | Content-Type |
|---|---|---|
| `https://<mcp-domain>/.well-known/mcp.json` | Server manifest referencing the attestation | `application/json` |

#### 4.4.2 HTTP Headers

The MCP server's HTTP responses SHOULD include:

```
Link: <https://<authority-domain>/.well-known/mcp-authority.json>; rel="authority-attestation"; type="application/json"
Link: <https://<authority-domain>/.well-known/did.json>; rel="attestor-did"; type="application/json"
X-MCP-Authority: did:web:<authority-domain>
X-MCP-Authority-Document: https://<authority-domain>/.well-known/mcp-authority.json
```

The `Link` header rel values are normative for AMCP. The `X-MCP-Authority*` headers are convenience accessors that SHOULD be exposed via `Access-Control-Expose-Headers` for browser-based verifiers.

---

## 5. Verification Protocol

A Verifier validates an attestation as follows:

```
Input:
  - candidateMcpServerUrl: URL of the MCP server claiming attestation

1. Fetch the MCP server's discovery manifest:
     manifest = GET https://<mcp-domain>/.well-known/mcp.json
   OR follow Link header rel="authority-attestation" from server response.

2. Extract the attestation document URL from manifest.attestedBy.attestationDocument

3. Fetch the attestation:
     vc = GET <attestationDocument>

4. Validate VC structure:
     - vc.type MUST include "VerifiableCredential" AND "MCPAuthorityAttestation"
     - vc.@context[0] MUST be "https://www.w3.org/ns/credentials/v2"
     - vc.issuer.id MUST be a valid DID
     - vc.credentialSubject.id MUST match candidateMcpServerUrl
     - vc.proof MUST exist and have cryptosuite = "eddsa-jcs-2022"

5. Resolve the DID to obtain the public key:
     - For did:web:<domain>: GET https://<domain>/.well-known/did.json
     - Find verificationMethod matching vc.proof.verificationMethod
     - Extract Ed25519 public key (from publicKeyJwk.x or publicKeyMultibase)

6. Reconstruct the signed hash data:
     - unsignedVC = vc without proof
     - proofOptions = vc.proof without proofValue
     - hashData = SHA-256(JCS(proofOptions)) || SHA-256(JCS(unsignedVC))

7. Decode signature:
     - signatureBytes = base58btc_decode(proof.proofValue[1:])  // strip 'z' prefix
     - signatureBytes MUST be exactly 64 bytes

8. Verify Ed25519 signature:
     - Ed25519.verify(publicKey, hashData, signatureBytes)
     - If valid: attestation is cryptographically authentic
     - If invalid: REJECT attestation

9. Validate temporal bounds:
     - now() MUST be >= vc.validFrom
     - If vc.validUntil exists: now() MUST be <= vc.validUntil

10. Validate trust chain (optional, policy-dependent):
     - Cross-check vc.evidence entries against external sources
     - Verify Wikidata anchors resolve to expected entities
     - Apply local policy on acceptable Authority categories
```

A Verifier MAY use any conforming W3C VC verification library (e.g., `@digitalbazaar/vc`, `@veramo/credential-w3c`, `ssi` Rust crate, `pyld`) to perform steps 4–8 in a standards-compliant manner.

---

## 6. Security Considerations

### 6.1 Key Management

The Authority's private signing key is the root of trust for all its attestations. Implementers MUST follow best practices for asymmetric key management:

- Generate keys on hardware where feasible (HSM, TPM, hardware token);
- Store private keys at rest with appropriate filesystem permissions (`chmod 600`) or in dedicated key vaults;
- Never transmit private keys over unencrypted channels;
- Never commit private keys to version control systems;
- Maintain offline backups in tamper-evident storage;
- Implement key rotation procedures and deprecation timelines.

### 6.2 Revocation

**v0.1 GAP:** This version does NOT specify a revocation mechanism. Once issued, an attestation remains cryptographically valid until `validUntil` is reached (if set) or the Authority's domain registration lapses.

For v0.1 deployments, Authorities SHOULD use short-lived attestations (e.g., 12-month `validUntil`) and renew via supersession. A formal revocation list mechanism (likely [StatusList2021](https://www.w3.org/TR/vc-status-list/)) is planned for v0.2.

### 6.3 DNS-Based Trust

The `did:web` method anchors trust in the Authority's DNS control. This creates the following attack surface:

- **DNS hijacking** of the Authority's domain compromises the DID document and thus the verification key;
- **Subdomain takeover** of `<authority-domain>` allows publishing of fraudulent `.well-known/did.json`;
- **Certificate Authority compromise** combined with DNS attacks allows traffic interception.

Mitigations: DNSSEC SHOULD be enabled. CAA records SHOULD restrict certificate issuance. Critical Authorities (governmental, regulatory) SHOULD consider hardware-backed keys with auditable issuance trails.

### 6.4 Replay and Substitution Attacks

The attestation document is intended to be public; it is not vulnerable to replay attacks in the traditional sense. However, an attacker MAY attempt to substitute a stale attestation. Mitigations:

- `validFrom` / `validUntil` timestamps bound the temporal validity;
- Verifiers SHOULD fetch the attestation directly from the canonical URL rather than accepting a presented copy;
- High-stakes verifiers SHOULD cross-check Wikidata anchors and trust-chain evidence.

### 6.5 Canonicalisation Stability

The `eddsa-jcs-2022` cryptosuite relies on RFC 8785 JCS canonicalisation. Implementers MUST use compliant JCS implementations to ensure signature verification interoperability. Edge cases (Unicode NFC normalisation, numeric edge cases, key ordering) MUST be tested against published JCS test vectors.

### 6.6 No Cryptographic Innovation

AMCP introduces NO new cryptographic primitives. All cryptography is delegated to W3C/IETF-standardised algorithms (Ed25519, SHA-256) and document formats (W3C VC, JCS, multibase). This is a deliberate design choice to maximise auditability and minimise novel attack surface.

---

## 7. Privacy Considerations

Attestation documents are intentionally public artifacts. The Authority, the MCP server, and the credential subject are designed to be discoverable.

Implementers MUST NOT include personal data (PII) in attestation documents beyond what is necessary for institutional identification (e.g., named curators with their explicit consent). The `credentialSubject.operator.curator` field is OPTIONAL; if used, it SHOULD reference institutional roles rather than personal data.

The attestation does NOT track or identify verifiers. Verification is performed offline from the verifier's perspective: only the attestation document, DID document, and (optionally) trust-chain links are fetched.

---

## 8. Reference Implementation

A live production reference implementation is operated jointly by the Jassverband Schweiz (Swiss Jass Federation) and JassWiki, with Agentic Relations as the specification owner:

| Component | URL |
|---|---|
| Authority DID document | `https://jassverband.ch/.well-known/did.json` |
| Attestation VC | `https://jassverband.ch/.well-known/mcp-authority.json` |
| MCP server endpoint (Streamable HTTP) | `https://us-central1-jassguru.cloudfunctions.net/mcp/http` |
| MCP server endpoint (SSE legacy) | `https://us-central1-jassguru.cloudfunctions.net/mcp/sse` |
| Server manifest | `https://jasswiki.ch/.well-known/mcp.json` |
| Server card (Glama-compatible) | `https://jasswiki.ch/.well-known/mcp/server-card.json` |
| Official MCP Registry entry | `https://registry.modelcontextprotocol.io/v0/servers/ch.jassverband/jasswiki` |
| Operating board resolution | `JVS-VS-2026-05-04-AMCP-01` |
| Public submission repository | `https://github.com/remoprinz/jasswiki-mcp` |

### 8.1 Verification Example (Python)

```python
import json, hashlib, urllib.request, base64
import nacl.signing
import base58  # pip install base58

# 1. Fetch live VC + DID document
vc = json.load(urllib.request.urlopen('https://jassverband.ch/.well-known/mcp-authority.json'))
did = json.load(urllib.request.urlopen('https://jassverband.ch/.well-known/did.json'))

# 2. JCS canonicalize unsigned credential and proof options
def jcs(obj):
    return json.dumps(obj, sort_keys=True, separators=(',', ':'),
                      ensure_ascii=False).encode('utf-8')

unsigned = {k: v for k, v in vc.items() if k != 'proof'}
proof_cfg = {k: v for k, v in vc['proof'].items() if k != 'proofValue'}
hash_data = (hashlib.sha256(jcs(proof_cfg)).digest()
             + hashlib.sha256(jcs(unsigned)).digest())

# 3. Decode signature (strip multibase 'z' prefix → base58btc)
sig = base58.b58decode(vc['proof']['proofValue'][1:])
assert len(sig) == 64

# 4. Decode Ed25519 public key from DID JWK
pub_b64 = did['verificationMethod'][0]['publicKeyJwk']['x']
pub_key = base64.urlsafe_b64decode(pub_b64 + '==')

# 5. Verify
nacl.signing.VerifyKey(pub_key).verify(hash_data, sig)
print('✓ AMCP attestation cryptographically verified')
```

### 8.2 Verification Example (JavaScript / Node.js)

Using `@digitalbazaar/vc` with `eddsa-jcs-2022` cryptosuite:

```javascript
import * as vc from '@digitalbazaar/vc';
import { cryptosuite } from '@digitalbazaar/eddsa-jcs-2022-cryptosuite';
import { DataIntegrityProof } from '@digitalbazaar/data-integrity';

const credential = await (await fetch(
  'https://jassverband.ch/.well-known/mcp-authority.json'
)).json();

const documentLoader = /* custom loader for DID:web resolution */;

const result = await vc.verifyCredential({
  credential,
  suite: new DataIntegrityProof({ cryptosuite }),
  documentLoader
});

console.log(result.verified ? '✓ verified' : '✗ failed');
```

---

## 9. Comparison to Related Work

| Specification | Scope | Relationship to AMCP |
|---|---|---|
| **W3C Verifiable Credentials v2.0** | General credential format | AMCP is a credential type within this framework. |
| **W3C DID Core** | Decentralized identity | AMCP uses `did:web` as default identity method. |
| **W3C Data Integrity 1.0 + eddsa-jcs-2022** | Cryptographic proofs on JSON | AMCP delegates all signing to this cryptosuite. |
| **MCP** (Anthropic) | Tool invocation protocol | AMCP does not modify or extend MCP itself; it adds an out-of-band trust layer. |
| **llms.txt** | Discovery hint for crawlers | Complementary; AMCP attestation URL MAY be referenced in `llms.txt`. |
| **C2PA** (Content Provenance) | Provenance for images/media | Similar pattern (signed manifests), different scope; AMCP does not address per-message provenance. |
| **SLSA** (Supply-chain levels) | Software build attestation | Adjacent; AMCP attests the running service, not the build pipeline. |
| **Sigstore / cosign** | Artifact signing | Adjacent; AMCP could in future incorporate Sigstore for keyless signing. |
| **OpenAPI Specification** | API description | Complementary; AMCP attests the operating authority, OpenAPI describes the interface. |
| **eIDAS 2.0 / EUDI Wallet** | EU digital identity for persons | Complementary; AMCP attests organisational MCP servers, EUDI attests persons. |

AMCP's contribution is NOT a new standard but the **combination** — specifying how an MCP server's authority is declared, signed, discovered, and verified using existing W3C primitives.

---

## 10. Open Questions and Future Work

The following items are deferred from v0.1:

1. **Revocation:** A `StatusList2021`-based mechanism for declaring revoked attestations without re-issuing fresh credentials.
2. **Multi-attestor support:** Joint attestations from federations of organisations (e.g., a national authority + sector regulator co-attesting an MCP server).
3. **Selective disclosure:** Profile for OID4VP presentations enabling Verifiers to request specific subsets of an attestation.
4. **Compliance mappings:** Companion documents mapping AMCP attestations to specific regulatory frameworks (EU AI Act Article 50, eIDAS 2.0 trust services, FADP/GDPR, FINMA Circular 2018/3).
5. **Hardware key bindings:** Profile for TPM/HSM-backed Authority keys with attestation.
6. **Time-stamping:** Integration with OpenTimestamps for tamper-evident attestation timing.
7. **Auto-renewal:** A pattern for automated attestation refresh ahead of `validUntil` expiry, with smooth supersession.
8. **Negative attestations:** A pattern for declaring that an MCP server is NOT operated by an organisation (counter-attestation against impersonation).

Contributions are welcomed at `https://github.com/remoprinz/jasswiki-mcp/issues` or via direct contact at `architect@agenticrelations.ch`.

---

## 11. Normative References

- [RFC 2119] Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119, March 1997.
- [RFC 8032] Josefsson, S. and I. Liusvaara, "Edwards-Curve Digital Signature Algorithm (EdDSA)", RFC 8032, January 2017.
- [RFC 8174] Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words", BCP 14, RFC 8174, May 2017.
- [RFC 8785] Rundgren, A., Jordan, B., and S. Erdtman, "JSON Canonicalization Scheme (JCS)", RFC 8785, June 2020.
- [W3C-DID] World Wide Web Consortium, "Decentralized Identifiers (DIDs) v1.0", W3C Recommendation, July 2022.
- [W3C-VC] World Wide Web Consortium, "Verifiable Credentials Data Model v2.0", W3C Recommendation, May 2025.
- [W3C-VCDI] World Wide Web Consortium, "Verifiable Credentials Data Integrity 1.0", W3C Recommendation, May 2025.
- [VC-DI-EDDSA] World Wide Web Consortium, "Data Integrity EdDSA Cryptosuites v1.0", W3C Recommendation, May 2025.

## 12. Informative References

- [DID-WEB] W3C Credentials Community Group, "did:web Method Specification".
- [MCP-SPEC] Anthropic, "Model Context Protocol Specification", `https://modelcontextprotocol.io`.
- [SCHEMA-ORG] schema.org, "Schema.org Vocabulary".
- [WIKIDATA] Wikimedia Foundation, "Wikidata: A Free Collaborative Knowledge Base", `https://www.wikidata.org`.
- [MULTIBASE] Multiformats, "Multibase Specification", IETF Draft.
- [STATUS-LIST] World Wide Web Consortium, "Bitstring Status List v1.0", Working Draft.
- [OID4VP] OpenID Foundation, "OpenID for Verifiable Presentations".
- [C2PA] Coalition for Content Provenance and Authenticity, "C2PA Technical Specification".

---

## 13. Authors and Contributors

**Architect & Specification Owner:**
Remo Prinz — Founder, Agentic Relations (`https://agenticrelations.ch`)
`architect@agenticrelations.ch`

**Reference Implementation Operator:**
Jassverband Schweiz (Swiss Jass Federation)
`did:web:jassverband.ch`
Resolution `JVS-VS-2026-05-04-AMCP-01`

**Co-Curator:**
Fabian Cadonau — JassWiki / Trumpf-As.ch

**Recognition:**
Jassen is recognised as Living Tradition by the Swiss Federal Office of Culture (BAK) since 2011 ([Lebendige Traditionen der Schweiz](https://www.lebendige-traditionen.ch)).

---

## Appendix A — Full Example Attestation

See live document: `https://jassverband.ch/.well-known/mcp-authority.json`

Mirror in this repository: [`examples/jvs-attestation.json`](./examples/jvs-attestation.json)

---

## Appendix B — Errata and Change Log

| Version | Date | Notes |
|---|---|---|
| 0.1 | 2026-05-11 | Initial working draft. Reference implementation live since 2026-05-04 (custom JSON-LD), refactored to W3C VC v2.0 envelope with `eddsa-jcs-2022` cryptosuite on 2026-05-05. |

---

*This document is published under the [Creative Commons Attribution 4.0 International License](https://creativecommons.org/licenses/by/4.0/). The reference implementation content is published under [CC-BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/). The attestation documents themselves are CC0.*
