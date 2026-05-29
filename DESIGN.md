It's been known that #[many chrome extensions are malicious](https://socket.dev/blog/108-chrome-ext-linked-to-data-exfil-session-theft-shared-c2). This repo is a proof-of-concept that uses LLMs to analyze the full surface of a Chrome extension and scores it ssecurity risk.

# Overview
 
- BYOK via an opencode-powered TUI
- Analyze the full surface of a Chrome extension
- Score the security risk posed by the extension

# Tech Stack

- [opencode](https://opencode.dev/)
- [rust](https://www.rust-lang.org/)
- [wasm](https://developer.mozilla.org/en-US/docs/WebAssembly)

# UI/UX Design

The UI/UX design for this project is specified in [`voltagent/DESIGN.md`](voltagent/DESIGN.md). It documents a Voltagent-inspired design language — a dark, near-black canvas with a single electric-green accent — covering the full token set (colors, typography, spacing, radii) and the component primitives (nav, buttons, hairline-bordered cards, code mockups, hero/content bands, footer) used to render the extension scoring surface.

# Threat Modeling

- [Supply chain attacks](https://socket.dev/blog/108-chrome-ext-linked-to-data-exfil-session-theft-shared-c2) have been known and become a real problem for Chrome extensions. Google account identity is a common target. Once a malicious extension is installed, the attacker can obtain temporary session tokens and read/write data from the user's account. Beyond that, Telegram/Youtube/Tiktok etc. session tokens are also targets for data exfiltration.
- By reading through the Chrome extension's permissions, we can score its potential security risk.
- In the future, we plan to analyze the full surface of the extension, including the permissions, the code, the UI, etc.

## Input and Output

- Input: Chrome extension's Chrome Web Store URL
- Output: A score of the security risk posed by the extension
    - Credential-interception capability implied by declared permissions (e.g., "with <all_urls> + webRequest, this extension can read form submissions on every site, including login forms")
    - Triggered deterministic rules (e.g., "permission set matches the classic credential-exfiltration pattern despite the extension describing itself as a calculator")
    - LLM-evaluated signals (e.g., description-vs-permission mismatch, ownership-change indicators, developer-reputation flags)
    - Recommended user action (install / install with caution / avoid / rotate-affected-credentials)
    - Confidence score and explicit refusal path when signals are ambiguous

- Governing rules:
    - The deterministic policy layer is implemented using a set of rules. It overrides the LLM-evaluated signals.
    - The LLM-evaluated signals are implemented using opencode-powered LLMs. It's invoked to handle ambiguity only if necessary.

## Architecture

The code is organized into a related group of rust modules:

- `permgrade-app/` — the actual extension-scoring application composed from the above primitives
- `permgrade-audit/` — append-only audit log with hashed inputs, replay tool for re-scoring historical decisions against new policy or model versions
- `permgrade-eval/` — eval harness with hand-labeled ground truth, scoring, diff tool for rubric/prompt changes; generic over any structured-output classifier
- `permgrade-llm/` — LLM invocation wrapper with strict JSON schema output, bounded score-adjustment range, refusal-on-low-confidence path, token budget, timeout, structured failure handling
- `permgrade-policy/` — deterministic rule engine, swappable rule set, generic over any "subject + signals → score + reasons" classification task

### Trust Boundary

- Data is fetched from the Chrome Web Store URL
- LLM invocation is implemented by calling a vanilla opencode server over localhost (`permgrade-llm`)
- Policy evaluation is a deterministic Rust crate (`permgrade-policy`) — in-process, consulting no LLM and making no network calls, which is what makes the Tier-1 short-circuit verifiable and auditable
- Completely run from localhost. No data is exfiltrated to the outside world
- The decision chain by the deterministic policy layer and the LLM-evaluated signals is logged by hashing the inputs and appending to an audit log which also lives on the localhost

## Scoring Rubric

permgrade turns an extension's Web Store listing + manifest into a **trustworthiness score**, a **recommended action**, a **confidence**, and a **reason chain** — the ordered list of signals that moved the score, logged to `permgrade-audit` by hashed input.

### Signal tiers

Each signal is tiered by one test — *can a legitimate extension ever produce this exact signal?* — which replaces a flat deterministic-vs-LLM split:

- **Tier 1 — Hard-gate** (`permgrade-policy`): deterministic *and* no legitimate extension can produce it → overrides the LLM. Scope is deliberately **strict**: (a) an exfiltration host on a known-C2/malware list, (b) logically-impossible permission combinations. Short-circuits to `avoid`.
- **Tier 2 — Raise + flag** (`permgrade-policy` → `permgrade-llm`): deterministic to *detect*, contextual to *judge*. Raises risk and hands the LLM a specific question. **Most signals live here.**
- **Tier 3 — LLM-only** (`permgrade-llm`): no deterministic anchor; bounded and confidence-gated.

Because Tier 1 is strict, hard-gating is rare and **the LLM is load-bearing for most verdicts** — its confidence and refusal-on-low-confidence path is central, not a fallback.

### Composite score

Risk is **likelihood × impact**, so a benign extension stays benign no matter how many users it has, while a malicious one scales with its blast radius:

```
contribution(sig) = direction × weight_band × custody_factor   # tamper-resistant signals weigh more;
                                                                # risk-lowering contributions are hard-capped
                                                                # and disabled while a tamper-resistant flag is live

L (likelihood) = Σ contribution(Tier-2)  +  bounded LLM delta(Tier-3)   # raise ≤ +X, lower ≤ −Y, with Y ≪ X
I (impact)     = g(install_count, target_domain_sensitivity)           # multiplier ≥ 1, never lowers risk

risk            = L × I
trustworthiness = invert(clamp(risk))
action          = install | install-with-caution | avoid | rotate-credentials
```

**LLM hardening:** all listing/review/manifest text reaches `permgrade-llm` framed as untrusted *data*, never instructions (pairs with its strict-JSON-schema output). The LLM's adjustment is **asymmetric** — it may raise risk freely but lower it only within a small bound, and not at all while a tamper-resistant flag (known-C2 host, harm-keyword review spike) is live. Injection detected, low confidence, or genuine ambiguity → **refusal path**.

**Temporal signals** (permission-escalation diff, dormancy spike, ownership-change, publisher fan-out) need version history or a cross-extension corpus, so they are **unavailable on a first scan** — they carry an explicit `unknown` state (never a default-safe `0`) and gain value as `permgrade-audit` accumulates re-scans. permgrade is therefore strongest as a re-scanning monitor, not only a one-shot checker.

### Signals

Categories: `INT` interception · `STO` storage-access · `EXF` exfiltration-vector · `—` none/meta. Direction: `↑` raises risk · `↕` non-monotonic · `×` impact multiplier. Weight bands (H/M/L) are starting points; `permgrade-eval` fits the numbers against hand-labeled ground truth to minimize false-negatives.

| # | Signal | Cat | Tier | Dir | Wt | Failure mode if the signal is wrong |
|---|--------|-----|------|-----|----|----|
| 1 | Host scope (`activeTab` vs `<all_urls>`) | INT | T2 | ↑ | M | Broad-but-legit false positive; meaningful only with #15 |
| 2a | `<all_urls>` + `webRequest` (read requests on every site) | INT | T2 | ↑ | H | Ad-blockers etc.; needs #15 to excuse |
| 2b | `cookies` + remote/unrelated host (read + ship session) | STO·EXF | T2 | ↑ | H | Session-sync tools; weighted marginal over #3 to avoid double-count |
| 2c | `webRequestBlocking`/`declarativeNetRequest` + content scripts on auth domains | INT | T2 | ↑ | H | Legit SSO / auth helpers |
| 3 | Cookie/session access (base `cookies` permission) | STO | T2 | ↑ | M | Password managers; #2b escalates this |
| 4 | Sleeper capability (powerful `optional_permissions` requested at runtime) | INT | T2 | ↑ | M-H | Legit lazy-loaders; weight by which permission is deferred |
| 5 | MV3 dynamic injection (`scripting` + broad host) | INT | T2 | ↑ | M | Legit dynamic UIs |
| 6 | Exfil-destination mismatch (`host_permissions`/`externally_connectable` unrelated to function; raw IP / odd TLD) | EXF | T2 | ↑ | H | Legit analytics/CDN → allowlist + host rarity |
| 7 | **Known-bad infrastructure** (exfil host ∈ known-C2 list) | EXF | **T1** | ↑ clamp | — | Stale/incorrect blocklist → maintained list + audit trail |
| 8 | **Logically-impossible permission combo** | * | **T1** | ↑ clamp | — | Definitional error → keep the set tiny and curated |
| 9 | Permission-escalation diff (version N+1 adds capability N lacked) | INT | T2 | ↑ | H | Honest feature growth → hand the diff to the LLM |
| 10 | Dormancy spike (long-silent → sudden update) | — | T2 | ↕ | M | Legit maintenance after hiatus → combine with #9 / #13 |
| 12 | Developer reputation (website present, domain age, prior takedowns) | — | T2+T3 | ↑ | M | Raise-only; thin-footprint indie devs → cold-start false positive |
| 13 | Ownership-change (developer name / support-email domain / privacy-policy host / homepage drift) | — | T2 detect → T3 judge | ↑ | M-H | Corporate rebrand or acquisition is legitimate → LLM judges the flagged change |
| 14 | Publisher fan-out / shared backend (rare host, contact email, boilerplate, asset hashes shared across many extensions) | EXF | T2 *(T1 if host ∈ C2)* | ↑ | H | Shared legit SaaS/SDK → host-rarity weighting + allowlist |
| 15 | Permission-fits-claimed-category (LLM infers category, judges in-band vs out-of-band; also covers description clarity) | — | T3 | ↑ | H | Novel/hybrid products misread → bounded LLM delta + confidence gate |
| 17 | Listing localization mismatch (declared locale vs actual language, machine-translation tells) | — | T3 | ↑ | L-M | Imperfect legit i18n → weak weight, combine with others |
| 18 | Harm-keyword negative-review spike (recent victim reviews: "hijacked", "redirect", "stole"…, ideally post-update) | — | T2 (vol/recency) + T3 (sentiment) | ↑ | H | Review-bombing / sabotage → require volume + recency + specificity. *Tamper-resistant ⇒ high weight* |
| 19 | Review/rating trust signals (volume, rating level, age) | — | T2 | ↑ | L | Bought reviews → positives ≈ 0 evidence; raise-only |
| 20 | **Install count → blast radius** | — | **IMP** | × | — | Impact multiplier only; multiplies existing likelihood, never manufactures risk from nothing |

_Companion impact term:_ **target-domain sensitivity** — whether host permissions reach known auth / banking / identity domains; feeds `I`, not `L`.

_Side advisory:_ **update cadence / abandonment** — staleness is an *unpatched-vulnerability* risk, a different axis from malice; reported alongside the score but excluded from trustworthiness.

# Todo

- [ ] Fetch the extension's manifest using its Chrome Web Store URL
- [ ] Analyze the manifest to score the security risk using opencode-powered LLMs (BYOK)
- [ ] (TBD) Fetch the extension's source code and score its security risk using opencode-powered LLMs (BYOK)
