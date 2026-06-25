# Fix: synthesize worker drops the required `findings` array

## Context

The `/rnd:ae-research` deep-research workflow (`research.js`) ends with a **Synthesize**
step: one agent is asked to turn the verified claims into a structured report matching
`REPORT_SCHEMA` (`summary` + `findings[]` + `caveats`). On the `behavior driven design`
run (workflow `wf_91c92b36-ccb`) this step misbehaved, observed directly in the agent
transcripts:

- The synthesize agent dumped the **entire report narrative into the `summary` string**
  (13,808 / 16,180 / 15,694 / 10,883 chars across attempts) and **never emitted the
  required `findings` array**.
- The schema validator rejected every attempt verbatim:
  `Output does not match required schema: root: must have required property 'findings'`.
- After 5 internal `StructuredOutput` retries the agent panicked into
  `{"summary":"test","caveats":"test caveats"}`, then `agent({schema})` threw. `withRetry`
  spun up the next synthesize agent and the cycle repeated across all 3 `MAX_ATTEMPTS`
  (agents `a834…`, `aaf2…`, `a3a5…`) — ~10 wasted agent calls.
- This run **recovered by luck** (attempt 3 happened to include `findings`). One worse
  roll and `withRetry` exhausts, hitting the salvage path at `research.js:440`, which
  currently returns `findings: []` → a near-empty report despite real verified claims.

**Two root causes, two fixes (defense in depth):**

1. **Prevention** — `REPORT_SCHEMA.summary` has no length bound and the prompt invites the
   model to treat `summary` as the whole report. Add a `maxLength` safety net + an explicit
   "summary is a brief intro; all detail goes in `findings`" output contract, so the model
   is constrained and the validator returns an *actionable* second error instead of only
   "missing findings."
2. **Containment** — when synthesis still fails, build `findings` from the already-confirmed
   claims instead of returning `[]`, so a synthesis failure degrades to "verified-but-unmerged
   findings," never an empty report. Extract this into a unit-tested pure-JS helper (per the
   repo's existing block-extraction test pattern + the user's TDD directive).

Measured calibration: the successful run's legitimate summary was **1,280 chars**; the failing
dumps were **10,883–16,180 chars**. A hard `maxLength: 2500` cleanly separates the two and
never threatens a real 3–5 sentence summary.

## Critical files (all under the rnd plugin SOURCE, not the cache)

Plugin source root: `/Users/russelltsherman/.claude/plugins/marketplaces/ae-skills/plugins/rnd/`

- `workflows/research.js` — 4 edits (schema, prompt, new testable block, salvage path)
- `workflows/__tests__/research.salvage.test.mjs` — **new** test file
- `.claude-plugin/plugin.json` — version bump `0.1.0` → `0.1.1`

> ⚠️ Do **not** edit the cache copy at `…/cache/ae-skills/rnd/0.1.0/`. The runtime uses the
> version-keyed cache; edits land there only via reinstall (see Propagation). Editing the
> cache without a version bump is the known [[plugin-cache-staleness]] trap.

---

## Edit 1 — `REPORT_SCHEMA`: bound `summary` length

File: `workflows/research.js` (the `REPORT_SCHEMA` block, ~line 193).
`summary: { type: "string" },` also appears in `SCOPE_SCHEMA`, so match with the surrounding
`REPORT_SCHEMA` context to stay unambiguous.

**old_string:**
```js
const REPORT_SCHEMA = {
  type: "object", required: ["summary", "findings", "caveats"],
  properties: {
    summary: { type: "string" },
```
**new_string:**
```js
const REPORT_SCHEMA = {
  type: "object", required: ["summary", "findings", "caveats"],
  properties: {
    summary: { type: "string", maxLength: 2500 },
```

---

## Edit 2 — Synthesize prompt: tighten summary + add explicit output contract

File: `workflows/research.js` (~lines 430–436). This `## Instructions` block is unique.

