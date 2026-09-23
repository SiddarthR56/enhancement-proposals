# Testplan — OSAC-4975

## Overview

- **Feature:** OSAC-4975 — BMaaS automatic ExternalIP allocation and lifecycle
- **PRD:** OSAC-1437 (scoped FRs: FR-6, FR-8, FR-9, FR-11, NFR-1)
- **Design:** `.artifacts/design/OSAC-4975/03-design.md`
- **Total test cases:** 14
- **Requirements covered:** 5 of 5 in epic scope (FR-1–FR-5, FR-7, FR-10, FR-12, NFR-2 remain under parent OSAC-1437 design — see Gaps)
- **Interface changes covered:** 4 of 4 (IC-1 … IC-4)

## Test Cases

### FR-6: Auto ExternalIP on BareMetalInstance create

#### TC-FR6-01: Successful auto reservation creates Pending EIP and EIPA

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-1 | critical | automated |

##### Preconditions

- READY ExternalIPPool with `available >= 1`
- Tenant with valid BMI create inputs and `auto_external_ip_attachment=true`

##### Steps

1. Call BareMetalInstances.Create with `auto_external_ip_attachment=true`.
2. List ExternalIPs filtered by `auto-created-for == <bmi-id>`.
3. List ExternalIPAttachments filtered by `auto-created-for == <bmi-id>` (or `spec.baremetal_instance.id`).

##### Expected Results

- Create returns OK with BMI id.
- Exactly one ExternalIP in `PENDING` with labels `auto-created=true`, `auto-created-for=<bmi-id>`, owner-reference annotation, tenant matching BMI.
- Exactly one ExternalIPAttachment in `PENDING` referencing that EIP and BMI, same auto labels.
- Pool `available` decreased by 1 versus pre-create.
- Create response BMI does not include a populated public ExternalIP address field implying Ready.

#### TC-FR6-02: Auto-created resources visible via public list filters

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-1, IC-4 | high | automated |

##### Preconditions

- TC-FR6-01 resources exist

##### Steps

1. As Tenant User, List ExternalIPAttachments with CEL `this.spec.baremetal_instance.id == "<bmi-id>"`.
2. Identify the item with `metadata.labels["osac.openshift.io/auto-created"] == "true"`.

##### Expected Results

- List returns the auto attachment.
- Manual-only tenants without access cannot see another tenant’s auto attachment.

### FR-8: Primary attachment IP visibility (DNAT precondition)

#### TC-FR8-01: EIPA stays Pending until BMI primary IP is present

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-2 | high | automated |

##### Preconditions

- BMI created with auto ExternalIP; EIP may be Allocated; BMI primary IP not yet written

##### Steps

1. Observe ExternalIPAttachment state before IP discovery completes.
2. After BMFO writes primary `networkAttachmentStatuses[].ipAddress` and feedback syncs, re-get the attachment until terminal state.

##### Expected Results

- Before primary IP: EIPA remains `PENDING` (not `READY`).
- After primary IP and DNAT success: EIPA becomes `READY`.

### FR-9: ExternalIPAttachment routes to BM primary IP

#### TC-FR9-01: Ready auto attachment exposes allocated address for BM target

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-2 | critical | automated |

##### Preconditions

- E2E (or IT) environment with fabric manager; BMI with auto ExternalIP reached Ready

##### Steps

1. Wait until auto ExternalIPAttachment state is `READY`.
2. Get referenced ExternalIP; read `status.address` / attachment `status.external_ip_address`.
3. Optionally SSH or HTTP probe via the external address to the BM (E2E).

##### Expected Results

- Attachment `READY` with non-empty external address.
- ExternalIP is `ALLOCATED` and attached.
- Traffic to the external address reaches the BM (E2E when fabric available).

### FR-11: Auto-cleanup on BMI deletion

#### TC-FR11-01: BMI delete removes auto EIPA then EIP and restores capacity

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-3 | critical | automated |

##### Preconditions

- BMI with auto EIP+EIPA (Pending or Ready); record pool `available`

##### Steps

1. Delete the BareMetalInstance.
2. List EIP/EIPA by `auto-created-for=<bmi-id>` until absent.
3. Read pool capacity.

##### Expected Results

- Auto ExternalIPAttachment is deleted.
- Auto ExternalIP is deleted.
- Pool `available` restored by 1 relative to post-create value.
- BMI is fully deleted (not stuck Terminating indefinitely under normal conditions).

#### TC-FR11-02: Manual ExternalIP is preserved on BMI delete

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-3 | critical | automated |

##### Preconditions

- BMI with `auto_external_ip_attachment=true` and its auto children
- Separately, a manually created ExternalIP (+ optional manual EIPA) in the same tenant, not labeled `auto-created-for` for this BMI

##### Steps

1. Delete the BMI.
2. Get the manual ExternalIP (and manual EIPA if present).

##### Expected Results

- Manual ExternalIP still exists.
- Manual ExternalIPAttachment (if any) still exists.
- Only auto-labeled children for the deleted BMI are gone.

