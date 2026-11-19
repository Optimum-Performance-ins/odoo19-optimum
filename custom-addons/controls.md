# Data Integrity Controls

This document catalogs all controls that prevent write, delete, or modification of records based on set conditions across all insurance modules, ensuring data integrity and consistency.

## Numbering Scheme

- **C-XX** — Control defined in the base module (`optimum_insurance_base`)
- **C-XX-01** — Extension of that control in a type-specific module (dash + sub-number)
- Controls unique to a type module get their own top-level number

---

## 1. Delete Prevention (`unlink()` overrides)

### C-01 — Cannot delete policy with endorsements

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.policy` |
| **File** | `models/insurance_policy.py` (line ~491) |
| **Condition** | Policy has one or more endorsements |
| **Error** | _"You cannot delete policy %s because it has %d endorsement(s) associated with it. Delete the endorsements first or cancel the policy instead."_ |

### C-02 — Cannot delete corporate client with active employees

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `client.corporate` |
| **File** | `models/client_corporate.py` (line ~229) |
| **Condition** | Corporate client has active employees (no end date or end date in the future) |
| **Error** | _"Cannot delete corporate client '%s' with %s active employee(s). Please terminate all employments first."_ |

### C-112 — Cannot delete offer with child offers

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.offer` |
| **File** | `models/insurance_offer.py` (line ~1060) |
| **Condition** | Offer has any records in `child_offer_ids` (i.e., it is the parent of one or more negotiation scenarios) |
| **Error** | _"You cannot delete offer '%s' because it has child offers. Delete child offers first or archive the offer instead."_ |
| **Tests** | `tests/test_insurance_offer.py::TestOfferUnlink::test_unlink_blocked_when_offer_has_child`, `tests/test_insurance_offer.py::TestOfferUnlink::test_unlink_allowed_for_child_without_children` |

### C-03 — Vehicle removal from offer cascades inspection cleanup

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **Model** | `offer.has.vehicle` |
| **File** | `models/offer_has_vehicle.py` (line ~160) |
| **Condition** | Always on delete — removes related `vehicle.inspection.line` and orphaned inspections |
| **Type** | Cascading cleanup (no error raised) |

### C-141 — Cannot delete a cleared cheque

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.cheque` |
| **File** | `models/insurance_cheque.py` (`unlink`) |
| **Condition** | Any cheque in the recordset has `state == 'cleared'` — a cleared cheque is what pays its installments |
| **Error** | _"Cheque %s is cleared and cannot be deleted. Move it out of cleared first."_ |
| **FR** | FR-BASE-013-06-04 |

### C-144 — Covered installment cannot be deleted

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `policy.payment.schedule` |
| **File** | `models/policy_payment_schedule.py` (`unlink`) |
| **Condition** | Any row in the recordset has `cheque_id` set (any cheque state); not bypassable by the endorsement engine — `_check_revert_allowed` refuses earlier with the same reason |
| **Error** | _"%(line)s is covered by cheque %(cheque)s and cannot be deleted. Remove it from the cheque first."_ |
| **FR** | FR-BASE-013-06-06 |

### C-145 — Policy with a covered installment cannot be deleted

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.policy` |
| **File** | `models/insurance_policy.py` (`unlink`) |
| **Condition** | Any of the policy's `payment_schedule_ids` has `cheque_id` set — the DB cascade would silently break the covering cheques' sums |
| **Error** | _"You cannot delete policy %(policy)s: %(count)d of its installments are covered by a cheque (e.g. %(cheque)s). Remove them from their cheques first."_ |
| **FR** | FR-BASE-013-06-06 |

### C-174 — A figure an incentive or a formula figure uses cannot be deleted

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.incentive.figure` |
| **File** | `models/insurance_incentive_figure.py` (`_unlink_except_in_use`, `@api.ondelete`; `user_figure_ids`). Python only, no database `restrict` (D-144) |
| **Condition** | any incentive picks the figure, or any formula figure uses it, read as superuser |
| **Error** | _"Figure %(code)s is in use and cannot be deleted. ⏎ المؤشر %(code)s مستخدم ولا يمكن حذفه."_ |
| **FR** | FR-BASE-009-05-19, FR-BASE-009-05-24, BR-BASE-010-03-04 |

### C-198 — A batched installment cannot be deleted

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `policy.payment.schedule` |
| **File** | `models/policy_payment_schedule.py` (`unlink`, first lines, before C-144) |
| **Mechanism** | an `unlink()` guard, not an FK `restrict`: the batch line's `installment_id` is `ondelete='set null'` (D-185), so a row named only by a cancelled batch can go and its line keeps its snapshot |
| **Condition** | any row in the recordset has `is_commission_batched` (a line of a draft, sent or confirmed batch holds it). Example: deleting POL-0042 #1 while it is in CB/2026/0005 → refused, the message names the batch. |
| **Error** | _"Installment %(inst)s is in commission batch %(batch)s and cannot be deleted. Remove it from the batch or cancel the batch first. ⏎ القسط %(inst)s مدرج في دفعة العمولة %(batch)s ولا يمكن حذفه. أزله من الدفعة أو ألغِ الدفعة أولاً."_ |
| **FR** | FR-BASE-013-02-01, FR-BASE-013-06-14 |
| **Tests** | `TestCommissionBatchLines.test_08` |

### C-190 — Only a draft batch can be deleted

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.commission.batch` |
| **File** | `models/insurance_commission_batch.py` (`unlink`) |
| **Mechanism** | `unlink()` guard (D-181). A draft's `line_ids` are unlinked through the ORM first, so their installments' `is_commission_batched` recomputes at once (plan Review Focus 1). |
| **Condition** | any batch in the recordset is not `draft`. Example: CB/2026/0005 sent, or cancelled with "Wrong insurer" → refused; a cancelled batch keeps its reason trail. Deleting a draft frees POL-0042 #2 for another batch. |
| **Error** | _"Only draft batches can be deleted. Cancel a sent batch instead. ⏎ لا يمكن حذف إلا الدفعات المسودة. ألغِ الدفعة المرسلة بدلاً من ذلك."_ |
| **FR** | FR-BASE-013-02-07 |
| **Tests** | `TestCommissionBatch.test_08_only_draft_batches_can_be_deleted` |

See C-168 (incentive payout).

### C-221 — Only a draft or cancelled trip can be deleted

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_collection` |
| **Model** | `insurance.collection.delivery` |
| **File** | `models/collection_trip.py` (`_unlink_only_before_it_starts`, `@api.ondelete`) |
| **Mechanism** | `@api.ondelete(at_uninstall=False)` via `_require_state`. On top of it the ACL gives delete on trips to `base.group_system` only (C-241). The trip's items stay and are pending again (the FK empties their trip) |
| **Condition** | any trip in the recordset is `assigned`, `in_progress` or `completed`. Example: TRIP/2026/0007 on the road → refused; cancel it first, then the administrator deletes it |
| **Error** | _"Only a draft or cancelled trip can be deleted: %s"_ |
| **FR** | FR-COL-002-07-01 (D-25) |
| **Tests** | `TestAccess.test_the_collection_manager_cancels_and_resets_only_the_admin_deletes` |

### C-222 — An item is removed only before its trip starts

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_collection` |
| **Model** | `insurance.collection.delivery.item` |
| **File** | `models/collection_trip_item.py` (`_unlink_only_before_the_trip_leaves`, `@api.ondelete`) |
| **Condition** | the item has a result, or its trip is `in_progress` or `completed`. Example: the CHQ-12 item on a trip that has started → refused; the same item pending, or on a draft trip → removed |
| **Error** | _"An item can be removed only before its trip starts: %s"_ |
| **FR** | FR-COL-002-07-03 |
| **Tests** | — (no test refuses it yet) |

---

## 2. Write Validation (`write()` overrides)

### C-04 — Offer company restriction validation on write

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.offer` |
| **File** | `models/insurance_offer.py` (line ~884) |
| **Condition** | When `insurance_company_id` or `insurance_request_id` is changed |
| **Behavior** | Calls `_validate_company_restrictions()` before write to enforce company-level restrictions |

### C-05 — Corporate client enforces is_company on partner

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `client.corporate` |
| **File** | `models/client_corporate.py` (line ~215) |
| **Condition** | After any write |
| **Behavior** | Ensures `partner_id.is_company` remains `True` |

### C-06 — Shipment request trip type field validation on write

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_shipment` |
| **Model** | `insurance.request.shipment` |
| **File** | `models/insurance_request_shipment.py` (line ~148) |
| **Condition** | On create and write — validates required location fields based on trip type |
| **Errors** | _"Export From Location is required for Export Only and Export & Import trip types!"_, _"Export From Country is required when Export From Location is Specific Country!"_, and similar for all import/export location combinations |

### C-140 — Cleared cheque is locked

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.cheque` |
| **File** | `models/insurance_cheque.py` (`write` → `_check_cleared_lock`) |
| **Condition** | Any cheque in the recordset has `state == 'cleared'` and the write touches a field of `_LOCKED_WHILE_CLEARED` (`name`, `bank_id`, `currency_id`, `amount`, `client_id`, `insurance_company_id`, `payment_date`, `line_ids`, `payment_method`, `cheque_document`, `cheque_document_filename`). Amended by D-175 and D-254 (one payment-proof pair; a transfer's number is its `name`). `state` and the chatter/activity fields stay writable so the cheque can leave cleared |
| **Error** | _"Cheque %s is cleared and locked. Move it out of cleared (bounced, or back to deposited or received) before editing it."_ |
| **FR** | FR-BASE-013-06-04 |

### C-142 — Covered installment is locked until removed from its cheque; nothing joins a cleared cheque

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `policy.payment.schedule` |
| **File** | `models/policy_payment_schedule.py` (`write` → `_check_covered_lock`) |
| **Condition** | Row has `cheque_id` set (any cheque state) and the write touches a field of `_LOCKED_WHEN_COVERED` (`amount`, `due_date`, `cheque_id`), and is not exactly a detach (`cheque_id = False` alone); or `vals['cheque_id']` points at a cheque that is itself cleared. If the row's own `cheque_id.state == 'cleared'`, that message wins instead, checked first, and refuses even a detach. Bypass: `_renumbering_cheques` context only — no `_cheque_allocating` bypass, so the cheque form's own One2many commands are held to the same rule. The form sends LINK / UNLINK, which the ORM applies only to rows that change; a `(6, 0, ids)` SET on a cheque with covered rows re-writes them and is refused — code must use LINK / UNLINK (D-067) |
| **Error** | _"%(line)s is paid by cheque %(cheque)s, which is cleared. Move the cheque out of cleared before changing the installment."_ / _"%(line)s is covered by cheque %(cheque)s. Remove it from the cheque before changing it."_ / _"Cheque %s is cleared; no installment can be added to it."_ |
| **FR** | FR-BASE-013-06-06, FR-BASE-013-06-04 |

### C-143 — Paid flag, payment date, net amount, bonus, receipt-pointer and batched fields are system-set

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `policy.payment.schedule` |
| **File** | `models/policy_payment_schedule.py` (`create`/`write` → `_check_system_set`) |
| **Condition** | `is_paid`, `payment_date`, `net_amount`, `bonus_tier_id`, `bonus_amount`, `has_premium_schedule`, `premium_schedule_reference_number` or `is_commission_batched` present in `vals`, unless the write runs in superuser mode **and** under the system's own `_installment_system_write` context. Both are required (D-073): the context of an RPC call is whatever the client sent, so the key alone is no proof the system is writing; superuser mode cannot be sent. Only `net_amount` is ever written that way; the others are stored computes, flushed by the ORM without `write()` (D-083). Amended (D-186): the two receipt-pointer fields (FR-BASE-006-11-27) joined the set, since the accountant has write on installments; so did `is_commission_batched` (FR-BASE-013-02-03) |
| **Error** | _"Cannot edit %s on an installment: set by the system."_ |
| **FR** | FR-BASE-006-11-21, FR-BASE-006-11-23, FR-BASE-006-11-25, FR-BASE-006-11-27, FR-BASE-013-02-03 |
| **Tests** | `TestPremiumSchedule.test_14_the_pointer_is_not_written_by_hand` (the receipt-pointer fields), `TestCommissionBatchLines.test_06` (`is_commission_batched`) |

### C-153 — Clearing a cheque re-asserts its coverage

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.cheque` |
| **File** | `models/insurance_cheque.py` (`write` → `_write_and_assert_cleared`) |
| **Method** | `write()` |
| **Condition** | `vals.get('state') == 'cleared'`: any record now cleared has an empty `line_ids`, or the cheque entering cleared fails its own `_check_lines_consistent()` (same insurer, same currency, Σ installment amounts = cheque amount, C-138) |
| **Error** | _"Cheque %s covers no installment and cannot be cleared."_ / the three C-138 messages, raised from `_check_lines_consistent()` |
| **FR** | FR-BASE-013-06-04 |
| **Purpose** | Clearing is the payment event — it is what marks every covered installment paid and earns the bonus. `_check_lines_consistent()` alone is not enough at that moment: it skips an empty `line_ids` (valid in every other state), and a row-side `cheque_id` write can skip its own re-check under the `_cheque_allocating` context (C-138), which exists precisely so the cheque's own commands are not rejected mid-batch. Without this control a cheque could be cleared covering nothing, or covering the wrong total, and the cleared lock (C-140) would then make it hard to correct. The write and the re-check run inside one savepoint so a refusal leaves the cheque untouched rather than tentatively cleared. |

### C-156 — The first installment covers the administrative fees

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `policy.payment.schedule` (rule), checked for `insurance.policy` and `insurance.policy.endorsement` |
| **File** | `models/policy_payment_schedule.py` (`_check_first_installment_covers_fees`), called from `insurance.commission.mixin._assign_installment_net_amounts()`, `accept.offer.wizard._action_accept()` and `insurance.policy.endorsement._validate_settlement()` |
| **Method** | `_check_first_installment_covers_fees()` |
| **Condition** | The owner's first installment (lowest due date, then creation order) is positive and smaller than the owner's administrative fees (`amount < fees`). A negative first installment — an endorsement's refund — is not checked (D-070). Administrative fees = `gross_premium − net_premium` (policy) or `gross_delta − net_delta` (endorsement). |
| **Error** | _"The first installment (due %(date)s) is %(amount)s but the administrative fees of %(owner)s are %(fees)s. The first installment must cover them."_ |
| **FR** | FR-BASE-006-11-24 |
| **Purpose** | D-065: the first installment absorbs the gap between gross and net, so it must be at least that large. Checked at schedule creation (before the policy exists, before the endorsement applies) and whenever an amount or due-date change makes a different installment first. Skipped while the endorsement engine is creating or deleting rows (`_endorsement_applying`); the engine checks once the schedule is complete. |

### C-160 — An issued policy's premiums are never edited

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.policy` |
| **File** | `models/insurance_policy.py` (`write`, `_ISSUED_LOCKED_FIELDS`) |
| **Method** | `write()` |
| **Condition** | `net_premium` or `gross_premium` present in `vals` on an existing policy, in any state, whatever the value (the bare rule, D-067). No bypass: superuser, system administrator and system code are refused alike. Creation is not affected. |
| **Error** | _"The net and gross premium of policy '%(name)s' were fixed when it was issued and cannot be edited. Change the premium through an endorsement."_ |
| **FR** | FR-BASE-006-13-01 |
| **Purpose** | D-084. The installments' net amounts, their commission slices and the first-installment rule (C-156) are all derived from the issued premiums; a later edit would leave every stored net amount wrong and, on a policy whose first installment is covered, make the next schedule edit hit C-157 with no way out but bouncing a paid cheque. An endorsement changes the premium through its own deltas, whose amounts are locked once applied (FR-BASE-007-08-08). A `write()` guard rather than `@api.constrains`: a constraint sees only the new value and also runs at creation. |

### C-163 — An incentive's contract is fixed at creation

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.incentive` |
| **File** | `models/insurance_incentive.py` (`write`) |
| **Condition** | `contract_id` is among the written keys |
| **Error** | _"An incentive's contract cannot be changed. Save it to the library and add it to the other contract instead. ⏎ لا يمكن تغيير عقد الحافز. احفظه في الحوافز المحفوظة ثم أضفه إلى العقد الآخر."_ |
| **FR** | FR-BASE-009-05-01, BR-BASE-010-03-02 |
| **Purpose** | A payout must never change contract behind its record. Reuse is a copy (FR-BASE-009-05-09). |

### C-168 — A claimed or paid payout is frozen

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.incentive.payout` |
| **File** | `models/insurance_incentive_payout.py` (`write`, `_unlink_except_claimed`, `action_recompute`, `_move`) |
| **Condition** | `state != 'computed'` and a field of `_EVALUATED_FIELDS` is written; or the payout is deleted; or Recompute is pressed. A status button pressed from the wrong status. |
| **Errors** | _"A claimed or paid payout cannot be changed. ⏎ لا يمكن تعديل دفعة حافز تمت المطالبة بها أو دُفعت."_ / _"A claimed or paid payout cannot be deleted. ⏎ لا يمكن حذف دفعة حافز تمت المطالبة بها أو دُفعت."_ / _"Only a payout that is still computed can be recomputed. ⏎ لا يُعاد حساب دفعة الحافز إلا وهي في حالة «محسوبة»."_ / _"Only a payout that is %(from_state)s can become %(to_state)s. ⏎ فقط دفعة الحافز في حالة «%(from_state)s» يمكن أن تصبح «%(to_state)s»."_ |
| **FR** | FR-BASE-013-07-03, FR-BASE-013-07-04, BR-BASE-010-04-02 |
| **Purpose** | What was claimed from the insurer stays what the record shows. |

### C-173 — A figure an incentive or a formula figure uses cannot change

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.incentive.figure` |
| **File** | `models/insurance_incentive_figure.py` (`write`, `is_locked`, `user_figure_ids`); `views/insurance_incentive_figure_views.xml` (`readonly="is_locked"` on every definition field, warning banner; view part: C-175) |
| **Condition** | any incentive picks the figure, or any formula figure uses it (read as superuser); every write is refused (picking happens on the incentive, D-130) |
| **Error** | _"This figure is in use: duplicate it to change it. ⏎ هذا المؤشر مستخدم: انسخه لتعديله."_ |
| **FR** | FR-BASE-009-05-20, FR-BASE-009-05-24, D-122 |

### C-196 — Only the system writes the حفظة التوريد

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.policy`, `insurance.policy.endorsement` (through `insurance.commission.mixin`) |
| **File** | `models/mixins/insurance_commission_mixin.py` |
| **Mechanism** | `create()`/`write()` of `insurance.commission.mixin` → `_check_premium_schedule_write(vals)` (D-190) |
| **Condition** | a key of `PREMIUM_SCHEDULE_FIELDS` (`premium_schedule_document`, `premium_schedule_document_filename`, `premium_schedule_reference_number`) without both `env.su` and ctx `_premium_schedule_attach` (D-172, D-187). The context alone is not enough: an RPC client can send any context but cannot get superuser mode. Example: customer service writing an endorsement's reference → refused; the sysadmin writing the policy's receipt → refused. |
| **Error** | _"Use «Attach Premium Instalment Schedule» to change the حفظة التوريد. ⏎ استخدم «إرفاق حفظة التوريد» لتغيير حفظة التوريد."_ |
| **FR** | FR-BASE-006-08-07 |
| **Tests** | `TestPremiumSchedule.test_02` |

### C-207 — An expired or cancelled policy is read-only

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.policy` |
| **File** | `models/insurance_policy.py` (`write`, `_TERMINAL_STATES`, `_TERMINAL_SAFE_FIELDS`) |
| **Condition** | any write on a policy in `expired` / `cancelled` except `_TERMINAL_SAFE_FIELDS` (the chatter and activity fields). Stored-compute flushes are caught too. |
| **Error** | _"Cannot modify policy '%(name)s': it is %(state)s. Contact an administrator if changes are required."_ |
| **Amended** | the three `PREMIUM_SCHEDULE_FIELDS` are safe (D-163): the insurer's receipt often arrives after the policy expired. Example: POL-0031, expired last month → `TW-2026-090` attached. |
| **FR** | FR-BASE-006-12-01 |
| **Tests** | `TestPremiumSchedule.test_06` |

### C-187 — Batch line snapshots are system-set

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.commission.batch.line` |
| **File** | `models/insurance_commission_batch_line.py` (`create`/`write` → `_check_system_fields`, `_LINE_SYSTEM_FIELDS`; `create`/`write`/`unlink` → `_check_batch_editable` / `batch._check_lines_editable`, `_LINE_FROZEN_WHEN_SENT`) |
| **Mechanism** | `create()`/`write()`/`unlink()` guards. The system fields pass only when `insurance.commission.batch._in_transition()` is true: `env.su` **and** ctx `_batch_transition` (D-186, D-187). The frozen-when-sent check has no bypass. |
| **Condition** | (a) a key of `_LINE_SYSTEM_FIELDS` in `vals`: the snapshots (`installment_name`, `policy_id`, `endorsement_id`, `payment_date`, `paid_amount`, `premium_schedule_reference_number`, `expected_commission`, `expected_bonus`) the computes `expected_amount`, `difference`, and `batch_state` (D-211: a readonly related field has no inverse, so a hand write would only take the line out of the C-185 index). Example: the accountant writing `paid_amount = 1` or `batch_state = cancelled` → refused, also under a crafted `_batch_transition` context. (b) Once the batch is not draft: a line created on it, a line unlinked from it, or `installment_id` / `batch_id` (`_LINE_FROZEN_WHEN_SENT`) written on its line, to or from that batch. Example: CB/2026/0005 is sent → adding POL-0042 #2, removing a line or re-pointing it → refused until it is reset to draft. |
| **Error** | (a) _"%(fields)s on a commission batch line are set by the system. ⏎ %(fields)s في سطر دفعة العمولة يحددها النظام."_ (b) _"Batch %(batch)s has been sent; reset it to draft first. ⏎ أُرسلت الدفعة %(batch)s؛ أعدها إلى مسودة أولاً."_ |
| **FR** | FR-BASE-013-02-06 |
| **Tests** | `TestCommissionBatchLines.test_05`; `TestCommissionBatch.test_18_sent_lines_are_frozen` |

### C-194 — A batched installment keeps its payment

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.cheque`, `policy.payment.schedule` |
| **File** | `models/insurance_cheque.py` (`write`, after C-140 `_check_cleared_lock`); `models/policy_payment_schedule.py` (`write` → `_check_payment_not_batched`, before C-142; message `_raise_payment_batched`) |
| **Mechanism** | `write()` guards on both sides (D-178). Only the move out of Cleared and the row's `cheque_id` are checked, so the payment engine (C-138, C-153) is untouched. |
| **Condition** | payment side: a `state` other than `cleared` written on a cleared payment one of whose rows has `is_commission_batched`; row side: any `cheque_id` in `vals` on a row with `is_commission_batched`. Example: bouncing CHQ 004512 of POL-0042 #1, or cash REC-0091 back to Received, while the rows are in CB/2026/0005 → refused. A cancelled batch releases them. A confirmed batch's payments can never bounce again: that is what confirmed means. |
| **Error** | _"Payment %(payment)s pays installments in commission batch %(batch)s. Remove them from the batch or cancel it first. ⏎ الدفعة %(payment)s تسدد أقساطاً في دفعة العمولة %(batch)s. أزلها من الدفعة أو ألغِ الدفعة أولاً."_ |
| **FR** | FR-BASE-013-06-14 |
| **Tests** | `TestCommissionBatchLines.test_09`, `test_10`, `test_11` |

