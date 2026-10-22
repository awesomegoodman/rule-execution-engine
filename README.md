# Rule Execution Engine

> An agentic AI system that retrieves regulatory rules, synthesises executable
> calculation logic, executes it safely in a sandbox, and returns verified
> results with full provenance.
>
> **Reference implementation:** Utility tariffs — 14,700+ real records from the
> OpenEI Utility Rate Database.

---

## Status

🚧 In active development. Nothing below is claimed as complete unless checked.

- [ ] Corpus ingestion + parsing (URDB, `source` URL staleness handling)
- [ ] Hand-written reference billing calculator (SWEPCO R-2 baseline)
- [ ] Sandbox execution environment
- [ ] Hybrid retrieval (BM25 + dense)
- [ ] Program synthesis (LLM-generated billing code)
- [ ] Static AST validation
- [ ] Planner + confidence gating
- [ ] Evaluation harness — **4 / 50 ground truth cases verified**
- [ ] Counterfactual analysis layer
- [ ] Deployment (FastAPI + Docker + LangSmith)
- [ ] Rule Change Impact Analysis (Project 2 — not started)

---

## Why This Exists

Large Language Models are good at reading regulations.

They are **not** good at performing deterministic calculations directly from
complex regulatory rules.

For domains like utility billing, tax, insurance, payroll, shipping, or
compliance, a wrong calculation is worse than no answer.

This project demonstrates a production-oriented approach:

- retrieve the correct regulatory rule
- generate executable calculation logic from it
- statically validate the generated program
- execute it safely inside a sandbox
- verify the result against deterministic invariants
- return an evidence-backed answer with full provenance

Instead of asking an LLM to *estimate* the answer, the model produces code that
performs the calculation. The result is either verified or the system says so
explicitly.

---

## Why This Isn't Just RAG

Traditional RAG looks like:

```
Documents → Embeddings → LLM → Answer
```

This project performs **reasoning through execution**:

```
Documents
    ↓
Hybrid Retrieval
    ↓
Program Synthesis
    ↓
Static Validation
    ↓
Sandbox Execution
    ↓
Deterministic Verification
    ↓
Grounded Answer
```

The hard engineering problems are no longer prompt engineering. They are
retrieval correctness, program synthesis, execution safety, deterministic
verification, and evaluation.

---

## What Makes This Hard

Three specific failure modes explain why "just ask the LLM" breaks for
regulatory calculations, and why each pipeline stage exists.

**Time-of-use schedule resolution.** URDB tariff records encode TOU periods as
a 12×24 matrix mapping month × hour-of-day to a rate period index, which then
maps to an entry in `energyratestructure`. Getting this assignment wrong for
even one timestamp produces a plausible but incorrect bill — the error is silent
because the output is still a valid number.

**Demand charge calculation.** Demand charges are assessed on the *peak demand*
(kW) observed over the tariff's defined demand interval, within the billing
period or within a TOU period of it — not on total or average consumption. Some
records also carry ratchet / lookback clauses that tie today's charge to past
peaks. A method that sums or averages usage instead of tracking the peak is
wrong for every record that carries a demand charge: 13,403 of 14,707 (91.1%) in
this corpus.

**Stateful tier accumulation.** Tiered energy charges require tracking
cumulative consumption across the billing period to know which tier applies at
each interval. This is inherently stateful — each interval's rate depends on
how much energy has already been consumed before it. A single-pass LLM
calculation cannot do this correctly.

Generated code executes in a sandbox where these operations can be implemented
correctly, verified against known outputs, and re-run on any usage profile. The
LLM's role is synthesis, not arithmetic.

---

## Architecture

```
                    User Query
                         │
                         ▼
                    Planner Agent
                         │
                         ▼
          Hybrid Retrieval (BM25 + Dense)
                         │
                         ▼
             Retrieval Confidence Gate
          (clarify if confidence is low)
                         │
                         ▼
        Program Synthesis (Python Generator)
                         │
                         ▼
              Static AST Validation
                         │
                         ▼
          Sandboxed Code Execution
                         │
                         ▼
        Deterministic Verification Layer
                         │
                         ▼
        Verified Result + Provenance + Confidence
```

---

## Reference Domain

The reference implementation uses the **OpenEI Utility Rate Database (URDB)**.

