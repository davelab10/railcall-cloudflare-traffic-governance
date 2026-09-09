# Command reference

This is the frozen 20-command surface declared in `module/module.json`. The
public marketplace identity is `dave/cloudflare-traffic-governance`; every
command's provider metadata resolves through
`dave-cloudflare-traffic-governance::cloudflare`.

Related reading: [README.md](README.md) for the trust boundary and setup, and
[TESTING.md](TESTING.md) for evidence levels and acceptance results.

Examples are sanitized shapes, not live credentials or production instructions.
Replace `ZONE_ID`, `RECORD_ID`, and state values only with exact values from a
fresh read and an explicitly bounded plan.

## Shared response contract

Every handler response begins with:

```json
{
  "status": "OK",
  "command": "cloudflare.traffic.<name>",
  "provider": "cloudflare",
  "as_of": "2026-09-10T00:00:00Z"
}
```

Fields are command-specific. When applicable, outputs also include:

- `complete`, `truncated`, `next_cursor`, `limit`, and `result_count` for
  bounded paginated reads;
- `plan_hash`, `expected_state_hash`, `state_hash`, and `exact_action_hash` for
  deterministic binding;
- `approval_state`, `quorum_state`, `execution_state`,
  `verification_state`, and `reconciliation_state` where that stage exists;
- `idempotency_reference` for governed writes;
- `receipt_reference` when the native Station wrapper supplies one.

`next_cursor` is the handler's name for the provider's next page number. A
non-complete result must not be described as a complete inventory.

Common bounds in `handler.py`:

- `limit`: integer 1–100; default 50;
- `page`: integer 1–10000; default 1;
- explicit target count: 1–50;
- risk policy batch cap: 25 targets;
- strings: maximum 4096 characters;
- supported record types: `A`, `AAAA`, `CNAME`, `TXT`, `MX`, `NS`, `SRV`,
  `CAA`, `PTR`.

Native Station adds the outer signed receipt, credential resolution, approval,
policy, identity, and provenance envelope. The handler does not print raw
credentials.

## Discovery / preflight

### 1. `cloudflare.traffic.account_preflight`

- **Purpose:** Check whether the configured Cloudflare credential can reach at
  least one accessible zone and report observed read readiness.
- **Authority/risk:** Read-only; LOW; no approval or quorum; provider effect
  `none`.
- **Inputs:** Manifest has no command input fields. Credential is resolved
  natively from the `cloudflare` slot. Handler defaults to `{}`.
- **Outputs:** `ready`, `token_ready`, `token_valid`, `accessible_zone`,
  `read_capability`, `write_capability`, `write_capability_verified`,
  `blocking_reason`, and `write_capability_note`.
- **Failure modes:** `NOT_READY`, `AUTH_ERROR`, provider/transport failure;
  write capability is not safely probed by this read command.
- **Example invocation:** `{}`
- **Response shape:**

```json
{"status":"OK","ready":true,"token_ready":true,"token_valid":true,"accessible_zone":true,"read_capability":true,"write_capability":null,"write_capability_verified":false}
```

### 2. `cloudflare.traffic.zone_list`

- **Purpose:** Bounded Cloudflare zone discovery.
- **Authority/risk:** Read-only; LOW; no approval/quorum; provider effect
  `read` (`GET /zones`).
- **Inputs:** Required: none. Optional: `limit` integer 1–100, `page` integer
  1–10000, `name` string.
- **Outputs:** `zones`, `limit`, `next_cursor`, `complete`, `truncated`,
  `result_count`, `truncation_disclosure`.
- **Failure modes:** `INVALID_INPUT`, `INPUT_CAP_EXCEEDED`, provider/auth
  error. `truncated=true` when the provider reports another page.
- **Example invocation:** `{"name":"davelabs.my.id","limit":50,"page":1}`
- **Response shape:**

```json
{"status":"OK","zones":[{"id":"ZONE_ID","name":"davelabs.my.id","status":"active"}],"complete":true,"truncated":false,"next_cursor":null,"result_count":1}
```

### 3. `cloudflare.traffic.zone_get`