**old_string:**
```js
  "## Instructions\n" +
  "1. Identify claims that say the same thing — merge them, combine their sources.\n" +
  "2. Group related claims into coherent findings. Each finding should directly address the research question.\n" +
  "3. Assign confidence per finding: high (multiple primary sources, unanimous votes), medium (secondary sources or split votes), low (single source or blog-quality).\n" +
  "4. Write a 3-5 sentence executive summary answering the research question.\n" +
  "5. Note caveats: what's uncertain, what sources were weak, what time-sensitivity applies.\n" +
  "6. List 2-4 open questions that emerged but weren't answered.\n\nStructured output only.",
```
**new_string:**
```js
  "## Instructions\n" +
  "1. Identify claims that say the same thing — merge them, combine their sources.\n" +
  "2. Group related claims into coherent findings. Each finding should directly address the research question.\n" +
  "3. Assign confidence per finding: high (multiple primary sources, unanimous votes), medium (secondary sources or split votes), low (single source or blog-quality).\n" +
  "4. Write a SHORT executive summary: 3-5 sentences, roughly 1200 characters and NEVER more than 2500. Do NOT put claim-level detail, quotes, evidence, or source lists in the summary — every claim belongs in `findings`, not in `summary`.\n" +
  "5. Note caveats: what's uncertain, what sources were weak, what time-sensitivity applies.\n" +
  "6. List 2-4 open questions that emerged but weren't answered.\n\n" +
  "## Output contract (read carefully)\n" +
  "Return ALL of these top-level fields: `summary` (string), `findings` (array), `caveats` (string), `openQuestions` (array). `findings` is REQUIRED and is the core payload: it MUST be a non-empty array with ONE object per finding, each having `claim`, `confidence` (high|medium|low), `sources` (array of URL strings), and `evidence`. NEVER omit `findings`. NEVER fold the report body into `summary` — `summary` is a brief intro only.\n\nStructured output only.",
```

---

## Edit 3 — Add the `salvageFindings` testable block (and de-duplicate `confRank`)

### 3a. Insert a new marker-delimited block after the fetch-label block (~line 137)

The block must define `confRank` (reused by the Synthesize block below — see 3b) and
`salvageFindings`. It is pure JS (no workflow globals), so the existing block-extraction
test trick applies.

**old_string:**
```js
// --- end fetch label ----------------------------------------------------------

// ─── Schemas ───
```
**new_string:**
```js
// --- end fetch label ----------------------------------------------------------

// --- salvage findings (testable) ----------------------------------------------
// When the synthesize agent exhausts its retries (returns null / throws), the run
// must NOT return an empty report — the Verify phase already confirmed real claims.
// salvageFindings turns each confirmed claim into a REPORT_SCHEMA-shaped finding so a
// synthesis failure degrades to "verified-but-unmerged findings" instead of nothing.
// Pure JS (reads only the confirmed-claim objects), so it is unit-tested in isolation
// by __tests__/research.salvage.test.mjs via the same block-extraction trick.
// confRank is defined HERE (once) and reused by the Synthesize block below — do not
// redeclare it there.
const confRank = { high: 0, medium: 1, low: 2 }
const salvageFindings = (confirmed) => (confirmed || []).map((c) => {
  const verdicts = Array.isArray(c.verdicts) ? c.verdicts : []
  const positive = verdicts.filter((v) => v && !v.refuted)
  const best = positive.slice().sort((a, b) => confRank[a.confidence] - confRank[b.confidence])[0]
  const refutedVotes = typeof c.refutedVotes === "number"
    ? c.refutedVotes
    : verdicts.filter((v) => v && v.refuted).length
  return {
    claim: c.claim,
    confidence: best ? best.confidence : "low",
    sources: c.sourceUrl ? [c.sourceUrl] : [],
    evidence: (best && best.evidence) || c.quote || "",
    vote: (verdicts.length - refutedVotes) + "-" + refutedVotes,
  }
})
// --- end salvage findings ------------------------------------------------------

// ─── Schemas ───
```

### 3b. Remove the now-duplicate `confRank` in the Synthesize section (~line 412)