Utility tariffs were chosen because they contain unusually complex rule
structures: nested tiered pricing, time-of-use schedules, demand charges,
seasonal rules, eligibility thresholds, and distributed generation credits.
This makes them an excellent stress test for a rule execution system.

The core pattern — retrieve a rule document, synthesise deterministic logic from
it, execute safely, verify the result — applies wherever a regulation governs a
calculation and a wrong answer has real consequences. The architecture is
intentionally domain-independent and can be adapted to:

- tax regulations
- insurance pricing
- payroll rules
- shipping tariffs
- compliance policies
- financial products

---

## Dataset

**Source:** OpenEI Utility Rate Database — https://openei.org

**Corpus composition (residential records):**

| Field | Count | Coverage |
|---|---|---|
| Active residential records | 14,707 | — |
| Records with TOU schedule matrix | 14,655 | 99.6% |
| Records with demand charges | 13,403 | 91.1% |
| Records with multi-period tiered structure | 7,488 | 50.9% |

The 7,488 multi-period records are the primary engineering targets. They require
the full parsing pipeline: schedule matrix lookup, rate structure array
traversal, tier threshold accumulation, and demand charge tracking. The 7,219
single-period records are used for unit test baselines.

**Known data quality issue:** the source URL on a URDB record (field `source`,
labelled "Source or Reference" in the OpenEI UI) can be stale even when the
record is marked active — confirmed during dataset verification (see
`docs/build-log.md`). The ingestion pipeline validates every source URL, detects
404s and redirects, and flags stale records for manual review rather than
treating the URL as authoritative.

---

## Agent Pipeline

### 1. Planning

The planner parses the query to determine customer class, tariff type, required
inputs, and any missing information. If the query is underspecified, the system
requests clarification instead of proceeding with assumptions.

### 2. Retrieval

Hybrid retrieval combines BM25, dense embeddings, and structured metadata
filtering to identify the most appropriate tariff record. Metadata filters
(utility name, customer sector, state, tariff type) are applied before
semantic search to constrain the candidate set.

### 3. Confidence Gate

A billing calculation should never proceed from an uncertain retrieval. The
engine evaluates retrieval confidence score, metadata consistency, and query
ambiguity. Below a configured threshold, execution stops and clarification is
requested. A low-confidence result returned with false certainty is a harder
failure to catch than an explicit refusal.

### 4. Program Synthesis

The LLM converts the retrieved tariff's structured fields — `energyratestructure`,
`energyweekdayschedule` (and the weekend schedule), `demandratestructure`,
`fixedchargefirstmeter` — into an executable Python billing calculator. The
synthesised code handles tier accumulation, schedule matrix resolution, seasonal
switching, and demand charge tracking as explicit, testable logic rather than
model inference.

### 5. Static AST Validation

Generated programs are parsed and inspected before execution. Code is rejected
if it contains filesystem access, network calls, dynamic imports, subprocess
creation, `exec`/`eval`, or other unsafe patterns. The program must conform to
an expected structural shape: a single callable that takes usage data and
returns a billing breakdown.

### 6. Sandboxed Execution

Validated code executes inside an isolated environment with CPU limits, memory
limits, an execution timeout, a read-only filesystem, and no network access.
The sandbox treats the generated code as untrusted regardless of what the AST
validator passed.

### 7. Deterministic Verification

Results are checked against two layers of invariants.

*Structural invariants* — enforced on every result: no component charge is
negative, tier consumption values sum to total usage, TOU period assignments
account for every interval in the billing period, demand charge is consistent
with the peak demand interval.

*Ground truth invariants* — for known benchmark cases, the computed result is
checked against the hand-verified expected output. Any deviation fails
verification and is surfaced explicitly rather than returned as a valid result.

**Scope of verification.** Ground truth exists only for benchmark cases. For any
other tariff, only the structural invariants run, and those cannot detect a
plausible-but-wrong rate or TOU assignment. Results for non-benchmark tariffs are
therefore labelled as *structurally verified only* in the provenance record. A
planned mitigation is differential testing of synthesised code against the
hand-written reference calculator.

**Numeric precision.** Money arithmetic uses `decimal.Decimal` with explicit
round-half-up at the end, not binary floats. This matters: the 1,000 kWh July
case in Case 1 below is exactly $133.055 before rounding. The rounding rule
printed in each filing should be confirmed per tariff.

The provenance record includes which checks ran and their outcomes.