- **Purpose:** Read one exact zone.
- **Authority/risk:** Read-only; LOW; no approval/quorum; provider effect
  `read` (`GET /zones/{zone_id}`).
- **Inputs:** Required: `zone_id` string. No optional fields.
- **Outputs:** `zone`, `zone_id`.
- **Failure modes:** `INVALID_INPUT`, provider/auth/zone-not-found error; no
  fallback zone selection.
- **Example invocation:** `{"zone_id":"ZONE_ID"}`
- **Response shape:**

```json
{"status":"OK","zone_id":"ZONE_ID","zone":{"id":"ZONE_ID","name":"davelabs.my.id","status":"active"}}
```

### 4. `cloudflare.traffic.dns_record_list`

- **Purpose:** List bounded records in one exact zone.
- **Authority/risk:** Read-only; LOW; no approval/quorum; provider effect
  `read` (`GET /zones/{zone_id}/dns_records`).
- **Inputs:** Required: `zone_id` string. Optional: `limit` 1–100, `page`
  1–10000, `name` string, `type` one supported DNS type.
- **Outputs:** `records`, `zone_id`, `limit`, `next_cursor`, `complete`,
  `truncated`, `result_count`, `truncation_disclosure`.
- **Failure modes:** `INVALID_INPUT`, `UNSUPPORTED_ACTION` for an unsupported
  type, provider/auth error, incomplete page disclosure.
- **Example invocation:** `{"zone_id":"ZONE_ID","name":"railcall-e2e-a.davelabs.my.id","type":"A","limit":100,"page":1}`
- **Response shape:**

```json
{"status":"OK","zone_id":"ZONE_ID","records":[{"record_id":"RECORD_ID","zone_id":"ZONE_ID","type":"A","name":"railcall-e2e-a.davelabs.my.id","content":"192.0.2.10","ttl":120,"proxied":false,"present":true}],"complete":true,"truncated":false,"result_count":1}
```

### 5. `cloudflare.traffic.dns_record_get`

- **Purpose:** Read one exact record or prove authoritative absence.
- **Authority/risk:** Read-only; LOW; no approval/quorum; provider effect
  `read` (`GET /zones/{zone_id}/dns_records/{record_id}`).
- **Inputs:** Required: `zone_id`, `record_id` strings.
- **Outputs:** `record`, `complete`; on exact 404, `status=NOT_FOUND`,
  `record=null`, `absence_authoritative=true`, `record_id`, `zone_id`.
- **Failure modes:** `INVALID_INPUT`, provider/auth error, malformed provider
  record. 404 is an explicit absence, not a generic success.
- **Example invocation:** `{"zone_id":"ZONE_ID","record_id":"RECORD_ID"}`
- **Response shape:** `{"status":"NOT_FOUND","record":null,"absence_authoritative":true,"complete":true}`

## State / analysis

### 6. `cloudflare.traffic.traffic_state_inspect`

- **Purpose:** Read and canonicalize an explicit bounded target set.
- **Authority/risk:** Read-only; LOW; no approval/quorum; provider effect
  `read`.
- **Inputs:** Required: `targets` non-empty array, maximum 50. Each target
  requires `zone_id` and either `record_id` or `name`; optional `type`.
- **Outputs:** `canonical_state`, `state_hash`, `fingerprint`,
  `observed_timestamp`, `target_count`, `blast_radius`, `complete`,
  `truncated`.
- **Failure modes:** `UNBOUNDED_TARGET_SET`, duplicate target,
  `AMBIGUOUS_TARGET`, provider error, incomplete exact read.
- **Example invocation:** `{"targets":[{"zone_id":"ZONE_ID","record_id":"RECORD_ID","name":"railcall-e2e-a.davelabs.my.id","type":"A"}]}`
- **Response shape:** `{"status":"OK","canonical_state":[{"zone_id":"ZONE_ID","record_id":"RECORD_ID","type":"A","name":"railcall-e2e-a.davelabs.my.id"}],"state_hash":"sha256:...","target_count":1,"blast_radius":{"record_count":1,"hostnames":["railcall-e2e-a.davelabs.my.id"],"bounded":true},"complete":true}`

