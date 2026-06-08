# ABC Score — System Design Document (SDD)

**Version:** 0.2 (incorporates prototype build report)
**Owner:** KPI research team — สถาบันพระปกเกล้า
**Funder:** วช. (NRCT), FY2568 · **PI:** รศ.ดร.อิสระ เสรีวัฒนวุฒิ
**End-user:** สำนักงาน ป.ป.ช. (NACC)
**Status of this doc:** source of truth. Deck, prototype, and any future build derive from here.

### Changelog v0.1 → v0.2
- Added **operator vs calibrator** role distinction (§6).
- Pillar A signals now specified as **derived features** with formulas (§5.1).
- Added **canonical missing-pillar renormalization rule** (§5.2).
- Extended **R1**: within-pillar signal weights are also an open calibration parameter.
- Added **§9.1 hard narration guardrail** for the simulated API-connect demo feature.

> **Read this first — scope guardrail.** This system is a **screening / triage engine**, not a corruption detector and not an investigation tool. It re-orders NACC's review queue and explains why. Deep investigation (ตรวจสอบเชิงลึก) is **out of scope**. AMLO (ปปง.) is a **candidate data source**, not the end-user.

---

## 1. Purpose & objectives

Prioritise which government officials' asset declarations should escalate from routine review (ตรวจสอบปกติ) to verification (ตรวจสอบยืนยัน), so NACC's limited review capacity is spent on the highest-risk cases first.

**Primary objective:** produce, per declaration, a 0–100 risk score + a ranked queue + machine-readable reason codes.
**This is NOT:** a tool that determines guilt, replaces human judgment, or conducts investigation. Every output is a recommendation to a human.

---

## 2. Scope

| In scope (MVP) | Out of scope (MVP) |
|---|---|
| Screening / triage scoring of declarations | Deep investigation (ตรวจสอบเชิงลึก) |
| Ranked queue + risk tiers (Red/Amber/Green) | Litigation / asset tracing / evidence management (AMLO functions, **not this project**) |
| Per-case reason codes (explainability) | Automated decisions without a human in the loop |
| Pillar A on data NACC controls (ACAS) | Production pipelines, real PII, live integrations |
| Pillars B & C **specified** (build phased) | Trained/supervised ML (blocked on labels — §8) |

---

## 3. Concept — the ABC framework

| Pillar | Name | Fraud Triangle | Question |
|---|---|---|---|
| **A** | Application | Opportunity | Does the declaration itself look anomalous? |
| **B** | Behavioral | Pressure | Do financial patterns signal pressure/anomaly? |
| **C** | Connection | Rationalization | Do relationships create conflict/risk? |

**Composite:** `ABC = wA·A + wB·B + wC·C`, scaled 0–100.
**Working weights (PROVISIONAL — pending expert panel):** wA = 0.40, wB = 0.35, wC = 0.25.

---

## 4. Architecture (logical)

```
 ACAS declarations (A)  ─┐
 AMLO / banking (B)*    ─┼─▶ normalize ─▶ feature ─▶ pillar ─▶ weighted ─▶ ranked queue
 e-GP / registry (C)*   ─┘  + validate    engineer    scorers   combiner    + reason codes
                                                                                  │
                                                                                  ▼
                                                              NACC officer dashboard (human-in-loop)
                                                                                  │
                                                                                  ▼
                                                              escalate top N% → verification
```
`*` B and C pending data-sharing agreements (§7). Pillars degrade gracefully (§5.2) — A alone is usable.

---

## 5. Scoring specification

**Composite:** `ABC = (0.40·A + 0.35·B + 0.25·C) × 100`. Normalization = min–max within cohort, clamped [0,1]. Pillar sub-score = Σ(normalized × weightInPillar), capped 1.0. Tiers: Red ≥ 70 · Amber 40–69 · Green < 40 (illustrative — to be calibrated). Reason codes = top-3 signals by (normalized × weightInPillar × pillarWeight).

### 5.1 Signals

**A — Application (ACAS — NACC-controlled). Specified as DERIVED features from declaration intake, not directly-entered indices:**
- `asset_income_ratio` = total assets ÷ (annual income × years in service)
- `networth_jump` = (current net worth − prior-filing net worth), relative to income
- `concealable_share` = concealable assets (cash/gold/foreign) ÷ total assets
- `declaration_inconsistency` = derived from issues/missing fields found at intake
- `position_opportunity` = procurement/budget/permit authority of the role
- Within-pillar weights (illustrative): 0.30 · 0.25 · 0.20 · 0.15 · 0.10

