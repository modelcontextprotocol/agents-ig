# AMEP Evidence Extension: Signed Effect Receipts for Tool Calls

- **Status**: Draft
- **Type**: Extensions Track
- **Created**: 2026-11-21
- **Author(s)**: Staford Titus Selvaraj ([@stafordtituss](https://github.com/stafordtituss))
- **Sponsor**: Luca Chang ([@LucaButBoring](https://github.com/LucaButBoring))
- **Associated Working Group**: MCP Agents Working Group
- **Proposed Extension Identifier**: `io.github.stafordtituss/amep-evidence`
- **PR**: [#31](https://github.com/modelcontextprotocol/agents-ig/pull/31)

## Abstract

This proposal defines an optional MCP extension that lets tool servers return **signed receipts of the effects they actually executed**. A client or gateway attaches two correlation references to each `tools/call`: an invocation identifier, and the identifier of the authority the call is made under. A server that supports the extension returns, in the result's `_meta`, a receipt describing what it did: the action, the resource, the effect class and the data released. The receipt is signed with a key the server publishes. An auditor can then check, offline and without trusting the agent or its host, whether each executed effect stayed within the authority the user delegated.

The receipt format is taken from the Agentic Mandate Evidence Protocol (AMEP), an open specification with an IETF Internet-Draft, a 118-vector conformance suite and reference verifiers in Python and JavaScript. The extension adds no methods and changes no core behavior. It uses only capability negotiation and `_meta`, and peers that don't advertise it behave exactly as today.

## Motivation

When an agent acts through MCP tools, the only record of what happened is usually written by the agent's own runtime. MCP gives no way to prove afterwards what a tool call actually changed:

1. **Tool annotations are self-description.** `readOnlyHint` and `destructiveHint` are hints a server gives about itself. Nothing records whether a particular call behaved as annotated.
2. **Results are not evidence.** A tool result is unsigned and may be summarized, truncated or rewritten before it reaches a log. An incident reviewer cannot tell "the server deleted three records" from "the agent said it did".
3. **The agent's own record can be wrong.** A prompt-injected agent, a buggy or compromised host, or someone editing logs afterwards can each make the agent-side record disagree with what really happened, and nothing outside the agent's stack can show it.
4. **Delegation is invisible at the server.** The server sees a credential, not the chain of authority behind the call, so audit records on the two sides cannot be joined reliably.

Organizations in regulated settings, such as financial services and manufacturing, need audit trails that survive these gaps. Today each gateway vendor invents its own log format, so evidence from one product cannot be checked by another.

The core protocol has no defined place for signed, server-attested effect records, or for correlation references that survive across hops. An optional extension fits: only deployments that want audit evidence pay for it, and everyone else can ignore it.

## Specification

The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are to be interpreted as described in RFC 2119 and RFC 8174.

### Extension Identifier

`io.github.stafordtituss/amep-evidence`

### Capability Negotiation

**Server settings.** A server advertises the extension under `capabilities.extensions`, in the `server/discover` result (protocol `2026-07-28` and later) or the `initialize` result (`2025-11-25` and earlier):

```json
"extensions": {
  "io.github.stafordtituss/amep-evidence": {
    "revision": "0.2",
    "effectReceipts": true,
    "keys": "https://tools.example/.well-known/amep-keys.json"
  }
}
```

| Field | Type | Meaning |
|---|---|---|
| `revision` | string | Revision of this extension. This document defines `"0.2"`. A peer that receives a revision it does not know SHOULD NOT enable the extension. |
| `effectReceipts` | boolean | The server returns effect receipts in tool results. |
| `keys` | string (HTTPS URL) | Where the server publishes its key set (see [Key Set](#key-set)). |

**Client settings.** A client advertises the extension in `_meta["io.modelcontextprotocol/clientCapabilities"].extensions` on each request (protocol `2026-07-28` and later) or in the `initialize` request (`2025-11-25` and earlier):

```json
"io.github.stafordtituss/amep-evidence": { "revision": "0.2", "wantsReceipts": true }
```

A server MUST NOT include receipts unless the client advertised `wantsReceipts: true` for the request. On `2026-07-28` requests, the per-request capabilities are authoritative.

### Discovery and Filtering

Because the server's settings appear in `server/discover` (and can be repeated in a Server Card or registry listing), a client can learn **before calling any tool** whether a server signs receipts. A client or host MAY use this to prefer or require servers that advertise `effectReceipts: true`, for example only for tools whose effects it considers high-impact.

### Request Metadata

A client or gateway that has advertised the extension SHOULD include in a `tools/call` request's `params._meta`:

| Key | Value |
|---|---|
| `io.github.stafordtituss/amep-invocation-id` | string, at most 256 characters: a unique identifier for this call |
| `io.github.stafordtituss/amep-mandate-ref` | string: the identifier of the mandate or delegation the call is made under |

A server MUST treat these values as correlation identifiers only. It MUST NOT use them for authorization, which comes from the transport's own credentials.

### Result Metadata: the Effect Receipt

A server that advertised `effectReceipts`, and received a request with `wantsReceipts: true` and an invocation identifier, SHOULD include a receipt in the result's `_meta`:

```json
"_meta": {
  "io.github.stafordtituss/amep-effect-receipt": {
    "id": "server:tickets.example:e-77",
    "kind": "effect",
    "invocation": "i-4",
    "observed_by": "server:tickets.example",
    "assurance": "L3",
    "executed": {
      "action": "saas.update",
      "resource": "tickets:issue/4821",
      "effect": "write_shared",
      "released": [],
      "reversible": true,
      "t": 1767225611
    },
    "proof": { "alg": "Ed25519", "kid": "tickets-observer-1", "sig": "…" }
  }
}
```

Requirements:

1. The receipt describes what the server **actually executed**, not what was requested.
2. `invocation` MUST equal the request's `amep-invocation-id`. If the request had none, the server MUST NOT return a receipt.
3. `effect` MUST be one of `read`, `write_local`, `write_shared`, `external_send`, `destructive` or `financial`. `resource` MUST be a canonical identifier (AMEP Section 6.2). `assurance` is `L3` when the server itself executed the effect.
4. The receipt MUST be signed with Ed25519 over the RFC 8785 (JCS) canonical form of the receipt without its `proof` member, by a key in the server's key set whose roles include `observer`. Numbers in a receipt MUST be within the I-JSON safe range (RFC 7493), so that every verifier canonicalizes them identically.
5. A server that executed nothing, for example after a validation error, SHOULD NOT return a receipt.
6. A server that executed more than one effect for one call MAY return an array of receipts under the same key.
7. The extension never changes a result's `content` or `structuredContent`.

### Key Set

The `keys` URL serves a JSON object mapping each key identifier to its public key:

```json
{
  "tickets-observer-1": {
    "kid": "tickets-observer-1",
    "alg": "Ed25519",
    "x": "<32-byte public key, hex>",
    "roles": ["observer"],
    "nbf": 1767225600,
    "exp": 1798761600,
    "revoked": false
  }
}
```

A client MUST fetch the key set over HTTPS. It MUST ignore keys that are revoked, outside their `nbf`–`exp` window, or without the `observer` role, and it MUST NOT trust a server's key set for any role other than `observer`.

### Client Handling

1. A client MUST NOT treat a receipt as evidence unless its signature verifies against the server's key set and its `invocation` matches the call.
2. A client or gateway SHOULD store verified receipts with its own record of the call, so that a verifier can later evaluate the call against the user's delegated authority.
3. A client that cannot verify a receipt, for example because the key set is unreachable, MUST NOT treat it as evidence, and SHOULD record that verification failed.
4. Clients SHOULD NOT place receipts in the model's context. Receipts are audit data, not tool output.

### Partial Adoption and Graceful Degradation

- If either peer does not advertise the extension, both behave as if it did not exist. Tool calls work exactly as in the core protocol.
- A deployment does not need every server to support the extension. Calls to servers without it carry no server-attested evidence, and an evaluator reports them as unresolved rather than verified. Missing evidence is never treated as permission.
- A malformed receipt MUST NOT cause the tool call to fail. The client ignores it and SHOULD log the problem.

## Rationale

**Why an extension, not core.** Most MCP deployments don't need signed evidence, and requiring every server to manage signing keys would add cost for no benefit. The intended destination is Stable, not Core.

**Why `_meta`, not new methods.** A receipt belongs to the call it describes. Returning it on that call's result keeps correlation trivial, needs no extra round trip, and leaves existing methods untouched.

**Why the server signs.** A receipt's value comes from who attests it. A record produced by the agent or its host is self-reporting. The tool server is a separate party, and the only one that knows what it actually did. In a reference demonstration with real MCP sessions, server-signed receipts expose:
- an agent hijacked by an instruction hidden in a support ticket;
- a host that misreports a call in its own log;
- log entries deleted or edited afterwards.

None of these is visible from the agent-side record alone.

**Alternatives considered.**

- *OpenTelemetry spans only:* good for correlation, but unsigned and mutable, so they cannot be checked offline. The extension is complementary: the invocation identifier can be carried as a span attribute.
- *SCITT transparency logs only:* complementary, not a substitute. Receipts defined here can be registered as SCITT signed statements.
- *A new `tools/receipt` method:* adds a round trip and a race between a call and its receipt, for no gain.
- *Tool annotations:* self-description, so they cannot serve as evidence of what a particular call did.

**Prior art.** The IETF AUDIT architecture (draft-kuehlewind-audit-architecture), SCITT, OAuth 2.0 Token Exchange (RFC 8693) for delegation chains, and OpenID AuthZEN for authorization decisions. AMEP builds on these and defines how to evaluate whether an executed effect stayed within delegated authority.

## Backward Compatibility

No backward-compatibility concerns. The extension is opt-in on both sides, adds no methods, and does not change the content of any existing message.

## Security Implications

- **Correlation identifiers are not credentials.** Servers must not grant or widen access based on `amep-mandate-ref` or `amep-invocation-id`.
- **Trust in the key set rests on HTTPS.** A receipt shows what the server attests it did. It protects against a misbehaving agent or host, not a dishonest server. Deployments needing more can require receipts from more than one independent observer, or anchor key sets in a transparency log.
- **Key compromise.** A stolen `observer` key could forge receipts for that server. Servers should keep observer keys separate from TLS and OAuth keys, rotate them, and mark revoked keys in their key set.
- **Untrusted content.** Receipts, like all tool results, are untrusted input and should be kept out of the model's context.
- **Privacy.** Receipts reveal which resources a user's agent touched. They carry canonical identifiers and data-class labels, not payloads. Servers should avoid personal data in resource identifiers, and stored receipts should be handled as sensitive audit data.
- **Replay.** A receipt binds to one invocation identifier and time. Verifiers reject a receipt whose invocation is not in the evidence they evaluate.

## Reference Implementation

- **TypeScript SDK (v2.4.0):** a server half that advertises the extension and signs receipts for the effects each tool records, and a client/gateway half that opts in, discovers support before calling tools, fetches and filters the key set, verifies receipts and evaluates the evidence. 24 integration tests run real MCP sessions on both protocol eras (`initialize` and `2026-07-28` with `server/discover` and per-request capabilities), over in-memory, HTTP and stdio transports, and cross-check verdicts against the Python verifier: https://github.com/stafordtituss/amep/tree/main/prototypes/mcp-amep-evidence-ts. A follow-up PR adds it to `packages/`, ported to the `{ id, setup(ctx) }` extension API (typescript-sdk#2820) once released: `ctx.capability(...)` for advertisement and `ctx.use('tools/call', ...)` to attach receipts without wrapping each tool.
- **Python SDK:** a prototype on the official MCP Python SDK's extension interface: https://github.com/stafordtituss/amep/tree/main/prototypes/mcp-amep-evidence
- **Evidence format and verifiers:** https://github.com/stafordtituss/amep, published as `amep-protocol` (PyPI) and `@stafordtituss/amep` (npm), with an IETF Internet-Draft (draft-stafordtituss-agentic-mandate-evidence). Both reference verifiers are by the same author; independent implementations are welcome.

## Discussion

- Presented at the MCP Agents Working Group call on 2026-11-20. Feedback from the call:
  - take it forward as an extension, using the TypeScript SDK's extension points;
  - support discovery and filtering of servers that sign receipts;
  - clarify partial adoption across servers;
  - consider regulated industries (financial services, manufacturing) as early adopters.

## Open Questions

1. Should Server Cards or the registry `server.json` carry the extension settings, so clients can filter servers before connecting?
2. Should multiple receipts per call stay in this revision, or be deferred?
3. Should a later revision let a server link each tool to a published effect mapping (which action, resource and effect class a tool produces), so clients can detect tool-definition changes?
4. Review from the Security Interest Group, whose charter includes "tamper-evident records of what a tool call did and under what authority", would be welcome before this moves beyond Draft.