### 7. `cloudflare.traffic.dns_conflict_check`

- **Purpose:** Reject structural conflicts in a proposed DNS state before a
  plan is created.
- **Authority/risk:** Read/local analysis; MEDIUM; no approval/quorum; provider
  effect `none`.
- **Inputs:** Required: `proposed_state` non-empty array. Fields are state
  objects; supported types are the module allowlist.
- **Outputs:** `valid`, `conflicts`, `proposal_hash`.
- **Failure modes:** `INVALID_INPUT`, `UNSUPPORTED_ACTION`, `status=CONFLICT`.
  A CNAME mixed with another record type for the same name is rejected.
- **Example invocation:** `{"proposed_state":[{"name":"railcall-e2e-a.davelabs.my.id","type":"A","content":"198.51.100.44"}]}`
- **Response shape:** `{"status":"VALID","valid":true,"conflicts":[],"proposal_hash":"sha256:..."}`

### 8. `cloudflare.traffic.change_plan`

- **Purpose:** Build a deterministic, non-executable exact plan.
- **Authority/risk:** Local planning; LOW; no approval/quorum; provider effect
  `none`.
- **Inputs:** Required: `intent` string, `targets` array, `old_state` array,
  `proposed_state` array. All three arrays must have the same target count.
- **Outputs:** `plan`, `plan_id`, `plan_hash`, `expected_state_hash`,
  `completeness`. The plan includes `ordered_actions`, `state_hash`, and
  `action_hash`.
- **Failure modes:** `INVALID_INPUT`, duplicate/unbounded target,
  `UNSUPPORTED_ACTION` for structural DNS conflict.
- **Example invocation:** `{"intent":"move disposable traffic","targets":[{"zone_id":"ZONE_ID","record_id":"RECORD_ID","name":"railcall-e2e-a.davelabs.my.id","type":"A"}],"old_state":[{"zone_id":"ZONE_ID","record_id":"RECORD_ID","type":"A","name":"railcall-e2e-a.davelabs.my.id","content":"192.0.2.10","ttl":120,"proxied":false}],"proposed_state":[{"zone_id":"ZONE_ID","record_id":"RECORD_ID","type":"A","name":"railcall-e2e-a.davelabs.my.id","content":"198.51.100.44","ttl":120,"proxied":false,"operation":"update"}]}`
- **Response shape:** `{"status":"OK","plan_id":"plan_...","plan_hash":"sha256:...","expected_state_hash":"sha256:...","completeness":{"complete":true,"target_count":1},"plan":{"ordered_actions":[...]}}`

### 9. `cloudflare.traffic.change_risk_assess`

- **Purpose:** Classify the exact plan and report blast radius and approval
  requirement.
- **Authority/risk:** Local analysis; MEDIUM; no approval/quorum to assess;
  provider effect `none`.
- **Inputs:** Required: `plan` object. Optional: `required_quorum` integer.
- **Outputs:** `risk`, `affected_record_count`, `affected_hostnames`,
  `destructive`, `blast_radius`, `required_approval`, `required_quorum`,
  `policy`.
- **Behavior:** `CRITICAL` for delete or more than 10 targets; `HIGH` for a
  non-empty non-destructive plan; `LOW` for an empty plan. A count above 25 is
  `POLICY_BLOCKED`.
- **Failure modes:** `INVALID_INPUT`, `POLICY_BLOCKED`.
- **Example invocation:** `{"plan":{"targets":[{"name":"railcall-e2e-a.davelabs.my.id"}],"ordered_actions":[{"operation":"update"}]},"required_quorum":1}`
- **Response shape:** `{"status":"OK","risk":"HIGH","affected_record_count":1,"destructive":false,"required_approval":true,"required_quorum":1,"policy":{"decision":"allow","ai_advisory_only":true}}`

### 10. `cloudflare.traffic.change_preview`

- **Purpose:** Produce the exact human approval surface.
- **Authority/risk:** Read/approval preparation; MEDIUM; provider effect `none`.
- **Inputs:** Required: `plan`, `plan_hash`. Optional: `risk`,
  `approval_expiry` string. `plan_hash` must match the canonical plan.
