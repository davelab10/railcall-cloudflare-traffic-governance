# Testing and evidence

This document separates what was inspected, mocked, exercised by the installed
RailCall runtime, proven against live Cloudflare, and verified offline. A
lower evidence level is never silently promoted to a higher one.

Related reading: [README.md](README.md) for the buyer-facing workflow and
[COMMANDS.md](COMMANDS.md) for the exact 20-command contracts.

## Test strategy

### Reviewer summary

| Evidence label | Meaning in this project |
|---|---|
| **PASS — LIVE PROVIDER** | Native RailCall reached the scoped Cloudflare provider and authoritative state was verified afterward. |
| **PASS — AUTOMATED / RUNTIME** | The handler regression suite or installed Station behavior passed; this is not live external-actor proof. |
| **NOT VERIFIED** | No defensible evidence was available in the audited environment. |

| Level | Meaning |
|---|---|
| STATIC | Manifest, handler, signed bundle, source, or contract inspection. |
| MOCK / UNIT | Deterministic handler test with a fake provider or fixture. |
| RUNTIME | Actual installed Station registry, validator, sandbox, governance, or command behavior. |
| LIVE PROVIDER | Native RailCall invocation reached the scoped Cloudflare provider and authoritative state was read after the operation. |
| OFFLINE RECEIPT VERIFY | A signed receipt was checked without contacting Cloudflare. |

The final Step 2 conclusion is intentionally mixed: bounded live Cloudflare
E2E is verified and cleaned up, while the full governance matrix remains
blocked for distinct Team approvers, approval expiry, and genuine external
out-of-band drift.

## Environment

- **Station baseline:** `station-v1.5.25`.
- **Module:** `dave/cloudflare-traffic-governance` version `0.1.0`.
- **Runtime integration:** `dave-cloudflare-traffic-governance::cloudflare`.
- **Credential method:** existing native RailCall Studio Integration credential;
  raw token was never read or printed.
- **Provider:** Cloudflare API host `api.cloudflare.com`.
- **Zone:** `davelabs.my.id`, zone ID `4d118d87fc91d996647673ae4233455c`.
- **Disposable scope:** only `railcall-e2e-a/b/c.davelabs.my.id`.
- **Documentation/test IPs:** `192.0.2.10`, `198.51.100.44`, and
  `203.0.113.77`.
- **Runtime sandbox:** network allowlist `api.cloudflare.com`, subprocess
  disabled, filesystem writes empty.
- **Final provider state:** all three disposable names absent after cleanup.

## Automated regression tests

Result: **19 passed, 0 failed** after the credential integration fix.

The test suite covers:

- deterministic canonical state and plan hashes;
- structural DNS conflict rejection;
- delete risk classification as `CRITICAL`;
- missing approval refusal;
- exact approved-payload tamper rejection;
- stale precondition with zero mutation;
- stale batch precondition with zero-of-N mutation;
- provider-assigned create record IDs;
- authoritative delete absence;
- exact record-not-found semantics;
- duplicate and unbounded target refusal;
- read caps and unsupported record types;
- verification and reconciliation refusal to fake green;
- timeout ambiguity with no blind retry;
- no arbitrary provider method/path escape hatch;
- all 20 handler functions present;
- marketplace slash ID preserved while provider uses normalized Station ID;
- normalized credential lookup;
- secret-free credential lookup output.

The tests are project regression evidence, not a substitute for live native
Team governance or external-actor evidence.

## Live provider evidence

### Step 2B read-only

Native RailCall receipts recorded:

- `account_preflight`: `ready=true`, `token_valid=true`,
  `read_capability=true`, provider was touched.
- `zone_list`: one complete active zone.
- `zone_get`: exact `davelabs.my.id` zone read.
- `dns_record_list`: complete final list, zero records after cleanup.
- `dns_record_get`: not run because no existing record ID was available.

The receipt references are recorded in `docs/STEP2_AUDIT.md`; they are not
reproduced here as raw receipt dumps.

### Step 2C bounded live E2E

The user-authorized live mutation scope was only A/B/C and only the three
documentation IPs. Native RailCall executed and then cleaned up:

