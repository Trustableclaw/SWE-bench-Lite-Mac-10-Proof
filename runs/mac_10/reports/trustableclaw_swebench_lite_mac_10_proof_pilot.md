# TrustableClaw SWE-bench Lite 10-Task Proof-Pipeline Pilot Report

This report documents a controlled **10-task SWE-bench Lite proof-pipeline pilot** for TrustableClaw.
---

## Executive Summary

TrustableClaw completed a 10-task SWE-bench Lite proof-pipeline pilot. The run produced agent-generated patches, test logs, result JSON files, cryptographic receipts, and tamper-evident verification records.

The run attempted **10 different SWE-bench Lite tasks** and resolved **0/10**. The value of this run is proof-pipeline validation: it demonstrates that TrustableClaw can record the attempted patch, test log, SWE-bench result, and verification outcome for each task without overstating the benchmark result.

### Performance Summary

| Metric | Value | Note |
| :--- | :--- | :--- |
| **Pilot Type** | 10-task proof-pipeline pilot | Not a full SWE-bench Lite benchmark score |
| **Total Tasks Attempted** | 10 | Lightweight `sqlfluff`, `marshmallow`, and `pvlib` tasks |
| **Inference Model** | Requested: `gpt-5.4-mini`; response logs may show a concrete dated snapshot such as `gpt-5.4-mini-2026-03-17` | The verifier now records and checks requested model and actual response model separately |
| **Execution Mode** | Sequential (`--max_workers 1`) | Conservative Mac run mode |
| **Resolved Tasks** | **0/10** | Actual SWE-bench result status |
| **Patch Apply Failures** | **9** | Model patches did not apply cleanly to the target repo snapshots |
| **Setup/Build Failures** | **0** | No task is currently classified as pure setup/build failure |
| **Tests Executed but Unresolved** | **1** | Patch applied and tests ran, but task remained unresolved |
| **Ledger Blocks Generated** | 60 | 6 chained receipts per task |
| **Audit Verification Status** | Passed | Required shape, self-contained proof manifest, all 60 receipts, manifest hashes, agent log hashes, exact prediction/patch consistency, model metadata consistency, artifact hashes, and outcome consistency verified |
| **Tamper Test Status** | Passed | Real verifier failed modified proof-package copies for artifact, agent-log, receipt, missing-receipt, missing-manifest, proof-manifest count, prediction/patch mismatch, trailing-whitespace mismatch, and model metadata mismatch tampering |

---

## Methodology & Execution Environment

The evaluation pilot was conducted on a local macOS environment under strict resource constraints.

### 1. Hardware & Environment Specifications

* **Host Operating System:** macOS
* **Virtualization Host:** Docker Desktop
* **Python Runtime:** Python `3.11.4`
* **Execution Constraint:** `--max_workers 1` / sequential execution

### 2. Allowlisted Task Profiles

The pilot used the first 10 IDs from `benchmark/task_sets/swebench_lite_mac_10.txt`. The run package also includes `proof_manifest.json`, which stores these expected task IDs so the proof package can be verified without relying on the repo-level allowlist:

* `sqlfluff__sqlfluff-1625`
* `sqlfluff__sqlfluff-2419`
* `sqlfluff__sqlfluff-1733`
* `sqlfluff__sqlfluff-1517`
* `sqlfluff__sqlfluff-1763`
* `marshmallow-code__marshmallow-1359`
* `marshmallow-code__marshmallow-1343`
* `pvlib__pvlib-python-1707`
* `pvlib__pvlib-python-1072`
* `pvlib__pvlib-python-1606`



## Cryptographic Proof & Ledger Architecture

For every evaluated task, TrustableClaw records six sequential receipts:

1. `01_task_selected`: task selection from the static allowlist.
2. `02_agent_started`: solver/provider metadata plus requested model, actual response model, agent log file, and agent log SHA-256 hash.
3. `03_patch_generated`: patch artifact and SHA-256 hash.
4. `04_tests_executed`: test log artifact and SHA-256 hash.
5. `05_result_recorded`: SWE-bench result JSON hash and actual `resolved` value.
6. `06_verification_completed`: verification recording for the task.