- **Outputs:** `status=AWAITING_APPROVAL`, `old_state`, `new_state`,
  `exact_targets`, `affected_hostnames`, `risk`, `blast_radius`, `plan_hash`,
  `expected_state_hash`, `required_quorum`, `exact_action_hash`,
  `approval_expiry`, `approval_state=AWAITING_APPROVAL`.
- **Failure modes:** `PLAN_MISMATCH`, invalid plan, incomplete targets.
  Optional `approval_expiry` is a preview field; native expiry was not proven
  in Step 2.
- **Example invocation:** `{"plan":{"plan_hash":"sha256:..."},"plan_hash":"sha256:...","risk":{"risk":"HIGH","blast_radius":{"record_count":1}}}`
- **Response shape:** `{"status":"AWAITING_APPROVAL","approval_state":"AWAITING_APPROVAL","plan_hash":"sha256:...","expected_state_hash":"sha256:...","required_quorum":1,"exact_action_hash":"sha256:..."}`

### 11. `cloudflare.traffic.precondition_check`

- **Purpose:** Reread every target immediately before mutation.
- **Authority/risk:** Read/gate; MEDIUM; no provider mutation.
- **Inputs:** Required: `plan`, `plan_hash`, `authority` object. Authority must
  include `approved=true` and matching `plan_hash`; if quorum is required,
  `quorum_satisfied=true` is required.
- **Outputs:** `expected_state_hash`, `live_state_hash`, `mismatches`,
  `mutation_count=0`, `decision` (`REFUSE` or `MAY_EXECUTE`).
- **Failure modes:** `HUMAN_APPROVAL_REQUIRED`, `AUTHORITY_INVALID`,
  `AWAITING_TEAM_APPROVAL`, `PLAN_MISMATCH`, `STALE_STATE`, provider error.
- **Example invocation:** `{"plan":{"plan_hash":"sha256:...","targets":[...]},"plan_hash":"sha256:...","authority":{"approved":true,"plan_hash":"sha256:..."}}`
- **Response shape:** `{"status":"STALE_STATE","expected_state_hash":"sha256:...","live_state_hash":"sha256:...","mismatches":[{"index":0}],"mutation_count":0,"decision":"REFUSE"}`

## Governed mutations

For commands 12–15, the manifest requires `plan`, `plan_hash`, and `authority`.
The plan must be canonical and the authority must match its hash. Native
Station is the outer approval/receipt authority. The examples below use
placeholder plan data and never include a credential.

### 12. `cloudflare.traffic.dns_record_create`

- **Purpose:** Create one exact planned record.
- **Authority/risk:** Write; HIGH; human approval required; policy quorum may
  apply; provider effect `POST /zones/{zone_id}/dns_records`.
- **Inputs:** Required: `plan`, `plan_hash`, `authority`; plan contains one
  `operation=create` action and expected absence.
- **Outputs:** `provider_accepted`, `provider_http_status`, `execution_state`,
  `idempotency_reference`, `expected`, `actual`, `mismatches`,
  `verification_state`, `receipt_reference`.
- **Failure modes:** `HUMAN_APPROVAL_REQUIRED`, `AUTHORITY_INVALID`,
  `PLAN_MISMATCH`, `STALE_STATE`, `UNKNOWN`,
  `EFFECT_LANDED_EVIDENCE_INCOMPLETE`, `NOT_RECONCILED`.
- **Example invocation:** `{"plan":{"plan_hash":"sha256:...","ordered_actions":[{"operation":"create","target":{"zone_id":"ZONE_ID","name":"railcall-e2e-a.davelabs.my.id","type":"A"},"old":{"present":false},"new":{"zone_id":"ZONE_ID","record_id":"","type":"A","name":"railcall-e2e-a.davelabs.my.id","content":"192.0.2.10","ttl":120,"proxied":false}}]},"plan_hash":"sha256:...","authority":{"approved":true,"plan_hash":"sha256:..."}}`
- **Response shape:** `{"status":"EXECUTED_AND_VERIFIED","provider_accepted":true,"provider_http_status":200,"execution_state":"EXECUTED","verification_state":"VERIFIED","idempotency_reference":"sha256:...","receipt_reference":null}`