### C-183 — A batched حفظة التوريد cannot change

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.policy`, `insurance.policy.endorsement` (through `insurance.commission.mixin`) |
| **File** | `models/mixins/insurance_commission_mixin.py` (`_check_premium_schedule_write`, after C-196; hook `_premium_schedule_dependents`, overridden in `models/insurance_policy.py` and `models/insurance_policy_endorsement.py`) |
| **Mechanism** | `create()`/`write()` of the mixin (D-190). Runs on the system path too: the attach wizard's sudo write is refused alike. |
| **Condition** | a key of `PREMIUM_SCHEDULE_FIELDS` is written while any row of `record._premium_schedule_dependents()` has `is_commission_batched` (D-171). Policy: its rows whose `_premium_schedule_holder()` is the policy (its own and the endorsement rows that fall back to it). Endorsement: all its own rows, receipt or not. Example: END-0007 has no receipt, END-0007 #1 is in CB/2026/0005 under the policy's `TW-2026-118` → attaching `TW-2026-201` to END-0007 is refused. A cancelled batch releases it. |
| **Error** | _"The Premium Instalment Schedule of %(record)s is used by commission batch %(batch)s and cannot be changed. ⏎ حفظة التوريد الخاصة بـ %(record)s مستخدمة في دفعة العمولة %(batch)s ولا يمكن تغييرها."_ |
| **FR** | FR-BASE-006-08-08 |
| **Tests** | `TestCommissionBatchLines.test_12`, `test_12b` (through the attach wizard, D-221), `test_13`, `test_14` |

### C-195 — A batched policy's commission settings cannot change

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.policy` |
| **File** | `models/insurance_policy.py` (`write`, after C-160; `_BATCH_LOCKED_COMMISSION_FIELDS`) |
| **Mechanism** | `write()` guard on a module-level frozenset (D-182). No bypass, superuser included. |
| **Condition** | `is_flat_commission`, `is_commission_exception` or `commission_rate` in `vals` while any row of `payment_schedule_ids` (endorsement rows included) has `is_commission_batched`. Example: POL-0042 #1 is in CB/2026/0005 → changing the rate to 12% or ticking flat is refused. A cancelled batch releases it. |
| **Error** | _"The commission settings of policy %(policy)s are used by commission batch %(batch)s and cannot be changed. ⏎ إعدادات العمولة للوثيقة %(policy)s مستخدمة في دفعة العمولة %(batch)s ولا يمكن تغييرها."_ |
| **FR** | FR-BASE-006-08-09 |
| **Tests** | `TestCommissionBatchLines.test_15` |

### C-186 — A batch's status changes only through its buttons

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.commission.batch` |
| **File** | `models/insurance_commission_batch.py` (`create`, `write` → `_refuse_state_write`; the one writer is `_transition`) |
| **Mechanism** | `write()` and `create()` guards; pass only when `_in_transition()` is true: `env.su` **and** ctx `_batch_transition` (D-187). `create()` also refuses a non-draft state (D-191). `state` is `copy=False`, so Duplicate starts as draft. |
| **Condition** | `state` in `vals` outside `_transition()`. Example: the accountant writing `state = 'confirmed'`, also under a crafted `_batch_transition` context, or creating a batch with `state = 'confirmed'` → refused. |
| **Error** | _"Use the batch buttons to change its status. ⏎ استخدم أزرار الدفعة لتغيير حالتها."_ |
| **FR** | FR-BASE-013-02-05 |
| **Tests** | `TestCommissionBatch.test_03_the_status_moves_only_through_the_buttons`, `test_11_a_duplicate_starts_as_draft` |

### C-197 — What a batch was sent with stays fixed; confirmed amounts are typed while Sent or Confirmed

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.commission.batch`, `insurance.commission.batch.line` |
| **File** | `models/insurance_commission_batch.py` (`write`, `_LOCKED_WHEN_SENT`); `models/insurance_commission_batch_line.py` (`create`/`write` → `_check_confirmed_amount_entry`, `_CONFIRMED_AMOUNT_STATES`) |
| **Mechanism** | `write()` guards (D-180). Batch side: no bypass, the moves never write these fields. Line side: passes only when `_in_transition()` is true (`env.su` **and** ctx `_batch_transition`, D-187), which is the send pre-fill. |
| **Condition** | batch: a key of `_LOCKED_WHEN_SENT` (`insurance_company_id`, `currency_id`, `date`) in `vals` on a batch that is not draft; line: `confirmed_amount` in `vals` (create or write) while the line's batch is neither Sent nor Confirmed (`_CONFIRMED_AMOUNT_STATES`), outside `_in_transition()` (D-229). Once the batch has an invoice, C-188 refuses the amount first. Example: CB/2026/0005 is sent → changing its insurer, currency or date is refused; typing 400 on a draft line is refused, also under a crafted `_batch_transition` context; typing 400 on POL-0042 #2 of the sent batch, or of the confirmed batch before Create Invoice, is accepted; in Confirmed the correction also meets the confirm checks again (C-189, D-239). |
| **Error** | batch: _"%(fields)s can be changed only while the batch is draft. ⏎ %(fields)s لا يمكن تغييره إلا والدفعة مسودة."_; line: _"Confirmed amounts are entered while the batch is sent or confirmed. ⏎ تُدخل المبالغ المؤكدة أثناء حالة «مرسلة» أو «مؤكدة» للدفعة."_ |
| **FR** | FR-BASE-013-02-06, FR-BASE-013-08-01, FR-BASE-013-08-05 |
| **Tests** | `TestCommissionBatch.test_17_a_sent_batch_header_is_frozen`, `test_23b_confirmed_amounts_are_corrected_until_the_invoice`, `test_23h_a_correction_in_confirmed_meets_confirm_checks_again` |

### C-208 — A batch's totals and dates are system-set

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.commission.batch` |
| **File** | `models/insurance_commission_batch.py` (`write`, `_BATCH_SYSTEM_FIELDS`) |
| **Mechanism** | `write()` guard (D-186); passes only when `_in_transition()` is true: `env.su` **and** ctx `_batch_transition` (D-187). The ORM stores the totals without `write()`. |
| **Condition** | a key of `_BATCH_SYSTEM_FIELDS` in `vals`: the stored totals `total_paid_amount`, `total_expected_amount`, `total_confirmed_amount`, `total_difference`, `line_count`, and the move dates `sent_date`, `confirmed_date`; the commission invoice link `invoice_id`, written by Create Invoice only (it freezes the batch, C-188, D-232). Example: the accountant writing `line_count = 9`, `sent_date = today`, or `invoice_id` pointing at another move or cleared → refused. |
| **Error** | _"%(fields)s is set by the system. ⏎ %(fields)s يحدده النظام."_ |
| **FR** | FR-BASE-013-02-06, FR-BASE-013-08-01 |
| **Tests** | `TestCommissionBatch.test_19_batch_system_fields_are_not_written_by_hand`, `test_23_a_confirmed_batch_is_locked` |

### C-188 — A confirmed batch cannot be changed but for its confirmed amounts; an invoiced batch is read-only

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.commission.batch`, `insurance.commission.batch.line` |
| **File** | `models/insurance_commission_batch.py` (`write` → `_check_not_invoiced`, `_CHATTER_FIELDS`, `_check_not_confirmed`, `_CONFIRMED_WRITABLE`); `models/insurance_commission_batch_line.py` (`write` → `batch_id._check_not_invoiced`, `batch_id._check_not_confirmed`, `_LINE_CONFIRMED_WRITABLE`) |
| **Mechanism** | `write()` guards, the first check in each `write()`, before the C-186 / C-187 / C-197 / C-208 bypass is read. No bypass: superuser mode and ctx `_batch_transition` do not open them. The ORM stores recomputed totals without `write()`. No move leaves `confirmed` (C-200). |
| **Condition** | **invoiced** (the batch has an `invoice_id`, D-229): batch: a key outside `_CHATTER_FIELDS` (`message_ids`, `message_follower_ids`, `message_partner_ids`, `activity_ids`) in `vals`, `invoice_id` included; line: any non-empty `vals`. **confirmed, no invoice:** batch: a key outside `_CONFIRMED_WRITABLE` (`_CHATTER_FIELDS` + `invoice_id` + `line_ids`); line: a key outside `_LINE_CONFIRMED_WRITABLE` (`confirmed_amount`). `line_ids` passes the batch guard because the form saves an inline edit through the batch (`web_save` → `line_ids: [Command.update(line, {'confirmed_amount': 400})]`, D-231); each command then meets the line guards: an update of the amount passes, any other key is refused here, and `Command.create` / `Command.delete` are refused by C-187. Line create and unlink are refused outside draft by C-187. A deleted draft commission invoice clears `invoice_id` (SQL set null), so the amounts open again. Example: CB/2026/0005 is confirmed → writing its note, or `paid_amount` on POL-0042 #1 as superuser with `_batch_transition`, is refused; POL-0042 #2 corrected from 440 to 400 is accepted (C-197, C-205 and the confirm checks of C-189 still apply, D-239); Create Invoice writing `invoice_id` is accepted; after it, 390 on POL-0042 #2, the note, a document reference or clearing the invoice link are refused, also as superuser with `_batch_transition`; a chatter post "Paid by the insurer on day 40" is accepted. |
| **Error** | confirmed: _"Confirmed batch %(batch)s cannot be changed. ⏎ لا يمكن تغيير الدفعة المؤكدة %(batch)s."_; invoiced: _"Batch %(batch)s is invoiced and cannot be changed. ⏎ الدفعة %(batch)s صدرت لها فاتورة ولا يمكن تغييرها."_ |
| **FR** | FR-BASE-013-08-05, FR-BASE-013-08-06 |
| **Tests** | `TestCommissionBatch.test_23_a_confirmed_batch_is_locked`, `test_23b_confirmed_amounts_are_corrected_until_the_invoice`, `test_23c_an_invoiced_batch_is_read_only`, `test_23e_the_form_saves_a_corrected_amount_through_the_batch`, `test_23f_deleting_the_draft_invoice_opens_the_amounts`, `test_23g_a_correction_in_confirmed_is_noted_on_the_batch` (a correction in Confirmed is noted on the batch chatter, D-233) |

### C-215 — A cancelled batch keeps only its chatter

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.commission.batch` |
| **File** | `models/insurance_commission_batch.py` (`write` → `_check_not_cancelled`, `_CHATTER_FIELDS`) |
| **Mechanism** | `write()` guard (F18, D-247). The system's own writes pass (superuser mode **and** ctx `_batch_transition`, `_in_transition()`, D-187), like C-186 / C-208; a context key alone or superuser mode alone does not open it. No move leaves `cancelled` (C-200), so no code path writes a cancelled batch today. The lines need no guard of their own: on a cancelled batch C-187 refuses `installment_id` / `batch_id`, a new line and a line delete, and C-197 refuses `confirmed_amount`. The form shows the note and the documents read-only (C-204). |
| **Condition** | a key outside `_CHATTER_FIELDS` (`message_ids`, `message_follower_ids`, `message_partner_ids`, `activity_ids`) in `vals` on a batch whose `state` is `cancelled`, outside `_in_transition()`. Example: CB/2026/0003 is cancelled → its note "late edit", a company invoice reference or the insurer's documents are refused for the accountant, the sysadmin, superuser mode and the accountant with a crafted `_batch_transition`; a chatter post "Insurer closed the file" is accepted. |
| **Error** | _"Cancelled batch %(batch)s cannot be changed. ⏎ لا يمكن تغيير الدفعة الملغاة %(batch)s."_ |
| **FR** | FR-BASE-013-02-05 |
| **Tests** | `TestCommissionBatch.test_04b_a_cancelled_batch_is_read_only` |

### C-212 — A contract's Renewed From is fixed at creation

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.company.contract` |
| **File** | `models/insurance_company_contract.py` (`write`, first lines) |
| **Mechanism** | `write()` guard, a bare refusal of the key (like C-163); an immutability rule, so not `@api.constrains`. Renew sets the field through `copy()` defaults, a `create()` |
| **Condition** | `renewed_from_id` is among the written keys, whatever its value. Example: re-pointing the 2032 contract to a 2036 contract, or clearing it → refused |
| **Error** | _"A contract's Renewed From cannot be changed. Renew the old contract instead. ⏎ لا يمكن تغيير العقد الأصلي لعقد مجدد. جدد العقد القديم بدلاً من ذلك."_ |
| **FR** | FR-BASE-009-04-13 |
| **Tests** | `TestContractRenewal.test_06_renewed_from_is_fixed` |

### C-223 — A completed trip does not change

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_collection` |
| **Model** | `insurance.collection.delivery` |
| **File** | `models/collection_trip.py` (`_LOCK_TIERS`, `write`) |
| **Mechanism** | base `insurance.lock.tier.mixin`: `priority`, `notes`, `cost`, the cost document, `item_ids`, `scheduled_date` and `assigned_employee_id` lock at `completed`; `sequence` never locks (reordering a day). The form's read-only conditions come from the same tiers |
| **Condition** | a locked field is written on a `completed` trip. Example: TRIP/2026/0007 completed with cost 60 → changing the cost or taking an item off is refused; dragging it to another place in the day is allowed |
| **Error** | _"%(trip)s is completed: it can no longer change (%(fields)s)."_ |
| **FR** | FR-COL-002-08 |
| **Tests** | `TestTripFlow.test_a_completed_trip_does_not_change`, `test_the_form_carries_the_lock_tiers` |

### C-224 — A trip's state changes only through its action

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_collection` |
| **Model** | `insurance.collection.delivery` |
| **File** | `models/collection_trip.py` (`write`, `STATE_ACTIONS`, ctx `VIA_ACTION`) |
| **Mechanism** | a `state` written from outside an action (a kanban drag, an import, a script) is popped and its action runs instead, so C-236 to C-239 apply either way. A value outside the five states is refused |
| **Condition** | `state` is not one of `draft`, `assigned`, `in_progress`, `completed`, `cancelled`. Example: an import writing `out_for_delivery` → refused; a drag from Draft to Completed → refused by C-236 |
| **Error** | _"A trip has no state %s."_ |
| **FR** | FR-COL-003-04, FR-COL-003-04-02 |
| **Tests** | `TestTripFlow.test_an_unknown_state_is_refused`, `test_a_draft_dragged_to_completed_is_refused`, `test_a_drag_to_assigned_without_an_agent_is_refused`, `test_a_drag_to_completed_needs_the_items_results` |

### C-225 — A closed trip keeps its agent

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_collection` |
| **Model** | `insurance.collection.delivery` |
| **File** | `models/collection_trip.py` (`_check_agent_change`, from `write`) |
| **Condition** | `assigned_employee_id` is written on a `completed` or `cancelled` trip. Until then the leader changes the agent, on the road too, and the papers out with the old agent follow the new one (FR-COL-004-02-02). Example: TRIP/2026/0007 cancelled → choosing Sayed is refused |
| **Error** | _"%s is closed: its agent can no longer change."_ |
| **FR** | FR-COL-003-08 |
| **Tests** | — (a completed trip is refused first by C-223; no test for a cancelled one) |

### C-226 — What a completed trip carried does not change

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_collection` |
| **Model** | `insurance.collection.delivery.item` |
| **File** | `models/collection_trip_item.py` (`write`, first check) |
| **Condition** | any field is written on an item whose trip is `completed`. Example: the notes of the CHQ-12 item on completed TRIP/2026/0007 → refused |
| **Error** | _"%s is completed: what it carried can no longer change."_ |
| **FR** | FR-COL-002-08 |
| **Tests** | `TestTripFlow.test_a_completed_trip_does_not_change` |

### C-227 — Items move between trips only before they start

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_collection` |
| **Model** | `insurance.collection.delivery.item` |
| **File** | `models/collection_trip_item.py` (`_check_trip_takes_items` from `create` and `write`; `write` on `trip_id`) |
| **Condition** | (a) the item already has a result; (b) the trip it leaves is `in_progress` or `completed`; (c) the trip it joins is not `draft` or `assigned`. Example: adding the policy document to TRIP/2026/0007 once it started → refused; moving it between two draft trips → allowed |
| **Error** | (a) _"An item with its result stays on its trip: %s"_; (b) _"Items are taken off a trip only while it is draft or assigned: %s"_; (c) _"Items are put on a trip only while it is draft or assigned: %s"_ |
| **FR** | FR-COL-012-04-01 |
| **Tests** | `TestPendingItems.test_items_come_and_go_until_the_trip_starts` |

### C-228 — An item gets its result on the road, once

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_collection` |
| **Model** | `insurance.collection.delivery.item` |
| **File** | `models/collection_trip_item.py` (`write` on `result` / `failure_reason`; `_set_result` behind the Complete, Left at Office and Failed buttons) |
| **Condition** | the item's trip is not `in_progress`, or (buttons) the item already has a result. Example: Complete on an item of an assigned trip → refused; Failed on an item already Completed → refused |
| **Error** | write: _"An item gets its result while its trip is on the road: %s"_; buttons: _"%s: its trip is not on the road."_, _"%s already has its result."_ |
| **FR** | FR-COL-012-05 |
| **Tests** | `TestTripFlow.test_an_item_gets_its_result_on_the_road_once` |

### C-229 — An item the system listed keeps its leg

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_collection` |
| **Model** | `insurance.collection.delivery.item` |
| **File** | `models/collection_trip_item.py` (`write` on `leg`) |
| **Condition** | the item has a `source_ref` (the system listed it) and the new leg differs. Example: the CHQ-12 item client → insurer changed to insurer pickup → refused |
| **Error** | _"%s was listed by the system for what it carries: its leg cannot change."_ |
| **FR** | FR-COL-002-01-04 |
| **Tests** | `TestPendingItems.test_a_listed_item_keeps_its_leg` |

### C-230 — Nobody types where a paper is

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_collection` |
| **Model** | `insurance.document.custody.mixin` (on `insurance.policy`, `insurance.policy.endorsement`, `insurance.cheque`, `insurance.claim`) |
| **File** | `models/mixins/document_custody_mixin.py` (`create`, `write`) |
| **Mechanism** | `custody_state`, `custody_agent_id` and `custody_since` are written only through `_custody_write`, with the system's rights and ctx `CUSTODY_MOVE` (D-24). The fields are `copy=False`. A settled claim's lock is skipped for this write only (`insurance_claim._custody_write`, `skip_settled_lock`, D-41) |
| **Condition** | any custody field is in the values of a `create` or `write` without ctx `CUSTODY_MOVE`, administrator included. Example: an import creating a cheque `with_insurer` → refused |
| **Error** | _"Where a paper is changes only with its trips: it cannot be typed."_ |
| **FR** | FR-COL-004-05 (D-34) |
| **Tests** | `TestDocumentTracking.test_nobody_types_the_place` |

### C-244 — A claim's fields lock at their state step

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.claim`, `insurance.claim.receipt`, `claim.coverage.line`, `claim.has.vehicle`, `vehicle.inspection.line.loss` |
| **File** | `models/insurance_claim.py` (`_check_claim_lock`, `write`); vehicle: `models/insurance_claim.py` (`VEHICLE_LOCK_TIERS`), `models/claim_has_vehicle.py`, `models/vehicle_inspection_line_loss.py` |
| **Mechanism** | `CLAIM_LOCK_TIERS` gives each field the state at which it locks; `lock_level` (not stored) follows the state, Settled and Rejected being Closed. The receipt, coverage line and vehicle line models ask the claim (`_check_claim_lock`). Skipped only in superuser mode with ctx `skip_settled_lock` (D-279, D-280) |
| **Condition** | a field is written at or after its lock level. Example: CLM/0042 in Fixing: `claim_date` (locks at Missing Documents) → refused; a new receipt (locks at Waiting for Payment) → allowed; the coverage analysis (locks at Settled / Rejected) → allowed. The expected settlement date (FR-BASE-008-10-02) locks at Settled / Rejected only. A mass write is refused when any one claim is locked. A Settled or Rejected claim cannot be deleted. Vehicle (D-287, D-290): the Vehicles tab stays editable until Waiting for Payment, but vehicles are added / removed only until Informed Insurance Company; the inspection link locks at Fixing; `labor_cost` and the losses (created, changed, deleted; a loss on a policy inspection has no claim and is not locked) at Waiting for Receipt; `repair_status`, `is_total_loss`, `service_centre_id` and `description` at Waiting for Payment; all locked when Settled or Rejected |
| **Error** | _"Cannot modify claim %(name)s in %(state)s. Locked field(s): %(fields)s."_ |
| **FR** | FR-BASE-008-05-05, FR-VEH-006-01-05, FR-VEH-006-01-07, FR-VEH-007-10-10 |
| **Tests** | `TestClaimLockGuard.test_one_field_per_tier_locks_at_its_step`, `test_receipts_lock_at_waiting_for_payment`, `test_coverage_lines_lock_when_closed`, `test_the_message_names_the_claim_state_and_fields`, `test_context_alone_does_not_bypass`, `test_a_closed_claim_cannot_be_deleted`, `test_closing_fields_are_system_written`; `TestClaimViewDrift`; vehicle: `TestClaimVehicleLockTiers.test_vehicles_add_and_remove_until_informed`, `test_inspection_link_locks_at_fixing`, `test_repair_fields_stay_open_until_waiting_for_payment`, `test_the_vehicle_pick_locks_with_the_lines`, `test_labor_locks_at_waiting_for_receipt`, `test_losses_lock_at_waiting_for_receipt`, `test_form_write_on_lines_follows_the_child_tier`, `test_a_loss_without_a_claim_is_not_claim_locked`, `test_everything_locks_on_a_rejected_claim`, `test_claim_vehicle_view_drift` |

### C-245 — A claim is settled only when its analysis and inspection are done

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.claim` |
| **File** | `models/insurance_claim.py` (`_check_settle_ready`, `write`; the Settle Claim button, `action_settle_claim`, is a plain write and calls the same one) |
| **Condition** | a write with `state = 'settled'` by anyone, whether the status bar, a kanban drag or the Settle Claim button. Example: CLM/0042 dragged to Settled with a coverage still "Used" and no amount → refused; with its inspection still Scheduled → refused; with the analysis complete and the inspection Done → Settled, and `settled_on` / `settled_by_id` stamped in that same write with the date and the user. The inspection is never written by settling (D-275, D-282) |
| **Error** | _"Cannot settle: these coverages still need an amount (or should be set to Unused): %(names)s"_; _"Cannot settle: the inspection is not completed yet."_ |
| **FR** | FR-BASE-008-05-09, FR-BASE-008-07-05 |
| **Tests** | `TestClaimStatus.test_settle_needs_a_complete_analysis`, `test_settle_needs_a_done_inspection`, `test_settle_stamps_date_and_user`, `test_the_settle_button_settles_like_the_status_bar` |

### C-248 — A new claim starts Fresh

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.claim` |
| **File** | `models/insurance_claim.py` (`create`) |
| **Condition** | a user (not superuser) creates a claim with a `state` other than Fresh. Example: a create with `state = 'settled'` → refused. Superuser creation (fixtures, imports) is unchanged |
| **Error** | _"A new claim starts Fresh."_ |
| **FR** | FR-BASE-008-05-02 |
| **Tests** | `TestClaimStatus.test_a_new_claim_cannot_start_closed` |

### C-249 — A claim needs an expected settlement date

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.claim` |
| **File** | `models/insurance_claim.py` (`expected_settlement_date`, `create`); `migrations/19.0.1.5/pre-migrate.py` |
| **Mechanism** | `required=True` on the plain field (NOT NULL in the database); `create()` fills it with the claim date + 30 days (`EXPECTED_SETTLEMENT_DAYS`) when it is left out; the migration does the same for existing claims. It locks at Settled / Rejected through `CLAIM_LOCK_TIERS` (C-244) |
| **Condition** | a claim is created or saved without an expected settlement date. Example: a claim dated 2026-01-01 created without one gets 2026-01-31; clearing the date on the form → refused |
| **Error** | Odoo's required-field error |
| **FR** | FR-BASE-008-10-01, FR-BASE-008-10-02 |
| **Tests** | `TestClaimExpectedSettlement.test_default_is_claim_date_plus_30`, `test_default_without_claim_date_is_today_plus_30`, `test_explicit_date_is_kept`, `test_writable_at_open_steps_locked_when_closed` |

### C-250 — The closing stamps exist only on a closed claim

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.claim` |
| **File** | `models/insurance_claim.py` (`_check_closing_fields_match_state`, `action_back_to_fresh`); `migrations/19.0.1.6/pre-migrate.py` |
| **Constrains** | `state`, `denial_reason_id`, `settled_on`, `settled_by_id` |
| **Mechanism** | `@api.constrains`, so it holds on create and on write, for the superuser too. Back to Fresh clears the three fields (C-247); the migration clears the stamps left on claims reopened before the rule |
| **Condition** | a rejection reason on a claim that is not Rejected, or a settlement date or user on a claim that is not Settled. Example: a create with `settled_on` filled on a Fresh claim → refused |
| **Error** | _"Claim %s has a rejection reason but is not Rejected."_; _"Claim %s has a settlement date or user but is not Settled."_ |
| **FR** | FR-BASE-008-05-10 (D-292) |
| **Tests** | `TestClaimStatus.test_closing_stamps_only_on_a_closed_claim`, `test_back_to_fresh_reopens_a_settled_claim` |

### C-251 — A claim's calculated fields are set only by the system

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base`, `optimum_insurance_vehicle` |
| **Model** | `insurance.claim`, `claim.has.vehicle` |
| **File** | base `models/insurance_claim.py` (`_raise_if_calculated`, called from `write`); vehicle `models/claim_has_vehicle.py` (`write`) |
| **Mechanism** | `write()` guard, at every state, for every caller but the superuser. It refuses any stored, read-only computed or related field in the values. The lock guard (C-244) skips such fields, and the ORM does not enforce `readonly` on `write()`; the system's own recalculation does not come through `write()` |
| **Condition** | a user's write names a calculated field. Example: `write({'reimbursed_loss_value': 0})` on a settled claim, or `write({'loss_value': 5})` on a claim vehicle line → refused |
| **Error** | _"Calculated by the system and cannot be set: %s."_ |
| **FR** | FR-BASE-008-05-11 |
| **Tests** | `TestClaimLockGuard.test_calculated_fields_are_not_written_by_hand`, `TestClaimVehicleLockTiers.test_line_totals_are_not_written_by_hand` |

