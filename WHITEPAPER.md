# Renker: Trusted Infrastructure for Autonomous AI

**A synthesis of the platform's architecture, threat model, and honest limitations**

Sebastian Renker · Version 0.1 · 2026-08

> **Status.** This is a research/prototype whitepaper. It synthesizes documents that
> already live in the Renker repositories and states, for every claim, what the code
> actually enforces today versus what is planned or out of scope. No Renker component
> has undergone an external security audit. Where a property is partial or planned, this
> document says so — a security claim you cannot falsify is not worth much.

---

## Abstract

AI systems are becoming capable of *acting* — operating a computer, sending messages,
running experiments — before a human reviews each step. Renker is infrastructure for
making those actions **controllable, verifiable, and accountable**. Its central design
commitment is simple: **capability must not imply permission**. The decision of what an
agent may do lives in a small, deterministic, auditable layer *outside* the model, which
reads only trusted grants — never the agent's own claims. Prompt injection can change
*what an agent requests*; it cannot change *what it is allowed to do*.

This document consolidates the platform's architecture (§3–4), its threat model (§5), the
secure-communication and accountable-learning slices (§6–7), and — deliberately — its
explicit non-guarantees (§8). It is meant to be cited and to be checkable against the
source repositories listed in §10.

---

## 1. Introduction

An LLM agent's guardrails, when they live *inside* the model, can be argued around by the
very inputs the agent processes: a web page, a file, a tool result. This is the structural
weakness behind prompt injection. Renker's response is not a better prompt but a different
place for the decision: a separate enforcement layer that treats the agent's request as
untrusted data and confines it to explicitly granted capabilities.

The platform is organized around three concrete questions about autonomous AI, each
answered by one project over a shared foundation:

- **ACT** — can an agent that operates a computer be confined to what it was permitted?
- **LEARN** — can an autonomous research system be prevented from asserting unverified
  claims as fact?
- **SECURE** — can agents and people communicate so that the infrastructure itself never
  sees plaintext?

## 2. Design principles

1. **Authorization outside the model.** The agent produces a *request*; a deterministic
   function decides. The request is never trusted to be within bounds.
2. **Least privilege by construction.** A capability grants exactly one action verb on one
   scope, immutable once issued. Nothing widens silently.
3. **Fail closed.** Errors, malformed input, missing grants, and every unmet condition
   resolve to `DENY`. This extends to configuration: an unreadable enforcement config
   denies rather than silently allowing.
4. **Tamper-evident accountability.** Every decision — allow and deny — is appended to a
   hash-chained audit log with a runnable `verify()`.
5. **Honesty over marketing.** Every repository ships a "what this does not protect
   against" section; this whitepaper collects those non-guarantees in §8.

## 3. Platform architecture

Three products consume one shared set of primitives — identity, capability, permission,
policy, and audit — so the security logic is written once, not reinvented per product.

```
                         RENKER
        trusted infrastructure for autonomous AI
                            │
     ┌──────────────┬───────┴───────┬──────────────┐
   ACT            LEARN           SECURE
  rencora        continuum       renkervault
  desktop agent  Phase-0 research E2E messaging
     └──────────────┴───────┬───────┴──────────────┘
                            │
                       renker-core
                    shared foundation (proprietary)
                            │
                     renker-core-authz
        public authorization slice · Apache-2.0 · 0 deps
              Identity · Capability · Policy · Audit
```

**Open-core split.** The broader platform foundation, `renker-core`, is proprietary. Its
authorization primitive — deterministic capability + policy + tamper-evident audit — is
published independently as **`renker-core-authz`** under Apache-2.0, dependency-free and
usable standalone. The most security-critical piece is therefore the most openly testable
one.

## 4. The authorization core

Every guarded action follows one path:

```
Identity → Capability → Policy → Decision → Execute / Audit
```

- **Identity.** An `Actor` (`kind:identifier`, e.g. `agent:session-1`) is validated and
  canonicalized. It is **validated, not cryptographically authenticated** — the caller must
  supply a trusted actor (see §8).
- **Capability.** An immutable grant: one dotted action verb, one path scope, an issuer, an
  issue/expiry time, and an approval policy (`auto` / `human` / `deny`). Least privilege is
  structural — a write grant cannot also permit reads elsewhere.