### 13. `cloudflare.traffic.dns_record_update`

- **Purpose:** Update one exact record from a complete pinned old state to an
  exact new state.
- **Authority/risk:** Write; HIGH; human approval/policy quorum; provider
  effect `PATCH /zones/{zone_id}/dns_records/{record_id}`.
- **Inputs:** Required: `plan`, `plan_hash`, `authority`; one exact update
  action with material old/new state.
- **Outputs:** Same governed single-write fields as create: provider result,
  idempotency reference, expected/actual state, mismatches, verification, and
  receipt reference.
- **Failure modes:** Same approval/plan/stale failures; timeout returns
  `UNKNOWN` with `retry_allowed=false`; a later read decides whether the effect
  landed.
- **Example invocation:** `{"plan":{"plan_hash":"sha256:...","ordered_actions":[{"operation":"update","target":{"zone_id":"ZONE_ID","record_id":"RECORD_ID","name":"railcall-e2e-a.davelabs.my.id"},"old":{...},"new":{...}}]},"plan_hash":"sha256:...","authority":{"approved":true,"plan_hash":"sha256:..."}}`
- **Response shape:** `{"status":"EXECUTED_AND_VERIFIED","provider_accepted":true,"provider_http_status":200,"execution_state":"EXECUTED","verification_state":"VERIFIED","mismatches":[]}`

### 14. `cloudflare.traffic.dns_record_delete`

- **Purpose:** Delete one exact record while preserving the complete prior
  snapshot for verification/recovery planning.
- **Authority/risk:** Write; CRITICAL; human approval and policy quorum;
  provider effect `DELETE /zones/{zone_id}/dns_records/{record_id}`.
- **Inputs:** Required: `plan`, `plan_hash`, `authority`; one exact delete
  action with complete old state and expected absence.
- **Outputs:** Governed write fields plus `actual` absence and verification
  state.
- **Failure modes:** Approval/plan/stale failures; `UNKNOWN` on provider
  ambiguity; deletion is never silently retried.
- **Example invocation:** `{"plan":{"plan_hash":"sha256:...","ordered_actions":[{"operation":"delete","target":{"zone_id":"ZONE_ID","record_id":"RECORD_ID","name":"railcall-e2e-a.davelabs.my.id"},"old":{...},"new":{"present":false}}]},"plan_hash":"sha256:...","authority":{"approved":true,"plan_hash":"sha256:..."}}`
- **Response shape:** `{"status":"EXECUTED_AND_VERIFIED","provider_accepted":true,"provider_http_status":200,"execution_state":"EXECUTED","actual":[{"present":false}],"verification_state":"VERIFIED"}`

### 15. `cloudflare.traffic.dns_batch_change`

- **Purpose:** Apply one bounded ordered set after checking every target before
  the first mutation.
- **Authority/risk:** Write; HIGH in the manifest; CRITICAL behavior may be
  assigned by risk policy for destructive/high-blast plans; approval/quorum
  required; provider effects are bounded POST/PATCH/DELETE calls.
- **Inputs:** Required: `plan`, `plan_hash`, `authority`; plan target count is
  explicit and bounded, with ordered actions and complete old/proposed state.
- **Outputs:** `execution_state`, `mutations_executed`, `mutation_count`,
  `provider_http_statuses`, `expected`, `actual`, `mismatches`,
  `verification_state`, `idempotency_reference`.
- **Failure modes:** `BATCH_PRECONDITION_FAILED` with `mutations_executed=0`
  and message `0/N mutations executed`; approval/plan failure; `UNKNOWN` after
  a provider ambiguity with count of completed actions; `NOT_RECONCILED` on
  post-state mismatch.
- **Example invocation:** `{"plan":{"plan_hash":"sha256:...","targets":[...],"old_state":[...],"proposed_state":[...],"ordered_actions":[...]},"plan_hash":"sha256:...","authority":{"approved":true,"plan_hash":"sha256:..."}}`
- **Response shape:** `{"status":"EXECUTED_AND_VERIFIED","execution_state":"EXECUTED","mutations_executed":3,"mutation_count":3,"provider_http_statuses":[200,200,200],"verification_state":"VERIFIED","mismatches":[]}`