#### TC-FR11-03: Default networking resources are not deleted

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-3 | high | automated |

##### Preconditions

- Tenant default VN/subnet/SG (and NATGateway if present) exist; BMI with auto EIP deleted

##### Steps

1. After BMI delete completes, Get default VN, subnet, SG (and NATGateway).

##### Expected Results

- Default networking resources still exist.

### NFR-1: Synchronous reservation; fail closed on pool exhaustion

#### TC-NFR1-01: Pool exhaustion returns error with no leaked BMI or capacity change

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-1 | critical | automated |

##### Preconditions

- No READY pool with `available > 0` (or pool with `available=0`)

##### Steps

1. Record BMI count and pool counters.
2. Create BMI with `auto_external_ip_attachment=true`.

##### Expected Results

- RPC fails with FailedPrecondition (or documented public equivalent) mentioning pool/capacity.
- No new BMI row for the request.
- No new EIP/EIPA with auto-created labels from this attempt.
- Pool counters unchanged.

#### TC-NFR1-02: Capacity race with available=1 allows exactly one winner

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-1 | critical | automated |

##### Preconditions

- Single READY pool with `available=1`

##### Steps

1. Concurrently (or back-to-back overlapping) create two BMIs with auto ExternalIP.

##### Expected Results

- Exactly one Create succeeds with Pending children.
- The other fails closed with no BMI leak and no over-allocation (`allocated` does not exceed pool size).

#### TC-NFR1-03: Mid-sequence child failure rolls back BMI

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-1 | critical | automated |

##### Preconditions

- Test double or fault injection causing ExternalIPAttachment create (or capacity update) to fail after EIP create

##### Steps

1. Attempt Create with auto flag under the fault.
2. Inspect DB for BMI id from the failed attempt and pool counters.

##### Expected Results

- Create returns error.
- No BMI left from the attempt.
- No orphan EIP/EIPA; capacity unchanged.

### IC-2 / Failed visibility (supports FR-9 lifecycle)

#### TC-FR9-02: Failed EIP or EIPA is observable without marking BMI create failed

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-2 | high | automated |

##### Preconditions

- BMI auto create succeeded; inject fabric allocate or DNAT failure (IT/envtest)

##### Steps

1. Confirm BMI still exists after create success.
2. Get EIP/EIPA until `FAILED` (or force Failed in envtest).
3. Read `status.message`.

##### Expected Results

- BMI remains (not deleted by async failure).
- Failed resource has `state=FAILED` and non-empty message.
- EIPA is not `READY`.

### IC-4: UI networking detail

#### TC-FR6-03: UI shows Pending placeholder without address after create

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-4 | high | automated |

##### Preconditions

- Component test harness with BMI `autoExternalIpAttachment=true` and EIPA `PENDING` without `externalIpAddress`

##### Steps

1. Render BareMetalNetworkingCard (or equivalent detail).

##### Expected Results

- UI shows in-progress/Pending external access copy.
- No Ready styling and no fabricated IP address.

#### TC-FR6-04: UI shows Ready address for auto attachment only

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-4 | high | automated |

##### Preconditions

- Auto EIPA Ready with address; optional second manual EIPA present

##### Steps

1. Render networking detail with list fixtures.

##### Expected Results

- Auto-provisioned label/address shown for the auto attachment.
- Manual attachment is not presented as the auto-provisioned external access row.

#### TC-FR9-03: UI shows Failed without crashing

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-4 | high | automated |

##### Preconditions

- Auto EIPA or EIP in `FAILED` with message

##### Steps

1. Render networking detail.

##### Expected Results

- Failed state and message visible.
- UI does not present Failed as Ready.

#### TC-FR11-04: UI handles missing auto attachment after cleanup

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-4 | medium | automated |

##### Preconditions

- BMI still displayed briefly with `autoExternalIpAttachment=true` but List EIPA returns empty (post-delete or race)

##### Steps

1. Render networking detail.

##### Expected Results

- No crash; empty/cleanup placeholder rather than stale Ready address.

## Gaps

- **FR-1–FR-5, FR-7, FR-10, FR-12, NFR-2:** Owned by parent OSAC-1437 BMaaS networking design/tests; not re-specified in this epic testplan.
- **Full fabric DNAT dataplane** in unit tests is out of reach; TC-FR9-01 relies on E2E/IT with fabric.
- **Open Question 9.1** (finalizer removal after N retries) has no TC until reviewers decide.
- **Open Question 9.2** (BMI condition for Failed) may add UI TCs if a condition is added.

---

## Provenance

Authored: draft @ design 0.11.3 - 9b25062, workspace OSAC-4975 @ dca3210df

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"design","workflow_version":"0.11.3","ai_workflows":"9b25062","source_repo":"dca3210df","source_repo_branch":"OSAC-4975","commits_behind_main":0,"commits_ahead_main":1448,"main_ref":"main","phases":["draft"],"authoring_modes":["skill"],"context_changed":false,"origin_untracked":false} -->
