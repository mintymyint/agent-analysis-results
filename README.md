# Passive network characterisation of AI agents — result data

Stage outputs, the frozen analysis plan and the task coding rubric for a thesis
measuring two AI coding agents from their network traffic alone.

Two agents, **OpenClaw** and **Hermes**, executed the same twenty prompts under
two independently collected deployments. Every feature is derived from passive
whole-host packet capture. The man-in-the-middle HTTP log was used for
validation and ground truth only, and never as an analysis input.

---

## The two corpora

| | Corpus 1 | Corpus 2 |
|---|---|---|
| Provider | OpenRouter | SiliconFlow and Together AI |
| Analysis package | `agentnet` 0.9.3 | `agentnet` 0.10.2 |
| Runs indexed | 3,499 | 1,999 |
| Runs analysed | **3,499** | **1,329** |
| Hermes / OpenClaw | 1,000 / 2,499 | 650 / 679 |
| Runs per prompt | Hermes 50, OpenClaw 124–125 | 24–40 |
| Agent × prompt cells | 40 of 40 | 40 of 40 |
| Balance | unbalanced by agent | balanced |

Both corpora ran under **analysis plan revision 7**. Corpus 1's figures were
produced at 0.9.3 and reproduced byte-identically at 0.10.0, so they do not
depend on the package version that produced them.

Corpus 2's funnel: 1,999 indexed → 1,500 in provider scope → 1,499 with a
feature row → −40 by the model-scope rule → −130 by the prompt exclusion →
**1,329**. The two rules overlap on 10 runs.

The twenty prompts are **byte-identical across corpora**, confirmed by SHA-256.
Class distribution in both: 6 `local_execution`, 6 `web_only`,
8 `web_then_local`.

---

## Layout

```
corpus1/     stage output CSVs, OpenRouter
corpus2/     stage output CSVs, SiliconFlow + Together AI
```

Stage outputs only. The analysis package, the capture infrastructure and the
raw captures are not here — see *Not in this repository* below.

---

## Reading the data — four things that will bite you

**`task_id` is corpus-local.** Identifiers are assigned per corpus in SHA order,
and only ten of the twenty coincide between the corpora. **Join on
`task_sha256`, never on `task_id`** — a join on identifier mislabels half the
set.

**`agent_a` is Hermes and `agent_b` is OpenClaw, in both corpora.** The sign
convention is identical throughout: **δ < 0 means OpenClaw higher.**

**The prompt is the unit of analysis, so n = 20.** Cliff's δ is computed within
each prompt and summarised as the median of the twenty prompt-level values. The
3,499 and 1,329 runs are not independent observations and must not be treated
as a sample size.

**`n_at_least_medium` counts |δ| ≥ 0.474**, which is the *large* boundary under
the Romano bands, not the medium one. The column name is misleading and is
retained for compatibility.

---

## What the data shows

Two research questions, and they came out in opposite directions.

### RQ1 — do the two agents differ?

Within either deployment, yes. Across deployments, **six of nine measurements
reversed sign**, and every effect that was medium or larger in corpus 1 is among
them. For five of the six the bootstrap intervals do not overlap.

| feature | corpus 1 δ | corpus 2 δ | |
|---|---|---|---|
| `bytes_ratio_down_up` | −0.899 | **+0.532** | reversed |
| `bytes_down_total` | −0.736 | **+0.597** | reversed |
| `median_flow_duration_s` | −0.613 | **+0.116** | reversed |
| `pktsize_down_median` | −0.565 | **+0.301** | reversed |
| `pktsize_up_median` | +0.400 | **−0.200** | reversed |
| `n_flows_tcp` | −0.160 | **+0.152** | reversed |
| `iat_down_median_ms` | −0.159 | −0.574 | held |
| `bytes_up_total` | +0.076 | +0.282 | held |
| `longest_gap_s` | +0.027 | +0.128 | held |

The reversal is **one-sided**. On `bytes_ratio_down_up`, Hermes moves 0.820 →
0.826 while OpenClaw falls 2.269 → 0.527. A swapped-directory error would have
moved both.

Three things changed together between the corpora — provider, agent client
version and served model — so the reversal is measured but **cannot be
attributed to any one of them**.

### RQ2 — does task type show in the traffic?

