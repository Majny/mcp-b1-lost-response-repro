# Retry recovery: collaboration plan and scenario matrix

Status: draft for collaborator review; 2026-10-16.
These are proposed fixture expectations, not adopted MCP semantics or completed tests.
This change adds planning documents only. No combined fixture has been implemented or run.

## Purpose and ownership

Build deterministic test material for the retry-identity work discussed in
[modelcontextprotocol/modelcontextprotocol#3394](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/3394).
Arjun owns the failure-case matrix, lost-response/recovery harness and effect-count oracle.
Fabio has agreed to contribute approval verification and the signed-approval adapter.
ndreno proposed taking the SEP specification and rationale. Sponsor, wire format and final
conformance expectations require their own review; this fixture does not predetermine them.
Other contributors' vectors must retain source attribution and their stated limitations.

The original root scripts and requirements remain unchanged. They use a real MCP SDK
client/server over in-memory streams and a synthetic in-process effect ledger, not a real
payment provider. The expanded fixture must describe its own persistence and transaction
boundaries explicitly; it must not turn a local ledger result into an exactly-once
guarantee for a remote payment.

## Fixed baselines

| Component | Pin | Use |
| --- | --- | --- |
| Original lost-response repository | `6373793aed3c98c281d6fa695afe9fb6fa541586` | Preserve both root scripts and root requirements |
| MCP Python SDK in original baseline | `mcp==2.2.0` | Preserve negotiated-version logging; no SDK upgrade in this work |
| Fabio's refund example | [`eb7cd4672addc02e1ea80a7a83fadf97d9b0a93e`](https://github.com/pic-standard/pic-standard/tree/eb7cd4672addc02e1ea80a7a83fadf97d9b0a93e/examples/refund-approval) | Example source, fixtures and workflow checks |
| PIC SDK dependency declared by that example | `330fdd817ef81d43461ea1787939e0d1cc87d589` | `pic-standard[crypto]` source pin; package version alone is insufficient |
| PIC transitive constraints | `examples/refund-approval/constraints.txt` at the example pin | Retain separately; combined dependency resolution is not yet tested |

Record Python version, both SDK source/version identities, dependency resolution and
negotiated MCP protocol version in future results. Do not assume combined-install
compatibility. If dependencies conflict, retain separate environments behind the adapter
boundary until an explicit, reviewed resolution exists.

## Shared definitions and independent oracles

- **Operation identity:** a fixture-owned stable identity scoped to authenticated tenant,
  principal and tool, with a bound semantic request. Approval references, attempt IDs,
  JSON-RPC IDs and mutable authorization epochs are distinct from it.
- **Effect oracle:** an append-only synthetic provider ledger independent of the
  deduplication store. Pruning the dedup store must not prune the oracle.
- **Recovery knowledge:** recorded completion, reserved/unknown, authoritative non-execution,
  or insufficient evidence. Actual provider completion may differ from what recovery knows.
- **Disclosure oracle:** inspect actual client-visible content, structured content and errors
  for a unique receipt marker. A denied result must not leak through an error or metadata.
- **New-execution permission:** current caller permission plus applicable approval/profile
  verification. It is distinct from current permission to disclose a historical receipt.
- **Recovery window:** fixture policy with explicit admission evidence. A caller-supplied
  creation time is not authenticated merely by being present.

Default recovery profile proposed for review: recognize an existing operation without
dispatch; authorize receipt disclosure independently; require current execution permission
for any new dispatch; never infer non-execution from a pruned-record lookup miss.
Recognition must not itself disclose operation existence across tenants.

Every row below is **planned / not run in this change**. Except the two baseline rows,
expectations describe candidate application-layer controls to test, not MCP conformance.

## Initial deterministic matrix

| ID | Setup / fault / transition | Expected effect count | Client-visible expectation / evidence |
| --- | --- | --- | --- |
| B01 | Original unguarded baseline: commit, drop response, fresh-ID application retry | 2 | Preserve original demonstration; observe timeout and second completion |
| B02 | Original positive control: same stable logical-operation key | 1 | Preserve original application-layer dedup demonstration |
| R01 | Commit and store receipt, drop response; same operation, valid approval and allowed disclosure | 1 | Return original receipt, no redispatch |
| R02 | Reserve operation, commit synthetic provider effect, interrupt before receipt storage; no provider lookup | 1 | Recovery remains unknown, no redispatch; oracle knows more than recovery |
| R03 | Same interrupted boundary as R02, provider supports authoritative operation lookup | 1 | Reconcile original receipt; log that lookup, not key alone, closed the gap |
| R04 | Deliberately unsafe effect-then-record control loses record; retry admitted | 2 | Demonstrate remote/local transaction gap; expected counterexample, not safe behavior |
| A01 | Approval expires after first commit; receipt retained; disclosure independently allowed | 1 | Return historical receipt without treating expired approval as new-effect permission |
| A02 | Same expiry, but current receipt disclosure denied | 1 | Denial without receipt marker, no dispatch |
| A03 | Signer key revoked after first commit; retained receipt; recovery identity independently authenticated | 1 | Reject that approval for new execution; disclose historical receipt only under explicit current recovery policy |
| A04 | Caller execution authority withdrawn after first commit | 1 | No new effect; vary disclosure permission independently and assert both outcomes |
| A05 | Same key/arguments presented by another tenant or principal | 1 for original; no new effect in rejection case | No original receipt/existence leak; reject fixture context change; separately defined genuinely new operations are not this case |
| A06 | Same scoped operation ID with changed payment, amount, currency or tool | 1 | Conflict/rejection, no cached receipt substitution and no dispatch; include signed-payload tampering control |
| A07 | New valid signed approval/ref, same authenticated decision-to-operation mapping | 1 | Recover same operation; new approval is not new operation identity |
| T01 | Receipt/dedup record pruned at 24h, approval still valid at day 2 of 7 days | 1 under safe profile | Reject recovery or report unknown according to chosen policy; lookup miss alone never authorizes dispatch |
| T02 | Same pruning plus approval reissuance | 1 under safe profile | Reissuance must not reset original operation admission window; retain independent mapping/tombstone or reject unverifiable recovery |
| T03 | Caller replaces claimed creation timestamp to appear fresh | 1 under safe profile | Reject changed/untrusted admission evidence; timestamp alone cannot restore eligibility |
| T04 | New, explicitly admitted operation after old window; distinguish from retry | 2 total only when independently authorized as new | Positive control: do not collapse legitimately distinct operations solely because arguments match |
| C01 | Concurrent requests with same scoped ID and semantic request | 1 under safe profile | Atomic reservation; other request gets retained result or in-progress/unknown; no unlocked check-then-act |
| X01 | Duplicate transport delivery with same operation identity | 1 under safe profile | Application effect suppression; log transport fault separately from model/application retry |
| X02 | Read-back returns empty while original request is in flight or provider visibility is delayed | 1 after controlled completion | Remain unknown until reliable evidence; missing list entry is not proof of non-execution |
| X03 | Transport success with empty payload | Assert actual oracle independently | Do not report confirmed success/non-execution solely from transport status; inspect operation outcome evidence |

For A03, denying all recovery under a deployment's policy is also a legitimate candidate.
Do not force PIC approval freshness or key trust to authorize disclosure. Agree the chosen
fixture policy and expected output before coding that row.

For T01-T03, demonstrate the unsafe lookup-miss behavior separately if useful, but never
describe the safe profile as achievable after deleting all recognition/admission evidence
while still allowing arbitrary fresh admission. Retained tombstones/mappings, server-bound
admission evidence or an explicit refusal policy are design options, not decided wire fields.

## Harness requirements

1. Commit exactly one synthetic effect before injecting the lost response; assert the
   first result is actually dropped and the client observes the intended SDK timeout.
2. Record both JSON-RPC IDs and stable operation identity. Keep semantic request arguments
   equal across retries. The original scripts vary `attempt_id`; any new semantic digest
   must explicitly exclude harness-only attempt metadata rather than claiming byte-identical
   arguments in the original scripts.
3. Inject transitions with barriers and a controllable clock. Do not wait 24 hours or use
   sleeps to create a race. Fabio should confirm clock injection supported by the pinned SDK;
   do not stub signature/freshness verification and report it as actual PIC verification.
4. Keep effect ledger, dedup state, approval verification, caller authorization and disclosure
   policy separately observable. No real payments, production credentials or personal data.
5. Missing/invalid approval bindings or adapter errors cannot reach a new-effect dispatch.
   Recovery can use a separately authenticated original operation mapping; an unverified
   claim inside a rejected approval is never sufficient for receipt access.
6. Concurrency and restart claims need their own exercised storage/locking path. Until then
   label in-process tests as in-process, not durable or crash-safe.
7. Archive per-attempt traces: transition, admission decision, approval result, lookup state,
   dispatch count, effect delta, disclosure decision and client-visible classification.
   Log receipt digests/test markers rather than private key material.
8. New test assertions must exit nonzero on failure. Baseline script stdout/exit alone is
   not sufficient: its unexpected-count branch currently only prints a message.

## Adapter and review

See [adapter-contract.md](adapter-contract.md). Agree its inputs/outputs in this draft PR.
Fabio can propose changes or submit an adapter PR from his fork; repository write access
is not required to contribute through a fork.

Suggested implementation sequence:
1. Agree matrix scope and adapter contract here.
2. Add the recovery harness and separate synthetic effect/dedup stores.
3. Integrate actual pinned PIC verification; run independent PIC baseline first.
4. Add expiry, the two revocation cases, disclosure and reissuance tests.
5. Add pruning/unknown-outcome cases, then concurrency and contributed fault definitions.
6. Publish actual results and limitations; map candidate vectors to a reviewed SEP draft.

Do not cite contributor registry/model statistics as fixture validation. The Oct 15
registry-sampling correction supersedes the earlier figures; raw methods/results require
review before reuse.

## Review questions

- Does the verifier expose a supported deterministic clock and keyring override at the pin?
- Which outputs can Fabio independently report without inferring caller authorization?
- Should denied recovery reveal any existence/status information in the chosen fixture policy?
- What immutable admission evidence survives pruning, and how is reissuance bound to it?
- Which first subset can we complete before considering an upstream conformance proposal?

Sources:
[ndreno work split](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/3394#issuecomment-6150734000),
[process routing](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/3394#issuecomment-6151318313),
[ryjen authorization vectors](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/3394#issuecomment-6107356108),
[Fabio expiry vector](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/3394#issuecomment-6122699649),
[retention vector](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/3394#issuecomment-6130163272),
[Arjun retention clarification](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/3394#issuecomment-6127493861),
[contributor sampling correction](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/3394#issuecomment-6192645077).