`const confRank` now lives in the block from 3a; a second `const confRank` in the same module
scope is a `SyntaxError`. Delete the redeclaration; the `block` builder keeps using `confRank`.

**old_string:**
```js
// ─── Synthesize ───
phase("Synthesize")
const confRank = { high: 0, medium: 1, low: 2 }
const block = confirmed.map((c, i) => {
```
**new_string:**
```js
// ─── Synthesize ───
phase("Synthesize")
// confRank is defined in the "salvage findings (testable)" block above.
const block = confirmed.map((c, i) => {
```

---

## Edit 4 — Salvage path: populate `findings` from confirmed claims

File: `workflows/research.js` (~lines 440–452). Replaces `findings: []` with the helper output
and drops the redundant raw `confirmed:` field (the success path at ~line 454 has no such
field — this keeps both return shapes consistent). Updates `afterSynthesis`.

**old_string:**
```js
if (!report) {
  // Synthesis skipped/errored — salvage the verified claims raw rather
  // than throwing on report.findings and discarding the whole run.
  return {
    question: QUESTION,
    summary: "Synthesis step was skipped or failed — returning " + confirmed.length + " verified claims unmerged.",
    findings: [],
    confirmed: confirmed.map(c => ({ claim: c.claim, source: c.sourceUrl, quote: c.quote, vote: (c.verdicts.length - c.refutedVotes) + "-" + c.refutedVotes })),
    refuted: killed.map(c => ({ claim: c.claim, vote: (c.verdicts.length - c.refutedVotes) + "-" + c.refutedVotes, source: c.sourceUrl })),
    sources: allSources.map(s => ({ url: s.url, quality: s.sourceQuality, claimCount: s.claims.length })),
    stats: { angles: scope.angles.length, sources: allSources.length, claims: allClaims.length, verified: voted.length, confirmed: confirmed.length, killed: killed.length, afterSynthesis: 0, retriesUsed },
  }
}
```
**new_string:**
```js
if (!report) {
  // Synthesis skipped/errored after all retries — DON'T discard the run. The Verify
  // phase already confirmed real claims, so degrade to those claims as unmerged
  // findings (built by the tested salvageFindings helper) instead of findings: [].
  const salvaged = salvageFindings(confirmed)
  return {
    question: QUESTION,
    summary: "Synthesis step was skipped or failed — returning " + confirmed.length + " verified claims as unmerged findings.",
    findings: salvaged,
    refuted: killed.map(c => ({ claim: c.claim, vote: (c.verdicts.length - c.refutedVotes) + "-" + c.refutedVotes, source: c.sourceUrl })),
    sources: allSources.map(s => ({ url: s.url, quality: s.sourceQuality, claimCount: s.claims.length })),
    stats: { angles: scope.angles.length, sources: allSources.length, claims: allClaims.length, verified: voted.length, confirmed: confirmed.length, killed: killed.length, afterSynthesis: salvaged.length, retriesUsed },
  }
}
```

---

## Edit 5 — New test file `workflows/__tests__/research.salvage.test.mjs`

Create this file verbatim. It mirrors `research.label.test.mjs` exactly (same imports,
same `START`/`END` `indexOf`/`slice` extraction, same `new Function` eval), so it tests the
REAL source block with zero duplication.

