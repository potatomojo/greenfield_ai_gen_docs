# Agent Article Retrieval Evaluation Process

## Purpose

This process measures whether the article-finding agent returns the expected article link at rank 1 for a prompt. It bootstraps a repeatable evaluation set from the existing article index and uses deterministic link comparison for scoring.

The target is 99.99% top-1 accuracy. A finite test run cannot prove that the agent is always 99.99% accurate. It can provide a benchmark score and, only when the evaluation cases are a sufficiently large, representative, independent random sample with reliable labels, a confidence bound on production accuracy. Keep those claims separate in every report.

## Scope and assumptions

- The agent, article index, and a way to run the agent against a fixed index snapshot already exist.
- The evaluated unit is one prompt and its single expected top article. The response passes only when the expected canonical article link is the first article returned.
- A link found lower in the top 10 is a top-1 failure. A missing, malformed, or unparseable first link is also a failure.
- The evaluation oracle must be independent of the agent response. If the index's own search result is used as the expected answer, the check measures agent-to-index integration and ranking parity; it does not establish that the index's relevance judgment is correct.
- Category-only prompts may have many relevant articles. To use exact-link scoring, specify a deterministic ranking rule or a reviewed expected article for each case. Otherwise evaluate category membership as a separate constraint, not as exact top-1 accuracy.
- Content-related prompts can have several reasonable answers. Exact-link scoring is meaningful only when a target article is explicitly defined and the label is defensible.

## Evaluation methods

### A. Target-known synthetic benchmark — primary, high-volume method

Generate cases locally from the indexed corpus, without calling a GenAI API. Build each case around a known target article ID, then create query variants using article metadata and text: category templates, title terms, selected body sentences, and deterministic term masking or reordering. Exclude the target URL, document ID, and exact title when they would trivially reveal the answer. Save the generation rule and source article ID with every case.

This creates labels by construction and scales quickly. It measures retrieval on the generated query distribution; it does not automatically represent real user language or prove semantic relevance. Deduplicate near-identical prompts and keep variants of one article together when splitting data, so the same source does not leak across development and held-out sets.

### B. Small adjudicated anchor set — quality check

Create a modest stratified sample across categories, common and rare articles, and both prompt types. Have a domain reviewer confirm that the prompt reasonably targets the expected article; resolve disagreements and mark genuinely ambiguous prompts as excluded or multi-answer. A second reviewer can adjudicate a smaller overlap sample to estimate label consistency.

This set checks whether synthetic generation creates plausible prompts and catches label errors. A set of a few hundred examples is useful for diagnosis, but it is far too small to establish 99.99% accuracy.

### C. Reference-index differential check — integration check

Where a stable standalone index-search operation is available, run the same normalized query through it and through the agent, then compare the first article IDs. Record this separately from the relevance benchmark. A match shows that the agent passes through or presents the reference ranking consistently; it does not independently validate the reference ranking.

### D. Existing traffic sample — production representativeness check

If permitted prompt logs already exist, sample cases by time, category, and prompt type. Use independently reviewed expected answers or sufficiently strong existing outcome evidence. Clicks and engagement are weak relevance signals and must not be treated as ground truth without validation. Do not send prompts to external services to generate labels.

## Recommended two-week execution

### Days 1–2: define and instrument

1. Freeze the agent version, prompt configuration, index snapshot, and URL canonicalization rules.
2. Define the target population (for example, all production article-search prompts during a stated period), top-1 pass rule, exclusions, and sampling unit.
3. Build the runner to capture the prompt, raw response, parsed first article ID/link, latency, agent/index versions, and case ID.
4. Confirm expected-link labels come from a recorded independent source. Separate relevance accuracy from index-parity checks.

### Days 3–5: generate and review cases

1. Generate target-known cases locally across categories and articles. Deduplicate and assign stable IDs.
2. Draw a small stratified subset for human plausibility and label review. Fix generation rules before freezing the held-out set.
3. Store the frozen benchmark and its manifest; do not tune against held-out failures and then report the same set as an untouched final evaluation.

### Days 6–8: run and diagnose

1. Run all benchmark cases against the frozen agent and index snapshot. Retry only infrastructure failures, using a documented retry policy; never retry a genuine wrong answer into a pass.
2. Score by exact canonical article ID at rank 1. Report total cases, passes, failures, exclusions, and rates by category, prompt type, generation method, and article frequency.
3. Inspect every failure for agent error, parser/canonicalization issue, bad label, ambiguous prompt, or index/ranking issue. Correct labels only with recorded evidence, then rerun and preserve both the original and corrected results.

### Days 9–10: independent sample and report

