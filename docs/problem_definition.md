# Bug-Hunting Agent: Problem Definition

**Version:** V4 (final)
**Scope:** Track 1, the vulnerability-triage classifier
**Status:** Fixed before any model results exist

## 0. Project Context

The project has two research tracks that start from the same base cases:

1. **Vulnerability triage classifier (this document).** A trained model that predicts which HTTP input parameters are worth manual security testing, benchmarked against a rule-based baseline.
2. **LLM robustness study (specified separately).** An LLM-based version of the same triage task, evaluated for susceptibility to prompt injection hidden in untrusted web content.

The classifier in this document never reads response content. The LLM track receives response context under separate controls and has its own specification.

## 1. Problem Statement

The objective is to develop an AI-assisted bug-hunting agent that analyses HTTP input parameters to identify potential web application vulnerability categories and prioritise inputs for further security testing.

The system classifies HTTP inputs using their structural and contextual characteristics. Predictions are evaluated on a labelled dataset, with lab targets used for validation in Month 7. The success criterion is **higher precision than the rule-based baseline at matched recall** on the `worth_testing` task (operationalised in Section 6), and higher per-class F1 on the multi-label task.

## 2. Unit of Classification and Input Scope

A single classification sample represents one HTTP input parameter. Each sample contains:

- HTTP method
- URL path and path context
- Parameter name
- Parameter value (`value_raw` and `value_clean`)
- Parameter source (`query`, `form`, or `json`)

```json
{
  "method": "GET",
  "path": "/search",
  "parameter": "q",
  "value_raw": "shoes",
  "value_clean": "<STRING>",
  "source": "query"
}
```

**V1 scope:**

- Query parameters, form body parameters and flat JSON keys: included
- Path segments: retained as context only, not independent samples
- Nested JSON keys, headers and cookies: deferred to future versions
- JavaScript and HTML snippets: not samples in this track; reserved as context content for the LLM robustness study

### Model features

The classifier uses only the following feature groups:

- **Parameter name:** tokens, character n-grams, length, entropy, presence of digits or separators
- **Path:** depth, segment tokens, numeric or UUID segments, route prefix, file extension (after label-token masking, see below)
- **Request context:** method, source, number of parameters, names of sibling parameters
- **Value:** the `value_clean` type only

Raw parameter values are never used as features.

**Path label-token masking.** Lab applications often place hints in the path (for example a directory named after the vulnerability class). A fixed list of label-revealing tokens (such as `sqli`, `xss`, `ssrf`, `lfi`, `idor`, `vuln`, `lab`, `exploit`, `challenge`) is masked in the path before feature extraction. The unmasked path is kept only in an audit column.

### Value handling (prevents payload leakage)

Disclosed reports often contain the attack payload as the parameter value, while benign samples contain ordinary values. A model could learn to detect payloads instead of parameter characteristics, which would not generalise to real recon data.

`value_raw` is retained only for source auditing and is excluded from model features and evaluation. `value_clean` is generated through a deterministic, label-independent normalisation procedure applied identically to all samples. It maps values to neutral types such as `<STRING>`, `<INTEGER>`, `<URL>` and `<BOOLEAN>`. No manual redaction based on vulnerability labels or knowledge of attack payloads is permitted.

Each type pattern is matched against the full value, using a fixed precedence order (`<EMPTY>`, `<BOOLEAN>`, `<INTEGER>`, `<URL>`, `<STRING>`). The type set is not extended with length, entropy or character-class features of the value. A leakage check trains a classifier on `value_clean` alone; performance clearly above chance means the mapping must be revised before results are reported.

## 3. Label Space

The system uses multi-label classification:

| Label | Definition |
|---|---|
| `sqli` | Input may influence SQL queries in an unsafe manner. |
| `xss` | Input may be rendered in a browser without appropriate output encoding. |
| `ssrf` | Input may influence server-side requests to unintended destinations. |
| `idor` | Input is an object reference for which a broken access-control vulnerability is documented in the source. |
| `open_redirect` | Input may influence redirects to untrusted destinations. |
| `lfi` | Input may influence unsafe local file access or inclusion. |
| `benign` | No vulnerability is documented for the sample in its source, and the sample is `verified_benign`. |

A sample may receive multiple vulnerability labels. `benign` is assigned only to `verified_benign` samples and is mutually exclusive with all vulnerability labels.

**Ground truth versus prediction.** Ground-truth labels are assigned only from documented evidence. The system's predictions are described as **candidates** (for example "IDOR candidate"), because a parameter name or a single request cannot confirm a vulnerability. The `idor` ground-truth label is assigned only when the source documents an access-control vulnerability associated with the specific object reference. Baseline and model predictions identify IDOR candidates only. Confirming IDOR requires ownership context, potentially involving multiple user accounts, which is out of scope for V1.

### Label provenance

Every sample carries a `label_provenance` field:

- `verified_vulnerable`: a vulnerability documented in a disclosed report, lab solution or CVE
- `verified_benign`: confirmed non-vulnerable within the defined scope, such as a documented safe endpoint in a lab application
- `assumed_benign`: no vulnerability documented, but not confirmed safe

Provenance records the reliability of the ground truth independently of model predictions and baseline rules.

### Per-parameter labelling rule

Reports usually describe a vulnerability at the request level, but samples are per parameter. Only the parameter named as vulnerable in the source receives `verified_vulnerable` and a vulnerability label. Sibling parameters from the same request are `assumed_benign` unless the source explicitly confirms them safe.

### Per-class annotation (positive / negative / unknown)

