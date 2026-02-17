# PQ Attestation Adapter (PQAA)

* **Specification Version:** 1.0.0
* **Status:** Implementation Ready
* **Date:** 2026
* **Author:** rosiea
* **Contact:** [PQRosie@proton.me](mailto:PQRosie@proton.me)
* **Licence:** Apache License 2.0 — Copyright 2026 rosiea
* **PQ Ecosystem:** STANDALONE — Evidence Producer Only

---

**Problem:** Every platform implements hardware trust differently — Secure Enclave, TPM, StrongBox, software keystores. Security evidence is fragmented, non-portable, and incomparable across devices.

**Solution:** PQAA normalises platform-specific trust anchors into canonical evidence artefacts. One attestation interface across all platforms. Evidence classification is deterministic. No platform lock-in.

PQAA defines evidence normalisation only. It does not enforce policy or grant authority.

Part of the [PQ Ecosystem](https://github.com/rosieRRRRR/pq-ecosystem).

---

## Summary

PQAA defines a governed migration layer for platform-native integrity and attestation signals.

It standardizes the ingestion and normalization of non-PQ-native platform evidence into canonical `ReceiptEnvelope` artefacts consumable by PQSEC. PQAA operates entirely within the Holder Execution Boundary and functions solely as an Evidence Producer under PQSF governance.

PQAA:

* Collects platform-native attestation material.
* Emits deterministic CBOR evidence artefacts.
* Binds evidence to `session_id`, `decision_id`, `intent_hash`, and tick windows.
* Classifies translated material as `platform_bridged`.
* Defaults to hash-only, privacy-preserving output.
* Enforces manifest-bound admission and fail-closed re-admission on update.

PQAA does not evaluate predicates, grant authority, or modify enforcement semantics. All enforcement decisions remain exclusively within PQSEC.

High-assurance deployments SHOULD prefer native PQ evidence producers where available.

---

## Dependencies

| Component   | Minimum Version | Purpose                                                                                           |
| ----------- | --------------- | ------------------------------------------------------------------------------------------------- |
| PQSF        | ≥ 2.0.3         | Canonical CBOR (§7), ReceiptEnvelope, EvidenceDescriptor (§32B), Annex AC governance              |
| PQSEC       | ≥ 2.0.3         | Enforcement consumption and policy gating; Annex AX admission discipline (deployment integration) |
| Epoch Clock | ≥ 2.0.0         | Tick binding (`issued_tick`, `expiry_tick`) for freshness                                         |

PQAA introduces no new enforcement primitives.

---

## 1. Purpose

PQAA acts as a migration compatibility layer for environments where operating systems and hardware vendors do not yet emit PQ-native evidence.

PQAA exists for two reasons:

1. To provide governed, deterministic platform integrity evidence where no PQ-native evidence producers exist.
2. To prevent proliferation of ungoverned, application-specific platform attestation shims that would fragment the evidence surface and undermine deterministic evaluation.

PQAA:

* Discovers platform-native attestation sources.
* Collects platform-native attestation material.
* Normalizes it into deterministic CBOR artefacts.
* Emits signed ReceiptEnvelope artefacts.
* Operates entirely within the Holder Execution Boundary.

PQAA does not:

* Evaluate enforcement predicates.
* Produce EnforcementOutcome artefacts.
* Grant authority.
* Implement degraded allow logic.

All enforcement decisions remain exclusively within PQSEC.

---

## 2. Architectural Position

```
Application → PQAA (local) → PQSEC (evaluation)
```

PQAA bridges platform-native sources (TPM, Secure Enclave, Android Keystore, OS integrity frameworks) into canonical ReceiptEnvelope artefacts.

PQAA MUST run inside the Holder Execution Boundary.

---

## 3. Threat and Trust Notes (Normative)

### 3.1 Host Integrity Dependency

PQAA does not independently verify the correctness of platform-native attestations. Policy MUST treat `platform_bridged` evidence as contingent on the integrity of the host operating system and platform APIs.

### 3.2 Intent Binding Security Property

Binding PQAA evidence to `intent_hash` and `decision_id` ensures that attestation material cannot be rebound to a different operation intent or enforcement attempt. This provides intent-specific platform co-signing even when the platform does not natively emit PQ-canonical evidence.

---

## 4. Admission and Update Discipline (Normative)

### 4.1 Manifest-Bound Installation

Installation or activation of PQAA MUST be manifest-bound and treated as an Authoritative operation.

```
PQAA_MANIFEST = {
  v:                     1,
  adapter_id:            tstr,
  adapter_version:       tstr,
  binary_hash:           bstr(32),
  update_authority_key:  bstr(32),
  source_map:            [+ SourceRule],
  issued_tick:           uint,
  expiry_tick:           uint,
  signature:             bstr
}

SourceRule = {
  source_id:             tstr,
  max_payload_bytes:     uint,
  allowed_operation_classes: [+ tstr],
  source_strength_hint:  "hardware_attested" / "software_measured" / "platform_reported"
}
```

Rules:

1. The manifest MUST be canonical CBOR.
2. The manifest MUST be signed by the update authority corresponding to `update_authority_key`.
3. The deployment MUST pin `update_authority_key` in policy.
4. `expiry_tick` MUST be strictly greater than `issued_tick`.
5. `source_map` MUST NOT exceed 16 entries.
6. Each `source_id` MUST be unique.
7. Each `max_payload_bytes` MUST be non-zero and MUST NOT exceed 65536 bytes.

### 4.2 Fail-Closed Re-admission on Update

If the PQAA binary hash changes or the manifest expands its scope, PQAA MUST undergo re-admission before evidence is accepted.

Until re-admission, PQAA evidence MUST be refused.

### 4.3 Update Authority Rotation (Normative)

Rotation of `update_authority_key` MUST be treated as an Authoritative operation and MUST be policy-approved.

If the pinned update authority is revoked by governance, all PQAA manifests signed under the revoked authority MUST be treated as invalid.

---

## 5. Evidence Producer Governance (Normative)

PQAA MUST register under PQSF Annex AC with:

```
producer_kind: "attestation_adapter"
```

All PQAA receipts MUST include:

* `producer_id`
* `producer_build_hash`
* canonical encoding
* signature under a holder-controlled key

### 5.1 Hardware-Backed Signing Key (Normative)

If the platform supports a hardware-backed non-exportable signing key, PQAA MUST use such a key for signing receipts.

PQAA MUST declare whether its signing key is hardware-backed in each bundle via `signing_key_class`.

---

## 6. Local API

### 6.1 Source Discovery (Informative)

```
SourceDescriptor = {
  source_id: tstr,
  category:  tstr,
  availability: "AVAILABLE" / "UNAVAILABLE"
}
```

Discovery output is informative only.

### 6.2 Evidence Request (Normative)

```
PQAARequest = {
  v:                1,
  session_id:       bstr(16),
  decision_id:      bstr(16),
  intent_hash:      bstr(32),
  operation_class:  tstr,
  requested_sources:[* tstr] / null,
  request_nonce:    bstr(16),
  issued_tick:      uint,
  expiry_tick:      uint,
  exporter_hash:    bstr(32) / null
}
```

Rules:

1. `issued_tick` and `expiry_tick` MUST use Epoch Clock ticks.
2. `expiry_tick` MUST be strictly greater than `issued_tick`.
3. `request_nonce` MUST be replay-protected locally.
4. Requests with `current_tick >= expiry_tick` MUST be refused.
5. All emitted evidence MUST bind to `session_id`, `decision_id`, `intent_hash`, and the tick window.

---

## 7. Evidence Output (Normative)

### 7.1 Receipt Type

```
ReceiptEnvelope.type = "pqaa.attestation_bundle"
```

### 7.2 Bundle Body

```
AttestationBundleBody = {
  v:                    1,
  session_id:           bstr(16),
  decision_id:          bstr(16),
  intent_hash:          bstr(32),
  request_nonce_hash:   bstr(32),
  issued_tick:          uint,
  expiry_tick:          uint,
  exporter_hash:        bstr(32) / null,
  producer_id:          tstr,
  producer_build_hash:  bstr(32),
  signing_key_class:    "hardware_backed" / "software_only",
  sources:              [+ SourceEvidence],
  signature:            bstr
}
```

Rules:

1. `request_nonce_hash` MUST equal `SHAKE256-256(request_nonce || u64be(issued_tick))`.
2. The `sources` array MUST be sorted lexicographically by `source_id`.

### 7.3 SourceEvidence

```
SourceEvidence = {
  source_id:            tstr,
  collection_status:    "collected" / "unavailable" / "error",
  claim_hash:           bstr(32),
  claim_summary:        tstr,
  source_version:       tstr / null,
  source_strength_hint: "hardware_attested" / "software_measured" / "platform_reported",
  reason_code:          tstr / null,
  evidence_descriptor:  EvidenceDescriptor
}
```

Rules:

1. `collection_status` describes harvesting outcome only.
2. If a source is harvested successfully, `collection_status` MUST be `"collected"` even if the platform-reported state indicates an insecure configuration.
3. `claim_hash` MUST equal `SHAKE256-256(DetCBOR(claim_object))`.
4. `claim_summary` MUST NOT include unique device identifiers.
5. `source_version` is informative and MUST NOT be evaluated by PQAA.
6. `evidence_descriptor` MUST include:

   * `producer_id`
   * `producer_build_hash`
   * `issued_tick`
   * `evidence_class = "platform_bridged"`
7. Raw platform artefacts MUST NOT be emitted unless explicitly permitted by policy.

---

## 8. Evidence Classification (Normative)

Add to PQSF §32B.2 Evidence Classes:

| Class              | Meaning                                                                                                                    |
| ------------------ | -------------------------------------------------------------------------------------------------------------------------- |
| `platform_bridged` | Evidence translated from a platform-native attestation API by a governed adapter. The original artefact was not PQ-native. |

PQAA MUST set:

```
EvidenceDescriptor.evidence_class = "platform_bridged"
```

---

## 9. Privacy Discipline (Normative)

1. Hash-only evidence by default.
2. Strict per-source size caps.
3. Immediate destruction of raw harvested bytes unless policy permits operation-scoped retention.
4. No cross-operation correlation.

---

## 10. Policy Interaction (Normative)

Policy determines:

* Whether PQAA evidence is admissible.
* Which `source_id` values are acceptable.
* Minimum acceptable `source_strength_hint`.
* Whether raw payload references are permitted.
* Any refusal rules based on `source_version`.

If required evidence is unavailable, consuming enforcement MUST fail closed for Authoritative operations.

### 10.1 Native Preference (Normative)

Where native PQ evidence producers are registered and available for the same predicate and operation class, policy SHOULD prefer native evidence over `platform_bridged` evidence.

---

## 11. Conformance (Normative)

PQAA is conformant if it:

1. Operates inside the Holder Execution Boundary.
2. Emits canonical ReceiptEnvelope artefacts.
3. Signs bundles under a holder-controlled key and declares signing key class.
4. Enforces request replay protection and binds `request_nonce_hash` to `issued_tick`.
5. Enforces manifest-bound admission and fail-closed re-admission.
6. Enforces size caps and privacy defaults.
7. Sorts sources deterministically by `source_id`.

---

## Annex A — Source ID Convention (Informative)

To reduce collisions across PQAA implementations, `source_id` SHOULD use a dotted prefix convention:

* `hw.tpm.*`
* `hw.secure_enclave.*`
* `os.android.*`
* `os.ios.*`
* `os.windows.*`
* `os.linux.*`

This convention is advisory and does not create a namespace authority surface.

---

## Annex B — Reason Code Vocabulary (Informative)

Recommended `reason_code` values:

* `E_PQAA_SOURCE_LOCKED`
* `E_PQAA_PAYLOAD_CAP`
* `E_PQAA_BRIDGE_FAILURE`
* `E_PQAA_NONCE_REPLAY`
* `E_PQAA_TICK_STALE`

These are descriptive diagnostics only and do not constitute PQSEC refusal codes.

---

## Funding (Non-Normative)

If you find this work useful and want to support continued development:

**Bitcoin:**
bc1q380874ggwuavgldrsyqzzn9zmvvldkrs8aygkw
