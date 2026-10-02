---
status: DRAFT          # DRAFT | SPECIFIED | RUNNABLE-SAMPLE | RUNNABLE-LIVE
todos_open: 4
last_gate: null
attestation: null      # no named-human attestation; set only by a human who signed one
recipe_version: 0.1.1
---

<!-- Why DRAFT and not RUNNABLE-SAMPLE: a full sample run exists (its output is
     in scripts/contrib/2026fa/riyask29-data-analyst-sponsor-boston/out/; the
     run entry goes in logs/runs/2026fa-riyask29-1.md), but SNICKERDOODLE requires zero open
     typed TODOs before SPECIFIED, and the four below are real gaps, not
     placeholders. Moving up needs: close or descope all four, then a logged
     sample run with a named human clearing G1–G4. -->

# riyask29-data-analyst-sponsor-boston — Data-Analyst Sponsor Check

## Executive summary

**What it does.** It takes a short list of job leads and, for each one, checks
three things against the engine's own data: whether the company has sponsored a
work visa for a *data-analyst-type* job (not only for jobs like "Cyber Analyst"
that share the word), whether the posting is still open, and whether hiring
could finish before the student's work permission or allowed unemployment days
run out. It then passes each lead that survives to the engine's existing
scorer.

**Who it is for.** An international master's student in the Boston area on F-1
post-completion OPT who is looking for Data Analyst or Business Intelligence
Analyst work. "Data Analyst" has no occupation code of its own, so a plain "does
this company sponsor?" list tells this student much less than it seems to.

**What it decides.** For each lead, one next action: tailor an application,
network into the company first, skip it, or stop and let a person check
something the data cannot settle. It never decides that a company "does not
sponsor" just because the company is missing from a list.

Two customers: this file is for the agent;
`recipes/cases/2026fa/riyask29-data-analyst-sponsor-boston.card.md` is for the
human.

## Run record and handoff

**Handoff condition (done when):** both outputs exist in the out-dir; in
`analyst-check-log.json`, `counts.scored` plus the sum of `counts.stopped`
equals `counts.candidates`; no role stopped at G1 carries a `sponsorship` value;
and every role has a `next_action`. The following command checks all four and
exits non-zero on failure:

```bash
node -e "const l=require('./scripts/contrib/2026fa/riyask29-data-analyst-sponsor-boston/out/analyst-check-log.json');const s=Object.values(l.counts.stopped).reduce((a,b)=>a+b,0);const ok=l.counts.scored+s===l.counts.candidates&&l.roles.every(e=>e.stopped_at!=='G1'||e.sponsorship===null)&&l.roles.every(e=>e.next_action&&e.next_action.action);console.log(ok?'handoff ok':'handoff FAIL',JSON.stringify(l.counts));process.exit(ok?0:1)"
```

"Looks right" is not the condition. The count identity and the
no-value-on-stop invariant are.

## Required reads

1. `SNICKERDOODLE.md`: gates, provenance, TODO closure.
2. `DOMAIN.md` → *Known gaps and defects* (items 3 and 9 are role quality and local wage).
3. `course/2026fa/submissions/riyask29/CHANGE-BRIEF.md`: the predictions this recipe is checked against.
4. `scripts/contrib/2026fa/riyask29-data-analyst-sponsor-boston/README.md`: labels and failure behaviour.

Prefer those local files over any external lookup.

## Source inventory

Every path below exists on a fresh clone (checked 2026-10-02).