The updated receipt generator derives `resolved` and `outcome_type` from the actual SWE-bench result JSON and test log. It no longer hard-codes `resolved: true`, and unresolved task receipts are recorded as `RECORDED_UNRESOLVED` with `assertions_passed: false` so the proof ledger does not imply benchmark success.

---

## Integrity Audit & Verification Results

The verification suite now validates nine classes of claims:

1. **Required package shape**: all 10 expected task IDs must be present in the self-contained `proof_manifest.json`.
2. **Receipt completeness**: the package must contain exactly 60 receipts: six expected receipt kinds per task.
3. **Ledger integrity**: receipt indexes, hashes, and `prev_hash` links must be valid.
4. **Manifest integrity**: `artifact_hashes.json` is required and every listed hash must match the file on disk, including `proof_manifest.json`.
5. **Artifact integrity**: agent inference log, patch, test log, result JSON, and predictions file hashes must match the files on disk.
6. **Exact prediction/patch consistency**: every `model_patch` in `predictions.jsonl` must exactly match `patches/<instance_id>.patch`, including trailing whitespace and newlines, because SWE-bench evaluates `predictions.jsonl` while receipts hash the patch files.
7. **Model metadata consistency**: every `agent_started` receipt requested model must match the corresponding `model_name_or_path` in `predictions.jsonl`.
8. **Agent log integrity**: every `agent_started` receipt must point to `agent_logs/<instance_id>_inference.log`, its SHA-256 must match the log file, and its `actual_response_model` must match the raw OpenAI response log model. This explicitly separates requested model aliases from provider-returned dated snapshots.
9. **Outcome integrity**: every `result_recorded` receipt must match the corresponding SWE-bench result JSON. If a receipt says `resolved: true` while the result JSON says `resolved: false`, verification fails.

Current verification output:

```json
{
  "ok": true,
  "required_shape_ok": true,
  "receipts_ok": true,
  "ledger_ok": true,
  "manifest_hash_match": true,
  "agent_log_hash_match": true,
  "agent_log_model_match": true,
  "patch_hash_match": true,
  "test_log_hash_match": true,
  "result_hash_match": true,
  "outcome_match": true,
  "verification_receipts_not_overstated": true,
  "prediction_patch_match": true,
  "prediction_model_match": true,
  "proof_package_ok": true,
  "expected_tasks": 10,
  "expected_receipts": 60,
  "actual_receipts": 60,
  "errors": []
}
```

---

## Real Artifact Tamper Testing

The tamper suite now creates a temporary copy of the real proof package, modifies that copied package, and then runs the real verifier against the modified copy.

It verifies that tampering is detected for:

* an actual agent inference log;
* an actual patch file;
* an actual test log;
* an actual result JSON file;
* an actual receipt JSON body;
* a missing receipt;
* a missing `artifact_hashes.json` manifest;
* a missing `proof_manifest.json` manifest;
* false count metadata inside `proof_manifest.json`;
* an agent log whose raw response model no longer matches the corresponding `agent_started` receipt;
* a `predictions.jsonl` patch that no longer matches the corresponding receipt-backed patch file;
* a trailing whitespace mismatch between `predictions.jsonl` and the corresponding receipt-backed patch file;
* a model metadata mismatch between `predictions.jsonl` and the corresponding `agent_started` receipt;
* an empty receipts directory;
* a missing receipts directory.

Current tamper output:

```json
{
  "ok": true,
  "patch_tamper_detected_by_real_verifier": true,
  "test_log_tamper_detected_by_real_verifier": true,
  "result_tamper_detected_by_real_verifier": true,
  "receipt_json_tamper_detected_by_real_verifier": true,
  "missing_receipt_detected_by_real_verifier": true,
  "missing_manifest_detected_by_real_verifier": true,
  "missing_proof_manifest_detected_by_real_verifier": true,
  "proof_manifest_count_tamper_detected_by_real_verifier": true,
  "agent_log_tamper_detected_by_real_verifier": true,
  "prediction_patch_mismatch_detected_by_real_verifier": true,
  "prediction_patch_trailing_whitespace_mismatch_detected_by_real_verifier": true,
  "prediction_model_metadata_mismatch_detected_by_real_verifier": true,
  "empty_receipts_dir_detected_by_real_verifier": true,
  "missing_receipts_dir_detected_by_real_verifier": true
}
```

