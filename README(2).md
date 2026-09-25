# Cancer Registry Hybrid NER + JEV

This project extracts cancer-registry entities from unstructured oncology notes with a hybrid workflow:

1. Spark NLP Healthcare produces high-recall entity candidates.
2. Narrow deterministic rules protect selected Grade and Pathology_Test mentions.
3. TypeSafe JEV reviews the remaining candidate spans in paragraph context and can confirm, relabel, or reject them.
4. JEV assigns assertion only to entity types for which assertion is clinically meaningful.

The notebook should send JEV a target-marked paragraph, not an isolated token. Each candidate is represented by its text, entity label, normalized offsets, paragraph context, and section hint.

## Frozen baseline

The current baseline is **Run 7** on 20 synthetic oncology notes.

| Check | Result |
| --- | ---: |
| Hybrid candidates in audit output | 1,209 |
| Final retained entities | 1,193 |
| Review-queue rows | 241 |
| Grade rule candidates retained | 118 / 118 |
| CBC/CMP Pathology_Test candidates retained | 40 / 40 |
| Candidates with valid target-context alignment | 1,209 / 1,209 |
| JEV entity or assertion API errors | 0 |

The offset validation is an important guardrail. Spark NLP NER chunks use split-relative offsets, while rule chunks can use document-relative offsets. Before JEV is called, rule offsets are converted to the same split-relative coordinate system and validated against the exact target text.

## Registry entity ontology

The current ontology contains 26 labels:

`Cancer_Dx`, `Tumor_Finding`, `Imaging_Test`, `Pathology_Test`, `Pathology_Result`, `Biomarker`, `Biomarker_Result`, `Clinical_Condition`, `Metastasis`, `Anatomical_Site`, `Lymph_Node`, `Histological_Type`, `Tumor_Size`, `Staging`, `Grade`, `Invasion`, `Cancer_Surgery`, `Radiotherapy`, `Chemotherapy`, `Targeted_Therapy`, `Immunotherapy`, `Hormonal_Therapy`, `Adverse_Event`, `Response_To_Treatment`, `Performance_Status`, and `Date`.

## Deterministic rules

- **Grade:** a narrow regex captures `Grade 1` through `Grade 5`, plus `low-grade`, `intermediate-grade`, and `high-grade` forms. These candidates are locked as `Grade` and do not receive assertion.
- **Pathology_Test:** the terminology list can be extended over time. `CBC`, `CMP`, `complete blood count`, and `comprehensive metabolic panel` are currently locked as `Pathology_Test` under the registry policy. They do not receive assertion.
- Other rule-derived pathology terms remain JEV candidates so that the model can reject or relabel them when context requires it.

## JEV entity review

For each non-locked candidate, JEV chooses one of the 26 entity labels or `NOT_ENTITY` using the target-marked paragraph and section information.

JEV is used for three decisions:

1. Confirm the original Spark NLP label.
2. Correct an ontology error, such as `Targeted_Therapy` versus `Immunotherapy`.
3. Reject non-entity or meta-language candidates while retaining an audit record.

Every candidate retains provenance in the audit output: original label, final label, rule source, target context, confidence and margin fields, JEV choices, and review flags.

## Assertion policy

JEV assertion labels are:

`Present`, `Planned`, `Past`, `Family`, `Absent`, `Hypothetical`, and `Possible`.

Assertion is applied only to:

`Cancer_Dx`, `Tumor_Finding`, `Clinical_Condition`, `Metastasis`, `Staging`, `Invasion`, `Cancer_Surgery`, `Radiotherapy`, `Chemotherapy`, `Targeted_Therapy`, `Immunotherapy`, `Hormonal_Therapy`, `Adverse_Event`, and `Response_To_Treatment`.

All other retained entity types, including `Grade` and `Pathology_Test`, receive `Not_Applicable`.

On the frozen Run 7 output, LLM-referee assertion agreement conditional on a correctly extracted and typed entity was **86.3% (784 / 908)**. This is a regression-test signal, not clinician-authored gold-standard accuracy. The main assertion tuning opportunity is distinguishing `Possible` from `Present` and `Past`, especially for adverse events.

## Notebook inputs and outputs

### Inputs

- One UTF-8 `.txt` oncology note per file.
- A trained Spark NLP Healthcare NER model.
- Spark NLP Healthcare credentials and TypeSafe JEV credentials, supplied through Colab Secrets or environment variables.

Never commit credentials, API keys, licenses, real clinical notes, or PHI-bearing outputs.

### Outputs

The notebook writes the following files for each run:

| File | Purpose |
| --- | --- |
| `cancer_registry_jev_audit.csv/json` | Every candidate, including rules, JEV decisions, offsets, provenance, and review flags. |
| `cancer_registry_jev_final.csv/json` | Retained final registry entities with final assertion status. |
| `cancer_registry_jev_review_queue.csv` | Candidates requiring entity or assertion review. |

## Suggested repository layout

```text
cancer-registry-jev/
├── README.md
├── notebooks/
│   └── cancer_registry_hybrid_jev_run7.ipynb
├── rules/
│   ├── grade_rule.json
│   └── pathology_patterns.json
├── docs/
│   └── evaluation_notes.md
└── .gitignore
```

Export the exact notebook that produced Run 7 and store it as `notebooks/cancer_registry_hybrid_jev_run7.ipynb`. Keep earlier notebook drafts only if they are clearly named as historical versions.

## Reproducibility and review

- Pin the Spark NLP model version, JEV model version, prompt version, and rule files.
- Cache JEV decisions using a key built from the normalized target context, span, candidate label, assertion label set, and prompt version.
- Keep the review queue rather than silently discarding low-confidence or rejected clinical candidates.
- Re-run the 20-note regression suite after any rule, prompt, model, or threshold change.
- Validate clinical performance with an MD-adjudicated set before making clinical accuracy claims.

## License and data note

This repository should contain only code, rule definitions, synthetic/de-identified examples that are approved for sharing, and documentation. It should not contain Spark NLP licenses, TypeSafe credentials, or identifiable clinical data.