Four measures — `bytes_up_total`, `bytes_down_total`, `n_flows_tcp`,
`distinct_dst_ips` — place the classes in the order
`local_execution < web_only < web_then_local` in **both agents and both
corpora**. Four measures × two agents × two corpora = **16 checks, all 16
agree**, across a fourteen-fold shift in absolute levels.

Two caveats that belong with that number. The four measures are **not
independent** — more external work produces more bytes, flows and destinations
together, so this is one phenomenon measured four ways. And with three classes
there are six possible orderings, so roughly two of eleven features would match
by chance. **No significance test was performed for RQ2**; the criterion is
replication across agents.

### RQ2b — task complexity

**The gate is closed in both corpora**, and was closed before any behaviour was
measured. `complexity_score` is derived from six factors that also determine
`taxonomy_class`, so a complexity effect cannot be separated from a class
effect. `CORPUS_CAN_ESTIMATE_COMPLEXITY_EFFECT = False`, frozen in advance.

Spearman correlations are present in the outputs. **They are descriptive only
and estimate no complexity effect, whatever their size.**

### Exploratory classification (corpus 2 only)

Task class predicted from the eleven passive features, leave-one-prompt-out:
**14 of 20 held-out prompts correct, 70% against a 40% majority-class baseline,
p = 0.0030** from 1,000 label permutations. The two pure classes are never
confused with one another; every error involves the composite class.

A pre-specified depth-3 decision tree reached 85%. Logistic regression was
fixed as primary before either ran, so **the reported figure is 70%**, phrased
as "at least 70%". This strand was **not preregistered** and ran on corpus 2
only.

---

## Verification

These outputs were checked by a standalone verifier, run as:

```bash
python3 verify_independently.py \
  --out corpus1/ \
  --expect-absent new_flows_per_active_min
```

The verifier imports nothing from `agentnet` and recomputes Cliff's δ by
brute-force double loop over every pair — the definition, not the rank identity
the pipeline uses for speed. It recomputed **80 per-prompt deltas, 160 cell
medians and 160 drift deltas** and reported **no disagreement at six significant
digits**, which is the documented output precision.

`new_flows_per_active_min` has no rows because plan revision 6 demoted it. Its
absence must be declared on the command line, or the verifier treats it as a
failure and exits 1.

---

## Two features demoted after producing results

Both timings are stated plainly rather than glossed, because post-hoc exclusion
is a fair target for criticism. What makes each defensible is a rationale that
does not depend on the outcome.

| feature | grounds | revision |
|---|---|---|
| `onset_median_ms` | rank correlation **0.094** against MITM-recorded latency over 3,458 runs; resolves a TLS handshake round trip, not application response onset | 4 |
| `new_flows_per_active_min` | passed all three RQ1 filters, dropped because its denominator `activity_span_s` had already been demoted | 6 |

**`onset_median_ms` is never a time-to-first-byte or time-to-first-token
measurement.** That is precisely the interpretation the validation refuted.

---

## Known limits

- **Twenty purposively chosen prompts.** No inference to prompts outside them.
- **Whole-host capture.** Background traffic is included and never subtracted;
  subtracting it would require knowing which flows belong to the agent, and the
  premise is that an observer does not.
- **Collection was blocked, not interleaved**, so agent identity is confounded
  with collection period. The drift control bounds that confound and cannot
  remove it. Four of nine RQ1 features move more over the collection period than
  they move between agents.
- **Three axes changed together** between the corpora.
- **Not a demonstrated attack.** This is evidence that the information is
  present, under conditions favourable to an observer.
- **No agent classifier was built**, so there is no accuracy figure anywhere for
  identifying which agent produced a capture.

---

## Not in this repository

- **Packet captures.** Whole-host capture means they contain traffic unrelated
  to the experiments.
- **MITM HTTP logs.** They contain decrypted request and response bodies,
  including provider credentials.
- **The capture and proxy infrastructure**, which is a separate codebase and not
  mine to publish.
- **The analysis package itself.** This repository holds what the analysis
  produced, not what produced it.

---

## Citing this

Reference a **commit hash**, not a branch:

```
https://github.com/mintymyint/agent-analysis-results/tree/<commit-sha>
```

A branch URL resolves to whatever the repository looks like when it is opened.
A commit resolves to the state the analysis actually ran against.