- **Policy.** `evaluate(actor, action, target, store)` reads **only** trusted grants and the
  requested (actor, action, target). It returns one of three decisions with an explainable
  reason:
  - **ALLOW** — a matching, unexpired, unrevoked capability permits the exact action on a
    target inside its scope.
  - **DENY** — no such capability, or any check fails (safe default).
  - **REQUIRE_APPROVAL** — the matching capability's policy requires a human to confirm.
- **Scope containment** is traversal-, prefix-confusion-, and case-safe: paths are resolved
  (collapsing `..`) and compared on normalized path *parts*, never string prefixes.
- **Audit.** Each decision appends one structured event to a SHA-256 hash chain with a
  separate head anchor; `verify()` detects modification, insertion, reordering, and tail
  truncation.

A runnable, dependency-light demonstration — an MCP server whose every tool call is routed
through this core, defeating a prompt-injection attack — is provided in `renker-agent-demo`
(§10).

## 5. Threat model

*Scope: the first platform slice — the guarded action `Identity → Capability → Policy →
Decision → Execute/Audit`. Claims are limited to what the code enforces, verified by the
security test suites.*

### 5.1 Assets

- **A1** — Files outside a granted scope (no unpermitted read/write).
- **A2** — Integrity of the authorization decision (no input turns DENY into ALLOW).
- **A3** — Integrity of the audit trail (later alteration is detectable).
- **A4** — Least privilege of grants (a grant does not silently widen).

Explicitly **not** protected here: file content confidentiality (no at-rest encryption in
this slice) and a fully compromised host OS account.

### 5.2 Actors and trust

| Actor | Trust | Notes |
|---|---|---|
| Human owner | trusted | issues / revokes capabilities |
| AI agent | **semi-trusted** | may be manipulated; its *intent* is untrusted |
| Local process | untrusted unless granted | must route through the guard to act |
| Remote service | untrusted | |
| Compromised agent | adversary | tries to exceed its capability |
| Malicious content / tool output | adversary | reaches the agent via prompt injection |

### 5.3 Trust boundaries

- **B1 — Agent reasoning ↔ enforcement.** The agent's request is never trusted to be in
  bounds; the enforcement layer decides.
- **B2 — External content ↔ instructions.** External content is data, never instructions;
  the core assumes the request may already be adversarial and still confines it.
- **B3 — Caller ↔ actor identity.** The core validates and canonicalizes an `Actor` but does
  not authenticate it.

### 5.4 Attack surfaces and enforcement

| Surface | Enforced? | Mechanism |
|---|---|---|
| Path traversal (`drafts/../secret`) | ✅ | resolve, then containment on path parts |
| Prefix confusion (`Documents2` vs `Documents`) | ✅ | compare resolved parts, never string prefix |
| Capability escalation (read→write, wider path) | ✅ | exact action match; scope check on resolved target |
| Confused deputy / wrong actor | ✅ | `granted_to == actor.urn` required |
| Expired / revoked capability | ✅ | UTC expiry check; revocation list |
| Malformed input | ✅ | any failed check → **DENY** (safe default) |
| Audit manipulation | ⚠️ partial | hash chain + head anchor detect edits; do not prevent deletion if both are rewritten |
| Replay | ⚠️ by design | a valid capability is reusable until expiry/revocation; no per-use nonce |
| Forged identity | ❌ open | validated and matched, not authenticated |
| Prompt injection | out of core scope | mitigated in the agent; the core guarantee is that injection changes the *request*, not the *capability* |

### 5.5 Guaranteed properties

Confinement, least privilege, explainability, auditability, tamper-evidence, and a safe
(fail-closed) default. Robustness additions verified in hardening: case-correct scope
containment (cross-checked against ground truth by property tests), input validation,
lock-guarded concurrent audit appends, `fsync` durability with atomic anchor replacement,
and corruption surfaced as a typed error rather than a raw parse failure.

## 6. Secure communication (SECURE / renkervault)

`renkervault` is a content-blind, end-to-end-encrypted messaging prototype whose relay only
ever sees ciphertext and routing metadata — whether the endpoint is a person or an agent. It
is **not** zero-knowledge in the cryptographic sense: no zero-knowledge proofs are used, and
the relay does observe connection metadata (account IDs, public keys, timing). The property is
content-blindness (no plaintext), not metadata-blindness.

- **Initial key agreement** is a hybrid X3DH-lite: classical X25519 ECDH **plus** ML-KEM-768
  (FIPS 203), mixed through HKDF-SHA-256 — the same "harvest-now-decrypt-later" defense
  Signal ships as PQXDH. Only the initial handshake is post-quantum; the ongoing ratchet is
  not, a documented, industry-standard boundary.