- A/B/C create: HTTP 200 and `EXECUTED_AND_VERIFIED`;
- single-record update: live verified;
- bounded three-target batch: precondition passed and 3/3 post-state verified;
- exact approved replay: blocked by consumed approval with no provider touch;
- rollback: live verified for the three disposable names;
- A/B/C delete cleanup: HTTP 200 and `EXECUTED_AND_VERIFIED`;
- exact-name cleanup reads: all returned `result_count=0`.

The detailed receipt suffixes and audit references remain in
`docs/STEP2_AUDIT.md`. No other hostname, zone setting, nameserver, security
setting, or Cloudflare resource was mutated.

## Stale-state evidence

The handler and regression suite prove:

- a pinned material field drift, including TTL, returns `STALE_STATE`;
- a stale batch returns `BATCH_PRECONDITION_FAILED` and `0/N` mutations;
- all batch preconditions are checked before the first mutation.

The Step 2 audit did not claim genuine external Cloudflare actor evidence:

- S1 genuine out-of-band stale single: blocked;
- S2 pinned non-content external drift: blocked;
- S3 genuine live batch stale `0/3`: blocked;
- H12 stale rollback after external drift: blocked.

Those cases are not presented as live provider tests.

## Idempotency and replay

The handler emits deterministic idempotency references for governed writes and
does not blindly retry a timeout. The regression suite verifies timeout output
as `UNKNOWN` with `retry_allowed=false`.

Step 2 also live-exercised exact approved batch replay: the consumed approval
blocked the replay and `external_api_touched=false`. This is live evidence for
that native replay boundary, not a claim that every possible Station Team
policy transition was tested.

## Receipt verification

- Native RailCall owns signed receipt production and identity/provenance
  boundaries.
- The final A/B/C cleanup write receipt independently verified PASS for
  integrity, explicit approval binding, and signature.
- Offline verification is therefore verified for that final cleanup receipt.
- A Station-wide sweep also reported pre-existing/other `FAIL_AUDIT` entries;
  those are not rewritten or hidden, and they do not alter the final cleanup
  receipt result.
- The module's `change_prove` output explicitly reports that Station remains
  the authority for native receipt and quorum cryptographic verification.

## Acceptance matrix

### Golden paths

| Case | Expected | Verification level | Result | Evidence | Notes |
|---|---|---|---|---|---|
| G1 single update | Exact update, post-read verified, later rollback | LIVE PROVIDER | PASS | Step 2C native receipts | Disposable A only within authorized scope |
| G2 governed create | A/B/C create with authoritative post-state | LIVE PROVIDER | PASS | Step 2C create receipts | Disposable names only |
| G3 governed delete | Delete and authoritative absence | LIVE PROVIDER | PASS | Step 2C cleanup receipts | Final state absent |
| G4 batch cutover | All preconditions before first write; 3/3 verified | LIVE PROVIDER | PASS | Step 2C batch receipts | Provider atomicity not claimed |

### Governance

| Case | Expected | Verification level | Result | Evidence | Notes |
|---|---|---|---|---|---|
| GV1 no approval | `HUMAN_APPROVAL_REQUIRED`, zero mutation | MOCK / UNIT | PASS | `test_missing_approval_is_honest` | Native execute not run |
| GV2 incomplete quorum | Awaiting or blocked native Team approval | NOT VERIFIED | NOT VERIFIED | No usable Cloudflare Team channel | No fake approvers |
| GV3 self approval | Self approval excluded | NOT VERIFIED | NOT VERIFIED | Native Team runtime unavailable | — |
| GV4 explicit deny | Denial blocks mutation | NOT VERIFIED | NOT VERIFIED | Native Team runtime unavailable | — |
| GV5 duplicate signer | Duplicate counts once | NOT VERIFIED | NOT VERIFIED | Native Team runtime unavailable | — |
| GV6 removed/ineligible approver | Ineligible signer excluded | NOT VERIFIED | NOT VERIFIED | Native Team runtime unavailable | — |
| GV7 quorum downgrade | Old approval invalidated | NOT VERIFIED | NOT VERIFIED | Native policy transition not executed | — |
| GV8 payload tamper | Plan hash mismatch blocks before provider | MOCK / UNIT; live bounded tamper receipt | PASS | Tampered update blocked before provider touch | Native full Team path not claimed |

### State integrity

