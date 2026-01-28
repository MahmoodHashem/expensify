## Proposal

### Please re-state the problem that we are trying to solve in this issue.

Expense - IOU thread is not bold in LHN after receiving a message about editing the expense and When a user sends a message or comment in an IOU thread, the **thread incorrectly appears bold** in the LHN for the  **sender themselves** 

### What is the root cause of that problem?

Unread/bold state in LHN is computed by `isUnread()` in `ReportUtils.ts`, which compares:
- `lastReadTime` from the parent report
- `lastVisibleActionCreated` (the latest visible action timestamp)


https://github.com/Expensify/App/blob/8f2427a4297d27d0415f50b0729753235c409f0d/src/libs/ReportUtils.ts#L8989-L9003

For one‑transaction IOUs, activity can live on the transaction thread (e.g., `MODIFIED_EXPENSE` when the amount is edited), but the unread logic mostly depends on parent report fields. This mismatch causes both bugs:

Bug 1 – “Sender sees bold after their own edit”
- When the sender edits the expense, we optimistically add a `MODIFIED_EXPENSE` action and update `lastReadTime` on the transaction thread (to mark the sender’s action as read).
- However, `isUnread()` only uses the parent report’s `lastReadTime`, which remains older.
- As a result, LHN still thinks there is unread activity and shows the row in bold for the sender.

Bug 2 – “Payer does not see bold after edit, but sees the New marker inside”
- The edit action is added to the transaction thread, and when the payer opens the thread, the “New message” marker appears because the thread’s action is newer than the read time.
- But LHN bolding depends on `lastVisibleActionCreated` on the parent report, which is not consistently updated for `MODIFIED_EXPENSE` actions on the thread.
- So LHN doesn’t “see” the newer action and stays unbolded, even though the thread itself correctly shows the unread marker.

In short:
Read state is stored on the thread, but LHN only checks the parent’s read time → sender sees bold incorrectly.
Latest action is on the thread, but LHN only checks parent’s last visible action → payer does not see bold, while the thread still shows “New message.”



### What changes do you think we should make in order to solve the problem?

- Determine the latest visible action time by checking actual visible report actions for both the parent report and its one‑transaction thread, then use the newest value for unread checks.
- Use the newest `lastReadTime` between the parent report and the transaction thread when determining unread status.
- Ensure callers pass the archived state into the last‑visible‑action calculation so visibility rules are consistent.

So we should update these functions:

1.

```diff
--- a/src/libs/ReportUtils.ts
+++ b/src/libs/ReportUtils.ts
@@ -9025,13 +9025,21 @@ function isUnread(report: OnyxEntry<Report>, oneTransactionThreadReport: OnyxEntr
     if (isEmptyReport(report, isReportArchived)) {
         return false;
     }
     // lastVisibleActionCreated and lastReadTime are both datetime strings and can be compared directly
-    const lastVisibleActionCreated = getReportLastVisibleActionCreated(report, oneTransactionThreadReport);
-    const lastReadTime = report.lastReadTime ?? '';
+    const lastVisibleActionCreated = getReportLastVisibleActionCreated(report, oneTransactionThreadReport, isReportArchived);
+    const reportLastReadTime = report.lastReadTime ?? '';
+    const threadLastReadTime = oneTransactionThreadReport?.lastReadTime ?? '';
+    const lastReadTime = reportLastReadTime > threadLastReadTime ? reportLastReadTime : threadLastReadTime;
     const lastMentionedTime = report.lastMentionedTime ?? '';
 
     // If the user was mentioned and the comment got deleted the lastMentionedTime will be more recent than the lastVisibleActionCreated
     return lastReadTime < (lastVisibleActionCreated ?? '') || lastReadTime < lastMentionedTime;
 }

 
-function getReportLastVisibleActionCreated(report: OnyxEntry<Report>, oneTransactionThreadReport: OnyxEntry<Report>) {
-    const reportLastVisibleActionCreated = report?.lastVisibleActionCreated ?? '';
-    const threadLastVisibleActionCreated = oneTransactionThreadReport?.lastVisibleActionCreated ?? '';
-    return reportLastVisibleActionCreated > threadLastVisibleActionCreated ? reportLastVisibleActionCreated : threadLastVisibleActionCreated;
+function getReportLastVisibleActionCreated(report: OnyxEntry<Report>, oneTransactionThreadReport: OnyxEntry<Report>, isReportArchived = false) {
+    const reportLastVisibleActionCreated = report?.lastVisibleActionCreated ?? '';
+    const reportLastVisibleAction = report?.reportID
+        ? getLastVisibleActionReportActionsUtils(report.reportID, canUserPerformWriteAction(report, isReportArchived))
+        : undefined;
+    const reportLastVisibleActionFromActions = reportLastVisibleAction?.created ?? '';
+    const resolvedReportLastVisibleActionCreated =
+        reportLastVisibleActionFromActions > reportLastVisibleActionCreated ? reportLastVisibleActionFromActions : reportLastVisibleActionCreated;
+
+    const threadLastVisibleActionCreated = oneTransactionThreadReport?.lastVisibleActionCreated ?? '';
+    const threadLastVisibleAction = oneTransactionThreadReport?.reportID
+        ? getLastVisibleActionReportActionsUtils(oneTransactionThreadReport.reportID, canUserPerformWriteAction(oneTransactionThreadReport, isReportArchived))
+        : undefined;
+    const threadLastVisibleActionFromActions = threadLastVisibleAction?.created ?? '';
+    const resolvedThreadLastVisibleActionCreated =
+        threadLastVisibleActionFromActions > threadLastVisibleActionCreated ? threadLastVisibleActionFromActions : threadLastVisibleActionCreated;
+
+    return resolvedReportLastVisibleActionCreated > resolvedThreadLastVisibleActionCreated ? resolvedReportLastVisibleActionCreated : resolvedThreadLastVisibleActionCreated;
 }

``` 

