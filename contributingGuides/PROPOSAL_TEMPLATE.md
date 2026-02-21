## Proposal

### Please re-state the problem that we are trying to solve in this issue.
After a workspace admin edits a tax code, older expenses that still store the old `taxCode` stop resolving correctly.  
This causes:
- Missing tax name in expense views
- No selected tax in the picker
- Incorrect `taxOutOfPolicy` validation/violation behavior

### What is the root cause of that problem?
When tax code is renamed, the policy key changes from old code to new code and the new tax stores `previousTaxCode`:
- https://github.com/Expensify/App/blob/476152dd009e098f8449f5f9dbee332882391e5d/src/libs/actions/TaxRate.ts#L577-L585

But several downstream paths still check direct key/code equality instead of resolving renamed codes first:
- `getTaxValue()` direct equality: https://github.com/Expensify/App/blob/476152dd009e098f8449f5f9dbee332882391e5d/src/libs/TransactionUtils/index.ts#L2131-L2133
- `getTaxName()` direct equality: https://github.com/Expensify/App/blob/476152dd009e098f8449f5f9dbee332882391e5d/src/libs/TransactionUtils/index.ts#L2145-L2151
- Confirmation validation direct key check: https://github.com/Expensify/App/blob/476152dd009e098f8449f5f9dbee332882391e5d/src/components/MoneyRequestConfirmationList.tsx#L1017-L1018
- Violation fix check direct key check: https://github.com/Expensify/App/blob/476152dd009e098f8449f5f9dbee332882391e5d/src/libs/Violations/ViolationsUtils.ts#L268-L271
- Violation recompute direct key check: https://github.com/Expensify/App/blob/476152dd009e098f8449f5f9dbee332882391e5d/src/libs/Violations/ViolationsUtils.ts#L406-L407

We already have a helper that resolves renamed tax IDs (`getCurrentTaxID`), but these paths are not consistently using it:
- https://github.com/Expensify/App/blob/476152dd009e098f8449f5f9dbee332882391e5d/src/libs/PolicyUtils.ts#L1596-L1598

### What changes do you think we should make in order to solve the problem?
Use `getCurrentTaxID` as the single resolver before tax lookup/validation in all affected paths.

1. `TransactionUtils`:
- In `getTaxValue`, resolve `taxCode` first via `getCurrentTaxID(policy, taxCode) ?? taxCode`
- In `getTaxName`, resolve `(transaction?.taxCode || defaultTaxCode)` first before matching rate code

2. `MoneyRequestConfirmationList`:
- Replace strict key existence check with `!getCurrentTaxID(policy, transaction.taxCode)` for `taxOutOfPolicy`

3. `ViolationsUtils`:
- In `getIsViolationFixed`, treat tax as valid if it matches direct key or renamed mapping
- In violation recompute logic, replace strict key existence with `!!getCurrentTaxID(policy, updatedTransaction.taxCode)`


Code implementations: 