## Verification / recovery

### 16. `cloudflare.traffic.change_verify`

- **Purpose:** Compare authoritative provider state with expected post-state.
- **Authority/risk:** Read/verification; MEDIUM; no approval/quorum; provider
  effect `read` through exact record reads.
- **Inputs:** Required: `targets` array and `expected_state` array. Optional:
  `old_state` array.
- **Outputs:** `expected`, `actual`, `mismatches`, `verified`; status is
  `VERIFIED` or `NOT_RECONCILED`.
- **Failure modes:** `INVALID_INPUT`, provider error, `NOT_RECONCILED`; 2xx is
  not sufficient.
- **Example invocation:** `{"targets":[{"zone_id":"ZONE_ID","record_id":"RECORD_ID","name":"railcall-e2e-a.davelabs.my.id","type":"A"}],"expected_state":[{"zone_id":"ZONE_ID","record_id":"RECORD_ID","type":"A","name":"railcall-e2e-a.davelabs.my.id","content":"198.51.100.44"}]}`
- **Response shape:** `{"status":"VERIFIED","expected":[...],"actual":[...],"mismatches":[],"verified":true}`

### 17. `cloudflare.traffic.change_reconcile`

- **Purpose:** Compare planned, approved, executed, actual, approval, and
  receipt evidence.
- **Authority/risk:** Local evidence analysis; MEDIUM; no approval/quorum;
  provider effect `none` for supplied evidence.
- **Inputs:** Optional: `planned`, `approved`, `executed`, `actual` arrays and
  `evidence` object. Missing values are reported, not inferred.
- **Outputs:** `planned`, `approved`, `executed`,
  `authoritative_provider_post_state`, `approval_evidence`, `quorum_evidence`,
  `receipt_evidence`, `missing_evidence`.
- **Failure modes:** `UNKNOWN` when required evidence is missing;
  `NOT_RECONCILED` when approved differs from planned, evidence is missing, or
  executed differs from actual.
- **Example invocation:** `{"planned":[...],"approved":[...],"executed":[...],"actual":[...],"evidence":{"approval":true,"receipt":true,"quorum":null}}`
- **Response shape:** `{"status":"RECONCILED","missing_evidence":[],"approval_evidence":true,"receipt_evidence":true,"authoritative_provider_post_state":[...]}`

### 18. `cloudflare.traffic.rollback_plan`

- **Purpose:** Plan a new restore action from current authoritative state to a
  saved original state.
- **Authority/risk:** Read/planning; HIGH in manifest; no provider mutation;
  fresh approval is required for later execution.
- **Inputs:** Required: `original_pre_state`, `current_authoritative_state`,
  `targets` arrays. Optional: `original_receipt_reference`, `required_quorum`.
  Original and current arrays must have equal length.
- **Outputs:** `plan`, `rollback_risk=CRITICAL`, `required_approval=true`,
  `required_quorum` (default 2), `current_state_hash`, `rollback_plan_hash`.
- **Failure modes:** `INVALID_INPUT`, count mismatch, unsupported/unbounded
  rollback target.
- **Example invocation:** `{"original_pre_state":[...],"current_authoritative_state":[...],"targets":[{"zone_id":"ZONE_ID","record_id":"RECORD_ID","name":"railcall-e2e-a.davelabs.my.id","type":"A"}],"original_receipt_reference":"receipt-ref","required_quorum":2}`
- **Response shape:** `{"status":"OK","rollback_risk":"CRITICAL","required_approval":true,"required_quorum":2,"current_state_hash":"sha256:...","rollback_plan_hash":"sha256:...","plan":{"intent":"restore prior state"}}`

### 19. `cloudflare.traffic.rollback_execute`

- **Purpose:** Execute a fresh approved rollback plan through the bounded batch
  mutation path.