```js
// Unit tests for the synthesize-failure salvage helper in ../research.js.
//
// Same block-extraction trick as research.label/retry/limiter tests: the workflow
// file can't be imported (its body runs at load and calls runtime globals), so we
// extract the self-contained "salvage findings (testable)" block and evaluate it
// in isolation. This tests the REAL source — no duplication.
//
// Run: node --test plugins/rnd/workflows/__tests__/research.salvage.test.mjs

import { test } from "node:test"
import assert from "node:assert/strict"
import { readFileSync } from "node:fs"
import { fileURLToPath } from "node:url"
import { dirname, join } from "node:path"

const __dirname = dirname(fileURLToPath(import.meta.url))
const SRC = readFileSync(join(__dirname, "..", "research.js"), "utf8")

const START = "// --- salvage findings (testable) "
const END = "// --- end salvage findings "
const startIdx = SRC.indexOf(START)
const endIdx = SRC.indexOf(END)
assert.ok(startIdx !== -1 && endIdx !== -1 && endIdx > startIdx,
  "could not locate the testable salvage-findings block markers in research.js")
const BLOCK = SRC.slice(startIdx, endIdx)

function loadSalvage() {
  const factory = new Function(BLOCK + "\nreturn { salvageFindings, confRank };")
  return factory()
}

test("maps each confirmed claim to a REPORT_SCHEMA-shaped finding", () => {
  const { salvageFindings } = loadSalvage()
  const out = salvageFindings([
    {
      claim: "X is true",
      sourceUrl: "https://example.com/x",
      quote: "the source says X",
      refutedVotes: 0,
      verdicts: [
        { refuted: false, confidence: "medium", evidence: "ev-medium" },
        { refuted: false, confidence: "high", evidence: "ev-high" },
      ],
    },
  ])
  assert.equal(out.length, 1)
  const f = out[0]
  assert.deepEqual(Object.keys(f).sort(), ["claim", "confidence", "evidence", "sources", "vote"])
  assert.equal(f.claim, "X is true")
  assert.equal(f.confidence, "high")            // best (highest) of medium/high
  assert.deepEqual(f.sources, ["https://example.com/x"])
  assert.equal(f.evidence, "ev-high")           // evidence from the best verdict
  assert.equal(f.vote, "2-0")                   // 2 valid, 0 refuted
})

test("vote string reflects refuted count; confidence from best non-refuted verdict", () => {
  const { salvageFindings } = loadSalvage()
  const [f] = salvageFindings([
    {
      claim: "Y",
      sourceUrl: "https://e.com/y",
      quote: "q",
      refutedVotes: 1,
      verdicts: [
        { refuted: true, confidence: "high", evidence: "refuting" },
        { refuted: false, confidence: "low", evidence: "supporting" },
      ],
    },
  ])
  assert.equal(f.confidence, "low")             // only non-refuted verdict is low
  assert.equal(f.evidence, "supporting")        // never use a refuting verdict's evidence
  assert.equal(f.vote, "1-1")
})

test("falls back to low confidence and quote evidence when no positive verdicts", () => {
  const { salvageFindings } = loadSalvage()
  const [f] = salvageFindings([
    { claim: "Z", sourceUrl: "https://e.com/z", quote: "quoted text", refutedVotes: 0, verdicts: [] },
  ])
  assert.equal(f.confidence, "low")
  assert.equal(f.evidence, "quoted text")
  assert.equal(f.vote, "0-0")
})

test("derives refutedVotes from verdicts when the field is absent", () => {
  const { salvageFindings } = loadSalvage()
  const [f] = salvageFindings([
    {
      claim: "W",
      sourceUrl: "https://e.com/w",
      quote: "q",
      verdicts: [
        { refuted: true, confidence: "high", evidence: "a" },
        { refuted: false, confidence: "medium", evidence: "b" },
        { refuted: false, confidence: "high", evidence: "c" },
      ],
    },
  ])
  assert.equal(f.vote, "2-1")                   // 3 verdicts, 1 refuted
  assert.equal(f.confidence, "high")
})

test("missing sourceUrl → empty sources array (never undefined)", () => {
  const { salvageFindings } = loadSalvage()
  const [f] = salvageFindings([
    { claim: "no source", quote: "", refutedVotes: 0, verdicts: [] },
  ])
  assert.deepEqual(f.sources, [])
  assert.equal(f.evidence, "")
})

test("empty / nullish input → empty array (no throw)", () => {
  const { salvageFindings } = loadSalvage()
  assert.deepEqual(salvageFindings([]), [])
  assert.deepEqual(salvageFindings(null), [])
  assert.deepEqual(salvageFindings(undefined), [])
})

test("every produced finding satisfies REPORT_SCHEMA's required finding fields", () => {
  const { salvageFindings } = loadSalvage()
  const out = salvageFindings([
    { claim: "a", sourceUrl: "https://e.com/a", quote: "qa", refutedVotes: 0,
      verdicts: [{ refuted: false, confidence: "medium", evidence: "ea" }] },
    { claim: "b", sourceUrl: "https://e.com/b", quote: "qb", refutedVotes: 0, verdicts: [] },
  ])
  for (const f of out) {
    for (const k of ["claim", "confidence", "sources", "evidence"]) {
      assert.ok(f[k] !== undefined, "finding missing required field: " + k)
    }
    assert.ok(["high", "medium", "low"].includes(f.confidence))
    assert.ok(Array.isArray(f.sources))
  }
})
```