**B — Behavioral (AMLO/banking — PENDING MOU):** cash structuring · txn velocity · debt stress · lifestyle mismatch (0.35 · 0.30 · 0.20 · 0.15)

**C — Connection (e-GP / registry / ACT Ai overlap — PARTIAL):** bidder link · nominee holdings · co-directorship · COI proximity (0.40 · 0.25 · 0.20 · 0.15)

### 5.2 Canonical missing-pillar rule (graceful degradation)
If a pillar's data is unavailable, score on the **present** pillars only, with their weights **renormalized to sum 1.0** over what's present; flag the result **partial** and render missing pillars as "รอ API / pending." With all three present, math is identical to the full formula above.

> **Explainability is a hard requirement.** Every score returns ranked reason codes. A black box is legally indefensible and will be rejected by reviewers.

---

## 6. Roles, governance, privacy & fairness

- **Operator (triage/intake)** sees the daily SOP screen — queue, escalation count, drill-down. **Calibrator (expert panel/admin)** uses the scoring-calibration drawer (pillar + within-pillar weights, thresholds). Calibration is **separated** from the officer's daily screen.
- **Human-in-the-loop:** tool recommends, officer decides.
- **PDPA:** declarations + financial data are sensitive personal data. Real data needs lawful basis, access controls, retention limits, audit logging. **Prototype uses synthetic data only.**
- **Fairness:** position-based signals may over-flag certain agencies → cohort normalization + bias review before any real-data use.

---

## 7. Data sources & dependency status

| Source | Pillar | Status | Risk if delayed |
|---|---|---|---|
| ACAS declarations | A | NACC-controlled | Low — A ships first |
| AMLO / banking | B | Agreement **not secured** | **High** — B blocked until MOU |
| e-GP / registry | C | Public-ish; ACT Ai overlap | Medium — partial via public data |

**Sequence the build A → C → B by data availability, not by weight.** Do not promise B as imminent.

---

## 8. Phasing — why v1 is not ML

| Phase | Model | Precondition |
|---|---|---|
| **v1 (now)** | Expert-weighted indicator model | None |
| **v2 (future)** | Supervised ML | Requires labeled confirmed-case outcomes — **do not exist yet** |

No labels → v1 makes **no predictive-accuracy claim**. This is the correct MVP. Earning labels from verification outcomes is itself a funded milestone.

---

## 9. The demo — scope

Interactive synthetic prototype proving engine + explainability end-to-end. Banner everywhere: **SYNTHETIC DATA — ILLUSTRATIVE**. Built features: derived Pillar A intake; live weight + threshold calibration drawer; ranked queue; case drill-down with reason codes; partial-pillar handling; plain-language Thai descriptions per factor.

### 9.1 HARD GUARDRAIL — the simulated "API connect" feature
The prototype simulates connecting AMLO (B) and registry (C) sources, after which the preview score climbs. This is a **labelled simulation — NO real network, NO real integration, NO existing data link.**

- **It must be narrated as:** "simulated — this demonstrates the value the *pending* data-sharing agreement would unlock."
- **It must NEVER be narrated as** an existing or imminent AMLO/registry integration.
- This is the project's known recurring failure (overstating AMLO). The on-screen label is not enough — the **spoken framing is the control.** Anyone demoing this must say "simulated" out loud at the moment of the click.

---

## 10. Risks & open decisions

| # | Item | State |
|---|---|---|
| R1 | Pillar weights **and within-pillar signal weights** | Provisional, pending expert panel |
| R2 | AMLO data-sharing (B) | Not secured |
| R3 | No labeled outcomes (blocks ML) | Open |
| R4 | Tier thresholds + escalation N% | Not calibrated |
| R5 | Fairness / cohort normalization | Designed, not tested |
| R6 | PDPA basis for real data | Not in place |

---

## 11. Roadmap

- **M1 (now):** SDD locked · synthetic prototype · funder deck.
- **M2:** panel calibrates weights + thresholds; ACAS access scoped.
- **M3:** Pillar A on de-identified ACAS data; fairness review.
- **M4:** AMLO MOU → Pillar B; e-GP/ACT Ai → Pillar C.
- **M5:** verification outcomes as labels → v2 supervised pilot.

---

*End v0.2. Update this doc before changing anything downstream.*
