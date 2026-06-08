# ABC Score — Autonomous Build Brief

**Consumes:** `ABC_Score_SDD_v0.1.md` (concept + scope source of truth).
**For:** Claude Design (visual) → Claude Code (autonomous build).
**Deliverable:** one interactive, synthetic, single-page demo for the วช. funder show.

> This document is the contract. Both Design and Code consume the **same data schema (§3) and score output (§5)**. Do not change the contract without updating this file first.

---

## 0. GUARDRAILS — read before building (autonomous = no one is watching)

**DO:**
- Build a **single-page static React app**. Tailwind for styling. `recharts` allowed for any chart.
- Generate **synthetic data client-side**, deterministically (fixed seed) so the demo is reproducible.
- Show a persistent banner: **"ข้อมูลจำลองเพื่อสาธิต — SYNTHETIC DATA, ILLUSTRATIVE ONLY"**.
- Implement scoring **exactly** as §5. Implement reason codes (§6) — they are mandatory.
- Thai-primary UI; English secondary in parentheses where useful.

**DO NOT (these are the drift failures):**
- ❌ Do not invent or imply a trained/ML model. v1 is a transparent weighted-indicator heuristic. No "AI confidence," no "neural," no fake accuracy %.
- ❌ Do not make the synthetic data look like real people or real agencies. Use obviously fake names/IDs.
- ❌ Do not build a backend, database, login, file upload, or any network call. Everything in-memory.
- ❌ Do not drop explainability because it's hard. Reason codes are the point.
- ❌ Do not change weights, thresholds, or signals from this spec. They are illustrative and locked for the demo.
- ❌ Do not claim Pillar B/C use live AMLO/registry data. They are synthetic stand-ins.

**Acceptance gate:** if any DO-NOT is violated, the build is rejected. Checklist in §9.

---

## 1. What to build (one sentence)

An officer's-eye triage screen that scores ~18 synthetic officials with the ABC model, ranks them, lets the user move the pillar weights and escalation threshold live, and explains each score with reason codes.

---

## 2. Tech constraints

- React SPA, single file acceptable. Tailwind core utilities only. No router needed.
- State in React (`useState`/`useReducer`). **No localStorage, no backend, no network.**
- Charts (if used): `recharts`. Thai font: load **Sarabun** (Google Fonts) for all Thai text.

---

## 3. Data contract (TypeScript — Design and Code both target this)

```ts
type Tier = "RED" | "AMBER" | "GREEN";

interface Signal {
  key: string;          // e.g. "asset_income_ratio"
  labelTh: string;      // Thai label shown in UI
  labelEn: string;
  raw: number;          // synthetic raw value
  normalized: number;   // 0..1 after normalization
  pillar: "A" | "B" | "C";
  weightInPillar: number; // see §5, sums to 1.0 within each pillar
}

interface PillarScore {
  pillar: "A" | "B" | "C";
  subScore: number;     // 0..1
  signals: Signal[];
}

interface Official {
  id: string;           // fake, e.g. "OFF-0007"
  nameTh: string;       // obviously synthetic
  positionTh: string;   // synthetic role
  grade: number;        // 1..11 (civil-service-like)
  cohort: string;       // grouping key for normalization
}

interface ABCResult {
  official: Official;
  pillars: { A: PillarScore; B: PillarScore; C: PillarScore };
  composite: number;    // 0..100
  tier: Tier;
  reasonCodes: { labelTh: string; contribution: number }[]; // top 3, desc
  escalate: boolean;    // composite >= threshold
}
```

---

## 4. Synthetic data — generation rules

- 18 officials, deterministic seed. Names obviously fake (e.g. "นาย ก. ทดสอบ", "OFF-0001").
- Ensure a **spread across tiers** at default weights: ~3 Red, ~7 Amber, ~8 Green. Tune raw values to achieve this so the demo isn't flat.
- Raw values realistic in *shape* but fictional. Normalization = min–max within the full synthetic set (cohort-aware optional), clamped to [0,1].

---

## 5. Scoring algorithm (exact — locked, illustrative)

**Pillar weights (composite):** `wA=0.40, wB=0.35, wC=0.25` (user-adjustable via sliders; default here).

**Within-pillar signal weights (sum to 1.0 each):**

- **A — Application:** asset_income_ratio 0.30 · networth_jump 0.25 · concealable_share 0.20 · declaration_inconsistency 0.15 · position_opportunity 0.10
- **B — Behavioral:** cash_structuring 0.35 · txn_velocity 0.30 · debt_stress 0.20 · lifestyle_mismatch 0.15
- **C — Connection:** bidder_link 0.40 · nominee_holdings 0.25 · codirectorship 0.20 · coi_proximity 0.15

**Compute:**
```
pillarSub(p)  = Σ ( signal.normalized × signal.weightInPillar )      // 0..1
composite     = 100 × ( wA·subA + wB·subB + wC·subC )                // 0..100
tier          = composite >= 70 ? RED : composite >= 40 ? AMBER : GREEN
escalate      = composite >= escalationThreshold                     // default 60
reasonCodes   = top 3 signals by ( signal.normalized × signal.weightInPillar × pillarWeight ), desc
```

---

## 6. Screens / components

1. **Header** — title (Thai), synthetic-data banner, short subtitle "การจัดลำดับความเสี่ยง — สาธิต".
2. **Control panel** — three sliders wA/wB/wC (constrained to sum ≈1.0, show live values), one escalation-threshold slider. Changing any re-sorts the table live.
3. **Ranked queue (main table)** — columns: rank, id, name, position, A / B / C sub-scores (small bars), composite (0–100), tier chip (red/amber/green), escalate flag. Sorted by composite desc.
4. **Case drill-down** (click a row) — shows the official, the three pillar sub-scores, and the **reason codes** (top contributing signals with their contribution). This is the explainability proof.
5. **Escalation summary** — "เสนอให้ตรวจสอบยืนยัน N ราย" count updates with the threshold.

---

## 7. Interaction spec

- Sliders → recompute all scores → re-rank → re-render, instantly.
- Threshold slider → highlights/escalation-flags the set above it; updates the count.
- Row click → opens drill-down (panel or modal); reason codes always visible there.

---

## 8. Visual direction (handoff brief for Claude Design)

- Tone: **institutional, trustworthy, restrained.** This is a Thai government anti-corruption context — NOT a startup dashboard. No purple gradients, no flashy hero. Think calm authority.
- Thai-first typography (Sarabun), generous legibility, high contrast (accessible). Tier colors muted, not neon (deep red / amber / forest green).
- Layout: control panel + queue dominant; drill-down secondary. Clarity over decoration.

---

## 9. Acceptance checklist (ruthless QA gate — verify before showing the funder)

- [ ] Synthetic-data banner visible at all times.
- [ ] Scoring matches §5 (spot-check one official by hand).
- [ ] Reason codes show and are correct (top-3 by contribution).
- [ ] Weight sliders re-rank the table live.
- [ ] Threshold slider changes the escalation count live.
- [ ] No backend / no network / no login / no storage.
- [ ] No language anywhere claiming ML, accuracy %, or real data.
- [ ] Thai text renders correctly (Sarabun, no tofu boxes).
- [ ] Tier spread is visible (not all one color) at default weights.

---

*If Claude Code completes this and every box in §9 is checked, the demo is funder-ready. Anything beyond this scope (real data, backend, ML) is OUT and must be rejected.*