---

## 3. Python Constraints (`@api.constrains`)

### C-07 — Policy commission validation

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.policy` |
| **File** | `models/insurance_policy.py` (line ~452) |
| **Constrains** | `is_commission_exception`, `commission_rate`, `contract_id` |
| **Errors** | _"For manual commission policies, you must specify a commission rate greater than 0."_ / _"No contract found for company '%s' on start date %s."_ |

### C-08 — Policy payment schedule must equal gross premium

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.policy` |
| **File** | `models/insurance_policy.py` (line ~476) |
| **Constrains** | `payment_schedule_ids`, `gross_premium` |
| **Error** | _"Payment schedule total (%(total)s) must equal the gross premium (%(premium)s)."_ |

### C-09 — Offer negotiation scenario must have parent offer

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.offer` |
| **File** | `models/insurance_offer.py` (line ~677) |
| **Constrains** | `is_negotiation_scenario`, `previous_offer_id` |
| **Errors** | _"A negotiation scenario must have a parent offer."_ / _"A negotiation scenario's parent must be an actual offer, not another scenario."_ |

### C-10 — Client must have exactly one specialized record

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `client` |
| **File** | `models/client.py` (line ~123) |
| **Constrains** | `individual_id`, `corporate_id` |
| **Errors** | _"Client '%s' must have exactly one specialized record (individual or corporate). Currently has none."_ / _"Client '%s' must have exactly one specialized record. Current: %s individual, %s corporate"_ / Type mismatch errors |

### C-11 — Client area must belong to district, district must belong to state

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `client` |
| **File** | `models/client.py` (line ~172) |
| **Constrains** | `area_id`, `district_id`, `state_id` |
| **Errors** | _"Area '%s' does not belong to district '%s'."_ / _"District '%s' does not belong to state '%s'."_ |

#### C-11-01 — CRM lead geographic validation

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `crm.lead` |
| **File** | `models/crm_lead.py` (line ~231) |
| **Constrains** | `district_id`, `state_id`, `area_id` |
| **Errors** | _"District '%s' does not belong to state '%s'."_ / _"Area '%s' does not belong to district '%s'."_ |

### C-12 — Corporate client disjoint type check

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `client.corporate` |
| **File** | `models/client_corporate.py` (line ~118) |
| **Constrains** | `client_id` |
| **Error** | _"Client '%s' is already registered as an individual client. A client cannot be both individual and corporate."_ |

### C-13 — Individual client disjoint type check

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `client.individual` |
| **File** | `models/client_individual.py` (line ~255) |
| **Constrains** | `client_id` |
| **Error** | _"Client '%s' is already registered as a corporate client. A client cannot be both individual and corporate."_ |

### C-14 — Individual birth date cannot be in the future

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `client.individual` |
| **File** | `models/client_individual.py` (line ~266) |
| **Constrains** | `birth_date` |
| **Error** | _"Birth date cannot be in the future. Please enter a valid date of birth."_ |

### C-15 — CRM lead email validation

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `crm.lead` |
| **File** | `models/crm_lead.py` (line ~200) |
| **Constrains** | `email_from` |
| **Errors** | _"Email cannot be a website URL (starting with 'www')."_ / _"Invalid email format."_ / _"Email domain '%s' is not allowed."_ |

### C-16 — Insurance company scores must be between 0 and 5

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.company` |
| **File** | `models/insurance_company.py` (line ~289) |
| **Constrains** | All 12 score fields (medical/general/motor prices, claims, endorsements, SLA) |
| **Error** | _"%s must be between 0 and 5. Current value: %s"_ |

### C-17 — Employment end date cannot be before start date

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `corporate.client.individual` |
| **File** | `models/corporate_client_individual.py` (line ~192) |
| **Constrains** | `start_date`, `end_date` |
| **Error** | _"Employment end date (%s) cannot be before start date (%s)"_ |

### C-18 — No overlapping employment periods

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `corporate.client.individual` |
| **File** | `models/corporate_client_individual.py` (line ~203) |
| **Constrains** | `corporate_client_id`, `individual_id`, `start_date`, `end_date` |
| **Error** | _"Employment period overlaps with existing employment..."_ |

### C-19 — Insurance request must have client or CRM lead

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.request` |
| **File** | `models/insurance_request.py` (line ~336) |
| **Constrains** | `client_id`, `crm_lead_id` |
| **Error** | _"Either a Client or a CRM Lead must be specified for the insurance request."_ |

### C-20 — Renewal request must have trigger

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.request` |
| **File** | `models/insurance_request.py` (line ~346) |
| **Constrains** | `is_renewal`, `renewal_trigger` |
| **Error** | _"Renewal trigger is required for renewal requests."_ |

### C-21 — Renewal request cannot have CRM lead

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.request` |
| **File** | `models/insurance_request.py` (line ~355) |
| **Constrains** | `is_renewal`, `crm_lead_id` |
| **Error** | _"CRM Lead must be empty for renewal requests."_ |

### C-22 — Offer deadline must be after request date

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.request` |
| **File** | `models/insurance_request.py` (line ~364) |
| **Constrains** | `offer_submission_deadline`, `request_date` |
| **Error** | _"The offer submission deadline must be after the insurance request date."_ |

### C-23 — Copay percentage must be between 0 and 100

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_medical` |
| **Model** | `offer.category.has.benefit.type` |
| **File** | `models/offer_category_has_benefit_type.py` (line ~66) |
| **Constrains** | `copay_percentage` |
| **Error** | _"Copay percentage must be between 0 and 100."_ |

#### C-23-01 — Copay percentage on offer coverage items

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_medical` |
| **Model** | `offer.benefit.has.coverage.item` |
| **File** | `models/offer_benefit_has_coverage_item.py` (line ~90) |
| **Constrains** | `copay_percentage` |
| **Error** | _"Copay percentage must be between 0 and 100."_ |

#### C-23-02 — Copay percentage on policy benefit types

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_medical` |
| **Model** | `policy.category.has.benefit.type` |
| **File** | `models/policy_category_has_benefit_type.py` (line ~88) |
| **Constrains** | `copay_percentage` |
| **Error** | _"Copay percentage must be between 0 and 100."_ |

#### C-23-03 — Copay percentage on policy coverage items

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_medical` |
| **Model** | `policy.benefit.has.coverage.item` |
| **File** | `models/policy_benefit_has_coverage_item.py` (line ~90) |
| **Constrains** | `copay_percentage` |
| **Error** | _"Copay percentage must be between 0 and 100."_ |

### C-24 — Coverage item parent must match benefit type line

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_medical` |
| **Model** | `offer.benefit.has.coverage.item` |
| **File** | `models/offer_benefit_has_coverage_item.py` (line ~65) |
| **Constrains** | `coverage_id`, `benefit_type_line_id` |
| **Error** | _"Coverage item '%(item)s' belongs to '%(item_parent)s' but is assigned under '%(line_parent)s'."_ |

#### C-24-01 — Same check on policy coverage items

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_medical` |
| **Model** | `policy.benefit.has.coverage.item` |
| **File** | `models/policy_benefit_has_coverage_item.py` (line ~65) |
| **Constrains** | `coverage_id`, `benefit_type_line_id` |
| **Error** | Same as C-24 |

### C-25 — Coverage item limit type consistency

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_medical` |
| **Model** | `offer.benefit.has.coverage.item` |
| **File** | `models/offer_benefit_has_coverage_item.py` (line ~78) |
| **Constrains** | `limit_type`, `limit_amount`, `limit_quantity` |
| **Error** | _"Limit type '%s' requires a limit amount/quantity."_ |

#### C-25-01 — Same check on policy coverage items

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_medical` |
| **Model** | `policy.benefit.has.coverage.item` |
| **File** | `models/policy_benefit_has_coverage_item.py` (line ~78) |
| **Constrains** | `limit_type`, `limit_amount`, `limit_quantity` |
| **Error** | Same as C-25 |

### C-26 — Insurable item must belong to correct client (client match)

This pattern is replicated across all type-specific junction tables:

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_medical` |
| **Model** | `offer.has.employee` |
| **File** | `models/offer_has_employee.py` (line ~36) |
| **Constrains** | `employee_id`, `offer_id` |
| **Error** | _"Employee '%s' does not belong to the offer's client."_ |

#### C-26-01 — Employee-policy client match (medical)

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_medical` |
| **Model** | `policy.has.employee` |
| **File** | `models/policy_has_employee.py` (line ~38) |
| **Error** | _"Employee '%s' does not belong to the policy holder."_ |

#### C-26-02 — Employee-request client match (medical)

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_medical` |
| **Model** | `request.has.employee` |
| **File** | `models/request_has_employee.py` (line ~35) |
| **Error** | _"Employee '%s' does not belong to the request's client."_ |

#### C-26-03 — Vehicle-offer client match (vehicle)

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **Model** | `offer.has.vehicle` |
| **File** | `models/offer_has_vehicle.py` (line ~177) |
| **Error** | _"Vehicle '%s' does not belong to the offer's client."_ |

#### C-26-04 — Vehicle-policy client match (vehicle)

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **Model** | `policy.has.vehicle` |
| **File** | `models/policy_has_vehicle.py` (line ~55) |
| **Error** | _"Vehicle '%s' does not belong to the policy holder."_ |

#### C-26-05 — Vehicle-request client match (vehicle)

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **Model** | `request.has.vehicle` |
| **File** | `models/request_has_vehicle.py` (line ~52) |
| **Error** | _"Vehicle '%s' does not belong to the request's client."_ |

#### C-26-06 — Shipment-offer client match (shipment)

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_shipment` |
| **Model** | `offer.has.shipment` |
| **File** | `models/offer_has_shipment.py` (line ~57) |
| **Error** | _"Shipment '%s' does not belong to the offer's client."_ |

#### C-26-07 — Shipment-policy client match (shipment)

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_shipment` |
| **Model** | `policy.has.shipment` |
| **File** | `models/policy_has_shipment.py` (line ~45) |
| **Error** | _"Shipment '%s' does not belong to the policy holder."_ |

#### C-26-08 — Shipment-request client match (shipment)

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_shipment` |
| **Model** | `request.has.shipment` |
| **File** | `models/request_has_shipment.py` (line ~42) |
| **Error** | _"Shipment '%s' does not belong to the request's client."_ |

#### C-26-09 — Estate-offer client match (general)

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_general` |
| **Model** | `offer.has.estate` |
| **File** | `models/offer_has_estate.py` (line ~58) |
| **Error** | _"Estate '%s' does not belong to the offer's client."_ |

#### C-26-10 — Estate-policy client match (general)

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_general` |
| **Model** | `policy.has.estate` |
| **File** | `models/policy_has_estate.py` (line ~46) |
| **Error** | _"Estate '%s' does not belong to the policy holder."_ |

#### C-26-11 — Estate-request client match (general)

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_general` |
| **Model** | `request.has.estate` |
| **File** | `models/request_has_estate.py` (line ~43) |
| **Error** | _"Estate '%s' does not belong to the request's client."_ |

---

### C-134 — Commission batch accepts only policies with a commission (Retired → C-184)

**Retired → C-184** (commission-batch-receipts plan, Task 18). The batch no longer has `policy_ids`; it is a list of installment lines, and C-184 refuses a line whose installment earns no commission. Kept for the record:

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.commission.batch` |
| **File** | `models/insurance_commission_batch.py` (removed: `_check_policies_have_commission`) |
| **Constrains** | `policy_ids` (field removed) |
| **Condition** | Any batched policy has `commission_amount <= 0` (checked as superuser: the field is accountant-only, C-136, and the sysadmin also runs batches; this replaced the former `policy_ids` domain clause) |
| **Error** | _"These policies have no commission and cannot be batched: %s"_ |
| **FR** | FR-BASE-013-02-02 |

---

### C-137 — Per-person limit is 0 when inactive and never negative when active

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.coverage.mixin` (persistent coverage rows: `offer.has.coverage`, `policy.has.coverage`; transient rows skip it unless `force_coverage_validation`), `endorsement.change.line.wizard` (its `wiz_cov_per_person_*` shadow fields) |
| **File** | `models/mixins/insurance_coverage_mixin.py` (`validate_dimension`, called by `_check_dimensions`), `wizards/endorsement_change_line_wizard.py` (`_validate_wiz_cov_dimensions`) |
| **Constrains** | `per_person_active`, `per_person_limit` (via `COVERAGE_DIMENSION_FIELDS`) |
| **Condition** | `per_person_active = False` and `per_person_limit != 0`; or `per_person_active = True` and `per_person_limit < 0`. No minimum above 0 while active: the currently supported insurance types need no person to be insured on. The activation onchange seeds 1 as a convenience only. |
| **Error** | _"%s is inactive; persons covered must be 0."_ / _"%s persons covered cannot be negative."_ |
| **FR** | FR-BASE-011-02-08 |

### C-138 — Cheque installments: same insurer, same policy holder, same currency, sum to the cheque amount

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.cheque` (`_check_lines_consistent`), `policy.payment.schedule` (`_check_cheque_company`, `_check_cheque_client`, `create()`, `write()`) |
| **File** | `models/insurance_cheque.py`, `models/policy_payment_schedule.py` (`_check_cheque_company`, `_check_cheque_client`, `_allocating_from_cheque()`, `create()`, `write()`) |
| **Constrains** | `insurance.cheque`: `line_ids`, `amount`, `currency_id`, `insurance_company_id`, `client_id`. `policy.payment.schedule`: `cheque_id` |
| **Condition** | Any covered installment belongs to another insurance company, or to a policy whose holder is not the cheque's client, or is in another currency, or Σ(installment amounts) ≠ cheque amount; also a `cheque_id` create/write on the installment outside the cheque's own One2many commands (`_cheque_allocating` context, read by the shared helper `_allocating_from_cheque()`) that would break the old or new cheque's sum. The aggregate/currency rule lives on the cheque because a cheque's installments change through One2many commands that link/unlink rows one write at a time; the insurer and holder rules are also enforced per row so they are safe mid-way through those commands. A row-side `cheque_id` create or write (not carrying `_cheque_allocating`) re-checks the cheque(s) involved explicitly from `policy.payment.schedule.create()`/`write()`, since `insurance.cheque`'s own `@api.constrains` never fires from a row create or write. Also re-asserted when a cheque enters `cleared` (C-153) |
| **Error** | _"Installment %(line)s belongs to %(company)s and cannot be covered by cheque %(cheque)s of %(other)s."_ / _"Installment %(line)s belongs to %(holder)s and cannot be covered by cheque %(cheque)s of %(other)s."_ / _"Cheque %(cheque)s is in %(currency)s; installment %(line)s is not."_ / _"The installments covered by cheque %(cheque)s total %(total)s, but the cheque amount is %(amount)s. They must match exactly."_ |
| **FR** | FR-BASE-013-06-03 |

### C-146 — Endorsement change lines for installments are add-only

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.policy.endorsement.line` |
| **File** | `models/insurance_policy_endorsement_line.py` (`_check_installment_lines_add_only`), also enforced in `wizards/endorsement_change_line_wizard.py` (`_validate_cheque_line`, `_onchange_operation`) |
| **Constrains** | `operation`, `target_record`, `before_values`, `after_values` |
| **Condition** | A line whose target model is `policy.payment.schedule` has `operation` other than `add` |
| **Error** | _"An endorsement can only add installments. Existing installments are never modified or removed; add a refund installment instead."_ |
| **FR** | FR-BASE-007-08-03 |

### C-151 — Cheque scan required, PDF or image

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.cheque` |
| **File** | `models/insurance_cheque.py` (`_check_method_fields`, was `_check_cheque_document`) |
| **Mechanism** | `@api.constrains`, shared with C-191. Files are read as their sizes through `with_file_sizes()` (`tools/document_checks.py`), never loaded (R6, D-245). |
| **Constrains** | `payment_method`, `cheque_document`, `cheque_document_filename` (and the transfer fields, C-191) |
| **Condition** | `cheque_document` is not set, or `cheque_document_filename` does not end with `.pdf`, `.png`, `.jpg` or `.jpeg` (case-insensitive); enforced through `default=False`, which puts the field in every create's values — `required=True` alone is a client hint for attachment binaries |
| **Error** | _"Cheque %s needs a scan of the physical cheque."_ / _"The scan of cheque %s must be a PDF or an image (.pdf, .png, .jpg, .jpeg)."_ |
| **FR** | FR-BASE-013-06-09 |
| **Extended by** | C-191 (cheques only since FR-BASE-013-06-10) |

### C-159 — An endorsement's installments have the sign of its direction

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.policy.endorsement` |
| **File** | `models/insurance_policy_endorsement.py` (`_check_settlement_shape`, run by the `_check_settlement` constraint and by `_validate_settlement()` at apply / schedule) |
| **Constrains** | `settlement_direction`, `net_premium`, `gross_premium`, `change_line_ids` (the constraint also fires when a change line is created for the endorsement) |
| **Condition** | An installment change line (`_cheque_lines()`) whose `after_values['amount']` is negative on an increase, or positive on a decrease |
| **Error** | _"Endorsement %(number)s is a premium increase and can carry only positive installments; %(amount)s due %(date)s is not one."_ / _"Endorsement %(number)s is a premium decrease and can carry only refund (negative) installments; %(amount)s due %(date)s is not one."_ |
| **FR** | FR-BASE-007-08-13 |
| **Purpose** | D-074 (increase side), extended by the user to the decrease side on 2026-10-03. A refund row inside an increase would earn a negative bonus (net amount × tier %, FR-BASE-006-11-14); a positive row inside a decrease would be money collected on a record that earns nothing (FR-BASE-007-09-04). |

### C-157 — A covered installment's net amount never moves

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `policy.payment.schedule` |
| **File** | `models/policy_payment_schedule.py` |
| **Method** | `_check_covered_net_amount()` |
| **Constrains** | `net_amount` |
| **Condition** | A `net_amount` is written on an installment that has a `cheque_id` (any cheque state). Only the system writes a net amount (C-143), and `_assign_installment_net_amounts()` writes only the rows whose net amount changes, so this is exactly: a create, delete, amount or due-date change on any installment of an owner would change the net amount of a covered installment of that owner. No bypass; the system's own superuser write is refused too. |
| **Error** | _"%(line)s is covered by cheque %(cheque)s. This change would move the administrative fees of %(owner)s onto or off it. Remove it from the cheque first."_ |
| **FR** | FR-BASE-006-11-23, FR-BASE-013-06-06 |
| **Purpose** | D-066 applied to the one figure another row's edit can move: a covered installment cannot be modified, and its net amount is the basis of its bonus. Without it, moving an uncovered installment before a cleared first one would silently change a paid bonus (D-069 A3). A constraint on the installment since D-085 (it used to be a check inside the assigner). |

### C-162 — An incentive's period settings match how it is evaluated

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.incentive` |
| **File** | `models/insurance_incentive.py` (`_check_period_settings`) |
| **Constrains** | `evaluation`, `period_type`, `period_months` |
| **Errors** | _"An incentive evaluated every period needs a period type. ⏎ الحافز الذي يُقيَّم كل فترة يحتاج إلى نوع فترة."_ / _"An incentive evaluated at contract end has no period type. ⏎ الحافز الذي يُقيَّم في نهاية العقد ليس له نوع فترة."_ / _"'Every N months' needs a number of months above zero. ⏎ «كل N شهر» يحتاج إلى عدد شهور أكبر من صفر."_ / _"The number of months is used only with 'Every N months'. ⏎ عدد الشهور يُستخدم فقط مع «كل N شهر»."_ |
| **FR** | FR-BASE-009-05-02, BR-BASE-010-03-01 |

### C-164 — A figure's fields match its kind; a record figure is a well-formed sum or count

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.incentive.figure` |
| **File** | `models/insurance_incentive_figure.py` (`_check_kind`, `_check_sum_field`, `_unique_code`) |
| **Constrains** | `kind`, `source`, `aggregate`, `sum_field`, `insurance_type_ids`, `formula` |
| **SQL** | `UNIQUE(code)` |
| **Errors** | _"A figure code is used once in the library. ⏎ رمز المؤشر يُستخدم مرة واحدة في مكتبة المؤشرات."_ / _"Figure %(code)s needs a field to sum. ⏎ المؤشر %(code)s يحتاج إلى حقل للجمع."_ / _"Figure %(code)s counts records, so it has no field to sum. ⏎ المؤشر %(code)s يعدّ السجلات، فليس له حقل للجمع."_ / _"Figure %(code)s cannot sum '%(field)s': it is not a field of its source. ⏎ لا يمكن للمؤشر %(code)s جمع '%(field)s': ليس حقلاً من مصدره."_ / _"Figure %(code)s reads records, so it has no formula. ⏎ المؤشر %(code)s يقرأ السجلات، فليس له معادلة."_ / _"Figure %(code)s needs a source and an operation. ⏎ المؤشر %(code)s يحتاج إلى مصدر وعملية."_ / _"Figure %(code)s is a formula, so it has no source, operation, field to sum or types. ⏎ المؤشر %(code)s معادلة، فليس له مصدر أو عملية أو حقل للجمع أو أنواع."_ |
| **FR** | FR-BASE-009-05-03, BR-BASE-010-03-03, FR-BASE-009-05-21, FR-BASE-009-05-19, FR-BASE-009-05-24 |

### C-177 — A formula figure's formula is well formed and names library figures

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.incentive.figure` |
| **File** | `models/insurance_incentive_figure.py` (`_check_formula`) |
| **Constrains** | `kind`, `formula` |
| **Errors** | _"Figure %(code)s is a formula: write its formula. ⏎ المؤشر %(code)s معادلة: اكتب معادلته."_ / _"Figure %(code)s: %(error)s ⏎ المؤشر %(code)s: %(error_ar)s"_ |
| **FR** | FR-BASE-009-05-24, D-138 |