```diff
diff --git a/src/components/MoneyRequestConfirmationList.tsx b/src/components/MoneyRequestConfirmationList.tsx
index e371db0..39e5d05 100755
--- a/src/components/MoneyRequestConfirmationList.tsx
+++ b/src/components/MoneyRequestConfirmationList.tsx
@@ -38,7 +38,7 @@ import Log from '@libs/Log';
 import {validateAmount} from '@libs/MoneyRequestUtils';
 import Navigation from '@libs/Navigation/Navigation';
 import {getIOUConfirmationOptionsFromPayeePersonalDetail, hasEnabledOptions} from '@libs/OptionsListUtils';
-import {getTagLists, isTaxTrackingEnabled} from '@libs/PolicyUtils';
+import {getCurrentTaxID, getTagLists, isTaxTrackingEnabled} from '@libs/PolicyUtils';
 import {isSelectedManagerMcTest} from '@libs/ReportUtils';
 import type {OptionData} from '@libs/ReportUtils';
 import {hasEnabledTags, hasMatchingTag} from '@libs/TagsOptionsListUtils';
@@ -1014,7 +1014,7 @@ function MoneyRequestConfirmationList({
                 return;
             }
 
-            if (shouldShowTax && !!transaction.taxCode && !Object.keys(policy?.taxRates?.taxes ?? {}).some((key) => key === transaction.taxCode)) {
+            if (shouldShowTax && !!transaction.taxCode && !getCurrentTaxID(policy, transaction.taxCode)) {
                 setFormError('violations.taxOutOfPolicy');
                 return;
             }
diff --git a/src/libs/TransactionUtils/index.ts b/src/libs/TransactionUtils/index.ts
index bd184c6..075b4bb 100644
--- a/src/libs/TransactionUtils/index.ts
+++ b/src/libs/TransactionUtils/index.ts
@@ -22,6 +22,7 @@ import {rand64, roundToTwoDecimalPlaces} from '@libs/NumberUtils';
 import {getLoginsByAccountIDs, getPersonalDetailsByIDs} from '@libs/PersonalDetailsUtils';
 import {
     getCommaSeparatedTagNameWithSanitizedColons,
+    getCurrentTaxID,
     getDistanceRateCustomUnit,
     getDistanceRateCustomUnitRate,
     getPolicy,
@@ -2129,7 +2130,8 @@ function transformedTaxRates(policy: OnyxEntry<Policy> | undefined, transaction?
  * Gets the tax value of a selected tax
  */
 function getTaxValue(policy: OnyxEntry<Policy>, transaction: OnyxEntry<Transaction>, taxCode: string) {
-    return Object.values(transformedTaxRates(policy, transaction)).find((taxRate) => taxRate.code === taxCode)?.value;
+    const resolvedTaxCode = getCurrentTaxID(policy, taxCode) ?? taxCode;
+    return Object.values(transformedTaxRates(policy, transaction)).find((taxRate) => taxRate.code === resolvedTaxCode)?.value;
 }
 
 /**
@@ -2147,7 +2149,9 @@ function getTaxName(policy: OnyxEntry<Policy>, transaction: OnyxEntry<Transactio
 
     // transaction?.taxCode may be an empty string
     // eslint-disable-next-line @typescript-eslint/prefer-nullish-coalescing
-    const taxRate = Object.values(transformedTaxRates(policy, transaction)).find((rate) => rate.code === (transaction?.taxCode || defaultTaxCode));
+    const requestedTaxCode = transaction?.taxCode || defaultTaxCode;
+    const resolvedTaxCode = requestedTaxCode ? (getCurrentTaxID(policy, requestedTaxCode) ?? requestedTaxCode) : requestedTaxCode;
+    const taxRate = Object.values(transformedTaxRates(policy, transaction)).find((rate) => rate.code === resolvedTaxCode);
 
     if (shouldFallbackToValue && transaction?.taxValue !== undefined && taxRate?.value !== transaction?.taxValue) {
         return transaction?.taxValue;
diff --git a/src/libs/Violations/ViolationsUtils.ts b/src/libs/Violations/ViolationsUtils.ts
index 589f001..ba0b09e 100644
--- a/src/libs/Violations/ViolationsUtils.ts
+++ b/src/libs/Violations/ViolationsUtils.ts
@@ -11,7 +11,7 @@ import DateUtils from '@libs/DateUtils';
 import {isReceiptError} from '@libs/ErrorUtils';
 import {getCurrentUserEmail} from '@libs/Network/NetworkStore';
 import Parser from '@libs/Parser';
-import {getDistanceRateCustomUnitRate, getPerDiemRateCustomUnitRate, getSortedTagKeys, isDefaultTagName, isTaxTrackingEnabled} from '@libs/PolicyUtils';
+import {getCurrentTaxID, getDistanceRateCustomUnitRate, getPerDiemRateCustomUnitRate, getSortedTagKeys, isDefaultTagName, isTaxTrackingEnabled} from '@libs/PolicyUtils';
 import {isCurrentUserSubmitter} from '@libs/ReportUtils';
 import * as TransactionUtils from '@libs/TransactionUtils';
 import {hasValidModifiedAmount, isViolationDismissed, shouldShowViolation} from '@libs/TransactionUtils';
@@ -267,7 +267,14 @@ function getIsViolationFixed(violationError: string, params: ViolationFixParams)
         },
         [`${CONST.VIOLATIONS_PREFIX}${CONST.VIOLATIONS.TAX_OUT_OF_POLICY}`]: () => {
             // Tax is fixed if it's empty or exists in policy tax rates
-            return !taxCode || Object.keys(policyTaxRates ?? {}).some((key) => key === taxCode);
+            if (!taxCode) {
+                return true;
+            }
+            return !!Object.keys(policyTaxRates ?? {}).find((taxIDKey) => {
+                const taxRate = policyTaxRates?.[taxIDKey];
+                const previousTaxCode = typeof taxRate === 'object' && taxRate !== null && 'previousTaxCode' in taxRate ? (taxRate.previousTaxCode as string | undefined) : undefined;
+                return taxIDKey === taxCode || previousTaxCode === taxCode;
+            });
         },
         [`${CONST.VIOLATIONS_PREFIX}${CONST.VIOLATIONS.MISSING_ATTENDEES}`]: () => {
             // Attendees violation is fixed if getIsMissingAttendeesViolation returns false
@@ -404,7 +411,7 @@ const ViolationsUtils = {
         const isPerDiemRequest = TransactionUtils.isPerDiemRequest(updatedTransaction);
         const isTimeRequest = TransactionUtils.isTimeRequest(updatedTransaction);
         const isPolicyTrackTaxEnabled = isTaxTrackingEnabled(true, policy, isDistanceRequest, isPerDiemRequest, isTimeRequest);
-        const isTaxInPolicy = Object.keys(policy.taxRates?.taxes ?? {}).some((key) => key === updatedTransaction.taxCode);
+        const isTaxInPolicy = !updatedTransaction.taxCode || !!getCurrentTaxID(policy, updatedTransaction.taxCode);
 
         // eslint-disable-next-line @typescript-eslint/prefer-nullish-coalescing
         const amount = hasValidModifiedAmount(updatedTransaction) ? Number(updatedTransaction.modifiedAmount) : updatedTransaction.amount;
```