| Source | Path | Used for | Label |
|---|---|---|---|
| H-1B sponsorship by company | `data/80-days-to-stay/80-days-csv/mapped_student_employment_targets_v3.csv` | `company_name`, `Total Approvals`, `Total Denials`, `Approval_Rate`, `median_salary_offered`, `top_job_titles_sponsored` | record |
| BLS wage by SOC | `data/bls/compact/soc_occupation_compact.csv` | `onet_soc_code`, `annual_median_wage`, `oews_year` | record |
| SEC Form D (samples) | `data/sec/form-d/processed/sample/companies-sec-*.sample.json` | company name, `filing.quarter`, `filing.date_filed` | record |
| Liveness | `scripts/ats/liveness-browser.mjs` (the same logic as `npm run ats:liveness`) | active / expired / uncertain + reason | record (dated) |
| Scorer | `scripts/score/role-scorer.mjs` (`npm run score`) | composite, Apply / Consider / Skip, audit trace | — (combines labels) |
| Prototype | `scripts/contrib/2026fa/riyask29-data-analyst-sponsor-boston/analyst_check.mjs`, `lib.mjs` | gates, `roles.json`, outputs | — |
| Title rules + tier mapping | `scripts/contrib/2026fa/riyask29-data-analyst-sponsor-boston/config/title-families.json` | data-analyst-family / ambiguous / other | your-input |
| Persona (fictional) | `scripts/contrib/2026fa/riyask29-data-analyst-sponsor-boston/inputs/persona.example.json` | OPT end date, unemployment days, buffer, target SOC | your-input |
| Candidate leads | `scripts/contrib/2026fa/riyask29-data-analyst-sponsor-boston/inputs/candidates.json` | company, title, URL, hiring lag, self-rated fit | your-input |
| G2 clearance (a person's posting checks) | `scripts/contrib/2026fa/riyask29-data-analyst-sponsor-boston/inputs/liveness.gate-cleared.json` | machine liveness results plus browser checks by a named person, which override `uncertain` | record / your-input (per entry) |

Not used, on purpose: `npm run bls:local-wage` (fails on a fresh clone with no
`.venv` or `requirements.txt`, and feeds no decision), and
`scripts/sec/validate-h1b-join-sample.py` (needs full data that does not ship).

## Inputs

| Field | Where | Required | Label |
|---|---|---|---|
| `needs_sponsorship` (true/false) | persona | yes; the run refuses if missing | your-input |
| `visa.current_status`, `opt_end_date` (YYYY-MM-DD), `unemployment_days_used`, `unemployment_ceiling`, `buffer_target_days` | persona | yes; the run refuses if any is missing | your-input |
| `target_soc` (O*NET codes, e.g. `15-2051.01`, `13-1161.00`) | persona | yes | your-input |
| `company`, `title`, `url` (or `null` when no posting is known) | candidates | yes | your-input |
| `hiring_lag_weeks` | candidates | yes; a missing value stops the lead at G3 | your-input |
| `fit_self_rating` (0–1) | candidates | optional; without it the fit vote is absent | your-input |

## Phase gates

Each lead stops at the first failed gate. A stopped lead is **not** sent to the
scorer and carries no invented value. Liveness and timeline are gates
(multipliers), never votes.

| Gate | Testable condition | Pass | Fail → | Human must see |
|---|---|---|---|---|
| **G1 Company identity** | Exactly one CSV row whose normalised `company_name` (case, punctuation, Inc/LLC/Corp/Co/Ltd/LP/PBC stripped) equals the query, **and** that row has a non-empty `Total Approvals`. | tier + wage + Form D | `not-in-dataset`, `no-h1b-record`, `ambiguous`, or `not-exact-near-matches` → stop; `sponsorship: null`; near matches listed | the near-match names, to pick one or accept "unknown" |
| **G3 Timeline** | `hiring_lag_weeks` is present. Factor = `clamp((min(days_to_opt_end, ceiling − used) − lag_days) / buffer, 0, 1)`. | factor goes to the scorer as a multiplier | lag missing → stop (`missing-hiring-lag`); OPT already ended or past the cliff → factor 0 → scorer Skip | the dates and the arithmetic line |
| **G2 Liveness** | The lead has a URL and liveness is `active` or `expired`. | `active` → 1, `expired` → 0, both scored | no URL → `no-posting` (not scored; network or skip); `uncertain` / not checked → stop | the URL; a person opens the page |
| **G4 Title review** | Every Proven or Likely tier lists the sponsored title(s) and the keyword that matched. | report lists them under *Gates a person must clear* | — (this gate never clears itself) | each matched title, to confirm it is work the student would do |

G3 is evaluated before G2 so that every lead whose company passed G1 shows its
timeline arithmetic, including leads with no posting.

Gate tests, run against the log of a finished run (each exits non-zero on failure):

```bash
# G1: no stopped lead carries a sponsorship value; list why each stopped
node -e "const l=require('./scripts/contrib/2026fa/riyask29-data-analyst-sponsor-boston/out/analyst-check-log.json');const g=l.roles.filter(e=>e.stopped_at==='G1');const bad=g.filter(e=>e.sponsorship!==null);g.forEach(e=>console.log(e.role_id,e.company_match.status,e.company_match.near_matches.join('; ')));process.exit(bad.length?1:0)"
# G2: only active/expired postings reached the scorer
node -e "const l=require('./scripts/contrib/2026fa/riyask29-data-analyst-sponsor-boston/out/analyst-check-log.json');const bad=l.roles.filter(e=>e.scorer&&!['active','expired'].includes(e.liveness.result));console.log('scored with bad liveness:',bad.length);process.exit(bad.length?1:0)"
# G3: every scored lead has a numeric timeline factor; factor 0 means Skip
node -e "const l=require('./scripts/contrib/2026fa/riyask29-data-analyst-sponsor-boston/out/analyst-check-log.json');const s=l.roles.filter(e=>e.scorer);const bad=s.filter(e=>typeof e.timeline.factor!=='number'||(e.timeline.factor===0&&e.scorer.recommendation!=='Skip'));s.forEach(e=>console.log(e.role_id,e.timeline.factor,e.scorer.recommendation));process.exit(bad.length?1:0)"
# G4: every Proven/Likely lead exposes its matched titles for review
node -e "const l=require('./scripts/contrib/2026fa/riyask29-data-analyst-sponsor-boston/out/analyst-check-log.json');const s=l.roles.filter(e=>e.sponsorship&&['Proven','Likely'].includes(e.sponsorship.tier));const bad=s.filter(e=>!(e.g4_review||[]).length);s.forEach(e=>e.g4_review.forEach(t=>console.log(e.role_id,JSON.stringify(t.title),'->',t.family,'('+t.matched+')')));process.exit(bad.length?1:0)"
```

**Where gate decisions are recorded.** `logs/gate-decisions/` does not exist in
this repo, so the person clearing G1–G4 writes their decision, name and date in
the run entry `logs/runs/2026fa-riyask29-<n>.md` (template below). A gate the
test passes is still not *cleared* until that line exists.

## Workflow

Verbatim, from the repo root.

1. One-time setup, needed only for live liveness checks:

```bash
npm install && npx playwright install chromium
```

2. Offline test: no network, fixtures cut from the real data files.

```bash
node --test scripts/contrib/2026fa/riyask29-data-analyst-sponsor-boston/analyst_check.test.mjs
```

3. Sample run, live liveness. Network goes only to the URLs in `candidates.json`.

```bash
node scripts/contrib/2026fa/riyask29-data-analyst-sponsor-boston/analyst_check.mjs
```

4. **Clear G2, then re-run.** A person opens every `uncertain` posting and writes
the result into a liveness file. Copy `out/liveness.json` from step 3, and for
each checked URL set `"result"`, a `"reason"` saying what they saw, and
`"source": "your-input"`. Then re-run without network (this repo's cleared
file is `inputs/liveness.gate-cleared.json`):

```bash
node scripts/contrib/2026fa/riyask29-data-analyst-sponsor-boston/analyst_check.mjs \
  --liveness-file scripts/contrib/2026fa/riyask29-data-analyst-sponsor-boston/inputs/liveness.gate-cleared.json --today 2026-10-02
```

An entry without `"source": "your-input"` is treated as a machine record.

5. Run the handoff and gate tests above, then conformance:

```bash
node scripts/conformance.mjs scripts/contrib/2026fa/riyask29-data-analyst-sponsor-boston/ recipes/cases/2026fa/
```

6. A person reads `out/analyst-check-report.md`, clears G1–G4, and records the
decisions in `logs/runs/2026fa-riyask29-<n>.md`. Never edit `logs/RUN_LOG.md`.

## What it can and cannot verify

| Claim | Status | Why |
|---|---|---|
| Company X has *n* H-1B approvals and *m* denials in this dataset | **verified (record)** | read from the CSV row |
| Company X's top sponsored titles include "Data Analyst" | **verified (record)** | read from `top_job_titles_sponsored` |
| That title is data-analyst work | **inferred (your-input rule)** | keyword list in `title-families.json`; G4 makes a person confirm |
| Sponsorship p = 0.9 / 0.6 / 0.3 / 0.15 | **inferred (your-input mapping)** | Proven/Likely copy `data/examples/ch11-roles.json`; Possible/Unknown are mine. Labeled `your-input` in `roles.json` |
| The posting was open at time T | **verified (record, dated)** | Playwright result + timestamp. It can change minutes later: Abacus was `active` at 17:23Z and `uncertain` (15 s timeout) at 17:48Z on 2026-10-02 |
| A person saw the posting open | **your-input (a person's check)** | carried as `source: your-input` from the liveness file into `roles.json`, never relabeled as a record |
| Hiring can finish inside the runway | **inferred (your-input)** | the formula is mine; hiring lag is the student's estimate |
| BLS median for 15-2051.01 / 13-1161.00 | **verified (record)** | compact CSV, OEWS 2024, national |
| What this company pays an analyst | **not verifiable** | `median_salary_offered` covers *all* the company's sponsored roles; OEWS is an occupation survey, not an offer |
| The company sponsored a data analyst at all (SOC-level) | **not verifiable** | the CSV stores titles, not SOC codes, and only a truncated "top" list |
| The company raised money recently | **not verifiable on samples** | Form D ships 50 companies per quarter; `not-in-sample` means nothing either way |
| The company will sponsor *this* student | **not verifiable** | no data source answers this |
| STEM-extension eligibility, status rules | **not verifiable** | legal question; DSO only |

## Facts that bite: how this recipe handles each

| Fact | How it is handled |
|---|---|
| 1. `role_quality` weight is 0.0 `[VERIFY]` | The wage check is **not** sent to the scorer and the weight is not changed. It appears in the human report as a ratio with its caveat, so the person can use it outside the score. |
| 2. `bls:local-wage` feeds nothing | Not used. National OEWS medians only, labeled national. |
| 3. Only Form D samples ship | Joined anyway, with coverage printed on every lead ("200 of 58,329 companies across 4 sample files"). `not-in-sample` is never read as "no funding", and there is no funding vote (the scorer has no funding weight). |
| 4. `data/raw/`, `data/verified/`, `logs/gate-decisions/` do not exist | Gates point at `out/analyst-check-log.json`; decisions go in `logs/runs/`. |
| 5. The `snickerdoodle` CLI is roadmap | No command in this recipe uses it. |
| 7. `bls:local-wage` needs `.venv` | Not needed; Node only. |
| 8. `validate-h1b-join-sample.py` needs full data | Not used. |
| Found in this work: `role-scorer.mjs` treats any authorization text containing "authorized" as *no sponsorship needed* | The prototype compares the scorer's `profile_needs_sponsorship` with the persona's explicit `needs_sponsorship` and exits 2 if they differ. A test covers it. The scorer itself is not patched. |

## Proposed additions

1. [TODO: DATA SOURCE] DOL LCA disclosure files (by employer **and** SOC code) under
   `data/` with a provenance note, to replace title matching with SOC-level
   evidence for 15-2051 / 13-1161. This would turn G4 from "is the title right?"
   into a record. Belongs here because title matching is this recipe's weakest
   inference.
2. [TODO: DATA SOURCE] Full SEC Form D quarters (gitignored; fetched by the
   existing `scripts/sec/` tooling per `DATA.md`), so the funding check can say
   something. Until then it is reported as coverage only.
3. [TODO: DEV] Liveness that a person does not have to redo. On 2026-10-02 the
   EverQuote posting (`careers.everquote.com`) returned `uncertain` ("no visible
   apply control"), but a person found an "Apply for this job" *form* on the page;
   the checker looks for a button. Abacus returned `active`, then `uncertain`
   25 minutes later (page-load timeout). Options: resolve `gh_jid` links through
   the Greenhouse boards API (a new network host, so it needs a logged approval
   first), recognise application forms, and retry once on timeout.
4. [TODO: DEV] Per-company hiring-lag evidence. Today `hiring_lag_weeks` is the
   student's guess and the timeline gate is only as good as that guess.

## Output contract

One run writes these to `--out-dir` (default
`scripts/contrib/2026fa/riyask29-data-analyst-sponsor-boston/out/`), never to a
tracked repo file outside that folder.

**For the agent: `analyst-check-log.json`**

```
_prototype, _version,
run { today, prototype, liveness_mode, inputs{...paths}, outputs[] },
persona { name, fictional, binding_clock [opt-end-date | unemployment-days], runway_days, source },
counts { candidates, scored, stopped { G1, G2, G3, no-posting }, form_d_in_sample },
scorer_stdout,
roles[] {
  role_id, company_query, title, url, stopped_at [null | G1 | G2 | G3 | no-posting],
  company_match { status, matched_name, near_matches[], source },
  sponsorship  null | { total_approvals, total_denials, approval_rate, counts_source,
                        sponsored_titles[{title, family, matched}], titles_source,
                        classification_source, tier, p, p_source, caveat },
  wage_check[] { soc, status [ok | no-occupation-row | no-wage-value], bls_annual_median,
                 company_median_salary_offered, ratio_company_to_bls, caveat, source },
  funding      { status [in-sample | not-in-sample], filings[], coverage, meaning, source },
  timeline     { factor | null, status, today, opt_end_date, runway_days, hiring_lag_days,
                 buffer_days, binding_clock, arithmetic, source },
  liveness     { result, reason, checked_at, source },
  fit          { p, source } | null,
  g4_review[]  { title, family, matched },
  scorer       { recommendation, composite, reason, arithmetic, source },
  next_action  { action, hours, why }
},
cannot_verify[]
```

**For the person: `analyst-check-report.md`**, with sections in this order:
*Executive summary* (plain language, counts by next action, the deadline that
binds first) → *Run record* → *Leads and next actions* (one table row per lead)
→ *Evidence per lead* (every value with its label) → *Gates a person must
clear* (G1–G4, with the G4 title list) → *What this run could not verify*.

**Written by the existing scorer, unchanged:** `role-scores.json`,
`role-scores.md`. **Written by the prototype as scorer input:** `roles.json`,
`scorer-profile.json`. Live runs also write `liveness.json`.

## Next action per result (the 3-3-2 hand-off)

| Result | `next_action` | Goes to |
|---|---|---|
| Scorer **Apply** | `tailor-application` | the 2 apply hours |
| Scorer **Consider** | `review-then-tailor` (G4 first) | the 2 apply hours, after review |
| Posting expired, tier Proven/Likely | `network` | the 3 networking hours: informational chat before the next opening |
| No posting, tier Proven/Likely | `network` | the 3 networking hours |
| Scorer **Skip** for any other reason (timeline 0, weak tier) | `skip` | none; time goes back to the day |
| Stopped at G1 | `resolve-company` | a person, once; then re-run |
| Stopped at G2 (`uncertain`) | `check-posting-by-hand` | a person opens the URL |
| Stopped at G3 (no lag) | `fix-input` | a person adds the estimate |

## Stop conditions

Stop, write nothing as if valid, and exit non-zero when:

- a data path in the source inventory is missing;
- the persona lacks `needs_sponsorship` or any required `visa` field, or `opt_end_date` is not YYYY-MM-DD;
- the scorer's `profile_needs_sponsorship` disagrees with the persona;
- the scorer exits non-zero.

Stop the **lead** (not the run) at G1, G2 or G3 as in the gates table.

Refuse, in this recipe, to:

- auto-pick a near-match company name. Near matches are shown, never chosen;
- write `sponsorship.p = 0` for a company that is missing or has no H-1B fields. Absent from a list is not "never sponsored";
- feed the wage ratio into the score or change `role_quality`. That is an authorial decision for the scorer's maintainer (DOMAIN.md item 3);
- treat `not-in-sample` Form D as "no funding";
- weaken a gate so that a liked lead passes, or add an `override` without a written reason.

## Logging rules and run-log template

One file per run: `logs/runs/2026fa-riyask29-<n>.md`. Never edit
`logs/RUN_LOG.md`. No real names, emails or phone numbers. The persona is
fictional.

```markdown
## YYYY-MM-DD — riyask29-data-analyst-sponsor-boston sample run <n>

- **Recipe:** recipes/cases/2026fa/riyask29-data-analyst-sponsor-boston.md v0.1.0 (DRAFT)
- **Inputs:** persona <path> (fictional) · candidates <path> (<k> leads) · data: 80-days CSV, BLS compact, Form D samples · liveness <live | recorded file>
- **Commands:** <verbatim>
- **Outputs:** <out-dir>/analyst-check-log.json, analyst-check-report.md, roles.json, role-scores.{json,md}
- **Result:** <k> leads · scored <s> · stopped G1 <a> · G2 <b> · G3 <c> · no-posting <d> · scorer: Apply <x> · Consider <y> · Skip <z>
- **Handoff test:** <pass | fail + output>
- **Gate decisions (human):**
  - G1 — <lead>: <decision> — <name>, <date>
  - G2 — <lead>: <decision> — <name>, <date>
  - G3 — <lead>: <decision> — <name>, <date>
  - G4 — <lead>: <titles confirmed / rejected> — <name>, <date>
- **Open issues:** <what did not work, what is still missing>
```