- **Message encryption** is a Double Ratchet (X25519 + HKDF-SHA-256 + AES-256-GCM): a fresh
  key per message (forward secrecy) and a fresh root key per round trip (post-compromise
  security).
- **At rest**, the local vault is Argon2id → key-encryption-key → AES-256-GCM with an HMAC
  integrity tag; no plaintext on disk.

The crypto layer is covered by an example-based and property-based test suite, including
known-answer vectors (RFC 7748 X25519, RFC 4231 HMAC-SHA-256, RFC 5869 HKDF, FIPS 203
ML-KEM-768 sizes and round-trip) and fast-check invariants for the ratchet state machine and
the hybrid handshake.

## 7. Accountable learning (LEARN / continuum)

`continuum` is a Phase-0 architecture prototype of a continuously learning research system,
run against a *simulated* target function rather than real lab hardware. Its safety posture
is architectural, so it need not be retrofitted later:

- **Evidence provenance.** Every assertion about a material, hypothesis, or model result must
  be tagged `EXPERIMENTAL`, `PREDICTED`, or `LITERATURE`; unlabeled claims are not mergeable.
  This is a structural barrier against emitting hallucinated results as fact.
- **Governance gate + audit.** Every simulated experiment approval passes through a governance
  gate and is logged; no silent state change.
- **Honest boundary.** The hazard-screening ruleset is an *example* only and must be reviewed
  and extended by domain experts before any use with real materials; the audit log is
  traceable, not tamper-proof.

## 8. Explicit non-guarantees

Collected here so they cannot be lost in the prose:

- **No cryptographic authentication of actors.** Forged identity is mitigated by matching, not
  proven; the caller must supply a trusted `Actor`.
- **No protection once the host account is fully compromised** — an attacker who can rewrite
  both the audit log and its head anchor can erase history, and could feed a forged trusted
  actor.
- **No content confidentiality in the authorization slice** — files are read/written in the
  clear there (distinct from renkervault's E2E layer, §6).
- **No enforcement outside the guard.** An agent that calls the OS directly, bypassing the
  guarded executor, is not constrained. Enforcement is only as good as the routing of every
  action through one chokepoint.
- **Audit is tamper-evident, not immutable**, and single-writer: the in-process lock does not
  coordinate multiple OS processes on one log file. A crash between the entry `fsync` and the
  anchor replacement leaves a detected, operator-recoverable head mismatch.
- **Post-quantum protection covers the handshake, not the ongoing ratchet.**
- **No external security audit** of any Renker component. Internal hardening reviews are
  documented per repository.

## 9. Reproducibility and verification

The design goal is that claims are checkable, not merely asserted:

- `renker-core-authz` ships with adversarial, property-based, and fuzz tests plus CI (format,
  lint, types, tests, build, dependency audit) and zero runtime dependencies.
- `renkervault` runs its client and server security suites, a Rust check, and a secret scan in
  CI on every pull request.
- `renker-agent-demo` is a runnable end-to-end demonstration of the authorization core
  stopping a prompt-injection attack, with its own tests and CI.

## 10. References — source repositories

The claims in this document are traceable to these repositories and their in-repo documents:

- **renker-core-authz** — public authorization core (Apache-2.0):
  <https://github.com/sebastianrenker/renker-core-authz> (`README.md`, `SECURITY.md`).
- **renker-core** — shared platform foundation:
  <https://github.com/sebastianrenker/renker-core>
  (`docs/THREAT_MODEL.md`, `SECURITY.md`, `docs/PRODUCTION_READINESS.md`).
- **rencora** — ACT, desktop agent:
  <https://github.com/sebastianrenker/rencora> (`SECURITY.md`).
- **continuum** — LEARN, Phase-0 research system:
  <https://github.com/sebastianrenker/continuum> (`ARCHITECTURE.md`, `ROADMAP.md`).
- **renkervault** — SECURE, E2E messaging:
  <https://github.com/sebastianrenker/renkervault>
  (`docs/THREAT_MODEL.md`, `SECURITY.md`, `SECURITY_AUDIT.md`).
- **renker-agent-demo** — runnable authorization demo (MCP):
  <https://github.com/sebastianrenker/renker-agent-demo>.
- **Platform landing page & documentation hub:** <https://sebastianrenker.github.io>.

---

*How to cite:* see [`CITATION.cff`](CITATION.cff). *License:* text under CC BY 4.0 — see
[`LICENSE`](LICENSE).