---

## Edit 6 — Version bump

File: `.claude-plugin/plugin.json`. Bumping forces a fresh version-keyed cache dir on
reinstall (reinstalling the *same* 0.1.0 may not re-copy — [[plugin-cache-staleness]]).

**old_string:** `  "version": "0.1.0",`
**new_string:** `  "version": "0.1.1",`

---

## Verification

Run from the marketplace root `cd /Users/russelltsherman/.claude/plugins/marketplaces/ae-skills`.

1. **New unit tests pass:**
   ```bash
   node --test plugins/rnd/workflows/__tests__/research.salvage.test.mjs
   ```
   Expect all 7 tests passing.

2. **No regression in existing tests** (the new block sits between fetch-label and Schemas;
   confirm the other extractors still locate their markers):
   ```bash
   node --test plugins/rnd/workflows/__tests__/research.retry.test.mjs \
               plugins/rnd/workflows/__tests__/research.limiter.test.mjs \
               plugins/rnd/workflows/__tests__/research.label.test.mjs
   ```

3. **Syntax sanity** (catches the duplicate-`confRank` class of error without running the
   workflow): `node --check plugins/rnd/workflows/research.js`.

4. **Propagate to the runtime cache** so `/rnd:ae-research` actually uses the new code:
   reinstall the plugin — `/plugin install rnd@ae-skills`. Because the version is now
   `0.1.1`, this creates `…/cache/ae-skills/rnd/0.1.1/`, and `${CLAUDE_PLUGIN_ROOT}` in the
   `/ae-research` command resolves to it on the next run. Confirm with:
   `cat ~/.claude/plugins/installed_plugins.json | grep -A2 'rnd@ae-skills'` → version `0.1.1`.

5. **End-to-end on the topic that triggered the bug:**
   ```
   /rnd:ae-research behavior driven design
   ```
   Success criteria, checked against the workflow's returned `stats` and the synthesize
   agent transcripts:
   - The report has a non-empty `findings[]` and a `summary` ≤ ~1300 chars.
   - `stats.retriesUsed` is low (synthesize no longer burns 5×3 StructuredOutput attempts).
   - In the synthesize agent transcript, the first `StructuredOutput` call already includes
     a `findings` key (the prevention fix working). If synthesis ever *does* fail, the
     report still carries `findings` from `salvageFindings` (the containment fix), never `[]`.

## Durability (requires the user — outward push)

The edits land in the local marketplace git clone
(`~/.claude/plugins/marketplaces/ae-skills/`, `autoUpdate: true`). A future
`/plugin marketplace update ae-skills` does a `git pull` and will revert uncommitted local
edits. To make the fix permanent, the changes must be committed and **pushed to the
`russelltsherman/ae-skills` GitHub repo**. That is an outward push to the user's repo — get
explicit confirmation before pushing. Until then, the fix is local-only and survives only
until the next marketplace update.

## Out of scope / explicitly NOT doing

- Not touching `review.js`, the batch driver (`research-batch.js`), or `cli.mjs`/`lib.mjs`.
- Not changing `VOTES_PER_CLAIM`, `MAX_ATTEMPTS`, `MAX_CONCURRENT_AGENTS`, or any other tuning.
- Not editing the cache copy directly (version bump + reinstall is the supported path).