- **Authority/risk:** Write; CRITICAL; fresh human approval/quorum required;
  provider effect is bounded POST/PATCH/DELETE according to the rollback plan.
- **Inputs:** Required: `plan`, `plan_hash`, `authority`; same exact plan and
  authority contract as `dns_batch_change`.
- **Outputs:** The current implementation delegates directly to
  `dns_batch_change`, so mutation output fields are batch fields:
  `execution_state`, `mutations_executed`, `mutation_count`, provider statuses,
  expected/actual, mismatches, verification, and idempotency reference.
- **Failure modes:** Same as bounded batch: approval/plan/stale refusal,
  `BATCH_PRECONDITION_FAILED`, `UNKNOWN`, `NOT_RECONCILED`.
- **Example invocation:** `{"plan":{"plan_hash":"sha256:...","targets":[...],"old_state":[...],"proposed_state":[...],"ordered_actions":[...]},"plan_hash":"sha256:...","authority":{"approved":true,"plan_hash":"sha256:..."}}`
- **Response shape:** `{"status":"EXECUTED_AND_VERIFIED","command":"cloudflare.traffic.dns_batch_change","execution_state":"EXECUTED","mutations_executed":3,"verification_state":"VERIFIED"}`
- **Runtime note:** The manifest command is `rollback_execute`, but the
  handler's delegated response currently carries
  `command=cloudflare.traffic.dns_batch_change`. Native Station invocation and
  receipt identity remain the authoritative outer command identity. This is a
  documented implementation mismatch, not a reason to rename the module or
  command.

### 20. `cloudflare.traffic.change_prove`

- **Purpose:** Return a human-readable proof/evidence surface without changing
  provider state.
- **Authority/risk:** Read/proof; MEDIUM; no approval/quorum; provider effect
  `none` for supplied evidence.
- **Inputs:** All optional in the manifest: `operation_identity`, `targets`,
  `prior_state`, `desired_state`, `plan_hash`, `state_hash`, `approval_chain`,
  `team_quorum`, `execution_evidence`, `verification`, `reconciliation`,
  `receipt_signature_integrity`, `offline_verification`.
- **Outputs:** Echoed evidence fields plus `limitations`. Defaults are
  `receipt_signature_integrity=not_verified_by_module` and
  `offline_verification=unavailable` when omitted.
- **Failure modes:** This handler does not independently cryptographically
  verify a receipt; native Station remains the authority. Missing/weak evidence
  must remain visible in output.
- **Example invocation:** `{"operation_identity":{"id":"operation-ref"},"targets":[...],"plan_hash":"sha256:...","verification":{"verified":true},"reconciliation":{"status":"RECONCILED"},"receipt_signature_integrity":"verified_by_station","offline_verification":"PASS"}`
- **Response shape:** `{"status":"OK","operation_identity":{"id":"operation-ref"},"plan_hash":"sha256:...","verification":{"verified":true},"reconciliation":{"status":"RECONCILED"},"receipt_signature_integrity":"verified_by_station","offline_verification":"PASS","limitations":["Station remains the authority for native receipt and quorum cryptographic verification","Provider 2xx is not authoritative post-state proof"]}`

## Authority and provider-effect summary

| Group | Commands | Provider effect | Approval |
|---|---|---|---|
| Discovery | 1–5 | Read or no provider effect | None |
| Analysis | 6–10 | Read/local/no effect | None; preview produces `AWAITING_APPROVAL` |
| Gate | 11 | Authoritative read, no mutation | Matching authority required |
| Mutation | 12–15 | Cloudflare write | Exact plan, human approval, and policy/quorum as required |
| Verification/recovery | 16–20 | Read/local, except rollback execute | Rollback execute requires fresh authority |

## Completeness and evidence notes

- Read pagination is explicit; `complete=false` or `truncated=true` means the
  caller must not claim a complete inventory.
- Exact target resolution rejects ambiguity and duplicate targets.
- A planned create may omit the provider-assigned record ID; the post-read
  binds that ID without weakening other pinned fields.
- A delete is verified by authoritative absence.
- Native receipts may contain more outer provenance than the handler response;
  use the native receipt reference from Station for final audit evidence.