| Case | Expected | Verification level | Result | Evidence | Notes |
|---|---|---|---|---|---|
| S1 provider drift | `STALE_STATE`, zero mutation | MOCK / UNIT | PASS | TTL drift regression | Genuine external actor blocked |
| S2 non-content drift | Pinned TTL/proxy drift blocks | MOCK / UNIT | PASS | Canonical material-field comparison | Live external change unavailable |
| S3 batch partial stale | `BATCH_PRECONDITION_FAILED`, `0/3` expected | MOCK / UNIT | PASS | `0/2` regression representative | Genuine live `0/3` blocked |
| S4 duplicate execution | Replay cannot duplicate effect | LIVE PROVIDER | PASS | Consumed approval replay | Native exact replay evidence |
| S5 network ambiguity | `UNKNOWN`, no blind retry | MOCK / UNIT | PASS | Timeout regression | Real landed ambiguity not reproduced |
| S6 provider mismatch | Non-green verification/reconciliation | MOCK / UNIT | PASS | Verification/reconciliation regressions | — |

### Hostile and failure cases

| Case | Expected | Verification level | Result | Evidence | Notes |
|---|---|---|---|---|---|
| H1 invalid token | Auth failure, no false readiness | NOT VERIFIED | NOT VERIFIED | Not deliberately invalidated | Existing credential was reused |
| H2 wrong zone scope | Access refusal | NOT VERIFIED | NOT VERIFIED | Requires separate scoped token | — |
| H3 nonexistent zone | Exact path fails, no fallback | STATIC | PASS | Source/contract inspection | Live response unavailable |
| H4 record disappears | Exact 404 is explicit absence | MOCK / UNIT | PASS | `dns_record_get` regression | Live race unavailable |
| H5 DNS conflict | Structural conflict refusal | MOCK / UNIT | PASS | CNAME conflict regression | — |
| H6 unbounded request | Empty/duplicate target refusal | MOCK / UNIT | PASS | Bounds regression | — |
| H7 policy cap | Batch above cap blocked | STATIC / MOCK | PASS | `MAX_BATCH=25` and risk path | Native policy cap not run |
| H8 unsupported action | No arbitrary endpoint/method/type | MOCK / UNIT | PASS | Unsupported method/type tests | — |
| H9 expired approval | Expired native approval blocks | NOT VERIFIED | NOT VERIFIED | Native expiry not executed | Optional preview field is not enforcement |
| H10 tampered receipt | Offline signature failure | NOT VERIFIED | NOT VERIFIED | No tampered native receipt produced | — |
| H11 effect/evidence failure | `UNKNOWN` or evidence-incomplete | MOCK / STATIC | PASS | Timeout/post-read mismatch paths | Signing failure not live-produced |
| H12 stale rollback | External drift blocks rollback | NOT VERIFIED | NOT VERIFIED | External actor unavailable | Rollback itself was live verified |

## Native runtime and package checks

- `module/module.json`, `module/module.sig`, and `module/handlers/handler.py`
  were signed and verified as the installed bundle.
- Source and installed SHA-256 values matched in the Step 2 audit.
- Station reported `dave/cloudflare-traffic-governance` loaded with 20
  commands and zero rejected.
- Every manifest command has a callable handler function.
- The canonical provider metadata for all commands is
  `dave-cloudflare-traffic-governance::cloudflare`.
- Marketplace module identity remains `dave/cloudflare-traffic-governance`.

## Known test-environment limitations

- No distinct native Cloudflare Team approver channel was available.
- Active native policy rules did not provide Cloudflare Team coverage.
- Native module write approval expiry was not available in the tested path.
- No authenticated external Cloudflare actor was used for genuine out-of-band
  drift because raw token access was prohibited.
- Receipt-chain cleanliness is partial at Station-wide scope because existing
  `FAIL_AUDIT` entries remain; the final cleanup receipt independently passed.

These limitations are recorded as evidence boundaries, not hidden failures.

## Cleanup

The final provider state recorded by Step 2 is clean:

```text
railcall-e2e-a.davelabs.my.id  absent
railcall-e2e-b.davelabs.my.id  absent
railcall-e2e-c.davelabs.my.id  absent
```

No additional DNS mutation is authorized by this documentation task.
