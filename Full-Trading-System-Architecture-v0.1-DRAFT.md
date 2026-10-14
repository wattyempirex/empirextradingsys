# Full Trading System Architecture — v0.1 DRAFT

**Status:** DRAFT for Pranav's review. Not frozen. Not a spec. A proposal to react to.
**Date:** 2026-10-13
**Context:** On 2026-10-13 Pranav decided to stop the Stage 2 crash-hardening loop ("let's put more energy into the full system design instead"). This document is that next step.

> The whole goal is to eventually get this to be a full trading system. — Pranav, 2026-10-13

---

## 1. Vision: what "full trading system" means

Right now we have excellent *instruments*: a discovery engine, a paper signal monitor, a pattern validator, a replay lab. But they are separate tools on a workbench. A **full trading system** is the closed loop:

**Find opportunities → decide entries and exits → manage risk across a portfolio → execute faithfully → measure everything → feed the measurements back into finding opportunities.**

Concretely, the full system does five things every session without human hands in the middle:

1. **Discovers** — scans the market and ranks what's worth attention (PM Edge does this today, in the morning).
2. **Signals** — says ENTER and EXIT at precise timestamps with a stated stop, target, and size (Stage 2 does this today, on paper).
3. **Executes faithfully** — turns signals into recorded positions without losing, duplicating, or mispricing anything, even if the machine crashes mid-thought (this is what the six repair rounds were about — and the single biggest gap between "research code" and "a system").
4. **Manages risk** — sizes positions, limits total portfolio heat, respects correlation, and has kill switches (nobody builds this yet — it's the largest missing piece).
5. **Learns** — every trade flows into per-pattern statistics, and those statistics change what the system looks for next (Pattern Farm measures this today, but nothing feeds the results back yet — the loop is open).

The difference between the workbench and the system is the **plumbing between the tools**: frozen interfaces, one source of truth for state, and a validation loop that closes. This document designs that plumbing.

**Paper-only until Pranav explicitly authorizes otherwise.** Nothing in this design assumes, requires, or sneaks toward live brokerage access. The system is built so that "going live" later means plugging a brokerage adapter into a frozen interface — not rewriting anything.

---

## 2. Current components and their roles

### PM Edge v0.2 — the discovery brain (Track S, swing)

**What it is:** Morning/midday/close scans over ~515 names. Agent A scores *Opportunity* (is this stock attractive for the next 2–10 days, /100); Agent B scores *Entry* (is this exact timestamp attractive, /100) and holds veto power. Commit-then-reveal keeps the two agents honest. Shadow trades track the setups we rejected, as a control group. Everything lands in frozen packets in `pm_edge.db`.

**Contributes:** the entire discovery-to-thesis half of the system — regime classification, catalyst scoring, the state machine (DISCOVERY → WATCH → CONFIRMATION → ENTER NOW → MANAGE → EXIT → REVIEW), and the four-way failure classification that tells us *where* alpha leaks.

**Lacks:** any execution (it only recommends), any portfolio awareness (each candidate is scored alone), and a closed feedback loop (farm-study results don't change its scoring yet).

### Intraday Sandbox Stage 1 — the replay lab

**What it is:** After the close, replays the day's bars and records what the six frozen strategies *would* have done. Explicitly labeled "forward-date simulation" — honest about being a replay, not live.

**Contributes:** clean, uncontaminated strategy observations; the qualified cohort is the only dataset allowed to speak about strategy quality.

**Lacks:** live evaluation by design — that's Stage 2's job.

### Stage 2 — the live signal monitor (research-grade)

**What it is:** A detached monitor that watches 5-minute bars during market hours and generates paper enter/exit signals in real time. After six adversarial repair rounds it has a write-ahead journal, honest timestamps, and labeled reference-only P&L — but Caliper never accepted it, so it runs as **diagnostics-only, research-grade**.

**Contributes:** the live signal-generation half of the system, plus the hard-won crash-safety lessons (Section 3.3) that the full system's execution layer will be built on — not retrofitted from.

**Lacks:** Caliper acceptance, genuinely-live data (still 60-second polling on delayed bars — the websocket proposal is unconfirmed), and executable fills (all P&L is next-open reference).

### Pattern Farm — the validation language

**What it is:** A frozen 20-pattern vocabulary (liquidity sweeps, failed breakouts, VWAP retests, …), a deterministic detector, append-only ledgers, and automated "farm studies" that measure each pattern's forward performance. First study: 694 instances, one session.

**Contributes:** the shared language the whole system will speak — every signal, everywhere, tagged with its pattern — and the statistical machinery to say which patterns earn their keep.

**Lacks:** integration. Patterns aren't wired into PM Edge scoring or Stage 2 signals yet, and per-signal live annotation is deferred.

### What's missing entirely

- **Risk / portfolio layer.** Position sizing exists per-trade (frozen: risk dollars ÷ risk per share), but nothing manages *the portfolio*: total heat, correlated positions, drawdown guards, kill switches.
- **A crash-safe execution core.** The journal work proved the concept; the full system needs it designed in from day one, not bolted on.
- **Live data path.** Genuinely-live bar-close evaluation (the standing requirement), still paper fills.
- **The closed learning loop.** Measurements flow into ledgers today; nothing flows back into discovery.

---

## 3. Target architecture

### 3.1 Component diagram

```
┌─────────────────────────────────────────────────────────────────┐
│ DATA INGESTION (versioned, timestamped, revision-archived)        │
│  Bars & quotes · News & social · Earnings calendar · Regime feeds │
└───────┬──────────────────┬────────────────────┬─────────────────┘
        │                  │                    │
        ▼                  ▼                    ▼
┌───────────────┐  ┌───────────────┐  ┌────────────────────┐
│ DISCOVERY     │  │ SIGNAL        │  │ RISK / PORTFOLIO   │
│ (PM Edge A)   │  │ (PM Edge B +  │  │                    │
│ Opportunity   │  │  Stage 2 +    │  │ Sizing · heat ·    │
│ scoring,      │  │  patterns)    │  │ correlation ·      │
│ regime,       │  │ ENTER / EXIT  │  │ drawdown guards ·  │
│ watchlist     │  │ at timestamps │  │ KILL SWITCHES      │
└───────┬───────┘  └───────┬───────┘  └─────────┬──────────┘
        │                  │                    │
        │   order intent   │   risk limits      │
        └────────┬─────────┴─────────┬──────────┘
                 ▼                   ▼
┌────────────────────────────────────────────────────────┐
│ CRASH-SAFE EXECUTION LAYER (§3.3)                        │
│  Write-ahead journal = SOLE source of truth (day one)   │
│  Single admission gate · persist-before-mutate           │
│  Bounded uncertainty · fail closed · stays small        │
│  Ledger interface: paper ledger now,                    │
│  brokerage adapter later — same interface, no rewrite   │
└────────────────────────┬───────────────────────────────┘
                         │ fills, outcomes (append-only)
                         ▼
┌────────────────────────────────────────────────────────┐
│ LEDGERS (frozen schemas, never rewritten)               │
│  Decisions · fills · outcomes · bar revisions · events  │
└────────────────────────┬───────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────┐
│ VALIDATION LOOP (Pattern Farm)                          │
│  Per-pattern farm studies · four-way failure attribution│
│  (selection / timing / stop / exit) · post-stop MFE     │
│  → promotion/demotion rules feed back into DISCOVERY    │
└────────────────────────────────────────────────────────┘
```

Data flows one way around the loop; learning flows back up. Every arrow is a **frozen interface** (Phase 1): a Discovery packet, a Signal packet (pattern-tagged, timestamped, with stop/target/size), an Order Intent (admitted or rejected by the risk layer with a reason code), a Fill record, an Outcome record. Components never reach into each other's internals.

### 3.2 What each layer owns (and doesn't)

- **Discovery** owns *what's worth attention*. It never places, sizes, or manages anything.
- **Signal** owns *when*: ENTER/EXIT timestamps, pattern tags, structural stops and targets. It never decides how much.
- **Risk / Portfolio** owns *how much and whether*: per-trade sizing from the frozen rule (risk dollars ÷ risk per share), portfolio heat caps, correlation blocks ("you already own this sector's risk"), drawdown-triggered stand-downs, and kill switches (manual + automatic). It can veto any order intent, and every veto is logged with a reason.
- **Execution** owns *faithfulness*: turning admitted intents into ledgered positions, exactly once, surviving crashes. It never has an opinion about direction.
- **Validation** owns *the truth about what worked*: per-pattern statistics, attribution, benchmark comparison. It never trades.

### 3.3 The crash-safe execution layer — designed from day one

Six rounds of adversarial review taught us how *not* to do this (retrofit crash-safety onto a polling script one probe at a time). The full system's execution layer is built on those lessons from the first line:

1. **Write-ahead journal as the sole source of truth.** Every state transition — signal observed, reservation made, fill committed, exit committed, cursor advanced — is appended to a durable journal *before* it takes effect anywhere else. Recovery is journal replay. There are no sidecars, cursors, or caches that can disagree with it, because there is nothing else.
2. **Single admission gate.** One function decides whether a new position may open. If anything is uncertain (an unconfirmed commit, a fence raised), the gate stays shut. No second path around it.
3. **Persist before mutate.** Nothing in memory changes until the journal entry for that change is durable. A crash at any point replays to the exact pre-crash state.
4. **Bounded uncertainty.** After any unconfirmed commit, the system assumes the conservative direction (capital is treated as committed, never as phantom free money) and blocks new admissions until the uncertainty is resolved against the ledger.
5. **Fail closed.** Corruption, unresolvable state, or a failed invariant → loud halt. Never a cheerful summary over broken books.
6. **Stay small.** The round-6 lesson: count the state-transition paths and keep the number tiny (one journal, one replay, one gate). Every new path is a new place to die; simplicity is a requirement, not a preference.
7. **Paper ledger now, brokerage adapter later.** The execution layer speaks to a *ledger interface*. Today the implementation is paper. When — and only when — Pranav authorizes live, a brokerage adapter implements the same interface. No rewrite, no parallel code path.

### 3.4 The risk / portfolio layer (new)

This is the largest missing piece, and the one that turns "a signal generator" into "a trading system":

- **Per-trade sizing** stays exactly as frozen: shares = max risk dollars ÷ risk per share, where the stop is structural (never tightened to manufacture size).
- **Portfolio heat:** cap on total open risk (e.g., sum of position risks ≤ X% of equity). New signals that breach heat are queued or rejected with a reason code — logged, auditable.
- **Correlation guard:** positions in the same sector/factor bucket share a sub-cap, so five "different" longs can't smuggle one giant sector bet.
- **Drawdown protocol:** tiered stand-downs (reduce size → no new entries → flatten) at defined drawdown thresholds. Defined in advance, not improvised.
- **Kill switches:** a manual halt Pranav can hit, plus automatic halts (data-feed staleness, execution-layer fence that won't clear, drawdown breach). A halted system closes or holds per its frozen halt policy — decided in Phase 1, not during a crisis.

### 3.5 The closed validation loop

Today Pattern Farm measures and humans read the report. The full system closes the loop mechanically:

- Every signal carries its **pattern tag** (from the frozen vocabulary).
- Every outcome flows into **per-pattern farm studies** automatically — win rate, expectancy, MFE/MAE capture, post-stop MFE.
- **Promotion/demotion rules** (frozen, versioned): patterns graduate into discovery scoring weight only after clearing sample thresholds; patterns that decay get down-weighted or retired. No human tweaks the weights on a hunch; the rules do it, and the rules themselves are versioned.
- The four-way failure attribution (selection / timing / stop / exit) routes improvements to the right layer: bad theses → Discovery, bad timestamps → Signal, bad captures → management rules.

---

## 4. Paper-to-live progression: what "proven" means

Borrowing Morrow's standing rules: **qualification ≠ profitability.** A profitable day is not proof of edge. The first milestone is always an *uncontaminated* observation, not making money. No quiet changes — anything meaningful bumps a version and starts a new cohort.

Proposed graduation ladder (numbers are proposals for Pranav to react to, not decrees):

| Stage | Name | What runs | Graduation criteria (proposal) |
|---|---|---|---|
| 0 | Research | Diagnostics, un-frozen experiments | Plumbing works; honest labels on everything |
| 1 | Qualified paper | Frozen rules, frozen interfaces | 100+ qualified trades, positive expectancy, benchmark-relative alpha (SPY/QQQ/sector), max drawdown within modeled bounds |
| 2 | Live-paper parallel | Live data feed, paper fills | 100+ more trades; live-paper results agree with research-paper within tolerance (catches feed/latency differences); no new failure modes |
| 3 | Live | Real brokerage adapter, real money | Explicit authorization from Pranav, small capital, kill switches armed, daily human review |

Sample-size guidance (from the v0.2 spec, unchanged): 10 trades = debugging, 25 = preliminary, 50 = useful signal, 100 = meaningful, 250 = interesting, 500+ = setup-family analysis. **Never declare victory after 12 trades.**

Two standing rules carry over untouched:

- **No strategy-rule change without a version bump.** A "small tweak" after three losses is how systems lie to themselves.
- **Shadow/control cohorts at every stage.** Rejected setups are tracked; if the filter doesn't beat the rejects, the filter is decorative.

"Proven," then, means: *a frozen version, qualified over hundreds of honest paper observations, reproduced on live data with paper fills, beating benchmarks net of modeled costs — with every number reproducible from append-only ledgers.* Anything less is research, labeled as such.

---

## 5. Phased build plan

### Phase 1 — Architecture + frozen interfaces
**Build:** This document, approved; then the interface contracts frozen in writing: Discovery packet → Signal packet → Order intent → Fill → Outcome → Validation feedback. Each contract lists its fields, its timestamp semantics (bar open vs close — the $0.02 lesson from Stage 2 applies everywhere), and its failure modes.
**Exit criteria:** Pranav approves the contracts. Nothing downstream starts until then.
**Why first:** every later phase is just implementing a contract. Arguments about fields happen on paper, not in code.

### Phase 2 — Crash-safe execution core (paper)
**Build:** A fresh, small execution layer implementing §3.3 — journal-first, single gate, paper ledger. Informed by all six repair rounds, but *not* a retrofit of the Stage 2 monitor (which stays as-is, diagnostics-only). Caliper reviews against a **bounded threat model** (open question #4 below) rather than unbounded kill -9.
**Exit criteria:** Caliper acceptance under the agreed threat model; 100+ simulated crash-recovery trials with zero accounting divergence.
**Why second:** everything else (risk, live data, learning) trusts this layer absolutely. It has to be right before anything leans on it.

### Phase 3 — Risk / portfolio layer (paper)
**Build:** §3.4 — heat caps, correlation guards, drawdown protocol, kill switches, per-trade sizing wired to portfolio state. Paper only.
**Exit criteria:** scripted stress scenarios pass (correlated crash, data outage mid-session, five simultaneous signals, drawdown breach) with the ledger proving the guards fired.
**Why third:** a signal engine without portfolio risk is a demo; this is what makes it a *system*.

### Phase 4 — Genuinely-live data path (paper fills)
**Build:** The standing "genuinely live" requirement: websocket/bar-close evaluation, live-push dashboard — data-feed-only. Signals evaluate on live bars; fills stay paper (reference basis, honestly labeled).
**Exit criteria:** live-paper parallel session agrees with research-paper within pre-defined tolerance for 20+ sessions.
**Why fourth:** live data changes signal timing; measuring that delta honestly is the whole point of the stage.

### Phase 5 — Closed validation loop
**Build:** §3.5 — pattern-tagged signals, automatic farm studies, frozen promotion/demotion rules feeding discovery weights.
**Exit criteria:** one full loop demonstrated end-to-end: a pattern measured → promoted → its weight changes discovery output → the change is attributed in the next study.
**Why fifth:** the system starts *learning* here. Everything before is plumbing; this is the compounding.

### Phase 6 — Live readiness (explicit authorization required)
**Build:** Brokerage adapter behind the frozen ledger interface, operational runbook, kill-switch drills, small-capital deployment.
**Exit criteria:** Pranav's explicit "go," in his own words, for a defined capital envelope. Paper parallel keeps running alongside as the control.
**Why last:** because it is. Nothing here moves without him.

---

## 6. Open questions for Pranav

1. **Live capital timeline:** is live trading a dated goal (e.g., sometime 2027), or "when the system proves itself, no date set"? (Sets the urgency of Phases 4–6.)
2. **Brokerage, when that day comes:** stay with Robinhood or move to a proper API broker (Alpaca/IBKR)? (Determines what the Phase 6 adapter looks like. No action now.)
3. **Which strategies graduate first:** the six intraday baselines or PM Edge swing? (Determines where Phases 2–3 focus their first paper cohorts.)
4. **The execution layer's acceptance bar:** bounded threat model (e.g., "survives crashes at poll boundaries, journal corruption fails loud") or unbounded Caliper kill -9 at every instruction? (This decides whether the hardening loop from October can ever end — and it needs your explicit answer, because it's your risk tolerance being encoded.)
5. **Genuinely-live data path:** approve the websocket + bar-close evaluation proposal (data-feed-only, fills stay paper)? (Unblocks Phase 4 design whenever we're ready.)

---

*End of DRAFT v0.1. Awaiting Pranav's review — especially the five questions above and the graduation numbers in §4, which are proposals, not decisions.*