---

## Evaluation

Each subsystem is evaluated independently before end-to-end evaluation runs.

| Component | Metric |
|---|---|
| Retrieval | Recall@k, MRR |
| Tariff selection accuracy | Exact match on utility + schedule |
| Program synthesis | Execution success rate |
| Static validation | Unsafe code detection rate |
| Billing accuracy | Mean absolute error, exact match rate |
| Change detection (Project 2) | Precision, Recall, F1 |
| End-to-end | Task completion rate |
| Performance | p50 / p95 latency |

Target: 50+ verified ground truth cases before the harness is considered
complete. Current: **4 verified** (see below).

---

## Ground Truth

Ground truth calculations are sourced from publicly filed utility commission
documents and are independently re-verifiable against the source PDFs. No
ground truth is self-generated.

"Base bill" below means the schedule's own customer and energy (and, for
Schedule 8, demand) charges as printed on the cited page. Riders and adjustment
clauses are out of scope for these cases.

### Benchmark cases

**Case 1 — SWEPCO Arkansas Schedule R-2 (tiered seasonal) — 3 cases verified**

Source: APSC Docket 25-003-U, Order 18, approved 2026-02-04.
URL: `https://www.swepco.com/lib/docs/ratesandtariffs/Arkansas/Arkansas_Compliance_Tariff_07-09-26.pdf`

Rate: Customer charge $17.97/month. Summer (May–Sep): $0.115085/kWh first 1,500
kWh, $0.137144/kWh additional. Winter (Oct–Apr): $0.092319/kWh flat.

| Usage | Month | Expected base bill |
|---|---|---|
| 1,000 kWh | July | $133.06 |
| 2,000 kWh | August | $259.17 |
| 1,200 kWh | January | $128.75 |

Arithmetic: $17.97 + (1,500 × $0.115085) + (500 × $0.137144) = $259.17 for the
2,000 kWh case. All three cases are independently verifiable from Schedule R-2,
page R-2.1 of the source PDF.

---

**Case 2 — SWEPCO Arkansas Schedule 8 (TOU + separate demand) — 1 case verified**

Source: Same filing, Schedule 8, pages R-8.1–R-8.4.

Rate: On-peak period 1:01 PM–7:00 PM weekdays, July/August/September. Secondary
voltage rates: on-peak energy $0.160041/kWh, off-peak energy $0.025710/kWh,
on-peak demand $33.39/kW, off-peak demand $9.29/kW.

Usage profile: 500 kWh on-peak, 1,200 kWh off-peak, 120 kW on-peak demand, 60
kW off-peak demand, August.

```
On-peak energy:    500 × $0.160041  =   $80.02
Off-peak energy: 1,200 × $0.025710  =   $30.85
On-peak demand:    120 × $33.39     = $4,006.80
Off-peak demand:   60  × $9.29      =  $557.40
─────────────────────────────────────────────────
Base total:                           $4,675.07
```

> **Note:** Schedule 8 includes a minimum on-peak demand charge of $1,669.50
> (50 kW × $33.39). The test case uses 120 kW, which exceeds this minimum, so
> the per-kW rate applies. Residential TOU records in the URDB do not have a
> demand minimum clause; the minimum-charge branch is absent from those
> calculators. The interval-level TOU assignment logic, demand peak tracking,
> and per-kW charge calculation are otherwise identical.

---

**Case 3 — ComEd RTOUPP (three-period split TOU) — structure confirmed, values pending, not yet counted**

Source: ILL. C.C. No. 10, fetched live 2026.

Period definitions:

| Period | Hours (CPT) |
|---|---|
| Off-Peak | 10:00 PM – 6:00 AM |
| Peak | 6:00 AM – 2:00 PM and 7:00 PM – 10:00 PM |
| Super Peak | 2:00 PM – 7:00 PM |

This is a split Peak period (two non-contiguous windows per day) with a Super
Peak embedded within the afternoon block — the most complex schedule matrix case
in the residential corpus. Structure and period definitions verified. Exact
¢/kWh values pending one fetch of Informational Sheet No. 57 of the same
filing.

---

### Sourcing standard

> Ground truth calculations come from publicly available utility commission filings, independently re-verifiable against the source documents. Ground truth is not self-generated. Exact ¢/kWh values are extracted from regulatory authority filings, not from the URDB record itself, to ensure the calculator is validated against the authoritative source rather than the dataset it is trained on.