Each vulnerability class is annotated as **positive**, **negative** or **unknown**. Unmentioned vulnerability classes are not automatically treated as negative. A class is marked negative only when:

- (a) lab documentation lists the parameter as non-vulnerable for that class, or
- (b) a source-code review shows a relevant control (parameterised queries, output encoding or an allow-list).

All `verified_benign` samples are negative for every vulnerability class. Unknown class labels are excluded from class-specific metric calculations.

### Handling of `assumed_benign`

`assumed_benign` samples are excluded from primary supervised training and primary evaluation. They may be retained for separate exploratory or unlabelled-data experiments.

## 4. Binary Classification: worth_testing

The secondary binary target determines whether a sample should be prioritised for further investigation. Its ground truth is derived exclusively from provenance:

- `verified_vulnerable` → `worth_testing = yes`
- `verified_benign` → `worth_testing = no`
- `assumed_benign` → `worth_testing = unknown`, excluded from primary binary evaluation

The agent still produces a yes/no prediction for every sample, but predictions on `unknown` samples are not treated as verified results. A negative prediction does not guarantee that an input is secure. The number of `unknown` samples is reported in the data card.

## 5. Baseline Rules

**Normalisation and tokenisation.** Parameter names are lowercased and split on underscores, hyphens, dots and camelCase boundaries (`user_id` and `userId` both become `[user, id]`). Short patterns use exact token matching; longer patterns use regular expressions with word boundaries.

**Emission policy.** Each rule fires independently. A parameter can receive several candidate categories (for example `order_id` triggers rules 1 and 7).

| No. | Matching condition | Candidate category |
|---|---|---|
| 1 | Any token equal to `id`, `uid` or `pid` after tokenisation | IDOR candidate |
| 2 | Any token equal to `url`, `uri` or `callback` | SSRF |
| 3 | Any token equal to `redirect`, `next` or `return` | Open redirect |
| 4 | Any token equal to `q`, `search`, `query` or `keyword` | XSS, SQLi |
| 5 | Word-boundary match for `comment`, `message` or `description` | XSS |
| 6 | Word-boundary match for `file`, `filename`, `path` or `template` | LFI |
| 7 | Word-boundary match for `sort`, `order`, `filter` or `column` | SQLi |
| 8 | Any token equal to `include`, `resource`, `fetch` or `proxy` | SSRF |
| 9 | See below | IDOR candidate |

**Rule 9.** Match a request whose path contains the complete segment `users`, `accounts` or `documents` at any position. The immediately following segment must be entirely numeric, a valid UUID, or a hexadecimal string of at least eight characters. When it matches, an IDOR candidate is emitted for every parameter of that request. Baseline results are reported both with and without rule 9.

Rule 1 emits only an IDOR candidate, not SQLi. A broad rule based on POST, PUT or PATCH alone was excluded because the HTTP method by itself is not meaningful evidence of SQLi or XSS. These rules produce candidates only and do not confirm vulnerabilities.

## 6. Evaluation and Dataset Considerations

**Dataset construction**

- Samples are collected from source inventories **independently of the baseline rules**; parameters are not selected using the name patterns the baseline uses.
- Verified-vulnerable and verified-benign samples are drawn from the same applications wherever the sources allow.
- Train, validation and test sets are split by source application or disclosure, not randomly.
- Ground-truth labels and provenance are fixed before any model evaluation. Samples of uncertain provenance are never treated as verified negatives.

**Sanity checks (all reported)**

- **Source-only baseline:** predicts the label from the source application alone. Strong performance indicates a source confound.
- **Value-only check:** a classifier trained on `value_clean` alone (Section 2).
- **Path-only ablation:** a classifier trained on the masked path alone. Strong performance indicates path leakage.

Any check performing clearly above chance means that feature group is leaking and must be revised before results are reported.

**Multi-label evaluation**

- Per-class precision, recall and F1 are calculated using only established positive and negative class labels.
- Class-specific metrics are reported only for classes with at least 30 positive test samples; other classes are reported as exploratory. (The threshold of 30 is a tunable parameter and may be adjusted once dataset size is known.)

**Binary (`worth_testing`) evaluation**

- Binary precision, recall and F1 are calculated on `verified_vulnerable` and `verified_benign` samples only.
- The binary decision threshold is selected using the validation set to match the rule-based baseline's recall. Final precision is compared on the held-out test set without test-set threshold tuning.
- Results are also reported as precision-recall curves with PR-AUC, and as a bootstrap 95% confidence interval for the precision difference against the baseline.
- Because the test set's class balance differs from real traffic, absolute precision values are not claimed to represent deployment performance; only the comparison against the baseline on the same test set is.
- Where the number of source applications allows, results are repeated with leave-one-source-out evaluation.

## 7. Limitations

- `verified_benign` samples will come mostly from lab applications, so the binary test set will be small and unlike real-world traffic. The Month 7 pilot is the check against this.
- Lab applications may still carry residual hints beyond the masked path tokens, which is why the path-only and source-only checks are reported.
- Parameter-level evidence cannot capture authorisation context, so IDOR performance reflects only how well object references are identified.
- `assumed_benign` labels may hide undiscovered vulnerabilities, which is why they are excluded from primary training and evaluation.
- Rare vulnerability classes may have too few positives for reliable class-specific metrics.

## 8. Scope and Ethics

V1 focuses on HTTP input analysis and vulnerability candidate prioritisation. It does not claim to confirm vulnerabilities from parameter names or a single request. Active testing and validation are restricted to explicitly authorised systems and controlled lab environments.
