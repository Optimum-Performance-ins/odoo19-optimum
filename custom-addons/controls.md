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
| **Condition** | Any cheque in the recordset has `state == 'cleared'` and the write touches a field of `_LOCKED_WHILE_CLEARED` (`name`, `bank_id`, `currency_id`, `amount`, `client_id`, `insurance_company_id`, `payment_date`, `line_ids`, `cheque_document`, `cheque_document_filename`). `state` and the chatter/activity fields stay writable so the cheque can leave cleared |
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

### C-143 — Paid flag, payment date and net amount are system-set

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `policy.payment.schedule` |
| **File** | `models/policy_payment_schedule.py` (`create`/`write` → `_check_system_set`) |
| **Condition** | `is_paid`, `payment_date` or `net_amount` present in `vals`, outside the system's own `_installment_system_write` context |
| **Error** | _"%s on an installment are set by the system and cannot be edited."_ |
| **FR** | FR-BASE-006-11-21, FR-BASE-006-11-23 |

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

### C-157 — A covered installment's net amount never moves

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `policy.payment.schedule` |
| **File** | `models/mixins/insurance_commission_mixin.py` (`_assign_installment_net_amounts`) |
| **Method** | `_assign_installment_net_amounts()` |
| **Condition** | A create, delete, amount or due-date change on any installment of an owner would change the `net_amount` of an installment of that owner that has a `cheque_id` (any cheque state). |
| **Error** | _"%(line)s is covered by cheque %(cheque)s. This change would move the administrative fees of %(owner)s onto or off it. Remove it from the cheque first."_ |
| **FR** | FR-BASE-006-11-23, FR-BASE-013-06-06 |
| **Purpose** | D-066 applied to the one figure another row's edit can move: a covered installment cannot be modified, and its net amount is the basis of its bonus. Without it, moving an uncovered installment before a cleared first one would silently change a paid bonus (D-069 A3). |

---

## 3. Python Constraints (`@api.constrains`)

### C-07 — Policy commission validation

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.policy` |
| **File** | `models/insurance_policy.py` (line ~452) |
| **Constrains** | `is_commission_exception`, `commission_rate`, `contract_id` |
| **Errors** | _"For manual commission policies, you must specify a commission rate greater than 0."_ / _"No active contract found for company '%s' on date %s."_ |

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

#### C-09-01 — Medical offer skips base coverage check

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_medical` |
| **Model** | `insurance.offer` (inherited) |
| **File** | `models/insurance_offer.py` (line ~134) |
| **Constrains** | `coverage_ids`, `stage` (must repeat the base triggers — re-declaring `@api.constrains` replaces them) |
| **Behavior** | Overrides base `_check_has_coverage` — skips coverage check for medical offers (uses medical categories instead); non-medical offers keep the base check |

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

### C-134 — Commission batch accepts only policies with a commission

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.commission.batch` |
| **File** | `models/insurance_commission_batch.py` |
| **Constrains** | `policy_ids` |
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
| **File** | `models/insurance_cheque.py` (`_check_cheque_document`) |
| **Constrains** | `cheque_document`, `cheque_document_filename` |
| **Condition** | `cheque_document` is not set, or `cheque_document_filename` does not end with `.pdf`, `.png`, `.jpg` or `.jpeg` (case-insensitive); enforced through `default=False`, which puts the field in every create's values — `required=True` alone is a client hint for attachment binaries |
| **Error** | _"Cheque %s needs a scan of the physical cheque."_ / _"The scan of cheque %s must be a PDF or an image (.pdf, .png, .jpg, .jpeg)."_ |
| **FR** | FR-BASE-013-06-09 |

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
| `first_check_completed`, `early_payment_bonus` | Computed |
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
| **Fields** | Mixin: `is_commission_exception`, `commission_rate`, `contract_id`, `commission_amount`, `early_payment_bonus`, `commission_bonus_basis`, `commission_bonus`, `total_commission_amount`. Policy roll-ups: `current_commission_amount`, `current_commission_bonus`, `current_total_commission_amount`. The endorsement's related contract/toggle/rate inherit the group from their policy target. |
| **Mechanism** | `groups='optimum_insurance_base.group_insurance_accountant'` on the field; the ORM refuses any Python attribute read, `read()`, fetch, `write()`/`create()` and search domains on it for non-members. The policy form's Commission page and the endorsement form's Commission page carry the same `groups=` so non-accountants see no empty heading. System paths run as superuser: `_earn_early_payment_bonus` (sudo), the policy's stored `current_*` roll-ups (stored computes run as superuser), the accept-offer wizard (C-135), the batch's invoice lines and C-134. |
| **Also** | Accountants hold read-only ACL on `insurance.policy`, `insurance.policy.endorsement` and `insurance.policy.endorsement.line` so the gated figures are reachable by someone. |
| **Error** | Odoo's standard field access error (_"You do not have enough rights to access the fields ..."_) |
| **FR** | FR-BASE-007-09-08 |

---

## Summary

| Category | Count |
|----------|-------|
| Delete Prevention (unlink) | 7 |
| Write Validation (write) | 7 |
| Python Constraints (@api.constrains) | 43 (with sub-controls) |
| SQL UNIQUE Constraints | 22 (with sub-controls) |
| SQL CHECK Constraints | 10 (with sub-controls) |
| Foreign Key Restrictions (ondelete restrict) | 51 rows (49 controls: C-148 covers three FKs) |
| Action Method Guards | 17 |
| Post-Policy Immutability | 13 (C-126 through C-132, with sub-controls C-130-01 to C-130-04, C-131-01, C-131-02) |
| Conditional View Readonly | 9 patterns (with 4 extensions) |
| Always-Readonly View Fields | 100+ fields across all modules |
| No-Create Relational Pickers | 1 pattern (with 2 extensions) |
| Field Access Controls (groups=) | 1 (C-136) |
| **Total named controls** | **C-01 through C-155 (with sub-controls)** |