---

## Example Query (Target Behaviour)

> What would a residential customer using 2,000 kWh in August pay under SWEPCO Schedule R-2?

Once complete, the engine will:

1. retrieve the SWEPCO Arkansas tariff record
2. synthesise a billing calculator from its structured fields
3. validate the generated code against AST safety rules
4. execute it inside the sandbox
5. verify the result against the Schedule R-2 ground truth invariant
6. return $259.17 with citations to the APSC filing

*(This exact case is already hand-verified — $259.17 — see Ground Truth above.
The synthesis pipeline that would produce it automatically is not yet built.)*

---

## Counterfactual Analysis (Planned)

The same execution engine supports optimisation scenarios by running the billing
calculator against alternative usage profiles:

- What if 40% of load shifted off-peak?
- Which available tariff is cheapest for this usage pattern?
- How much would demand reduction save?
- What would next year's proposed tariff cost against current usage?

These are re-runs of the same code path with modified inputs — not separate
LLM calls.

---

## Rule Change Impact Analysis (Project 2 — Planned)

A second capability will compare successive tariff revisions once Project 1 is
fully shipped.

```
Old Tariff → Canonical Rule Graph ─┐
                                   ├─→ Graph Diff → Change Classification
New Tariff → Canonical Rule Graph ─┘
                                          │
                                          ▼
                                  Run Billing Engine
                                          │
                                          ▼
                                 Impact Quantification
                                          │
                                          ▼
                                    Cited Summary
```

Questions this will support:

- What changed between this tariff and the prior version?
- Which customer classes are affected?
- What is the bill impact across the usage distribution?
- Which specific rule clause caused the increase?

---

## Technology Stack

| Layer | Technology |
|---|---|
| Language | Python |
| LLM orchestration | Plain Python (LangGraph optional) |
| Models — local | Ollama (free, offline) |
| Models — hosted | Gemini and Groq free tiers (limits change; check provider docs) |
| Retrieval | pgvector + BM25 |
| API | FastAPI |
| Containerisation | Docker |
| Experiment tracking | Weights & Biases (free tier) |
| Tracing | LangSmith |
| Testing | Pytest |

No paid model APIs are required to run this project.

---

## Safety Design

Generated code is never trusted. Planned controls enforced before and during
execution:

- AST validation rejects unsafe patterns before any code runs
- Sandbox provides CPU limits, memory limits, execution timeout
- Read-only filesystem — no writes permitted
- No network access from inside the sandbox
- Deterministic verification checks output before it is returned

*(Design target — enforced once Sandbox and Static Validation stages are
complete; see Status.)*

---

## Project Structure

```
rule-execution-engine/

├── app/
│   ├── main.py           # FastAPI entrypoint
│   ├── planner/          # Query parsing, input extraction
│   ├── retrieval/        # BM25 + dense hybrid retrieval
│   ├── synthesis/        # LLM-based code generation
│   ├── validation/       # AST safety validation
│   ├── sandbox/          # Isolated execution environment
│   ├── verification/     # Deterministic result checking
│   ├── api/              # FastAPI endpoints
│   └── evaluation/       # Harness, metrics, ground truth runner
│
├── data/
│   ├── urdb/             # URDB bulk download + ingestion scripts
│   └── ground_truth/     # Verified benchmark cases
│
├── tests/
│   ├── unit/
│   └── integration/
│
├── notebooks/            # Exploration and analysis
│
├── docs/
│   └── build-log.md      # Engineering decisions and verification trail
│
├── docker/
├── docker-compose.yml
│
└── readme.md
```

---

## Running Locally

*(Will be accurate once deployment stage is complete — see Status.)*

```bash
git clone https://github.com/[username]/rule-execution-engine
cd rule-execution-engine
docker compose up
python data/urdb/load_urdb.py
uvicorn app.main:app --reload
```

---

> Built an agentic Rule Execution Engine that retrieves regulatory rules, synthesises executable calculation logic, executes it safely inside a sandbox, and verifies results against deterministic invariants — with full provenance at every step. Utility tariffs are the reference implementation across 14,700+ real records from the OpenEI URDB. The architecture is domain-independent: the same pipeline applies to tax rules, insurance pricing, payroll, and any domain where a regulation governs a calculation and a wrong answer has consequences.