Why this approach:
- Minimal and low risk (no data migration, no schema change)
- Uses existing remap behavior already intended by the codebase
- Fixes both UI rendering and validation/violation lifecycle, not only display

### What alternative solutions did you explore? (Optional)
1. Update all existing transactions when tax code is renamed.
- Rejected: hard to guarantee complete coverage from frontend cache and higher risk.

2. Add new alias history fields and chain resolution.
- Rejected for this issue: larger model change than needed and not minimal.

3. Only patch `getTaxName`/`getTaxValue`.
- Rejected: leaves false `taxOutOfPolicy` in confirmation and violations flow.

**Reminder:** Please use plain English, be brief and avoid jargon. Feel free to use images, charts or pseudo-code if necessary. Do not post large multi-line diffs or write walls of text. Do not create PRs unless you have been hired for this job.

<!---
ATTN: Contributor+

You are the first line of defense in making sure every proposal has a clear and easily understood problem with a "root cause". Do not approve any proposals that lack a satisfying explanation to the first two prompts. It is CRITICALLY important that we understand the root cause at a minimum even if the solution doesn't directly address it. When we avoid this step, we can end up solving the wrong problems entirely or just writing hacks and workarounds.

Instructions for how to review a proposal:

1. Address each contributor proposal one at a time and address each part of the question one at a time e.g. if a solution looks acceptable, but the stated problem is not clear, then you should provide feedback and make suggestions to improve each prompt before moving on to the next. Avoid responding to all sections of a proposal at once. Move from one question to the next each time asking the contributor to "Please update your original proposal and tag me again when it's ready for review".

2. Limit excessive conversation and moderate issues to keep them on track. If someone is doing any of the following things, please kindly and humbly course-correct them:

- Posting PRs.
- Posting large multi-line diffs (this is basically a PR).
- Skipping any of the required questions.
- Not using the proposal template at all.
- Suggesting that an existing issue is related to the current issue before a problem or root cause has been established.
- Excessively wordy explanations.

3. Choose the first proposal that has a reasonable answer to all the required questions.
-->