### C-178 — A formula figure never contains itself

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.incentive.figure` |
| **File** | `models/insurance_incentive_figure.py` (`_check_not_contains_itself`, `_contains`, `write`) |
| **Constrains** | `formula` |
| **Errors** | _"Figure %(code)s cannot use itself. ⏎ لا يمكن للمؤشر %(code)s أن يستخدم نفسه."_ / _"Figure %(code)s cannot use %(part)s: %(part)s contains %(code)s. ⏎ لا يمكن للمؤشر %(code)s استخدام %(part)s: المؤشر %(part)s يحتوي على %(code)s."_ |
| **Note** | The figure `write()` also refuses `component_ids` and `user_figure_ids` (D-140): the figures a formula uses are read from its formula. |
| **FR** | FR-BASE-009-05-24, D-138 |

### C-165 — An incentive's formulas are well formed and use its own figures

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.incentive`; `insurance.incentive.figure` |
| **File** | `models/insurance_incentive.py` (`_check_formulas`); `models/insurance_incentive_figure.py` (`write`) |
| **Constrains** | `condition_formula`, `payout_formula`, `figure_ids`; the figure's `write()` refuses `incentive_ids` (D-130) |
| **Errors** | _"%(incentive)s — %(formula)s: %(error)s ⏎ %(incentive)s — %(formula_ar)s: %(error_ar)s"_ (the error is the formula error, e.g. "Unknown figure Z."; every formula error is English then Arabic, FR-BASE-009-05-11; an emptied group is refused with "A group needs at least one condition.", the same message as before, FR-BASE-009-05-13; e.g. _"Choose a figure or a number for every empty value. ⏎ اختر مؤشراً أو رقماً لكل قيمة فارغة."_) / (raised when a write takes the figure out of the incentive; checked before the formula errors) _"Figure %(code)s is used by a formula of '%(incentive)s'. Take it out of the formula first. ⏎ المؤشر %(code)s مستخدم في معادلة الحافز '%(incentive)s'. أزله من المعادلة أولاً."_ / _"Pick figures from the incentive. ⏎ اختر المؤشرات من الحافز."_ (D-130: the figure's `write()` refuses `incentive_ids`) |
| **FR** | FR-BASE-009-05-05, FR-BASE-009-05-06, FR-BASE-009-05-12, FR-BASE-009-05-13, BR-BASE-010-03-04, FR-BASE-009-05-19 |

### C-179 — A flat commission type has no early-payment tiers on its contract

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.company.contract` |
| **File** | `models/insurance_company_contract.py` |
| **Mechanism** | `_check_flat_declarations()` from `@api.constrains('commission_line_ids', 'early_payment_reward_ids')` (`_check_flat_declarations_on_save`). It runs once, after every line and tier command of a contract save (D-182). Known limit, as C-180: a line or tier written directly (import, RPC) is not checked (D-253). |
| **Condition** | a type declared flat on the contract has a reward row on the same contract. **In practice:** declare a type flat before any policy earns on it; an earned tier cannot be deleted (C-158). |
| **Error** | _"Insurance type %(type)s is a flat commission (عموله مقتوعه) on this contract and cannot have early-payment rewards. ⏎ نوع التأمين %(type)s عمولته مقتوعة في هذا العقد ولا يجوز أن تكون له مكافآت سداد مبكر."_ |
| **FR** | FR-BASE-009-04-12, BR-BASE-010-02-05 |
| **Tests** | `TestContractFlatCommission.test_07`, `test_08`, `test_11`–`test_13` |

---

### C-180 — Every non-flat commission type has early-payment tiers

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.company.contract` |
| **File** | `models/insurance_company_contract.py` |
| **Constrains** | `commission_line_ids`, `early_payment_reward_ids` (`_check_early_payment_coverage`) |
| **Condition** | a **non-flat** commission line type has no early-payment reward row. A flat line (`is_flat_commission`, «عموله مقتوعه») is exempt. Known limit (unchanged): a direct child write skips it. |
| **Error** | the existing _"The following insurance types have commission lines but no early payment reward entries…"_ |
| **FR** | FR-BASE-009-04-08, BR-BASE-010-01-05, FR-BASE-009-04-11 |
| **Tests** | `TestContractFlatCommission.test_03`, `test_04` |

---

### C-181 — A non-manual flat policy needs its contract's flat declaration

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.policy` (part (a)); `insurance.company.contract` (part (b)) |
| **File** | (a) `models/insurance_policy.py` (`_check_flat_commission_declared`); (b) `models/insurance_company_contract.py` (`_check_flat_declarations`, run by the contract constraint, as C-179) |
| **Constrains** | (a) `is_flat_commission`, `is_commission_exception`, `contract_id`, `offer_id` (the policy's `insurance_type_id` is related to the offer, so the offer is the trigger). Read as superuser: the fields are accountant-only (C-136). (b) the contract's `commission_line_ids` (a line written or deleted directly is not checked, D-253); its policy search runs as `sudo()` (D-184). |
| **Condition** | (a) a flat policy (`is_flat_commission`, «عموله مقتوعه») that is not manual (`is_commission_exception = False`) whose contract is missing or does not declare its insurance type flat (`_is_flat_commission_for`). A manual flat policy is accepted on any contract. (b) a contract line stops being flat (unticked or deleted) while a non-manual flat policy of that contract and type exists. Example: Motor unticked while POL-0050 (Motor, flat, not manual) uses the contract → refused. |
| **Error** | (a) _"Policy %(policy)s is a flat commission but its contract does not declare %(type)s as flat. Make the commission manual or declare the type flat on the contract. ⏎ الوثيقة %(policy)s عمولتها مقتوعة لكن عقدها لا يحدد %(type)s كعمولة مقتوعة. اجعل العمولة يدوية أو حدد النوع كعمولة مقتوعة في العقد."_ (b) _"Insurance type %(type)s cannot stop being flat on this contract: flat policies %(policies)s use it. ⏎ لا يمكن إلغاء العمولة المقتوعة لنوع التأمين %(type)s في هذا العقد: الوثائق المقتوعة %(policies)s تستخدمه."_ |
| **FR** | FR-BASE-006-08-06 |
| **Tests** | (a) `TestFlatCommissionPolicy.test_01`–`test_03`; (b) `TestContractFlatCommission.test_15`–`test_17` |

---

### C-182 — A حفظة التوريد is a PDF with its reference, on a record that has installments

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.policy`, `insurance.policy.endorsement` (through `insurance.commission.mixin`) |
| **File** | `models/mixins/insurance_commission_mixin.py` (`_check_premium_schedule`) |
| **Constrains** | `premium_schedule_document`, `premium_schedule_document_filename`, `premium_schedule_reference_number` |
| **Mechanism** | `check_pdf_pair` in `tools/document_checks.py` (shared with the batch documents), then the own-installments check. Files are read as their sizes through `with_file_sizes()` (`tools/document_checks.py`), never loaded (R6, D-245). |
| **Condition** | document xor reference; filename not `.pdf` (any case: `TW-118.PDF` is accepted, `tw.docx` refused); a receipt on a record with no own installments (D-173): a policy with no schedule, an endorsement not applied. |
| **Error** | _"The Premium Instalment Schedule (حفظة التوريد) needs both a PDF and its reference number. ⏎ حفظة التوريد يحتاج إلى ملف PDF ورقمه المرجعي معاً."_ / _"The Premium Instalment Schedule (حفظة التوريد) must be a PDF. ⏎ حفظة التوريد يجب أن يكون ملف PDF."_ / _"The Premium Instalment Schedule (حفظة التوريد) can be attached only once the payment schedule exists. ⏎ لا يمكن إرفاق حفظة التوريد إلا بعد إنشاء جدول السداد."_ |
| **FR** | FR-BASE-006-08-07 |
| **Tests** | `TestPremiumSchedule.test_03`–`test_05` |

---

### C-191 — A payment record carries its method's fields, and only those

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.cheque` |
| **File** | `models/insurance_cheque.py` (`_check_method_fields`; constant `PROOF_METHODS`) |
| **Constrains** | `payment_method`, `cheque_document`, `cheque_document_filename` |
| **Mechanism** | `@api.constrains`. Files are read as their sizes through `with_file_sizes()` (`tools/document_checks.py`), never loaded (R6, D-245). |
| **Condition** | a cheque or a bank transfer has no payment proof (`cheque_document`: the cheque's scan or the transfer's photo), or its proof is not a PDF or an image, or cash carries one; runs again when the method changes (D-175, D-254). Example: cash REC-0091 with a cheque scan → refused. |
| **Error** | _"Cash payment %(name)s cannot carry a payment proof. ⏎ الدفعة النقدية %(name)s لا يجوز أن تحمل إثبات سداد."_ / _"Cheque %s needs a scan of the physical cheque."_ / _"Bank transfer %(name)s needs a photo of the transfer. ⏎ التحويل البنكي %(name)s يحتاج صورة التحويل."_ / _"The payment proof of %(name)s must be a PDF or an image (.pdf, .png, .jpg, .jpeg). ⏎ إثبات السداد للدفعة %(name)s يجب أن يكون PDF أو صورة (.pdf, .png, .jpg, .jpeg)."_ |
| **FR** | FR-BASE-013-06-11 |
| **Tests** | `TestPaymentMethods.test_02`–`test_04`, `test_06` |

---

### C-192 — Only a cheque is deposited or bounced

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.cheque` |
| **File** | `models/insurance_cheque.py` (`_check_cheque_only_states`; constant `CHEQUE_ONLY_STATES`) |
| **Constrains** | `payment_method`, `state` |
| **Condition** | a state in `CHEQUE_ONLY_STATES` (`deposited`, `bounced`) with `payment_method != 'cheque'`. Cash and bank transfers go Received → Cleared (D-175). Catches the Deposit/Bounce buttons, a direct state write, and switching a deposited or bounced cheque to another method. Example: cash REC-0091 → Deposit refused; Clear accepted. |
| **Error** | _"Only a cheque can be deposited or bounced. ⏎ لا يمكن إيداع أو ارتداد إلا الشيك."_ |
| **FR** | FR-BASE-013-06-12 |
| **Tests** | `TestPaymentMethods.test_08` |

---

### C-193 — A cheque names its bank

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.cheque` |
| **File** | `models/insurance_cheque.py` (`_check_cheque_bank`) |
| **Constrains** | `payment_method`, `bank_id` |
| **Condition** | `payment_method == 'cheque'` and no `bank_id`. A bank transfer may name its bank; cash does not use it (D-176). Replaces the former `required=True` on `bank_id` (the column is nullable now). |
| **Error** | _"Cheque %(name)s needs its bank. ⏎ الشيك %(name)s يحتاج إلى البنك."_ |
| **FR** | FR-BASE-013-06-13 |
| **Tests** | `TestPaymentMethods.test_03` |

---

### C-202 — The attach wizard names exactly one record

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.premium.schedule.wizard` |
| **File** | `wizards/insurance_premium_schedule_wizard.py` (`_check_one_record`) |
| **Constrains** | `policy_id`, `endorsement_id`, `reference_number` (required, so the check also runs when neither record is given) |
| **Condition** | neither a policy nor an endorsement, or both. Example: a wizard on policy 42 and endorsement 7 at once is refused. |
| **Error** | _"Attach the receipt to one policy or one endorsement. ⏎ أرفق الحفظة بوثيقة واحدة أو بملحق واحد."_ |
| **FR** | FR-BASE-006-08-07 |
| **Tests** | `TestPremiumSchedule.test_08` |

---

### C-184 — Only an eligible installment goes in a batch

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.commission.batch.line` |
| **File** | `models/insurance_commission_batch_line.py` (`_check_eligible`) |
| **Constrains** | `installment_id`, `batch_id` |
| **Mechanism** | `_eligibility_errors()` runs `_ineligibility_reason()` as superuser (commission and bonus are accountant-only, the sysadmin batches too); later re-run by send and confirm. Replaces C-134 (retired in plan Task 18, when the batch moved onto its lines). |
| **Condition** | on a batch that is not cancelled, the line's installment: was deleted; belongs to another insurance company; is in another currency; is not paid; has no حفظة التوريد (read from `_premium_schedule_holder()`, never the stored flag, D-186); or earns no commission (slice + bonus is zero). A return (refund) installment is eligible (D-209, F2: the former "return installment" reason and the picker's `amount > 0` were removed). Example: POL-0042 #3, not paid → refused; a paid −450 refund (expected −45) → accepted. |
| **Error** | _"These installments cannot be put in a commission batch: ⏎ لا يمكن إدراج هذه الأقساط في دفعة عمولة: ⏎ - <installment>: <reason>"_, one reason per line |
| **FR** | FR-BASE-013-02-02 |
| **Tests** | `TestCommissionBatchLines.test_03`, `test_03b_a_return_installment_is_batched`, `test_07` |

---

### C-201 — A backward or cancel move needs a reason

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.commission.batch.reason.wizard`, `insurance.commission.batch` |
| **File** | `wizards/insurance_commission_batch_reason_wizard.py` (`reason`, `required=True`); `models/insurance_commission_batch.py` (`_transition`, `_REASON_MOVES`) |
| **Mechanism** | required field (NOT NULL) on the wizard, plus the `_transition()` check for a blank reason; the reason is posted on the batch chatter (D-158) |
| **Condition** | a move in `_REASON_MOVES` (sent → draft, draft → cancelled, sent → cancelled) with no reason or only spaces. Example: Cancel with reason `"  "` → refused; with "Insurer asked for a correction" → the chatter reads "Draft → Cancelled: Insurer asked for a correction". |
| **Error** | _"Give a reason to move batch %(batch)s back or to cancel it. ⏎ اذكر سبب إرجاع الدفعة %(batch)s أو إلغائها."_; on the wizard, Odoo's missing required value error |
| **FR** | FR-BASE-013-02-05 |
| **Tests** | `TestCommissionBatch.test_04_cancel_needs_a_reason_and_is_final` |

---

### C-205 — The batch's documents are PDFs with their references; a confirmed amount keeps its line's sign

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.commission.batch`, `insurance.commission.batch.line` |
| **File** | `models/insurance_commission_batch.py` (`_check_documents`); `models/insurance_commission_batch_line.py` (`_check_confirmed_amount`) |
| **Constrains** | batch: `company_invoice`, `company_invoice_filename`, `company_invoice_reference`, `commission_receipt`, `commission_receipt_filename`, `commission_receipt_reference_number`; line: `confirmed_amount`, `paid_amount` |
| **Mechanism** | `check_pdf_pair` in `tools/document_checks.py` (shared with C-182), once per document; a plain sign check on the line. Files are read as their sizes through `with_file_sizes()` (`tools/document_checks.py`), never loaded (R6, D-245). |
| **Condition** | per document: file xor reference, or a filename not `.pdf` (any case). Example: `INV-2026-55.pdf` + `INV-2026-55` → accepted; reference `INV-1` without a file → refused; `inv.xlsx` → refused. Line (D-209): `confirmed_amount < 0` on an ordinary line (`paid_amount` 0 or more), or `confirmed_amount > 0` on a return line (`paid_amount` below 0); an insurer that refuses a line confirms 0 (D-180). Example: expected 390 → −1 refused; expected −45 → −30 accepted, +1 refused. |
| **Error** | _"The company invoice needs both a PDF and its reference number. ⏎ فاتورة شركة التأمين يحتاج إلى ملف PDF ورقمه المرجعي معاً."_ / _"The commission receipt (حفظة العمولة) must be a PDF. ⏎ حفظة العمولة يجب أن يكون ملف PDF."_ (same two messages for each document) / _"A confirmed amount cannot be negative. ⏎ لا يجوز أن يكون المبلغ المؤكد سالباً."_ / _"A return installment's confirmed amount cannot be positive. ⏎ لا يجوز أن يكون المبلغ المؤكد لقسط مرتجع موجباً."_ |
| **FR** | FR-BASE-013-08-03 |
| **Tests** | `TestCommissionBatch.test_13_batch_documents_are_pdfs_with_references`, `TestCommissionBatch.test_13b_a_return_line_is_confirmed_at_zero_or_below`, `TestCommissionBatch.test_14_a_confirmed_amount_is_never_negative`, `test_23b_confirmed_amounts_are_corrected_until_the_invoice` (in Confirmed, D-229) |

### C-210 — An endorsement is applied only with its document

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.policy.endorsement` |
| **File** | `models/insurance_policy_endorsement.py` (`_check_applied_document`, `_check_document_attached`) |
| **Constrains** | `is_applied`, `policy_document` |
| **Mechanism** | `@api.constrains` on the endorsement; `_schedule_endorsement()` and `_do_apply_changes()` call the same `_check_document_attached()` after their other validations, so Apply and Schedule are refused before any change line runs. Files are read as their sizes through `with_file_sizes()` (`tools/document_checks.py`), never loaded (R6, D-245). |
| **Condition** | the endorsement is applied (or being applied or scheduled) and `policy_document` (from `insurance.commission.mixin`, D-219) is empty. A draft may be saved without it; an applied endorsement may replace it, not remove it; a reverted one may drop it. Example: END-0007 with no PDF → Apply refused; with `END-0007.pdf` → applied |
| **Error** | _"Endorsement %(name)s cannot be applied without its document. Upload the endorsement document in the Documents page first. ⏎ لا يمكن تطبيق الملحق %(name)s بدون مستنده. ارفع مستند الملحق في صفحة المستندات أولاً."_ |
| **FR** | FR-BASE-007-08-14 |
| **Tests** | `TestEndorsementDocument.test_02_applying_without_the_document_is_refused`, `test_03_scheduling_without_the_document_is_refused`, `test_05_an_applied_endorsement_keeps_its_document`, `test_06_a_reverted_endorsement_may_drop_its_document` |

### C-217 — A Lost request has its reason

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.request` |
| **File** | `models/insurance_request.py` (`_check_lost_has_reason`) |
| **Constrains** | `state`, `failure_reason_id` |
| **Condition** | `state` is `failed` and `failure_reason_id` is empty. Example: `write({'state': 'failed'})` on REQ-0012 → refused; the Lost button with the reason "Too expensive" → Lost |
| **Error** | _"Request %s cannot be Lost without a reason."_ |
| **FR** | FR-BASE-004-09-02 |
| **Tests** | `TestRequestStages.test_lost_needs_its_reason` |

### C-218 — A Won request has its policy

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.request` |
| **File** | `models/insurance_request.py` (`_check_won_has_policy`) |
| **Constrains** | `state` |
| **Condition** | `state` is `won` and `policy_generated` is false. Example: `write({'state': 'won'})` on an open REQ-0012 → refused. Deleting POL-0042, the policy of a Won request, is allowed: `_reopen_requests` moves the offer, then the request, back (FR-BASE-006-12-06) |
| **Error** | _"Request %s cannot be Won without a policy."_ |
| **FR** | FR-BASE-004-02-07, FR-BASE-006-12-06 |
| **Tests** | `TestRequestStages.test_won_needs_its_policy`, `test_deleting_the_policy_reopens_its_request` |

### C-253 — A vehicle request needs a vehicle before it gets an offer

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **Model** | `insurance.offer` |
| **File** | `models/insurance_offer.py` (`_check_request_has_vehicle`) |
| **Constrains** | `insurance_request_id`, `insurance_type_id` |
| **Condition** | The offer's type applies to vehicles and its request has no vehicles. Example: an offer created on REQ-0031 (Private Motor, 0 vehicles) → refused; after one vehicle is added to REQ-0031 → created |
| **Error** | _"Request '%s' has no vehicles. Add at least one vehicle before creating offers."_ |
| **FR** | FR-VEH-004-02-01 |
| **Tests** | `TestVehicleConstraints.test_offer_refused_on_request_without_vehicles`, `TestVehicleConstraints.test_offer_cannot_move_to_request_without_vehicles` |

### C-254 — A number plate is an Egyptian plate of the 2008 format or later

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **Model** | `client.insurable.vehicle` |
| **File** | `models/client_insurable_vehicle.py` (`_check_number_plate_format`); rule in `tools/number_plate.py` |
| **Constrains** | `number_plate` |
| **Condition** | The plate is filled and is not 1 to 3 letter-section symbols (at least one letter, from أ ب ج د ر س ص ط ع ف ق ل م ن ه و ى; D-297) followed by 1 to 4 digits. `create()` and `write()` first rewrite the typed plate to its stored spelling. Digits are always stored as Arabic-Indic digits; Latin digits are converted on save (D-302). Example: `ا ب ج 1234` → saved as `أ ب ج ١٢٣٤`; `أ 1` → saved as `أ ١` (custom); `231 1 أ س` → saved as `١ أ س ٢٣١` (diplomatic); `أبج 1234` and `231 1أس` (no space between the letters, D-299) → refused; `12345` (pre-2008, no letter), `ش ش 1` (ش is not a plate letter), `ABC-1234`, `أ ب ج د ١٢٣` → refused. An empty plate is allowed. The vehicle import applies the same rule per row (FR-VEH-008-02-05). Records saved before this rule are checked only when their plate is next written |
| **Error** | _"'%(plate)s' is not a valid Egyptian number plate (vehicle %(name)s). Accepted plates: ..."_ then the same in Arabic: _"«%(plate)s» ليست لوحة معدنية مصرية صحيحة (المركبة %(name)s). اللوحات المقبولة: ..."_ |
| **FR** | FR-VEH-001-04-01, FR-VEH-001-04-02, FR-VEH-001-04-03, FR-VEH-001-04-05, FR-VEH-001-04-06 |
| **Tests** | `TestVehicleNumberPlate.test_accepted_plates_are_saved_in_the_stored_spelling`, `test_refused_plates`, `test_empty_plate_is_allowed`, `test_write_follows_the_same_rule`, `test_import_refuses_a_plate_that_is_not_egyptian` |

### C-246 — A rejected claim has its reason

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.claim` |
| **File** | `models/insurance_claim.py` (`_check_rejected_has_reason`, `write`) |
| **Constrains** | `state`, `denial_reason_id` |
| **Condition** | `state` is `rejected` and `denial_reason_id` is empty; and a user write of `state = 'rejected'` (a drag or status-bar move) is refused in `write()`, only the Reject wizard (`_reject`, superuser) rejects. `_reject` first runs `check_access('write')` as the caller, so a user who can only read claims (ACL or record rule) gets an `AccessError` and the claim stays open. Example: dragging the card of CLM/0042 onto Rejected → refused; the Reject button with the reason "Not covered by the policy" → Rejected |
| **Error** | _"Use the Reject button: it asks for the reason."_ (write); _"Claim %s cannot be Rejected without a reason."_ (constraint) |
| **FR** | FR-BASE-008-05-04 |
| **Tests** | `TestClaimStatus.test_drag_to_rejected_is_refused`, `test_the_reason_is_required`, `test_any_open_state_can_be_rejected`, `test_reject_refused_without_write_access` |

### C-231 — An item names what its purpose carries

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_collection` |
| **Model** | `insurance.collection.delivery.item` |
| **File** | `models/collection_trip_item.py` (`_check_document_required`) |
| **Constrains** | `purpose`, `related_document_id` |
| **Condition** | the purpose is in `PURPOSE_MODEL_MAP` and `related_document_id` is empty. `other` needs none, but such a trip is then not assigned (C-237). Example: a Cheque item with no cheque → refused |
| **Error** | _"A related document is required for '%s' items."_ |
| **FR** | FR-BASE-015-02-01, FR-COL-002-02-04 |
| **Tests** | `TestPendingItems.test_a_cheque_item_needs_its_cheque` |

### C-232 — A trip carries one team's items

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_collection` |
| **Model** | `insurance.collection.delivery.item` |
| **File** | `models/collection_trip_item.py` (`_check_one_team`; also checked first in `action_create_trip`, C-240) |
| **Constrains** | `trip_id`, `team` |
| **Condition** | the item's team differs from its trip's. Example: claim papers put on a collection-team trip → refused |
| **Error** | _"%(item)s is the %(team)s team's: it cannot go on %(trip)s."_ |
| **FR** | FR-COL-013-02-01 (D-40) |
| **Tests** | `TestClaimNeeds.test_a_trip_carries_one_teams_items` |

### C-233 — Failed needs its reason; Left at Office only on a direct leg

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_collection` |
| **Model** | `insurance.collection.delivery.item` |
| **File** | `models/collection_trip_item.py` (`_check_result`) |
| **Constrains** | `result`, `failure_reason`, `leg` |
| **Condition** | `result` is `failed` with a blank `failure_reason`; or `result` is `left_at_office` and the leg is not in `ONWARD` (insurer → client or client → insurer, straight). Example: Failed on the policy document with no reason → refused; Left at Office on an office → client item → refused |
| **Error** | _"Write the failure reason before marking it failed: %s"_; _"Only an item going straight from the insurer to the client, or back, is left at the office: %s"_ |
| **FR** | FR-COL-012-05-02, FR-COL-001-06 |
| **Tests** | `TestTripFlow.test_an_item_fails_only_with_its_reason`, `TestLeftAtOffice.test_only_a_direct_item_on_the_road_is_left_at_the_office` |

### C-234 — A report period runs forward

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_collection` |
| **Model** | `collection.payment.report.wizard`, `collection.route.plan.wizard` |
| **File** | `wizard/collection_payment_report_wizard.py` and `wizard/collection_route_plan_wizard.py` (`_check_period`) |
| **Constrains** | `date_from`, `date_to` |
| **Condition** | `date_from > date_to`. Example: 1 Nov to 31 Oct → refused |
| **Error** | _"The period ends before it starts."_ |
| **FR** | FR-COL-011-01-01, FR-COL-009-01-01 |
| **Tests** | `TestPaymentReport.test_the_period_is_this_month_by_default_and_runs_forward` (the route plan's is not tested) |

---

## 4. SQL Constraints (`models.Constraint`)

### 4a. UNIQUE Constraints

### C-27 — Policy number must be unique

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.policy` |
| **File** | `models/insurance_policy.py` |
| **SQL** | `UNIQUE(policy_number)` |
| **Error** | _"Policy number must be unique!"_ |

### C-28 — Endorsement number must be unique

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.policy.endorsement` |
| **File** | `models/insurance_policy_endorsement.py` |
| **SQL** | `UNIQUE(endorsement_number)` |
| **Error** | _"Endorsement number must be unique!"_ |

### C-29 — Insurance company partner must be unique

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.company` |
| **File** | `models/insurance_company.py` |
| **SQL** | `UNIQUE(partner_id)` |
| **Error** | _"A partner can only be associated with one insurance company."_ |

### C-30 — Insurance company name must be unique

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.company` |
| **File** | `models/insurance_company.py` |
| **SQL** | `UNIQUE(name)` |
| **Error** | _"Insurance company name must be unique!"_ |

### C-31 — Each client can have only one corporate record

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `client.corporate` |
| **File** | `models/client_corporate.py` |
| **SQL** | `UNIQUE(client_id)` |
| **Error** | _"Each client can have only one corporate record!"_ |

### C-32 — Each client can have only one individual record

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `client.individual` |
| **File** | `models/client_individual.py` |
| **SQL** | `UNIQUE(client_id)` |
| **Error** | _"Each client can have only one individual record!"_ |

### C-33 — Individual ID number must be unique

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `client.individual` |
| **File** | `models/client_individual.py` |
| **SQL** | `UNIQUE(id_number)` |
| **Error** | _"ID number must be unique!"_ |

### C-34 — Employment start date unique per individual and corporate

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `corporate.client.individual` |
| **File** | `models/corporate_client_individual.py` |
| **SQL** | `UNIQUE(corporate_client_id, individual_id, start_date)` |
| **Error** | _"Each individual can only have one employment with the same company starting on the same date!"_ |

### C-35 — Claim denial reason name must be unique

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `claim.denial.reason` |
| **File** | `models/claim_denial_reason.py` |
| **SQL** | `UNIQUE(name)` |
| **Error** | _"Claim denial reason must be unique!"_ |

### C-36 — Unique insurable item per offer/policy/request

This pattern is replicated across all type-specific junction tables:

| Sub-Control | Module | Model | SQL | Error |
|-------------|--------|-------|-----|-------|
| **C-36** | `medical` | `offer.has.employee` | `UNIQUE(offer_id, employee_id, COALESCE(category_id, 0))` | _"Each employee can only be added once per offer and category!"_ |
| **C-36-01** | `medical` | `policy.has.employee` | `UNIQUE(policy_id, employee_id, COALESCE(category_id, 0))` | _"Each employee can only be added once per policy and category!"_ |
| **C-36-02** | `medical` | `request.has.employee` | `UNIQUE(insurance_request_id, employee_id, COALESCE(category_id, 0))` | _"Each employee can only be added once per request and category!"_ |
| **C-36-03** | `vehicle` | `offer.has.vehicle` | `UNIQUE(offer_id, vehicle_id)` | _"Each vehicle can only be added once per offer!"_ |
| **C-36-04** | `vehicle` | `policy.has.vehicle` | `UNIQUE(policy_id, vehicle_id)` | _"Each vehicle can only be added once per policy!"_ |
| **C-36-05** | `vehicle` | `request.has.vehicle` | `UNIQUE(insurance_request_id, vehicle_id)` | _"Each vehicle can only be added once per request!"_ |
| **C-36-06** | `shipment` | `offer.has.shipment` | `UNIQUE(offer_id, shipment_id)` | _"Each shipment can only be added once per offer!"_ |
| **C-36-07** | `shipment` | `policy.has.shipment` | `UNIQUE(policy_id, shipment_id)` | _"Each shipment can only be added once per policy!"_ |
| **C-36-08** | `shipment` | `request.has.shipment` | `UNIQUE(insurance_request_id, shipment_id)` | _"Each shipment can only be added once per request!"_ |
| **C-36-09** | `general` | `offer.has.estate` | `UNIQUE(offer_id, estate_id)` | _"Each estate can only be added once per offer!"_ |
| **C-36-10** | `general` | `policy.has.estate` | `UNIQUE(policy_id, estate_id)` | _"Each estate can only be added once per policy!"_ |
| **C-36-11** | `general` | `request.has.estate` | `UNIQUE(insurance_request_id, estate_id)` | _"Each estate can only be added once per request!"_ |

### C-37 — Medical category number unique per parent

| Sub-Control | Module | Model | SQL | Error |
|-------------|--------|-------|-----|-------|
| **C-37** | `medical` | `insurance.offer.medical.category` | `UNIQUE(offer_id, category_number)` | _"Category number must be unique per offer!"_ |
| **C-37-01** | `medical` | `insurance.request.medical.category` | `UNIQUE(insurance_request_id, category_number)` | _"Category number must be unique per request!"_ |
| **C-37-02** | `medical` | `insurance.policy.medical.category` | `UNIQUE(policy_id, category_number)` | _"Category number must be unique per policy!"_ |

### C-38 — Unique benefit type per medical category

| Sub-Control | Module | Model | SQL | Error |
|-------------|--------|-------|-----|-------|
| **C-38** | `medical` | `offer.category.has.benefit.type` | `UNIQUE(category_id, coverage_id)` | _"Each benefit type can only appear once per category!"_ |
| **C-38-01** | `medical` | `policy.category.has.benefit.type` | `UNIQUE(category_id, coverage_id)` | _"Each benefit type can only appear once per category!"_ |
| **C-38-02** | `medical` | `request.category.has.benefit.type` | `UNIQUE(category_id, coverage_id)` | _"Each benefit type can only appear once per category!"_ |

### C-39 — Unique coverage item per benefit type line

| Sub-Control | Module | Model | SQL | Error |
|-------------|--------|-------|-----|-------|
| **C-39** | `medical` | `offer.benefit.has.coverage.item` | `UNIQUE(benefit_type_line_id, coverage_id)` | _"Each coverage item can only appear once per benefit type line!"_ |
| **C-39-01** | `medical` | `policy.benefit.has.coverage.item` | `UNIQUE(benefit_type_line_id, coverage_id)` | _"Each coverage item can only appear once per benefit type line!"_ |

### C-40 — Vehicle inspection line unique per inspection

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **Model** | `vehicle.inspection.line` |
| **File** | `models/vehicle_inspection_line.py` |
| **SQL** | `UNIQUE(inspection_id, vehicle_id)` |
| **Error** | _"Each vehicle can only appear once per inspection!"_ |

### C-41 — Vehicle model name unique per manufacturer

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **Model** | `vehicle.model` |
| **File** | `models/vehicle_model.py` |
| **SQL** | `UNIQUE(name, manufacturer_id)` |
| **Error** | _"Model name must be unique per manufacturer!"_ |

### C-42 — Vehicle manufacturer name must be unique

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **Model** | `vehicle.manufacturer` |
| **File** | `models/vehicle_manufacturer.py` |
| **SQL** | `UNIQUE(name)` |
| **Error** | _"Manufacturer name must be unique!"_ |

### C-43 — Vehicle part name must be unique

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **Model** | `vehicle.part` |
| **File** | `models/vehicle_part.py` |
| **SQL** | `UNIQUE(name)` |
| **Error** | _"Vehicle part name must be unique!"_ |

### C-44 — Year of production unique per vehicle model

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **Model** | `vehicle.year.of.prod` |
| **File** | `models/vehicle_year_of_prod.py` |
| **SQL** | `UNIQUE(vehicle_model_id, year)` |
| **Error** | _"Year of production must be unique per model!"_ |

### C-45 — Service centre unique per manufacturer

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **Model** | `vehicle.manufacturer.service.centre` |
| **File** | `models/vehicle_manufacturer_service_centre.py` |
| **SQL** | `UNIQUE(service_centre_id, vehicle_manufacturer_id)` |
| **Error** | _"Service centre is already linked to this manufacturer!"_ |

### C-46 — One shipment record per insurance request

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_shipment` |
| **Model** | `insurance.request.shipment` |
| **File** | `models/insurance_request_shipment.py` |
| **SQL** | `UNIQUE(request_id)` |
| **Error** | _"Only one shipment record is allowed per insurance request!"_ |

### C-47 — Packing method name must be unique

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_shipment` |
| **Model** | `shipment.packing.method` |
| **File** | `models/shipment_packing_method.py` |
| **SQL** | `UNIQUE(name)` |
| **Error** | _"Packing method name must be unique!"_ |

### C-48 — Estate type name must be unique

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_general` |
| **Model** | `estate.type` |
| **File** | `models/estate_type.py` |
| **SQL** | `UNIQUE(name)` |
| **Error** | _"Estate type name must be unique!"_ |

### C-185 — One active batch per installment

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.commission.batch.line` |
| **File** | `models/insurance_commission_batch_line.py` (`_unique_live_installment`, `batch_state`) |
| **SQL** | `models.UniqueIndex("(installment_id) WHERE batch_state != 'cancelled'")`, index `insurance_commission_batch_line_unique_live_installment`. `batch_state` is the stored related `batch_id.state`, system-set (C-187). Moved here from the Python constraints by F3 (D-211): the "already in commission batch" search in `_ineligibility_reason()` and the send lock `_lock_installments()` are gone. |
| **Condition** | the installment is on another line of a batch that is not cancelled, this batch included. Postgres also refuses it across transactions: two batches saved at the same moment cannot both hold the row (the former "Known limit" is closed). Example: POL-0042 #1 in CB/2026/0001 (draft), added to a second batch → refused; once CB/2026/0001 is cancelled, or its line removed, it is accepted. `is_commission_batched` reads the same condition. |
| **Error** | _"This installment is already in a commission batch that is not cancelled. ⏎ هذا القسط مدرج بالفعل في دفعة عمولة غير ملغاة."_ (an `IntegrityError` in Python; the RPC layer shows the message) |
| **FR** | FR-BASE-013-02-03 |
| **Tests** | `TestCommissionBatchLines.test_04`, `test_04b_a_sent_batch_holds_its_row` |

### 4b. CHECK Constraints

### C-49 — Contract early payment reward days must be positive

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `contract.early.payment.reward` |
| **File** | `models/contract_early_payment_reward.py` |
| **SQL** | `CHECK(days_valid > 0)` |
| **Error** | _"Days valid must be greater than zero!"_ |

### C-50 — Employment end date >= start date (SQL level)

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `corporate.client.individual` |
| **File** | `models/corporate_client_individual.py` |
| **SQL** | `CHECK(end_date IS NULL OR end_date >= start_date)` |
| **Error** | _"Employment end date must be on or after start date!"_ |

### C-51 — Medical category number must be positive

| Sub-Control | Module | Model | SQL | Error |
|-------------|--------|-------|-----|-------|
| **C-51** | `medical` | `insurance.offer.medical.category` | `CHECK(category_number > 0)` | _"Category number must be positive!"_ |
| **C-51-01** | `medical` | `insurance.request.medical.category` | `CHECK(category_number > 0)` | _"Category number must be positive!"_ |
| **C-51-02** | `medical` | `insurance.policy.medical.category` | `CHECK(category_number > 0)` | _"Category number must be positive!"_ |

### C-52 — Benefit type annual limit cannot be negative

| Sub-Control | Module | Model | SQL | Error |
|-------------|--------|-------|-----|-------|
| **C-52** | `medical` | `offer.category.has.benefit.type` | `CHECK(annual_limit >= 0 OR annual_limit IS NULL)` | _"Benefit type annual limit cannot be negative!"_ |
| **C-52-01** | `medical` | `policy.category.has.benefit.type` | `CHECK(annual_limit >= 0 OR annual_limit IS NULL)` | _"Benefit type annual limit cannot be negative!"_ |

### C-53 — Waiting period months cannot be negative

| Sub-Control | Module | Model | SQL | Error |
|-------------|--------|-------|-----|-------|
| **C-53** | `medical` | `offer.category.has.benefit.type` | `CHECK(waiting_period_months >= 0 OR waiting_period_months IS NULL)` | _"Waiting period months cannot be negative!"_ |
| **C-53-01** | `medical` | `policy.category.has.benefit.type` | `CHECK(waiting_period_months >= 0 OR waiting_period_months IS NULL)` | _"Waiting period months cannot be negative!"_ |

### C-54 — Additional premium cannot be negative

| Sub-Control | Module | Model | SQL | Error |
|-------------|--------|-------|-----|-------|
| **C-54** | `medical` | `offer.category.has.benefit.type` | `CHECK(additional_premium >= 0 OR additional_premium IS NULL)` | _"Additional premium cannot be negative!"_ |
| **C-54-01** | `medical` | `policy.category.has.benefit.type` | `CHECK(additional_premium >= 0 OR additional_premium IS NULL)` | _"Additional premium cannot be negative!"_ |

### C-55 — Vehicle price must be positive

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **Model** | `vehicle.year.of.prod` |
| **File** | `models/vehicle_year_of_prod.py` |
| **SQL** | `CHECK(price >= 0)` |
| **Error** | _"Price must be positive!"_ |

### C-56 — Vehicle minimum price must not exceed maximum

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **Model** | `vehicle.year.of.prod` |
| **File** | `models/vehicle_year_of_prod.py` |
| **SQL** | `CHECK(minimum_price <= maximum_price)` |
| **Error** | _"Minimum price must not exceed maximum price!"_ |

### C-57 — Vehicle average price must be within range

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **Model** | `vehicle.year.of.prod` |
| **File** | `models/vehicle_year_of_prod.py` |
| **SQL** | `CHECK(minimum_price <= price AND price <= maximum_price)` |
| **Error** | _"Average price must be between minimum and maximum price!"_ |

### C-139 — Cheque amount cannot be zero

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.cheque` |
| **File** | `models/insurance_cheque.py` |
| **SQL** | `CHECK(amount <> 0)` |
| **Error** | _"A cheque amount cannot be zero."_ |
| **FR** | FR-BASE-013-06-01 — the amount is signed (a refund cheque is negative), so only zero is invalid |

### C-161 — A request, an offer and a policy have at least one cheque

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.request`; `insurance.base.mixin`, and through it `insurance.offer` and `insurance.policy` |
| **File** | `models/insurance_request.py`, `models/mixins/insurance_base_mixin.py` (`_check_number_of_cheques` on both) |
| **SQL** | `CHECK(number_of_cheques >= 1)` |
| **Error** | _"The number of cheques must be at least 1."_ (request) / _"The number of payment cheques must be at least 1."_ (offer, policy) |
| **FR** | FR-BASE-004-05-03, FR-BASE-005-05-04, FR-BASE-006-02-05 |
| **Purpose** | D-090. The number of cheques is the number of installments the accept-offer wizard builds (FR-BASE-006-11-01); zero or a negative number is not a payment plan. Both fields default to 1. A request saved before the rule can hold no number (NULL), which the check does not look at; an offer created for such a request keeps the default of 1 instead of copying 0. |

### C-166 — One payout per incentive and period

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.incentive.payout` |
| **File** | `models/insurance_incentive_payout.py` (`_unique_period`) |
| **SQL** | `UNIQUE(incentive_id, period_start)` |
| **Error** | _"An incentive has one payout per period. ⏎ للحافز دفعة واحدة لكل فترة."_ |
| **FR** | FR-BASE-013-07-01, BR-BASE-010-04-01 |

### C-211 — A contract is renewed once

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.company.contract` |
| **File** | `models/insurance_company_contract.py` (`_unique_renewed_from`) |
| **SQL** | `UNIQUE(renewed_from_id)` (NULLs do not clash, so contracts that renew nothing are free). No Python pre-check: the index is the rule (D-226); the Renew button hides once renewed (C-214) |
| **Condition** | a second contract with the same `renewed_from_id`. Example: the 2031 contract already renewed as 2032 → a second Renew is refused; once the 2032 contract is deleted, Renew works again |
| **Error** | _"This contract has already been renewed. ⏎ تم تجديد هذا العقد بالفعل."_ (an `IntegrityError` in Python; the RPC layer shows the message) |
| **FR** | FR-BASE-009-04-13 |
| **Tests** | `TestContractRenewal.test_05_a_contract_is_renewed_once` |

### C-220 — A policy has one renewal request, Lost or not

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.request`, `insurance.policy` |
| **File** | `models/insurance_request.py` (`_one_renewal_per_policy`); `models/insurance_policy.py` (`_create_renewal_request` pre-check, `_cron_create_renewal_requests` domain `('renewal_request_id', '=', False)`) |
| **SQL** | `UNIQUE(previous_policy_id) WHERE previous_policy_id IS NOT NULL` (a partial unique index: requests that renew nothing are free). The Python pre-check gives the readable message first; the index is the final guard |
| **Condition** | a second request with the same `previous_policy_id`. Example: POL-0042 has REQ-A, draft or Lost → the daily job skips POL-0042, and `_create_renewal_request` is refused; a Lost REQ-A comes back with Reopen by an administrator (FR-BASE-004-09-03), never as a second request |
| **Error** | Pre-check: _"Policy %(policy)s already has its renewal request (%(request)s). A Lost one is reopened by an administrator."_ Index: _"This policy already has its renewal request."_ (an `IntegrityError` in Python) |
| **FR** | BR-BASE-004-02-05, FR-BASE-004-06-05 |
| **Tests** | `TestPolicyRenewal.test_one_renewal_request_per_policy`, `test_job_creates_the_renewal_request`, `test_not_renewing_on_the_renewal_request` |

---

## 5. Foreign Key Restrictions (`ondelete='restrict'`)

These prevent deletion of parent records when child records reference them.

### Base Module (`optimum_insurance_base`)

| Control | Model | Field | Prevents Deletion Of | File |
|---------|-------|-------|---------------------|------|
| **C-58** | `insurance.policy` | `policy_holder_id` | `client` with policies | `models/insurance_policy.py` |
| **C-59** | `insurance.policy` | `offer_id` | `insurance.offer` with policies | `models/insurance_policy.py` |
| **C-60** | `insurance.policy` | `insurance_type_id` | `insurance.type` with policies | `models/insurance_policy.py` |
| **C-61** | `insurance.offer` | `insurance_request_id` | `insurance.request` with offers | `models/insurance_offer.py` |
| **C-62** | `insurance.offer` | `insurance_type_id` | `insurance.type` with offers | `models/insurance_offer.py` |
| **C-63** | `insurance.policy.endorsement` | `policy_id` | `insurance.policy` with endorsements | `models/insurance_policy_endorsement.py` |
| **C-64** | `crm.lead` | `district_id` | `res.country.state.district` with leads | `models/crm_lead.py` |
| **C-65** | `crm.lead` | `area_id` | `res.country.state.district.area` with leads | `models/crm_lead.py` |
| **C-66** | `client` | `district_id` | `res.country.state.district` with clients | `models/client.py` |
| **C-67** | `client` | `area_id` | `res.country.state.district.area` with clients | `models/client.py` |
| **C-68** | `client.corporate` | `industry_id` | `res.partner.industry` with corporates | `models/client_corporate.py` |
| **C-69** | `corporate.client.individual` | `insurance_type_id` | `insurance.type` with employments | `models/corporate_client_individual.py` |
| **C-70** | `insurance.company.has.employee` | `insurance_type_id` | `insurance.type` with company employees | `models/insurance_company_employee.py` |
| **C-148** | `insurance.cheque` | `bank_id` | `res.bank` with cheques | `models/insurance_cheque.py` |
| **C-148** | `insurance.cheque` | `client_id` | `client` with cheques | `models/insurance_cheque.py` |
| **C-148** | `insurance.cheque` | `insurance_company_id` | `insurance.company` with cheques | `models/insurance_cheque.py` |
| **C-158** | `policy.payment.schedule` | `bonus_tier_id` | `contract.early.payment.reward` (a contract's early payment tier, and so its contract, which cascades to it) that an installment earned its bonus on. D-030: the installment links the tier row instead of copying its numbers, so the row must outlive the link (FR-BASE-006-11-25, D-069 A8) | `models/policy_payment_schedule.py` |
| **C-167** | `insurance.incentive.payout` | `incentive_id` | `insurance.incentive` with payouts. An incentive that has payouts cannot be deleted, nor can its contract (BR-BASE-010-03-05). | `models/insurance_incentive_payout.py` |
| **C-176** | `insurance.incentive.figure` | `insurance_type_ids` (relation `incentive_figure_type_rel.type_id` → `insurance_type`) | `insurance.type` that a figure uses. Deleting it would silently drop it from the figure (a cascade), and a figure limited to that one type would then read every type, changing a locked figure's value (C-173). FR-BASE-009-05-03, D-133. | `models/insurance_incentive_figure.py` |
| **C-213** | `insurance.company.contract` | `renewed_from_id` | `insurance.company.contract` that has a renewal. The renewal chain is an audit link (odoo-workflow-integrity); deleting the renewal itself is allowed and frees the old contract for a new Renew (FR-BASE-009-04-13, D-226). Test: `TestContractRenewal.test_07_a_renewed_contract_cannot_be_deleted` | `models/insurance_company_contract.py` |

### Medical Module (`optimum_insurance_medical`)

| Control | Model | Field | Prevents Deletion Of | File |
|---------|-------|-------|---------------------|------|
| **C-71** | `offer.category.has.benefit.type` | `coverage_id` | `insurance.coverage` with benefit types | `models/offer_category_has_benefit_type.py` |
| **C-72** | `offer.benefit.has.coverage.item` | `coverage_id` | `insurance.coverage` with coverage items | `models/offer_benefit_has_coverage_item.py` |
| **C-73** | `offer.has.employee` | `employee_id` | `client.insurable.employee` in offers | `models/offer_has_employee.py` |
| **C-74** | `policy.category.has.benefit.type` | `coverage_id` | `insurance.coverage` with policy benefits | `models/policy_category_has_benefit_type.py` |
| **C-75** | `policy.benefit.has.coverage.item` | `coverage_id` | `insurance.coverage` with policy items | `models/policy_benefit_has_coverage_item.py` |
| **C-76** | `policy.has.employee` | `employee_id` | `client.insurable.employee` in policies | `models/policy_has_employee.py` |
| **C-77** | `request.category.has.benefit.type` | `coverage_id` | `insurance.coverage` with request benefits | `models/request_category_has_benefit_type.py` |
| **C-78** | `request.has.employee` | `employee_id` | `client.insurable.employee` in requests | `models/request_has_employee.py` |
| **C-79** | `client.insurable.employee` | `client_id` | `client` with insurable employees | `models/client_insurable_employee.py` |
| **C-80** | `offer.beneficiary` | `employee_id` | `client.insurable.employee` as beneficiaries | `models/offer_beneficiary.py` |
| **C-81** | `policy.beneficiary` | `employee_id` | `client.insurable.employee` as beneficiaries | `models/policy_beneficiary.py` |

### Vehicle Module (`optimum_insurance_vehicle`)

| Control | Model | Field | Prevents Deletion Of | File |
|---------|-------|-------|---------------------|------|
| **C-82** | `offer.has.vehicle` | `vehicle_id` | `client.insurable.vehicle` in offers | `models/offer_has_vehicle.py` |
| **C-83** | `policy.has.vehicle` | `vehicle_id` | `client.insurable.vehicle` in policies | `models/policy_has_vehicle.py` |
| **C-84** | `request.has.vehicle` | `vehicle_id` | `client.insurable.vehicle` in requests | `models/request_has_vehicle.py` |
| **C-85** | `vehicle.inspection.line` | `vehicle_id` | `client.insurable.vehicle` in inspections | `models/vehicle_inspection_line.py` |
| **C-86** | `vehicle.model` | `manufacturer_id` | `vehicle.manufacturer` with models | `models/vehicle_model.py` |
| **C-87** | `client.insurable.vehicle` | `client_id` | `client` with insurable vehicles | `models/client_insurable_vehicle.py` |
| **C-88** | `client.insurable.vehicle` | `vehicle_year_of_prod_id` | `vehicle.year.of.prod` with vehicles | `models/client_insurable_vehicle.py` |
| **C-89** | `insurance.company.service.centres` | `insurance_company_id` | `insurance.company` with service centres | `models/insurance_company_service_centres.py` |
| **C-90** | `insurance.company.service.centres` | `service_centre_id` | service centre with company authorizations | `models/insurance_company_service_centres.py` |

### Shipment Module (`optimum_insurance_shipment`)

| Control | Model | Field | Prevents Deletion Of | File |
|---------|-------|-------|---------------------|------|
| **C-91** | `offer.has.shipment` | `shipment_id` | `client.insurable.shipment` in offers | `models/offer_has_shipment.py` |
| **C-92** | `policy.has.shipment` | `shipment_id` | `client.insurable.shipment` in policies | `models/policy_has_shipment.py` |
| **C-93** | `request.has.shipment` | `shipment_id` | `client.insurable.shipment` in requests | `models/request_has_shipment.py` |
| **C-94** | `insurance.request.shipment` | `export_from_country_id` | `res.country` in export config | `models/insurance_request_shipment.py` |
| **C-95** | `insurance.request.shipment` | `export_from_state_id` | `res.country.state` in export config | `models/insurance_request_shipment.py` |
| **C-96** | `insurance.request.shipment` | `export_to_country_id` | `res.country` in export config | `models/insurance_request_shipment.py` |
| **C-97** | `insurance.request.shipment` | `export_to_state_id` | `res.country.state` in export config | `models/insurance_request_shipment.py` |
| **C-98** | `insurance.request.shipment` | `import_from_country_id` | `res.country` in import config | `models/insurance_request_shipment.py` |
| **C-99** | `insurance.request.shipment` | `import_from_state_id` | `res.country.state` in import config | `models/insurance_request_shipment.py` |
| **C-100** | `insurance.request.shipment` | `import_to_country_id` | `res.country` in import config | `models/insurance_request_shipment.py` |
| **C-101** | `insurance.request.shipment` | `import_to_state_id` | `res.country.state` in import config | `models/insurance_request_shipment.py` |
| **C-102** | `insurance.request.shipment` | `packing_method_id` | `shipment.packing.method` in requests | `models/insurance_request_shipment.py` |

### General Module (`optimum_insurance_general`)

| Control | Model | Field | Prevents Deletion Of | File |
|---------|-------|-------|---------------------|------|
| **C-103** | `offer.has.estate` | `estate_id` | `client.insurable.estate` in offers | `models/offer_has_estate.py` |
| **C-104** | `policy.has.estate` | `estate_id` | `client.insurable.estate` in policies | `models/policy_has_estate.py` |
| **C-105** | `request.has.estate` | `estate_id` | `client.insurable.estate` in requests | `models/request_has_estate.py` |

---

## 6. Action Method Guards

### C-106 — Cannot terminate already-terminated employment

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `corporate.client.individual` |
| **File** | `models/corporate_client_individual.py` (line ~269) |
| **Method** | `action_terminate_employment()` |
| **Condition** | Employment already has end date in the past |
| **Error** | _"Employment is already terminated (ended on %s)"_ |

### C-107 — Cannot extend employment with no end date

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `corporate.client.individual` |
| **File** | `models/corporate_client_individual.py` (line ~297) |
| **Method** | `action_extend_employment()` |
| **Condition** | Employment has no end date (already active indefinitely) |
| **Error** | _"Employment is already active (no end date set)"_ |

### C-108 — Cannot apply already-applied endorsement

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.policy.endorsement` |
| **File** | `models/insurance_policy_endorsement.py` (line ~200) |
| **Method** | `action_apply_changes()` |
| **Condition** | Endorsement `is_applied` is already True |
| **Error** | _"This endorsement has already been applied."_ |

### C-109 — Cannot reset applied endorsement to draft

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.policy.endorsement` |
| **File** | `models/insurance_policy_endorsement.py` (line ~252) |
| **Method** | `action_reset_to_draft()` |
| **Condition** | Endorsement `is_applied` is True |
| **Error** | _"Cannot reset an applied endorsement."_ |

### C-113 — Cannot schedule inspection without scheduled date/time

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.inspection` |
| **File** | `models/insurance_inspection.py` |
| **Method** | `action_schedule()` |
| **Condition** | `scheduled_datetime` is not set |
| **Error** | _"Please set a scheduled date/time before scheduling."_ |

### C-114 — Cannot schedule re-inspection without re-inspection scheduled date/time

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.inspection` |
| **File** | `models/insurance_inspection.py` |
| **Method** | `action_schedule_reinspection()` |
| **Condition** | `reinspection_scheduled_datetime` is not set |
| **Error** | _"Please set a re-inspection scheduled date/time before scheduling."_ |

### C-120 — Cannot complete inspection without actual datetime

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.inspection` |
| **File** | `models/insurance_inspection.py` |
| **Method** | `action_complete()` |
| **Condition** | `inspection_datetime` is not set |
| **Error** | _"Please set the actual inspection date/time before completing."_ |

### C-121 — Cannot complete inspection without inspection report

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.inspection` |
| **File** | `models/insurance_inspection.py` |
| **Method** | `action_complete()` |
| **Condition** | `inspection_document` is not uploaded |
| **Error** | _"Please upload the inspection report before completing."_ |

### C-122 — Cannot confirm re-inspection without actual datetime

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.inspection` |
| **File** | `models/insurance_inspection.py` |
| **Method** | `action_confirm_reinspection()` |
| **Condition** | `reinspection_datetime` is not set |
| **Error** | _"Please set the actual re-inspection date/time before confirming."_ |

### C-123 — Cannot confirm re-inspection without re-inspection report

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.inspection` |
| **File** | `models/insurance_inspection.py` |
| **Method** | `action_confirm_reinspection()` |
| **Condition** | `reinspection_document` is not uploaded |
| **Error** | _"Please upload the re-inspection report before confirming."_ |

### C-124 — Cannot complete vehicle inspection without body parts on every line

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **Model** | `insurance.inspection` (inherited) |
| **File** | `models/vehicle_inspection.py` |
| **Method** | `action_complete()` |
| **Condition** | Any `vehicle.inspection.line` has no `body_part_ids` |
| **Error** | _"Cannot complete inspection: the following vehicles have no body parts recorded: %s"_ |

### C-135 — Accept-offer wizard: only pricing team leaders and sysadmin

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `accept.offer.wizard`, `accept.offer.wizard.payment.line`, `insurance.offer`, `insurance.policy` |
| **File** | `security/ir.model.access.csv`, `security/security.xml`, `wizards/accept_offer_wizard.py`, `models/insurance_offer.py` |
| **Mechanism** | ACL: the two wizard models are accessible only to `group_insurance_pricing_team_leader` (RWC) and `base.group_system` (RWCU); no other group can read them. Gate in `accept.offer.wizard.action_accept()`, as the acting user: `insurance.offer._check_accept_authorization()` (also called by the button `action_accept_offer()`), `offer.check_access('write')` (keeps the leader record rule on offers effective) and `offer._check_acceptable()` (the button's preconditions, shared). Past the gate `_action_accept()` builds the policy as superuser; type modules extend `_action_accept()` only. Record rule `rule_insurance_policy_pricing_leader`: leaders read only policies whose `offer_id.pricing_team_id.leader_id` is them. `create_uid` stays the acceptor. |
| **Error** | _"Only pricing team leaders and sysadmin can accept an offer."_ / _"Cannot accept offer: The client must accept the offer first."_ / standard record-rule AccessError |
| **FR** | FR-BASE-020-13 (-01 to -04) |

### C-149 — Endorsement cannot be reverted while an installment it created is covered

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.policy.endorsement` |
| **File** | `models/insurance_policy_endorsement.py` (`_check_revert_allowed`, called by `action_revert_changes`) |
| **Method** | `action_revert_changes()` |
| **Condition** | Any of the endorsement's schedule lines has a `target_record` with `cheque_id` set (any cheque state); every such line is an `add` under C-146 |
| **Error** | _"Endorsement %(number)s cannot be reverted: installment %(line)s is covered by cheque %(cheque)s."_ |
| **FR** | FR-BASE-007-08-07, FR-BASE-013-06-06 |

### C-150 — Clearing a cheque with mixed due dates is confirmed first

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.cheque` |
| **File** | `models/insurance_cheque.py` (`action_clear` → `insurance.cheque.clear.wizard`) |
| **Method** | `action_clear()` |
| **Condition** | `len(set(line_ids.due_date)) > 1` and the context does not carry `skip_due_date_warning` |
| **Behavior** | Not an error, a confirmation — opens `insurance.cheque.clear.wizard` listing the installments and their due dates; `action_confirm_clear()` re-calls `action_clear()` with `skip_due_date_warning` set. Clearing through code (`write({'state': 'cleared'})`) never goes through here |
| **FR** | FR-BASE-013-06-08 |

### C-152 — Register Cheque refuses a bad selection

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `policy.payment.schedule` |
| **File** | `models/policy_payment_schedule.py` (`action_register_cheque`, list-header button of `view_policy_payment_schedule_list` and `view_policy_payment_schedule_list_all`, `groups=` accountant) |
| **Method** | `action_register_cheque()` |
| **Condition** | Empty selection; any selected row already has `cheque_id` set; the selection spans more than one `policy_id.insurance_company_id`; the selection spans more than one `policy_id.policy_holder_id`; the selection spans more than one `currency_id`; or the selected installments total zero (`currency.is_zero(Σ amount)`). The button's `groups=` is a client-side filter only — `action_register_cheque()` itself has no server-side accountant check; the cheque `create()` it leads to (accountant/sysadmin ACL only, D-060) is what actually refuses a non-accountant who reaches the method some other way |
| **Error** | _"Select the installments the cheque covers first."_ / _"%(line)s is already covered by cheque %(cheque)s."_ / _"One cheque covers installments of one insurance company only; the selection spans %s."_ / _"One cheque is in one currency only; the selection spans %s."_ / _"The selected installments total zero; a cheque cannot be for nothing."_ |
| **FR** | FR-BASE-013-06-05 |

### C-154 — Endorsement settlement shape gate at apply / schedule

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.policy.endorsement` |
| **File** | `models/insurance_policy_endorsement.py` (`_validate_settlement`, called from `action_apply_changes()` and the scheduling path) |
| **Method** | `_validate_settlement()` |
| **Condition** | `settlement_direction != 'none'` and the endorsement's installment lines (`_cheque_lines()`) are empty, or `settlement_balanced` is false (installment lines do not sum to the gross premium change) |
| **Error** | _"Endorsement %s changes the premium but adds no installment."_ / _"Endorsement %(number)s is not balanced: its installment lines move %(total)s but the gross premium change is %(delta)s."_ |
| **FR** | FR-BASE-007-08-06 |
| **Purpose** | This branch reworded the refusal and, by merging the pre-existing C10 into it, made `_validate_settlement()` the only settlement-shape gate left at apply and at schedule (D-057) — `_check_settlement_shape()` (C1) only guards direction/amount coherence at save, not the balance, since the wizard adds installment lines one at a time. |

### C-155 — Corporate lead conversion needs a gender on every contact

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `crm.lead.to.client` |
| **File** | `wizards/crm_lead_to_client.py` (`action_convert()`); contact popup marks `gender` required in `wizards/crm_lead_to_client_views.xml` |
| **Method** | `action_convert()` |
| **Condition** | `action == 'create'`, `client_type == 'corporate'`, and any contact line has no `gender` |
| **Error** | _"Please set the gender of these contacts before converting: %s"_ |
| **FR** | BR-BASE-003-02-06 |
| **Purpose** | Each contact line becomes a `client.individual`, whose `gender` is required (NOT NULL). The line's own field stays optional because `default_get` auto-fills a contact from the lead without one; this guard turns the database error into a readable message. |

### C-172 — Only an accountant loads real figure values into "Try with values"

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.incentive` |
| **File** | `models/insurance_incentive.py` (`action_current_period_values`) |
| **Condition** | The user is in neither `group_insurance_accountant` nor `base.group_system` |
| **Error** | _"Only an accountant can load real figure values. ⏎ تحميل القيم الفعلية للمؤشرات متاح للمحاسب فقط."_ (AccessError) |
| **FR** | FR-BASE-009-05-16 |
| **Purpose** | Figure values are money figures; payouts are already accountant-only (C-169). `try_formulas` stays open: it reads no business data. |

See C-168 (incentive payout).

---

### C-206 — Collection trips carry cheques only

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_collection` |
| **Model** | `insurance.cheque`, `insurance.collection.delivery.item`, `insurance.collection.custody` |
| **File** | `models/insurance_cheque.py` (`_collection_is_paper`, `_collection_waiting`, `_collection_ensure_payment_letter`, `write`), `models/collection_trip_item.py` (`_generate_cheque_handover_needs`, `_source_still_wanted`), `models/collection_custody.py` (`init`, the SQL view), `views/insurance_cheque_views.xml` |
| **Mechanism** | predicate `_collection_is_paper` (`payment_method == 'cheque'`) used by `_collection_waiting`, the payment-letter hook and `_source_still_wanted` (so neither the daily onward listing from the office nor a redo after a failure lists it again); domain leaf `('payment_method', '=', 'cheque')` in the handover cron search (the letter cron starts from completed cheque items, so the hook's check covers it); `write()` withdraws a record switched away from cheque from what has not set out and re-runs the hooks on a `payment_method` change; the Document Tracking view takes `payment_method = 'cheque'` rows only, and the cheque form hides the trip and custody fields otherwise (D-177) |
| **Condition** | A cash or bank-transfer record never gets an item to the insurer or a payment letter, and is not on Document Tracking, whether it was created that way or corrected later; its installments still count as having their payment, so the policy's cheques are listed without it. A record with no method (written before the base field existed) is treated as not a cheque |
| **Error** | — (silently not listed) |
| **FR** | FR-COL-001-03-10 |
| **Tests** | `TestChequeNeeds.test_a_cash_payment_is_never_listed`, `test_a_cash_row_counts_as_in_but_only_cheques_are_listed`, `test_a_cheque_switched_to_cash_leaves_a_trip_still_to_leave`, `test_a_handed_over_cheque_switched_to_cash_gets_no_letter`, `test_a_cheque_left_at_the_office_then_switched_to_cash_goes_no_further`, `test_a_cash_row_switched_to_cheque_is_listed_at_once`; `TestDocumentTracking.test_a_cash_record_is_not_a_paper` |

---

### C-199 — Only a confirmed batch without an invoice is invoiced

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.commission.batch` |
| **File** | `models/insurance_commission_batch.py` (`action_create_invoice`) |
| **Condition** | `state != 'confirmed'` or `invoice_id` already set. Write access on the batch is checked first; the link is then written as the system (C-208, D-232). Example: Create Invoice on draft CB/2026/0005 → refused. Creating the invoice no longer changes the state. The invoice is created as the acting user, so Odoo's `account.move` ACL also applies: the accountant has Invoicing rights (implied); a sysadmin without them is refused, and nothing is linked (D-242). |
| **Error** | _"Only a confirmed batch without an invoice can be invoiced. ⏎ لا يمكن إصدار فاتورة إلا لدفعة مؤكدة ليست لها فاتورة."_ |
| **FR** | FR-BASE-013-08-06 |
| **Tests** | `TestCommissionBatch.test_07_only_a_confirmed_batch_is_invoiced`, `test_07b_the_invoice_needs_invoicing_rights` |

---

### C-200 — The batch moves only along its diagram

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.commission.batch` |
| **File** | `models/insurance_commission_batch.py` (`_check_transition`, called by `_transition` and first by `action_confirm`; `_MOVES`) |
| **Condition** | the acting user has no write access on the batch (`check_access('write')`, checked before any sudo); or a (from, to) pair outside `_MOVES` = {draft → sent, sent → draft, sent → confirmed, draft → cancelled, sent → cancelled}. Cancelled is final; Confirmed goes nowhere. `action_confirm` checks this before the documents (C-189). Example: a read-only user (the QA agent) pressing Confirm on a sent batch → AccessError, the batch stays Sent; Reset to Draft on a draft batch, Cancel on a confirmed batch, cancelled → draft, or Confirm on a draft batch without documents → refused with the move message. |
| **Error** | no write access: Odoo's AccessError; a move outside `_MOVES`: _"Batch %(batch)s cannot go from %(old)s to %(new)s. ⏎ لا يمكن نقل الدفعة %(batch)s من %(old)s إلى %(new)s."_ |
| **FR** | FR-BASE-013-02-05, FR-BASE-013-08-04 |
| **Tests** | `TestCommissionBatch.test_04_cancel_needs_a_reason_and_is_final`, `test_05_reset_to_draft_from_sent_only_and_warns`, `test_06_a_confirmed_batch_cannot_be_cancelled`, `test_22_a_draft_batch_cannot_be_confirmed`, `test_24_a_read_only_user_cannot_confirm` |

---

### C-189 — A batch is sent with eligible lines and confirmed with the insurer's documents; a correction in Confirmed meets the confirm checks again

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.commission.batch`, `insurance.commission.batch.line` |
| **File** | `models/insurance_commission_batch.py` (send: `action_send` checks write access first (`check_access('write')`), then `_check_sendable`, before the composer opens and renders the PDF; `_action_mark_sent` calls `_check_sendable` again when the email is posted; confirm: `_check_confirmable`, called by `action_confirm`); `models/insurance_commission_batch_line.py` (correction: `@api.constrains('confirmed_amount')` `_check_confirmed_batch_still_confirmable` → the batch's `_check_confirmable`, F14, D-239) |
| **Condition** | **send:** the acting user has no write access on the batch (Send only; a reader such as the QA agent never gets the composer); the batch has no line, or a line fails C-184 again (C-185 is the database's, D-211). A line that fails when the email is posted refuses the post: nothing is posted, the batch stays Draft. **confirm:** the company invoice or its reference is missing, the «حفظة العمولة» PDF or its reference is missing, or a line fails C-184 again. The access and state rules (only from Sent) are `_check_transition`'s (C-200) and run first. Only the references are read: C-205 allows a reference only with its PDF. **correction in Confirmed (D-239, supersedes D-234):** a line's `confirmed_amount` is written while its batch is `confirmed` (no invoice yet, C-188) and the batch fails the confirm checks above; one helper, `_check_confirmable`, serves both. Every writer, superuser and the form's `line_ids` commands included; Sent is not checked. Example: CB/2026/0005 sent without documents → Confirm lists both; with both attached and POL-0042 #2 accepted at 400 → Confirmed, 960 expected, 920 confirmed, difference −40; then POL-0042 #2 corrected 440 → 400 → accepted; the same with the «حفظة العمولة» reference emptied outside the ORM, or POL-0042 #1's installment gone → refused, 440 stays. |
| **Error** | send: no write access: Odoo's AccessError; _"Batch %(batch)s has no installments to send. ⏎ الدفعة %(batch)s ليس بها أقساط لإرسالها."_; confirm: _"Batch %(batch)s cannot be confirmed yet. Missing: ⏎ لا يمكن تأكيد الدفعة %(batch)s بعد. الناقص: ⏎ - the company invoice and its reference / فاتورة شركة التأمين ورقمها ⏎ - the commission receipt (حفظة العمولة) and its reference / حفظة العمولة ورقمها"_ (only the missing items are listed); a line failing eligibility: the C-184 message; a correction in Confirmed: the same confirm messages |
| **FR** | FR-BASE-013-08-01, FR-BASE-013-08-04, FR-BASE-013-08-05 |
| **Tests** | `TestCommissionBatch.test_16_an_empty_batch_is_not_sent`, `test_28_send_refuses_an_empty_batch`, `test_36_a_refused_batch_posts_nothing`, `test_38_a_read_only_user_cannot_open_send`, `test_21_confirm_needs_both_documents`, `test_22_a_draft_batch_cannot_be_confirmed`, `test_23h_a_correction_in_confirmed_meets_confirm_checks_again` |

---

### C-209 — The statement email matches the batch it was prepared for, and is never scheduled

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.commission.batch`, `mail.compose.message` |
| **File** | `models/insurance_commission_batch.py` (`_statement_signature`, stored by `action_send` in ctx `mark_batch_as_sent`; `_check_statement_unchanged`, called by `message_post` before the move); `wizards/mail_compose_message.py` (`_action_schedule_message`) |
| **Condition** | The composer renders the statement PDF when it opens, not when it posts. With ctx `mark_batch_as_sent`: the batch is cancelled or confirmed; or it is draft and its date or its installments differ from the signature taken at Send; or the composer's Schedule is used. A batch already sent is left alone (D-159). Nothing is posted and the batch keeps its state. Example: Send on CB/2026/0005 with POL-0042 #1, then POL-0042 #2 added → the post is refused; CB/2026/0005 cancelled while the composer is open → refused; Schedule → refused. |
| **Error** | changed: _"Batch %(batch)s changed since the statement was prepared. Press Send again. ⏎ تغيرت الدفعة %(batch)s بعد إعداد الكشف. اضغط إرسال مرة أخرى."_; cancelled / confirmed: _"Batch %(batch)s is %(state)s and cannot be sent. ⏎ الدفعة %(batch)s %(state)s ولا يمكن إرسالها."_; scheduled: _"A commission statement cannot be scheduled. Send it now. ⏎ لا يمكن جدولة كشف العمولة. أرسله الآن."_ |
| **FR** | FR-BASE-013-08-01 |
| **Tests** | `TestCommissionBatch.test_33_a_line_added_after_the_composer_opened_refuses_the_post`, `test_34_a_cancelled_batch_refuses_the_post`, `test_35_the_statement_cannot_be_scheduled` |

### C-219 — An offer is not accepted while its request is Lost

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.offer`, `accept.offer.wizard` |
| **File** | `models/insurance_offer.py` (`_check_acceptable`, called by `action_accept_offer` and by the wizard's `action_accept`) |
| **Condition** | the offer's request is in `failed`. Example: REQ-0012 is Lost with its offer at Client Accepted → Accept refused from the button and the wizard; after Reopen by an administrator → accepted |
| **Error** | _"Cannot accept offer: Its request is Lost. An administrator must reopen it first."_ |
| **FR** | FR-BASE-005-07-08 |
| **Tests** | `TestAcceptOfferWizardDocument.test_no_offer_is_accepted_once_its_request_is_lost` |

### C-253 — A policy is created only by accepting an offer

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.policy` |
| **File** | `models/insurance_policy.py` (`create`, `_issue_from_offer`); the only caller is `accept.offer.wizard._action_accept` |
| **Mechanism** | `create()` guard. `_issue_from_offer()` puts a Python object in the context as a pass; RPC and import carry JSON, so neither can send it, and the method is private, so neither can call it. No bypass, superuser included (D-294) |
| **Condition** | `create()` is reached without the pass. Example: an administrator imports a policy file, or types a new number in a policy picker and presses "Create and edit" → refused; a pricing team leader presses Accept on OFF-0031 and confirms the wizard → POL-0031 is created |
| **Error** | _"A policy cannot be created directly: it is issued only by accepting an offer."_ |
| **FR** | FR-BASE-006-16-01 |
| **Tests** | `TestAcceptOfferWizardDocument.test_a_policy_is_created_only_by_accepting_an_offer` |

### C-235 — A trip cannot be duplicated

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_collection` |
| **Model** | `insurance.collection.delivery` |
| **File** | `models/collection_trip.py` (`copy`) |
| **Condition** | always: a copy would carry the same cheques and documents a second time. Example: Duplicate on TRIP/2026/0007 → refused |
| **Error** | _"A trip cannot be duplicated: a copy would carry the same cheques and documents twice. Create a new trip instead."_ |
| **FR** | FR-COL-002-07-02 (D-25) |
| **Tests** | `TestTripFlow.test_a_trip_cannot_be_duplicated` |

### C-236 — Each trip action starts from its own states

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_collection` |
| **Model** | `insurance.collection.delivery` |
| **File** | `models/collection_trip.py` (`_require_state`, in `action_assign`, `action_start`, `action_complete`, `action_cancel`, `action_reset_to_draft`) |
| **Condition** | Assign from `draft`; Start from `assigned`; Complete from `in_progress` (a closed trip gets "Already closed"); Cancel from `draft`, `assigned` or `in_progress`; Reset to Draft from `cancelled`. Through C-224 a board drag gets the same checks. Example: a draft dragged to Completed → refused |
| **Error** | _"Only a draft trip can be assigned: %s"_; _"Only an assigned trip can start: %s"_; _"Already closed: %s"_; _"Start the trip before completing it: %s"_; _"Already closed, so it cannot be cancelled: %s"_; _"Only a cancelled trip goes back to draft: %s"_ |
| **FR** | FR-BASE-015-05-01, FR-COL-003-04-02, FR-COL-012-06-01, FR-COL-012-07 |
| **Tests** | `TestTripFlow.test_a_draft_dragged_to_completed_is_refused`, `test_only_a_cancelled_trip_goes_back_to_draft`, `test_a_completed_trip_stays_closed` |

### C-237 — A trip says everything before an agent takes it

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_collection` |
| **Model** | `insurance.collection.delivery` |
| **File** | `models/collection_trip.py` (`_check_ready`, on Assign and again on Start; `action_assign`) |
| **Mechanism** | choosing the agent runs Assign (D-37), so the check fires on the agent field too. Checked again on Start because items may be added in between |
| **Condition** | no `scheduled_date`; no item; an item without its sender, or without its receiver (a bank leg needs none); an item naming no document; on Assign, no agent. Example: TRIP/2026/0009 with the policy document but no date → choosing Sayed is refused, naming Scheduled Date |
| **Error** | _"Trip %(trip)s cannot be assigned before it has: %(fields)s."_; _"Trip %(trip)s cannot be assigned: %(item)s has no %(fields)s."_; _"Trip %(trip)s cannot be assigned: %(count)d item(s) do not name what they carry. …"_; _"Please assign an employee before confirming."_ |
| **FR** | FR-COL-002-04-05, FR-COL-002-02-05, FR-COL-003-02-02 (D-37) |
| **Tests** | `TestTripFlow.test_a_trip_missing_an_end_or_its_date_is_not_assigned`, `test_a_trip_with_nothing_on_it_is_not_assigned`, `test_a_drag_to_assigned_without_an_agent_is_refused`, `TestPendingItems.test_a_trip_with_an_unnamed_item_is_not_assigned` |

### C-238 — A trip is completed with every item's result and its cost

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_collection` |
| **Model** | `insurance.collection.delivery` |
| **File** | `models/collection_trip.py` (`action_complete`, `_check_cost`) |
| **Condition** | an item of the trip has no result; or `cost <= 0`. Example: TRIP/2026/0007 with CHQ-13 still without a result → refused, naming it; all results given but no cost → refused |
| **Error** | _"Trip %(trip)s cannot be completed: give every item its result first (%(items)s)."_; _"Enter the trip cost before completing it: %s"_ |
| **FR** | FR-COL-012-06-01, FR-COL-002-04-09 (D-32) |
| **Tests** | `TestTripFlow.test_the_trip_is_completed_once_every_item_has_its_result`, `test_no_trip_is_completed_without_its_cost`, `test_a_drag_to_completed_needs_the_items_results` |

### C-239 — A trip with a result is completed, not cancelled

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_collection` |
| **Model** | `insurance.collection.delivery` |
| **File** | `models/collection_trip.py` (`action_cancel`) |
| **Condition** | any item of the trip has a result. Example: TRIP/2026/0007 with CHQ-12 Completed → Cancel refused |
| **Error** | _"%s has items with their result: give the others theirs and complete it instead."_ |
| **FR** | FR-COL-012-07-01 |
| **Tests** | `TestTripFlow.test_a_trip_with_a_result_is_not_cancelled` |

### C-240 — Create Trip takes free items of one team

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_collection` |
| **Model** | `insurance.collection.delivery.item` |
| **File** | `models/collection_trip_item.py` (`action_create_trip`) |
| **Condition** | nothing is selected; a selected item has a result, or is on a trip that is not cancelled; the selection mixes the claims team's items with the collection team's (C-232). Example: Create Trip on CHQ-12, already on draft TRIP/2026/0007 → refused |
| **Error** | _"Select the items to put on the trip."_; _"Already on a trip, or done: %s"_; _"A trip carries the items of one team: select the claims' items or the others, not both."_ |
| **FR** | FR-COL-012-03-01, FR-COL-013-02-01 |
| **Tests** | `TestPendingItems.test_an_item_on_a_trip_or_done_is_not_taken_twice`, `TestClaimNeeds.test_a_trip_carries_one_teams_items` |

### C-247 — Only the team leader and the administrator reopen a closed claim

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.claim` |
| **File** | `models/insurance_claim.py` (`action_back_to_fresh`; the button also carries `groups=` in `views/insurance_claim_views.xml`) |
| **Condition** | the user is neither superuser, nor an administrator (`base.group_system`), nor in Customer Service Team Leader. Example: Mona, Customer Service, presses Back to Fresh on settled CLM/0042 → refused; the team leader presses it → CLM/0042 is Fresh, its rejection reason cleared, `settled_on` kept until it settles again (D-283). After the group gate, `check_access('write')` runs as the caller before the superuser write, so write access on the claim is also required (every group that passes the gate has it, so it is a safeguard, not tested separately) |
| **Error** | _"Only the Customer Service Team Leader or an administrator can reopen a claim."_ (`AccessError`) |
| **FR** | FR-BASE-008-05-06 |
| **Tests** | `TestClaimStatus.test_back_to_fresh_refused_for_customer_service`, `test_back_to_fresh_reopens_a_rejected_claim`, `test_back_to_fresh_reopens_a_settled_claim`, `test_a_closed_claim_leaves_by_back_to_fresh_only`, `test_resettled_claim_is_stamped_again`, `test_header_and_kanban` |

---

## 7. View Readonly Controls

### 7a. Conditional Readonly (state-dependent)

These fields become readonly based on business state, preventing modification at certain stages.

### C-110 — Offer financial fields readonly when has child offers

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **View** | `views/insurance_offer_views.xml` |
| **Condition** | `readonly="has_child_offers"` |
| **Fields affected** | `insurance_request_id`, `insurance_company_id`, `insurance_type_id`, `net_premium`, `gross_premium`, `sum_insurance`, `gross_rate`, `insurance_duration`, `number_of_cheques`, `coverage_ids` |
| **Purpose** | Prevents modification of core offer data after negotiation scenarios have been created |

#### C-110-01 — Medical categories readonly when has child offers

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_medical` |
| **View** | `views/insurance_offer_views.xml` |
| **Condition** | `readonly="has_child_offers"` |
| **Fields affected** | `medical_category_ids`, `employee_ids` |

#### C-110-02 — Vehicle list readonly when has child offers

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **View** | `views/insurance_offer_views.xml` |
| **Condition** | `readonly="has_child_offers"` |
| **Fields affected** | `vehicle_ids` |

#### C-110-03 — Shipment list readonly when has child offers

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_shipment` |
| **View** | `views/insurance_offer_views.xml` |
| **Condition** | `readonly="has_child_offers"` |
| **Fields affected** | `shipment_ids` |

#### C-110-04 — Estate list readonly when has child offers

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_general` |
| **View** | `views/insurance_offer_views.xml` |
| **Condition** | `readonly="has_child_offers"` |
| **Fields affected** | `estate_ids` |

### C-111 — Vehicle inspection body part original fields readonly outside scheduled state

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **View** | `views/vehicle_inspection_views.xml` |
| **Condition** | `readonly="parent.inspection_state != 'scheduled'"` |
| **Fields affected** | `vehicle_part_id`, `custom_part_name`, `status`, `photo`, `damage_description`, `loss_value`, `reimbursed_value` |
| **Purpose** | Original inspection data can only be edited during the scheduled (active inspection) state; during re-inspection only `negotiation_status` and `resolved_photo` are editable |

### C-115 — Inspection scheduled datetime readonly after scheduling

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **View** | `views/insurance_inspection_views.xml` |
| **Condition** | `readonly="state != 'draft'"` |
| **Fields affected** | `scheduled_datetime` |
| **Purpose** | Prevents modification of scheduled date after inspection has been scheduled |

### C-116 — Inspection datetime and document required when completing

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **View** | `views/insurance_inspection_views.xml` |
| **Condition** | `required="state == 'scheduled'"` |
| **Fields affected** | `inspection_datetime`, `inspection_document` |
| **Purpose** | Ensures actual inspection time and report are provided before marking inspection complete |

### C-117 — Re-inspection scheduled datetime readonly after scheduling

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **View** | `views/insurance_inspection_views.xml` |
| **Condition** | `readonly="state != 'negotiation'"` |
| **Fields affected** | `reinspection_scheduled_datetime` |
| **Purpose** | Prevents modification of re-inspection scheduled date after re-inspection has been scheduled |

### C-118 — Re-inspection datetime and document required when completing

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **View** | `views/insurance_inspection_views.xml` |
| **Condition** | `required="state == 'reinspection_scheduled'"` |
| **Fields affected** | `reinspection_datetime`, `reinspection_document` |
| **Purpose** | Ensures actual re-inspection time and report are provided before confirming re-inspection done |

### C-119 — Vehicle inspection body parts One2many locked outside active states

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **View** | `views/vehicle_inspection_views.xml` |
| **Condition** | `readonly="inspection_state not in ('scheduled', 'reinspection_scheduled', 'negotiation')"` |
| **Fields affected** | `body_part_ids` (One2many on `vehicle.inspection.line` form) |
| **Purpose** | Prevents adding/deleting body part rows outside active inspection, re-inspection, or negotiation states. Individual field editability is controlled by C-111 |

### C-125 — Inspection fields readonly after completion

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **View** | `views/insurance_inspection_views.xml` |
| **Condition** | `readonly="state not in ('draft', 'scheduled')"` |
| **Fields affected** | `client_id`, `inspection_datetime`, `inspector_name`, `inspector_phone`, `inspector_email`, `notes`, `inspection_document` |
| **Purpose** | Prevents modification of original inspection data once inspection is completed |

### C-147 — Cheque and paid-installment fields readonly

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **View** | `views/insurance_cheque_views.xml` (cheque form: every business field and `line_ids`); `views/insurance_policy_views.xml` (Manage Payments list); `views/policy_payment_schedule_views.xml` (`view_policy_payment_schedule_list_all`, same readonly rules) |
| **Condition** | `views/insurance_cheque_views.xml`: `readonly="state == 'cleared'"`. `views/insurance_policy_views.xml`: `cheque_id`, `amount`, `sequence` readonly in the arch; `is_paid` and `payment_date` readonly from the Python field definition (C-143); `due_date` readonly="cheque_id" in both lists (a covered installment's due date locks in every cheque state, not only cleared, FR-BASE-013-06-06) |
| **Fields affected** | Cheque form: `name`, `bank_id`, `client_id`, `insurance_company_id`, `payment_date`, `currency_id`, `amount`, `line_ids`, `cheque_document`, `cheque_document_filename` (invisible, but in `_LOCKED_WHILE_CLEARED`). Manage Payments list: `due_date`, `cheque_id`, `is_paid`, `payment_date`, `amount`, `sequence`. All-installments list (`view_policy_payment_schedule_list_all`): same fields, plus `policy_holder_id`, `insurance_company_id` readonly |
| **Purpose** | A cleared cheque has paid its covered installments; the cheque's own data locks until it leaves cleared, and the installment's due date locks as soon as it is covered by a cheque, in any cheque state (C-142) — not only while cleared, while paid flag, payment date, cheque link, amount and installment number are always system-set. The Manage Payments list's Register Cheque button is accountant-only (`groups="optimum_insurance_base.group_insurance_accountant"`); its own refusals (empty selection, a covered installment, or a selection spanning two insurance companies, two currencies, or totalling zero) live in `action_register_cheque` (FR-BASE-013-06-05), not in the view |
| **FR** | FR-BASE-013-06-04, FR-BASE-013-06-06, FR-BASE-006-11-21, FR-BASE-013-06-05 |

### C-175 — Incentive figure form readonly when the figure is locked

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **View** | `views/insurance_incentive_figure_views.xml` |
| **Condition** | `readonly="is_locked"` |
| **Fields affected** | `code`, `name`, `kind`, `source`, `aggregate`, `sum_field`, `insurance_type_ids`, `formula` |
| **Purpose** | A figure an incentive or a formula figure uses cannot change; its form is read-only (the kind and the formula too) and shows a warning banner (see C-173) |
| **FR** | FR-BASE-009-05-20, FR-BASE-009-05-24 |

### 7b. Always-Readonly Fields (computed / reference / system-generated)

These fields are always `readonly="1"` because they are computed, system-generated, or reference fields that should not be manually edited.

#### Offer Form (`optimum_insurance_base` — `insurance_offer_views.xml`)

| Fields | Purpose |
|--------|---------|
| `name` | Auto-generated offer name |
| `prev_net_premium`, `prev_gross_premium`, `prev_sum_insurance`, `prev_gross_rate`, `prev_insurance_duration`, `prev_number_of_cheques` | Previous version comparison values |
| `reviewer_id`, `reviewed_date` | Reviewer and review timestamp |
| `inspection_id`, `previous_offer_id`, `child_offer_count` | Reference / computed |
| `mandatory_coverage_status` | Computed widget |
| `prev_coverage_*`, `prev_deductible_*`, `prev_copayment_*` fields | Previous version comparison values in coverage/deductible/copayment lines |
| `limit_review_status`, `coverage_review_notes` | Review status |
| `deductible_review_status`, `deductible_review_notes` | Review status |
| `copayment_review_status`, `copayment_review_notes` | Review status |
| Predefined descriptions, company wording, standard wording | Reference text |
| `has_copay` | Computed flag |
| `pricing_contact_ids` | Reference |

#### Policy Form (`optimum_insurance_base` — `insurance_policy_views.xml`)

| Fields | Purpose |
|--------|---------|
| `first_check_completed` | Computed |
| `predefined_description`, `is_mandatory` | Reference |
| `sequence` (check #) | Auto-generated |
| `contract_id` | System-determined |

#### Endorsement Form (`optimum_insurance_base` — `insurance_policy_endorsement_views.xml`)

| Fields | Purpose |
|--------|---------|
| `endorsement_number` | Auto-generated |
| `policy_holder_id`, `insurance_company_id`, `insurance_type_id` | Inherited from policy |
| `description` | System-generated |
| `is_applied` | State toggle |

#### Request Form (`optimum_insurance_base` — `insurance_request_views.xml`)

| Fields | Purpose |
|--------|---------|
| `name` | Auto-generated |
| Status fields, `is_confirmed`, `offer_count` | Computed |
| `declared_company_count`, `offers_received_count`, `offers_completion_percentage` | Computed statistics |

#### Inspection Form (`optimum_insurance_base` — `insurance_inspection_views.xml`)

| Fields | Purpose |
|--------|---------|
| `name`, `inspection_datetime` | Auto-generated |

#### KYC History Form (`optimum_insurance_base` — `insurance_kyc_history_views.xml`)

| Fields | Purpose |
|--------|---------|
| `policy_duration_months`, `loss_ratio`, `overall_rating` | Computed |
| `create_uid`, `create_date`, `write_date` | Audit fields |

#### Other Base Views

| View | Fields | Purpose |
|------|--------|---------|
| `client_corporate_views.xml` | `active` (in employment table) | Status display |
| `client_individual_views.xml` | `current_employer_id` | Computed |
| `corporate_client_individual_views.xml` | `display_name`, `email`, `phone`, `end_date` | Reference / computed |
| `insurance_collection_delivery_views.xml` | `name` | Auto-generated |
| `crm_lead_views.xml` | `client_contact_ids` | Reference |
| `insurance_public_terms_views.xml` | `display_name` | Computed |
| `offer_has_copayment_views.xml` | `copayment_review_status`, `copayment_review_notes` | Review status |
| `offer_has_service_views.xml` | `has_copay` | Computed |
| `policy_has_service_views.xml` | `has_copay` | Computed |
| `request_client_company_preference_views.xml` | `name` | Reference |

#### Medical Module Junction Table Views

| View | Fields | Purpose |
|------|--------|---------|
| `offer_has_employee_views.xml` | `full_name`, `gender`, `relation`, `id_card_number`, `offer_id`, `client_id` | Derived from employee |
| `policy_has_employee_views.xml` | `full_name`, `gender`, `relation`, `id_card_number`, `policy_id`, `policy_holder_id`, `policy_state` | Derived from employee |
| `request_has_employee_views.xml` | `full_name`, `gender`, `relation`, `id_card_number`, `insurance_request_id`, `client_id` | Derived from employee |
| `client_insurable_employee_views.xml` | `name`, `is_insured` | Computed |
| `insurance_offer_medical_category_views.xml` | `offer_id` | Parent reference |
| `insurance_request_medical_category_views.xml` | `insurance_request_id` | Parent reference |
| `insurance_policy_medical_category_views.xml` | `policy_id` | Parent reference |

#### Vehicle Module Junction Table Views

| View | Fields | Purpose |
|------|--------|---------|
| `offer_has_vehicle_views.xml` | `manufacturer_id`, `vehicle_model_id`, `number_plate`, `chassis_number`, vehicle prices, `vehicle_inspection_line_id`, `offer_id`, `client_id` | Derived from vehicle |
| `policy_has_vehicle_views.xml` | Same pattern + `policy_id`, `policy_holder_id`, `policy_state` | Derived from vehicle |
| `request_has_vehicle_views.xml` | Same pattern + `insurance_request_id`, `client_id` | Derived from vehicle |
| `client_insurable_vehicle_views.xml` | `name`, `is_insured`, `manufacturer_id`, `vehicle_model_id` | Computed |
| `vehicle_inspection_views.xml` | `inspection_id`, `number_plate`, `chassis_number`, `manufacturer_id`, `vehicle_model_id` | Reference fields in inspection lines |
| `vehicle_model_views.xml` | `normalised_name` | Computed |
| `vehicle_year_of_prod_views.xml` | `name` | Computed |

#### Shipment Module Junction Table Views

| View | Fields | Purpose |
|------|--------|---------|
| `offer_has_shipment_views.xml` | `shipment_type`, `weight`, `from_country_id`, `to_country_id`, `method_of_packing_id`, `offer_id`, `client_id` | Derived from shipment |
| `policy_has_shipment_views.xml` | Same + `policy_id`, `policy_holder_id`, `policy_state` | Derived from shipment |
| `request_has_shipment_views.xml` | Same + `insurance_request_id`, `client_id` | Derived from shipment |
| `client_insurable_shipment_views.xml` | `name`, `is_insured` | Computed |

#### General Module Junction Table Views

| View | Fields | Purpose |
|------|--------|---------|
| `offer_has_estate_views.xml` | `estate_type_id`, `address`, `area_sqm`, `estimated_value`, `country_id`, `city`, `offer_id`, `client_id` | Derived from estate |
| `policy_has_estate_views.xml` | Same + `policy_id`, `policy_holder_id`, `policy_state` | Derived from estate |
| `request_has_estate_views.xml` | Same + `insurance_request_id`, `client_id` | Derived from estate |
| `client_insurable_estate_views.xml` | `name`, `is_insured` | Computed |

### C-170 — Incentive payouts are created by the system and their evaluated fields are read-only

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **View** | `views/insurance_incentive_payout_views.xml` — `insurance.incentive.payout` form and list |
| **Condition** | `create="0"` on the form and the list; `readonly="1"` on the fields below |
| **Fields affected** | `incentive_id`, `period_start`, `period_end`, `evaluated_on`, `holds`, `is_undefined`, `amount`, `condition_text`, `payout_text` |
| **Purpose** | A payout is created by the daily job and changed only by Recompute (FR-BASE-013-07-01, FR-BASE-013-07-03). |
| **FR** | FR-BASE-013-07-01, FR-BASE-013-07-03 |

### C-171 — Irreversible incentive actions ask for confirmation

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **View** | `views/insurance_incentive_payout_views.xml` — payout form, `action_claim` and `action_mark_paid` (`confirm=`) |
| **View** | `static/src/views/incentive_formula_widget/` — "Remove condition" opens a `ConfirmationDialog` (`removeCondition`) |
| **Condition** | The user presses Claim or Mark Paid, or "Remove condition" in the builder |
| **Message** | _"Claiming freezes this payout's figures and amount. This cannot be undone. Claim it? ⏎ المطالبة تجمّد قيم مؤشرات دفعة الحافز هذه ومبلغها، ولا يمكن التراجع عن ذلك. هل تريد المطالبة بها؟"_ / _"Mark this payout as paid? This cannot be undone. ⏎ هل تريد تسجيل دفعة الحافز هذه كمدفوعة؟ لا يمكن التراجع عن ذلك."_ / _"The incentive will then pay every period. ⏎ سيُصرف الحافز عندها في كل فترة."_ (Remove condition) |
| **FR** | FR-BASE-013-07-06; FR-BASE-009-05-13 (Remove condition) |
| **Purpose** | Claiming freezes the figures (C-168); a mis-click must not freeze a wrong payout; marking paid is final (FR-BASE-013-07-04). |

### 7c. No-Create Relational Pickers (`options="{'no_create': True}"`)

These Many2one pickers may only select records that already exist. Both the dropdown quick-create and the "Create and edit..." dialog are disabled.

### C-133 — Endorsement change line target record pickers cannot create records

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **View** | `wizards/endorsement_change_line_wizard_views.xml` |
| **FR** | - |
| **Condition** | `options="{'no_create': True, 'no_create_edit': True}"` |
| **Fields affected** | `target_coverage_id`, `target_deductible_id`, `target_service_id`, `target_shared_limit_id`, `target_beneficiary_id`, `target_private_term_id`, `target_exclusion_id` |
| **Purpose** | The Target Record pickers identify which existing policy record an endorsement line edits or removes. Creating a record from the picker would write directly to the policy, bypassing the endorsement apply/revert engine and leaving an untracked change. Records are added to a policy only through an endorsement line with operation `add` |

#### C-133-01 — Medical target record pickers cannot create records

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_medical` |
| **View** | `wizards/endorsement_change_line_wizard_views.xml` |
| **Fields affected** | `target_employee_id`, `target_medical_category_id`, `target_benefit_type_id`, `target_coverage_item_id` |

#### C-133-02 — Vehicle target record picker cannot create records

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **View** | `wizards/endorsement_change_line_wizard_views.xml` |
| **Fields affected** | `target_vehicle_id` |

---

### 7d. Feature Screen Rules (`readonly` / `invisible` / `required`, one table per feature)

One control lists every view rule of one feature's screens, row by row.

### C-204 — Readonly and visibility rules of the batch-receipt screens

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Mechanism** | view `readonly` / `invisible` / `required` / `groups` |
| **FR** | FR-BASE-006-08-06, FR-BASE-007-09-09, FR-BASE-013-06-10..-13, FR-BASE-013-02-05, FR-BASE-013-02-06, FR-BASE-013-08-01, FR-BASE-013-08-03, FR-BASE-013-08-04, FR-BASE-013-08-06 |
| **Tests** | `TestFlatCommissionAccept.test_04_the_tier_preview_hides_while_flat`, `TestPaymentMethodForm.test_01`–`test_03`, `test_05`; the attach button's opener: `TestPremiumSchedule.test_09`; the attach wizard's current receipt, warning and lock: `TestPremiumSchedule.test_17`, `test_18`, `test_18b`, `test_20`; Send's server side: `TestCommissionBatch.test_28`–`test_39`; the batch form's line list, confirmed amount and note: `TestCommissionBatch.test_23d`; who gets the four move buttons: `TestCommissionBatch.test_04c` |

| Rule | View | Field / button | Condition |
|---|---|---|---|
| The flat flag is set at acceptance and never edited (D-182) | `views/insurance_policy_views.xml` (Commission page, group `commission_type`) | `is_flat_commission` | `readonly="1"` |
| The endorsement shows its policy's flag | `views/insurance_policy_endorsement_views.xml` (Commission page, group `commission_type`) | `is_flat_commission` | readonly by the related field (FR-BASE-007-09-09) |
| A flat commission earns no early-payment bonus, so its tiers are not previewed | `wizards/accept_offer_wizard_views.xml` (group `early_payment_info`) | the tier preview `early_payment_reward_ids` | `invisible="not contract_id or is_flat_commission"` |
| Only a cheque is deposited or bounced (C-192) | `views/insurance_cheque_views.xml` (payment form header) | buttons `action_deposit`, `action_bounce` | `invisible="state == '<that state>' or payment_method != 'cheque'"` |
| The bank is unused for cash, required for a cheque, optional for a transfer (C-193, D-176) | `views/insurance_cheque_views.xml` (payment form, group `parties`) | `bank_id` | `invisible="payment_method == 'cash'"`, `required="payment_method == 'cheque'"` |
| The payment proof (cheque scan or transfer photo) is required for a cheque and a transfer, hidden for cash (C-151, C-191, D-175, D-254) | `views/insurance_cheque_views.xml` (payment form, group `money`) | `cheque_document` | `invisible="payment_method == 'cash'"`, `required="payment_method != 'cash'"` |
| The method is locked once cleared (C-140) | `views/insurance_cheque_views.xml` (payment form, group `parties`) | `payment_method` | `readonly="state == 'cleared'"` |
| A receipt goes only on a policy with its own installments (C-182) | `views/insurance_policy_views.xml` (Documents page, group `premium_schedule`, D-215) | button `action_attach_premium_schedule` | `invisible="not payment_schedule_ids"`; `groups` accountant, sysadmin |
| An endorsement's receipt waits until it is applied (C-182, D-173) | `views/insurance_policy_endorsement_views.xml` (Documents page, group `premium_schedule`, D-217) | button `action_attach_premium_schedule` | `invisible="not is_applied or not installment_ids"`; `groups` accountant, sysadmin |
| Only the attach wizard writes the receipt (C-196) | `views/insurance_policy_views.xml` (Documents page, group `premium_schedule`, D-215), `views/insurance_policy_endorsement_views.xml` (Documents page, group `premium_schedule`, D-217) | `premium_schedule_document`, `premium_schedule_reference_number` | `readonly="1"` |
| The attach wizard shows the record's current receipt before replacing it, never as an input: file name, reference and a Download link to the record's own PDF (D-221, D-223) | `wizards/insurance_premium_schedule_wizard_views.xml` (group `current_receipt`) | `current_filename`, `current_reference_number`, `current_document_url` (`widget="url"`) | `readonly="1"`; the group `invisible="not current_reference_number"` |
| The attach wizard warns that attaching replaces the current receipt (D-221) | `wizards/insurance_premium_schedule_wizard_views.xml` | the `alert-warning` "Attaching replaces it" | `invisible="not current_reference_number or is_current_locked"` |
| A receipt a live batch uses is shown as locked and Attach is hidden; C-183 stays the enforcement (D-223) | `wizards/insurance_premium_schedule_wizard_views.xml` | the `alert-danger` "… cannot be changed"; button `action_attach` | the alert `invisible="not is_current_locked"`; the button `invisible="is_current_locked"` |
| A batch's header changes only while draft (C-197) | `views/insurance_commission_batch_views.xml` (form sheet) | `date`, `insurance_company_id`, `currency_id` | `readonly="state != 'draft'"` |
| A batch's lines are added, removed or re-pointed only while draft (C-187) | `views/insurance_commission_batch_views.xml` (form sheet, page `installments`) | `line_ids`; its `installment_id` column | `line_ids` `readonly="state == 'cancelled' or invoice_id"` (D-229), `options="{'create': [('state', '=', 'draft')], 'delete': [('state', '=', 'draft')]}"`; `installment_id` `readonly="parent.state != 'draft'"` |
| Confirmed amounts are typed while sent or confirmed, until the invoice (C-197, C-188, D-229) | `views/insurance_commission_batch_views.xml` (page `installments`, line list) | `confirmed_amount` | `readonly="parent.state not in ('sent', 'confirmed') or parent.invoice_id"` |
| A confirmed or cancelled batch's note is final (C-188, D-229; C-215, D-247) | `views/insurance_commission_batch_views.xml` (page `notes`) | `note` | `readonly="state in ('confirmed', 'cancelled')"` |
| The move dates are shown, never edited (C-208) | `views/insurance_commission_batch_views.xml` (form sheet, right group) | `sent_date`, `confirmed_date` | `readonly="1"`, `invisible="not <field>"` |
| Send only from Draft; the batch becomes Sent when the email is posted (C-189, D-159, D-201) | `views/insurance_commission_batch_views.xml` (form header, first button) | button `action_send` | `invisible="state != 'draft'"`; `groups` accountant, sysadmin |
| Confirm only from Sent (C-189, C-200) | `views/insurance_commission_batch_views.xml` (form header) | button `action_confirm` | `invisible="state != 'sent'"`, `groups="optimum_insurance_base.group_insurance_accountant,base.group_system"` |
| Reset to Draft only from Sent (C-200); accountant and sysadmin only, like Send and Confirm (F18, D-248) | `views/insurance_commission_batch_views.xml` (form header) | button `action_reset_to_draft` | `invisible="state != 'sent'"`, `groups="optimum_insurance_base.group_insurance_accountant,base.group_system"` |
| Cancel only from Draft or Sent (C-200); accountant and sysadmin only, like Send and Confirm (F18, D-248) | `views/insurance_commission_batch_views.xml` (form header) | button `action_cancel` | `invisible="state not in ('draft', 'sent')"`, `groups="optimum_insurance_base.group_insurance_accountant,base.group_system"` |
| Create Invoice only once, from Confirmed (C-199) | `views/insurance_commission_batch_views.xml` (form header) | button `action_create_invoice` | `invisible="state != 'confirmed' or invoice_id"` |
| The insurer's documents are frozen once the batch is confirmed or cancelled (C-205) | `views/insurance_commission_batch_views.xml` (form sheet, page `documents`) | `company_invoice`, `company_invoice_reference`, `commission_receipt`, `commission_receipt_reference_number` | `readonly="state in ('confirmed', 'cancelled')"` |
| The reason wizard's warning shows only on Reset to Draft (D-180) | `wizards/insurance_commission_batch_reason_wizard_views.xml` | the `warning` alert | `invisible="action != 'draft'"` |

### C-214 — Readonly and visibility rules of contract renewal

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Mechanism** | view `readonly` / `invisible` / `groups` |
| **FR** | FR-BASE-009-04-13 |
| **Tests** | `TestContractRenewal.test_10_the_form_shows_renew_until_renewed` |

| Rule | View | Field / button | Condition |
|---|---|---|---|
| Renew once, accountant and sysadmin only (C-211, D-225); a confirm dialog first | `views/insurance_company_contract_views.xml` (form header) | button `action_renew` | `invisible="renewal_id"`, `groups="optimum_insurance_base.group_insurance_accountant,base.group_system"` |
| The renewal chain is shown, never edited (C-212) | `views/insurance_company_contract_views.xml` (Contract Period group) | `renewed_from_id`, `renewal_id` | `readonly="1"`, `invisible="not <field>"` |

### C-252 — A client pre-filled by the opening screen is read-only

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base`, `optimum_insurance_vehicle`, `optimum_insurance_medical`, `optimum_insurance_general`, `optimum_insurance_shipment` |
| **Mechanism** | view `readonly` on the context default, with `force_save="1"` so the pre-filled value is still saved. View-level only: `write()` does not refuse a change. |
| **FR** | FR-BASE-001-03-08 |
| **Tests** | none (view-level) |

| Rule | View | Field / button | Condition |
|---|---|---|---|
| The owner of an insurable opened from a client, request, offer, policy, endorsement, negotiated-offer or inspection screen is not edited | `views/client_insurable_vehicle_views.xml`, `views/client_insurable_employee_views.xml`, `views/client_insurable_estate_views.xml`, `views/client_insurable_shipment_views.xml` (form) | `client_id` | `readonly="context.get('default_client_id')"` |
| The side of an employment record given by the client form it was opened from is not edited | `views/corporate_client_individual_views.xml` (form) | `corporate_client_id`, `individual_id` | `readonly="context.get('default_corporate_client_id')"`, `readonly="context.get('default_individual_id')"` |

---

## 8. Post-Policy Generation Immutability

When a policy is generated from an accepted offer on an `insurance.request`, all request and offer data becomes immutable. The `policy_generated` computed field on `insurance.request` (stored, depends on `offer_ids.stage` and `offer_ids.final_policy_id`) gates all these controls.

### C-126 — Offer action methods blocked after policy generation

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.offer` |
| **File** | `models/insurance_offer.py` |
| **Method** | `_check_request_policy_generated()` called by: `action_negotiate_offer()`, `action_client_accept_offer()`, `action_accept_offer()`, `action_create_negotiated_offer()`, `action_send_offer_to_client()`, `action_send_to_insurance_company()` |
| **Condition** | `insurance_request_id.policy_generated` is True |
| **Error** | _"Cannot perform this action: a policy has already been generated for request '%s'. All offer data is now immutable."_ |

### C-127 — Cannot create new offers for a request with a generated policy

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.offer` |
| **File** | `models/insurance_offer.py` |
| **Method** | `create()` |
| **Condition** | Target `insurance_request_id.policy_generated` is True |
| **Error** | _"Cannot create new offers for request '%s': a policy has already been generated."_ |

### C-128 — Cannot modify or delete offers after policy generation

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.offer` |
| **File** | `models/insurance_offer.py` |
| **Method** | `write()`, `unlink()` |
| **Condition** | `insurance_request_id.policy_generated` is True |
| **Error** | _"Cannot modify/delete offer '%s': a policy has already been generated for request '%s'."_ |

### C-129 — Cannot modify request business fields after policy generation

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.request` |
| **File** | `models/insurance_request.py` |
| **Method** | `write()` |
| **Condition** | `policy_generated` is True and write touches fields other than `insurance_request_stage_id`, `failure_reason_id`, or messaging fields |
| **Error** | _"Cannot modify request '%s': a policy has already been generated. The request data is now immutable."_ |

### C-130 — Cannot modify request junction tables after policy generation

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Models** | `request.coverage.preference`, `request.term.preference`, `request.has.shared.limit`, `request.has.service`, `request.expected.company`, `request.client.company.preference` |
| **Files** | `models/request_coverage_preference.py`, `models/request_term_preference.py`, `models/request_has_shared_limit.py`, `models/request_has_service.py`, `models/request_expected_company.py`, `models/request_client_company_preference.py` |
| **Methods** | `create()`, `write()`, `unlink()` on each model |
| **Condition** | `request_id.policy_generated` is True |
| **Note** | `request.expected.company.write()` allows system-level updates to `has_offer` and `offer_count` fields |

### C-130-01 — Cannot modify medical submodule junction tables after policy generation

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_medical` |
| **Models** | `offer.has.employee`, `insurance.offer.medical.category`, `offer.category.has.benefit.type`, `offer.benefit.has.coverage.item`, `request.has.employee`, `insurance.request.medical.category`, `request.category.has.benefit.type` |
| **Methods** | `create()`, `write()`, `unlink()` on each model |
| **Condition** | Traverses to `insurance_request_id.policy_generated` (offer-side via `offer_id`, indirect via `category_id`) |
| **Error** | _"Cannot modify [entity]: a policy has already been generated for request '%s'."_ |

### C-130-02 — Cannot modify vehicle submodule junction tables after policy generation

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **Models** | `offer.has.vehicle`, `request.has.vehicle` |
| **Methods** | `create()`, `write()`, `unlink()`, `action_create_inspection()` on each model |
| **Condition** | `offer_id.insurance_request_id.policy_generated` / `insurance_request_id.policy_generated` |
| **Error** | _"Cannot modify [entity]: a policy has already been generated for request '%s'."_ |

### C-130-03 — Cannot modify shipment submodule junction tables after policy generation

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_shipment` |
| **Models** | `offer.has.shipment`, `request.has.shipment`, `insurance.request.shipment` |
| **Methods** | `create()`, `write()`, `unlink()` on each model |
| **Condition** | `offer_id.insurance_request_id.policy_generated` / `insurance_request_id.policy_generated` / `request_id.policy_generated` |
| **Error** | _"Cannot modify [entity]: a policy has already been generated for request '%s'."_ |

### C-130-04 — Cannot modify general submodule junction tables after policy generation

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_general` |
| **Models** | `offer.has.estate`, `request.has.estate` |
| **Methods** | `create()`, `write()`, `unlink()` on each model |
| **Condition** | `offer_id.insurance_request_id.policy_generated` / `insurance_request_id.policy_generated` |
| **Error** | _"Cannot modify [entity]: a policy has already been generated for request '%s'."_ |

### C-131 — Request form fields readonly after policy generation

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **View** | `views/insurance_request_views.xml` |
| **Condition** | `readonly="policy_generated"` |
| **Fields affected** | `client_id`, `insurance_type_id`, `request_date`, `offer_submission_deadline`, `enforce_company_list`, `currency_id`, `sum_insurance`, `loss_ratio`, `number_of_cheques`, `renewal_trigger`, `coverage_preference_ids`, `term_preference_ids`, `shared_limit_ids`, `service_ids`, `declared_company_ids`, `company_preference_ids` |
| **Purpose** | Prevents UI modification of request data after a policy has been generated |

### C-131-01 — Submodule request form fields readonly after policy generation

| Attribute | Value |
|-----------|-------|
| **Modules** | `optimum_insurance_medical`, `optimum_insurance_vehicle`, `optimum_insurance_shipment`, `optimum_insurance_general` |
| **Views** | `insurance_request_views.xml` in each submodule |
| **Condition** | `readonly="policy_generated"` |
| **Fields affected** | `employee_ids`, `medical_category_ids` (medical); `vehicle_ids` (vehicle); `shipment_ids`, `trip_type`, `shipment_location_type`, `shipment_packing_method_id`, `shipment_min_value`, `shipment_max_value`, export/import route fields (shipment); `estate_ids` (general) |

### C-131-02 — Submodule offer form One2many fields readonly after policy generation

| Attribute | Value |
|-----------|-------|
| **Modules** | `optimum_insurance_medical`, `optimum_insurance_vehicle`, `optimum_insurance_shipment`, `optimum_insurance_general` |
| **Views** | `insurance_offer_views.xml` in each submodule |
| **Condition** | `readonly="has_child_offers or request_policy_generated"` |
| **Fields affected** | `employee_ids`, `medical_category_ids` (medical); `vehicle_ids` (vehicle); `shipment_ids` (shipment); `estate_ids` (general) |

### C-132 — Offer action buttons hidden after policy generation

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **View** | `views/insurance_offer_views.xml` |
| **Condition** | `invisible="request_policy_generated or ..."` |
| **Buttons affected** | `action_client_accept_offer`, `action_record_response_not_ready`, `action_accept_offer`, `action_create_negotiated_offer`, `action_negotiate_offer`, `action_send_offer_to_client`, `action_send_not_ready`, `action_send_to_insurance_company` |
| **Purpose** | Hides all workflow action buttons on offers when the parent request already has a generated policy |

## 9. Field Access Controls (`groups=`)

### C-136 — Commission fields are accountant-only

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.commission.mixin` (so `insurance.policy` and `insurance.policy.endorsement`), `insurance.policy` |
| **File** | `models/mixins/insurance_commission_mixin.py`, `models/insurance_policy.py`, `views/insurance_policy_views.xml`, `views/insurance_policy_endorsement_views.xml` |
| **Fields** | Mixin: `is_commission_exception`, `commission_rate`, `is_flat_commission` (FR-BASE-006-08-06, «عموله مقتوعه»), `contract_id`, `commission_amount`, `commission_bonus`, `total_commission_amount`. Policy roll-ups: `current_commission_amount`, `current_commission_bonus`, `current_total_commission_amount`. The endorsement's related contract/toggle/rate/flat flag inherit the group from their policy target. Installment (`policy.payment.schedule`): `commission_amount`, `bonus_tier_id`, `bonus_percentage`, `bonus_amount`. |
| **Mechanism** | `groups='optimum_insurance_base.group_insurance_accountant'` on the field; the ORM refuses any Python attribute read, `read()`, fetch, `write()`/`create()` and search domains on it for non-members. The policy form's Commission page and the endorsement form's Commission page carry the same `groups=` so non-accountants see no empty heading. System paths run as superuser: the installment's and the owner's stored bonus computes (D-083), `_assign_installment_net_amounts` (sudo), the policy's stored `current_*` roll-ups (stored computes run as superuser), the accept-offer wizard (C-135) and the batch line's snapshots and eligibility check (C-184; the batch totals and invoice lines read the ungated snapshots). |
| **Also** | Accountants hold read-only ACL on `insurance.policy`, `insurance.policy.endorsement` and `insurance.policy.endorsement.line` so the gated figures are reachable by someone. |
| **Error** | Odoo's standard field access error (_"You do not have enough rights to access the fields ..."_) |
| **FR** | FR-BASE-007-09-08, FR-BASE-006-11-25, FR-BASE-006-11-26, FR-BASE-020-08-05 |
| **Tests** | `TestFlatCommissionPolicy.test_05` (the flat flag) |

### C-169 — Incentive payouts are accountant-only

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.incentive`, `insurance.incentive.figure`, `insurance.company.contract` |
| **File** | `models/insurance_incentive_figure.py` (`PAYOUT_GROUPS`), `models/insurance_incentive.py`, `models/insurance_company_contract.py` |
| **Fields** | `insurance.incentive`: `payout_ids`, `payout_count`; `insurance.incentive.figure`: `current_value_text`; `insurance.company.contract`: `incentive_payout_count` |
| **Mechanism** | `groups=PAYOUT_GROUPS` (`optimum_insurance_base.group_insurance_accountant`, `base.group_system`) on all four fields; the ORM refuses any read of them for non-members. The payout model itself (`insurance.incentive.payout`) has ACL rows for these two groups only. |
| **Error** | Odoo's standard field access error (_"You do not have enough rights to access the fields ..."_) |
| **FR** | FR-BASE-013-07-01, FR-BASE-009-05-18 |
| **Purpose** | The pricing team leader reads incentives but not what they paid (FR-BASE-013-07-01). |

### C-203 — The batch-receipt models are accountant and sysadmin only

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Mechanism** | ACL rows in `security/ir.model.access.csv`; no other group has a row, so the ORM refuses them every access |
| **Error** | Odoo's standard access error (_"You are not allowed to create ..."_) |
| **FR** | FR-BASE-006-08-07, FR-BASE-013-02-01, FR-BASE-013-02-05 |
| **Tests** | `TestPremiumSchedule.test_07`, `test_10`; `TestAdminAccountingAccess.test_pricing_cannot_read_commission_batch_line`, `test_accountant_can_read_commission_batch_line`, `test_pricing_cannot_create_commission_batch_reason_wizard`, `test_accountant_can_create_commission_batch_reason_wizard` |

| Model | Group | Read | Write | Create | Delete |
|---|---|---|---|---|---|
| `insurance.premium.schedule.wizard` | `optimum_insurance_base.group_insurance_accountant` | 1 | 1 | 1 | 0 |
| `insurance.premium.schedule.wizard` | `base.group_system` | 1 | 1 | 1 | 1 |
| `insurance.commission.batch.line` | `optimum_insurance_base.group_insurance_accountant` | 1 | 1 | 1 | 1 |
| `insurance.commission.batch.line` | `base.group_system` | 1 | 1 | 1 | 1 |
| `insurance.commission.batch.reason.wizard` | `optimum_insurance_base.group_insurance_accountant` | 1 | 1 | 1 | 0 |
| `insurance.commission.batch.reason.wizard` | `base.group_system` | 1 | 1 | 1 | 1 |

### C-216 — States, districts and areas: everyone reads, only the administrator edits

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Mechanism** | ACL rows in `security/ir.model.access.csv`. For states, the module overrides Odoo core's row `base.access_res_country_state_group_user` so *Contact Creation* (`base.group_partner_manager`) keeps read only, and adds a full row for `base.group_system`, which doesn't imply *Contact Creation*. Countries need no row: core already makes them admin-only. |
| **Error** | Odoo's standard access error (_"You are not allowed to modify ..."_) |
| **FR** | FR-BASE-017-01-04, FR-BASE-017-01-05 |
| **Tests** | `TestGeographyAccess` |

| Model | Group | Read | Write | Create | Delete |
|---|---|---|---|---|---|
| `res.country.state` | `base.group_partner_manager` (core row, overridden) | 1 | 0 | 0 | 0 |
| `res.country.state` | `base.group_system` | 1 | 1 | 1 | 1 |
| `res.country.state.district` | `base.group_user` | 1 | 0 | 0 | 0 |
| `res.country.state.district` | `base.group_system` | 1 | 1 | 1 | 1 |
| `res.country.state.district.area` | `base.group_user` | 1 | 0 | 0 | 0 |
| `res.country.state.district.area` | `base.group_system` | 1 | 1 | 1 | 1 |

### C-241 — Collection roles: who may touch trips and items

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_collection` |
| **Mechanism** | ACL rows in `security/ir.model.access.csv`. Base models (`insurance.cheque`, `policy.payment.schedule`, `insurance.policy`, `insurance.policy.endorsement`, `client`; `insurance.claim` for the claims team) are read-only to the collection groups. Trips and items are written with the user's own rights; listing and the paper's place run with the system's (D-24, C-230) |
| **Error** | Odoo's standard access error |
| **FR** | FR-COL-010-01-01 to FR-COL-010-01-05 |
| **Tests** | `TestAccess`, `TestClaimNeeds.test_each_team_sees_its_own` |

| Model | Group | Read | Write | Create | Delete |
|---|---|---|---|---|---|
| `insurance.collection.delivery` | scheduler, manager, claims | 1 | 1 | 1 | 0 |
| `insurance.collection.delivery` | `base.group_system` | 1 | 1 | 1 | 1 |
| `insurance.collection.delivery.item` | scheduler, manager, claims, `base.group_system` | 1 | 1 | 1 | 1 |
| `insurance.collection.custody` (Document Tracking) | scheduler, manager, claims, `base.group_system` | 1 | 0 | 0 | 0 |

Open: the scheduler's and the manager's rows are the same, and no button carries `groups=`, while FR-COL-010-01-02 gives assigning, results and costs to the manager only (review 2026-11-10).

### C-242 — Each team sees its own trips, items and papers

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_collection` |
| **Model** | `insurance.collection.delivery`, `insurance.collection.delivery.item`, `insurance.collection.custody` |
| **Mechanism** | record rules in `security/security.xml`: `[('team', '=', 'collection')]` for the scheduler (and so the manager), `[('team', '=', 'claims')]` for the claims group, `[(1, '=', 1)]` for `base.group_system` |
| **Condition** | a user reads or writes a record of the other team. Example: the claims team does not see the CHQ-12 item; the collection lead does not see a claim's papers |
| **Error** | records are filtered out; a direct access raises Odoo's standard access error |
| **FR** | FR-COL-013-02-02, FR-COL-013-02-03 |
| **Tests** | `TestClaimNeeds.test_each_team_sees_its_own`, `TestDocumentTracking` |

### C-243 — The claims team reads claims, never writes them

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_collection` |
| **Model** | `insurance.claim` |
| **Mechanism** | ACL read row plus a read-only record rule `rule_insurance_claim_collection_claims` (`[(1, '=', 1)]`, read only) for `group_insurance_collection_claims` |
| **Condition** | a claims-team member writes, creates or deletes a claim → refused. Reading covers every claim |
| **Error** | Odoo's standard access error |
| **FR** | FR-COL-010-01-05 |
| **Tests** | `TestClaimNeeds.test_each_team_sees_its_own` |

Open: FR-COL-010-01-05 says the team reads "the claims its items name"; the rule reads all claims (review 2026-11-10).

---

## Summary

| Category | Count |
|----------|-------|
| Delete Prevention (unlink) | 12 (C-190, C-198 added by commission-batch-receipts; C-221, C-222 collection trips and items) |
| Write Validation (write) | 36 (C-251 a claim's calculated fields are set only by the system; C-245 a claim is settled only when ready, C-248 a new claim starts Fresh, C-249 a claim needs an expected settlement date; C-183, C-186, C-187, C-188, C-194, C-195, C-196, C-197, C-207, C-208, C-212, C-215 added; C-223 to C-230 collection; C-244 claim lock tiers, D-279) |
| Python Constraints (@api.constrains) | 70 (with sub-controls; C-254 a number plate is an Egyptian plate, D-295; C-250 the closing stamps exist only on a closed claim, D-292; C-246 a rejected claim has its reason, D-281; C-217, C-218 added; C-177, C-178 are formula figure constraints; C-179 to C-182, C-184, C-191 to C-193, C-201, C-202, C-205 added; C-134 retired → C-184, not counted; C-185 moved to SQL UNIQUE, D-211; C-210 added, D-219; C-231 to C-234 collection) |
| SQL UNIQUE Constraints | 26 (with sub-controls; C-185 partial unique index, D-211; C-211 a contract is renewed once, D-226; C-220 a policy has one renewal request) |
| SQL CHECK Constraints | 11 (with sub-controls) |
| Foreign Key Restrictions (ondelete restrict) | 55 rows (53 controls: C-148 covers three FKs; C-213 added) |
| Action Method Guards | 32 (C-253 a policy is created only by accepting an offer, D-294; C-247 only the team leader and the administrator reopen a closed claim, D-283; C-189, C-199, C-200, C-206, C-209, C-219 added; C-235 to C-240 collection; C-189's confirm checks also re-run from a line `@api.constrains` on a correction in Confirmed, D-239, counted here once) |
| Post-Policy Immutability | 13 (C-126 through C-132, with sub-controls C-130-01 to C-130-04, C-131-01, C-131-02) |
| Conditional View Readonly | 10 patterns (with 4 extensions); C-252, a client pre-filled by the opening screen |
| Always-Readonly View Fields | 100+ fields across all modules (incl. C-170, incentive payouts) |
| No-Create Relational Pickers | 1 pattern (with 2 extensions) |
| Feature Screen Rules (7d) | 2 (C-204, the batch-receipt screens; C-214, contract renewal) |
| Field Access Controls (groups=, ACL) | 7 (C-136, C-169; C-203 the batch-receipt models' ACL; C-216 geography is admin-edited; C-241 to C-243 collection roles, team rules, claims read) |
| Amended by commission-batch-receipts | C-136 (flat flag gated), C-140 (the cleared lock also covers the method and transfer fields), C-143 (receipt pointer and `is_commission_batched` system-set), C-151 (the scan is a cheque's only) |
| **Total named controls** | **C-01 through C-254 (with sub-controls; C-134 retired)** |