1. If real prompt logs and reliable labels are available, run a separate representative random sample as the production-oriented estimate. Keep it distinct from synthetic benchmark results.
2. Calculate the one-sided 95% exact binomial confidence bound. Publish the sample design, independence assumptions, label source, system/index versions, and limitations beside the result.
3. Make the release decision using the gates below. If the sample or label quality is insufficient, report “99.99% not established” and state the evidence needed next.

## Statistical evidence for a 99.99% claim

Accuracy is the fraction of evaluated prompts whose expected article ID is the agent's first result. For a representative independent random sample with zero failures, the one-sided 95% exact lower confidence bound on accuracy is 0.05^(1/n), where n is the number of cases. To put that lower bound at or above 99.99%, at least 29,956 independent cases must pass with zero failures. This is roughly 3,000 cases per business day over ten days.

Illustrative zero-failure bounds:

| Independent cases | One-sided 95% lower accuracy bound |
| ---: | ---: |
| 1,000 | 99.70% |
| 10,000 | 99.97% |
| 29,956 | 99.99% |

These numbers do not apply to a convenience sample, synthetic-only cases, repeated paraphrases of the same article/query, correlated cases, or cases with uncertain labels. Correlation reduces effective sample size; a bootstrap or cluster-aware analysis may be needed, and a simple binomial claim is not justified. With any observed failures, 29,956 is no longer sufficient; compute the exact binomial bound for the observed pass/fail count. No finite sample proves universal accuracy.

Therefore, the two-week outcome should be one of:

- **99.99% supported at 95% confidence:** at least 29,956 independent, representative, correctly labeled cases with zero failures, under a frozen evaluation setup and defensible sampling assumptions.
- **Benchmark passed, production claim not established:** the synthetic and/or smaller adjudicated benchmark passes its stated threshold, but representative statistical evidence is insufficient.
- **Target missed:** the measured benchmark or confidence bound falls below the agreed threshold, or important categories have unresolved failures.

Do not describe a high synthetic benchmark score as proof of 99.99% production accuracy.

## Case format and scoring

Keep one immutable record per case with these fields:

| Field | Purpose |
| --- | --- |
| case_id | Stable unique identifier |
| prompt | Exact user prompt sent to the agent |
| prompt_type | Category, content-related, or other defined class |
| expected_article_id | Stable index identifier used as the primary comparison key |
| expected_canonical_url | Human-readable expected link |
| category | Article category or evaluation stratum |
| source_article_id | Corpus item used to construct the case, when applicable |
| generation_method | Template/rule or reviewed-label source |
| label_status | Constructed, reviewed, adjudicated, or excluded |
| benchmark_split | Development, validation, or held-out |
| index_snapshot and agent_version | Reproducibility metadata |

Canonicalize links through a fixed, documented mapping to article IDs. Ignore only known tracking parameters or harmless URL formatting differences. Do not collapse distinct article IDs, silently follow arbitrary redirects, or accept the expected link if it appears below rank 1.

For each case, record the raw response, parsed top article ID, pass/fail, failure reason, and run timestamp. Overall top-1 accuracy is passes divided by eligible cases. Report exclusions separately; never count them as passes. Also report per-stratum results so a strong overall score cannot hide a weak category or prompt type.

## Automation and safeguards

Keep the pipeline in separable stages so each can be rerun and audited: case generation, case validation/deduplication, agent runner, deterministic link parser/oracle, scorer, and report writer. Pin the index and agent versions for each run. Store prompts and outputs with access controls appropriate to their content, and avoid credentials in the benchmark files.

Use local deterministic generation for the two-week bootstrap. Automated requests to GenAI services for prompt or label generation require separate approval under the project instructions. Preserve random seeds, templates, sampling code/configuration, and manifests so a run can be reproduced.

## Release gates and ongoing monitoring

- The held-out exact-link benchmark has no unexplained failures; all failures have an assigned cause and disposition.
- Category and prompt-type slices meet separately agreed minimums; do not rely on the aggregate alone.
- The 99.99% production claim is made only if its independent representative sample and confidence-bound requirements are met.
- If not met, publish the benchmark score and confidence bound plainly, label the target “not established,” and continue collecting independently sampled, reviewed cases.
- Re-run the frozen benchmark after agent, ranking, parsing, or index changes. Monitor a sampled stream of real prompts with a documented review process and refresh the benchmark when categories or traffic change materially.

## Final report checklist

Include the evaluation period; agent and index versions; dataset source and case counts; generation and sampling methods; label review process; exact top-1 rule; exclusions; overall and per-stratum pass rates; confidence interval and assumptions; failure analysis; known limitations; and the release decision. State separately what the synthetic benchmark demonstrates and what the production sample supports.