- **isUnread** now uses the newest `lastReadTime` between the parent report and the one‑transaction thread. This prevents the sender from seeing bold after their own edit (thread read time is updated for the sender
- **getReportLastVisibleActionCreated** now resolves the latest visible action from actual report actions for both parent + thread, so edits on the thread are considered even if` lastVisibleActionCreated `wasn’t updated.


2. Pass `isReportArchived` so visibility rules are correct when computing the latest visible action.
```diff
--- a/src/pages/inbox/report/ReportActionsList.tsx
+++ b/src/pages/inbox/report/ReportActionsList.tsx
@@ -348,7 +348,7 @@ function ReportActionsList({
 
     const lastActionIndex = lastAction?.reportActionID;
     const reportActionSize = useRef(sortedVisibleReportActions.length);
-    const lastVisibleActionCreated = getReportLastVisibleActionCreated(report, transactionThreadReport);
+    const lastVisibleActionCreated = getReportLastVisibleActionCreated(report, transactionThreadReport, isReportArchived);
     const hasNewestReportAction = lastAction?.created === lastVisibleActionCreated || isReportPreviewAction(lastAction);
```


3. pass `isReportArchived `so the “latest visible action” is computed using proper visibility rules.

```diff
--- a/src/components/MoneyRequestReportView/MoneyRequestReportActionsList.tsx
+++ b/src/components/MoneyRequestReportView/MoneyRequestReportActionsList.tsx
@@ -246,7 +246,7 @@ function MoneyRequestReportActionsList({
 
     const reportActionSize = useRef(visibleReportActions.length);
     const lastAction = visibleReportActions.at(-1);
     const lastActionIndex = lastAction?.reportActionID;
     const previousLastIndex = useRef(lastActionIndex);
@@ -249,7 +249,7 @@ function MoneyRequestReportActionsList({
     const scrollingVerticalBottomOffset = useRef(0);
     const scrollingVerticalTopOffset = useRef(0);
     const wrapperViewRef = useRef<View>(null);
     const readActionSkipped = useRef(false);
-    const lastVisibleActionCreated = getReportLastVisibleActionCreated(report, transactionThreadReport);
+    const lastVisibleActionCreated = getReportLastVisibleActionCreated(report, transactionThreadReport, isReportArchived);
```



### What alternative solutions did you explore? (Optional)

**Reminder:** Please use plain English, be brief and avoid jargon. Feel free to use images, charts or pseudo-code if necessary. Do not post large multi-line diffs or write walls of text. Do not create PRs unless you have been hired for this job.