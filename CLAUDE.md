# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Status

permgrade is **design-stage**: the only tracked source artifacts are `DESIGN.md`, `voltagent/DESIGN.md`, `LICENSE`, and `.gitignore`. There is **no code, no `Cargo.toml`, and no build system yet**. The architecture below is the committed plan; implement against it rather than inventing a new structure.

permgrade is a localhost-only, BYOK proof-of-concept that scores a Chrome extension's credential-exposure / security risk from its Chrome Web Store listing + manifest. It emits a **trustworthiness score**, a **recommended action** (`install` | `install-with-caution` | `avoid` | `rotate-credentials`), a **confidence**, and a **reason chain** of the signals that moved the score.

## Commands

No build system is scaffolded yet. The intended stack (per `DESIGN.md` → Tech Stack) is a **Rust** workspace compiled to **WASM**, with **opencode** servers running the LLM and policy invocations.

Once the Cargo workspace exists, standard tooling applies (don't hand-run these to "verify" — show them):
- Build: `cargo build`
- Test all: `cargo test`
- Test one crate: `cargo test -p permgrade-policy`
- Test a single test: `cargo test -p permgrade-policy <test_name>`
- Lint: `cargo clippy --all-targets`
- Format: `cargo fmt`

When you create the first crate, prefer a workspace (`[workspace]` root `Cargo.toml`) with one member per module below.

## Architecture

### Crate layout (planned)

| Crate | Responsibility |
|---|---|
| `permgrade-app` | The extension-scoring application composed from the primitives below. |
| `permgrade-policy` | Deterministic rule engine. Generic over "subject + signals → score + reasons". Owns Tier-1 hard-gates and Tier-2 detection. Swappable rule set. |
| `permgrade-llm` | LLM invocation wrapper: strict JSON-schema output, bounded score-adjustment, refusal-on-low-confidence path, token budget, timeout, structured failure handling. |
| `permgrade-audit` | Append-only audit log keyed by **hashed inputs**; replay tool to re-score historical decisions against new policy/model versions. |
| `permgrade-eval` | Eval harness with hand-labeled ground truth; fits the weight bands; diff tool for rubric/prompt changes. Generic over any structured-output classifier. |

### Scoring pipeline

The full 20-signal rubric is the source of truth in **`DESIGN.md` → Scoring Rubric** — read it before touching scoring code. The shape:

Each signal is placed in one of three tiers by a single test — *can a legitimate extension ever produce this exact signal?*
- **Tier 1 — Hard-gate** (`permgrade-policy`): deterministic *and* no legit producer → **short-circuits to `avoid`, LLM is NOT consulted**. Scope is deliberately strict: (a) exfil host on a known-C2 list, (b) logically-impossible permission combos.
- **Tier 2 — Raise + flag** (`permgrade-policy` → `permgrade-llm`): deterministic to *detect*, contextual to *judge*. **Most signals live here.**
- **Tier 3 — LLM-only** (`permgrade-llm`): no deterministic anchor; bounded and confidence-gated.

Composite score (risk is **likelihood × impact**):
```
contribution(sig) = direction × weight_band × custody_factor
L (likelihood)    = Σ contribution(Tier-2)  +  bounded LLM delta(Tier-3)
I (impact)        = g(install_count, target_domain_sensitivity)   # multiplier ≥ 1
risk              = L × I
trustworthiness   = invert(clamp(risk))
```

### Invariants — preserve these; they are easy to violate and central to the design

- **Tier 1 short-circuits** before any LLM call — return `avoid` with high confidence and a single-rule reason chain.
- **All listing/review/manifest text reaches `permgrade-llm` framed as untrusted *data*, never instructions** (injection hardening; pairs with strict-JSON-schema output).
- **The LLM's adjustment is asymmetric**: it may raise risk freely (≤ +X) but lower it only within a small bound (≤ −Y, with Y ≪ X), and **not at all while a tamper-resistant flag is live** (known-C2 host, harm-keyword review spike). Risk-lowering contributions are hard-capped.
- **Temporal / corpus signals** (permission-escalation diff, dormancy spike, ownership-change, publisher fan-out) are unavailable on a first scan → they carry an explicit **`unknown` state, NEVER a default-safe `0`**. They gain value as `permgrade-audit` accumulates re-scans — permgrade is strongest as a re-scanning monitor, not a one-shot checker.
- **Install count is an impact multiplier only** (≥ 1) — it never lowers risk and never manufactures risk from nothing.
- **Weight bands (H/M/L) are placeholders**, fitted by `permgrade-eval` against hand-labeled ground truth to minimize false-negatives. Don't hardcode tuned decimals.
- **Every decision logs a reason chain to `permgrade-audit` by hashed input.**
- **localhost-only**: data is fetched from the Web Store and scored locally; nothing is exfiltrated.
- **Refusal path** on injection-detected, low LLM confidence, or genuine ambiguity.

### Build order

The signals cluster into four layers that also give the shipping sequence:
1. **Deterministic manifest scorer** (MVP): `permgrade-policy` + score composition. Manifest-only signals, `I = 1` until install count is fetched, known-C2 (#7) stubbed. A working, explainable scorer with no LLM and no history.
2. **LLM layer** (the false-positive cure): `permgrade-llm` category-fit (#15), localization (#17), review sentiment (#18), ownership-change judging (#13), plus the untrusted-data / asymmetric-bound / refusal hardening. Without this, Phase 1 over-flags every password manager.
3. **Corpus / temporal layer**: needs `permgrade-audit` history + a cross-extension corpus — escalation diff (#9), dormancy (#10), fan-out (#14), real known-C2 (#7). Plumb the `unknown` state.
4. **Eval & tune**: `permgrade-eval` — labeled corpus fits the weight bands (X / Y / custody_factor); diff tool for rubric/prompt changes.

## Key Documents

- `DESIGN.md` — product overview, threat model, trust boundary, and the authoritative **Scoring Rubric** (20-signal matrix). The contract for all scoring work.
- `voltagent/DESIGN.md` — the UI/UX design language (Voltagent-inspired: dark canvas, single electric-green accent, hairline cards, Inter + SF Mono) and component primitives for the scoring surface.

---

## Working Guidelines

Behavioral guidelines to reduce common LLM coding mistakes. Merge with the project-specific instructions above as needed.

**Tradeoff:** These guidelines bias toward caution over speed. For trivial tasks, use judgment.

### 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

### 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

### 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

### 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

---

**These guidelines are working if:** fewer unnecessary changes in diffs, fewer rewrites due to overcomplication, and clarifying questions come before implementation rather than after mistakes.
