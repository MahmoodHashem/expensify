## Proposal

### Please re-state the problem that we are trying to solve in this issue.
When splitting a distance expense, the distance map step does not draw the route line, and the confirmation step shows an empty split/total with “Pending” for distance, even after entering valid start/stop waypoints.

### What is the root cause of that problem?
Flow-wise, this is what happens in the split distance path:
1) The split flow initializes a draft transaction for the distance expense with the generic type (`iouRequestType: distance`) rather than the map-specific type (`src/libs/actions/IOU/index.ts:1129-1187`).
2) The distance map step and the confirmation step both rely on `useFetchRoute` to request a route when waypoints are valid (`src/hooks/useFetchRoute.ts:14-42`).
3) `useFetchRoute` only fetches when `isMapDistanceRequest(transaction)` is true (`src/hooks/useFetchRoute.ts:28-34`). `isMapDistanceRequest` only returns true for `iouRequestType: distance-map` (pre‑save) or for saved distance transactions (`src/libs/TransactionUtils/index.ts:215-222`).
4) Because the split draft is `iouRequestType: distance`, the route fetch never triggers, so no route is written into `transaction.routes.route0.geometry.coordinates`.
5) Without route coordinates, the map cannot draw the green line and the confirmation step has no distance/amount data to show (stays pending).

### What changes do you think we should make in order to solve the problem?
Allow draft distance transactions that are still marked as the generic request type (`iouRequestType: distance`) to be treated as route-eligible in the fetch hook. The smallest change is in `src/hooks/useFetchRoute.ts`, expanding the `shouldFetchRoute` condition to include draft transactions with `iouRequestType: distance`.

Solution code (diff). Apply in `src/hooks/useFetchRoute.ts` near the `isMapDistanceRequest` / `shouldFetchRoute` logic:
```diff
 const isMapDistanceRequest = isMapDistanceRequestTransactionUtils(transaction);
-const shouldFetchRoute = isMapDistanceRequest && (isRouteAbsentWithoutErrors || haveValidatedWaypointsChanged) && !isLoadingRoute && Object.keys(validatedWaypoints).length > 1;
+const isDraftGenericDistanceRequest =
+    transactionState === CONST.TRANSACTION.STATE.DRAFT &&
+    transaction?.iouRequestType === CONST.IOU.REQUEST_TYPE.DISTANCE;
+const shouldFetchRoute =
+    (isMapDistanceRequest || isDraftGenericDistanceRequest) &&
+    (isRouteAbsentWithoutErrors || haveValidatedWaypointsChanged) &&
+    !isLoadingRoute &&
+    Object.keys(validatedWaypoints).length > 1;

### What alternative solutions did you explore? (Optional)
- Forcing the split distance flow to set `iouRequestType: distance-map` in the draft. This would work but changes semantics of the generic distance type and risks side effects in other flows.
- Fetching the route directly in the split distance screen. This duplicates logic already handled by `useFetchRoute` and would be harder to maintain.

**Reminder:** Please use plain English, be brief and avoid jargon. Feel free to use images, charts or pseudo-code if necessary. Do not post large multi-line diffs or write walls of text. Do not create PRs unless you have been hired for this job.
