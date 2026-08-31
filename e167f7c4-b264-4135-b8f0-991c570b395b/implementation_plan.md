# Implementation Plan: Magento Order Creation Failure & Reconciliation

## Goal
Add a dedicated, highly detailed section to the `magento_deep_dive.md` document that answers exactly what happens when a payment succeeds but Magento order creation fails. This will be framed as a major production reliability problem you solved, ready for an Engineering Director interview.

## Research Findings (Confirmed from Repository)

I have thoroughly investigated the codebase and found the exact sequence of events for this failure scenario. Here is what I will document:

1. **The Core Problem**: A customer pays via the Euler/JusPay gateway, Vayu receives the success webhook (or polling), but the synchronous call to Magento's `createOrderAPI` fails (e.g., 500 error, timeout). The customer is charged, but no Magento order exists.
2. **Detection & ProcessTracker**: When payment succeeds, Vayu spawns a ProcessTracker (PT) task called `RunnerEnum_PLATOFORM_ORDER_CREATION_WORKFLOW`.
3. **The Worker Flow**: The PT worker (`processPlatformOrderWorkflow`) picks this up and tries to create the Magento order asynchronously.
4. **The Ghost Order Problem (Race Condition / Dropped Response)**: If Vayu retries the Magento API, it risks creating a duplicate order if Magento actually *did* create the order on the first attempt but the HTTP response dropped.
5. **The Idempotency / Recovery Mechanism**: I found `MissingOrderResolution.hs` and `PlatformMagento.prefetchMagentoOrder`. Before retrying order creation, Vayu calls `searchOrderByQuoteId` on Magento. If Magento already has an order for this cart, Vayu simply syncs the ID and marks the task as successful, avoiding duplicates!
6. **Ultimate Failure Handling**: If the API fails repeatedly and hits the max retry limit (`MANDATE_PLATFORM_ORDER_MAX_RETRY_COUNT`), Vayu marks the Breeze order status as `AUTO_CANCELLED` and uses `sendMerchantNotificationEmail` to alert the merchant ("Action Required: Missing Order on Your Platform") with customer contact and order value details so they can manually refund or fulfill it.

## Proposed Additions to the Document

I will insert a new major section into the existing artifact:
**"Challenge 4: Magento Order Creation Failures (Payment Success vs Platform Failure)"**

It will include:
1. **The Scenario**: Step-by-step breakdown of the failure.
2. **Mermaid Diagrams**: 
   - Diagram 1: The standard failure + retry loop.
   - Diagram 2: The "Ghost Order" dropped-response scenario and how `prefetchMagentoOrder` prevents duplicate creation.
3. **The Resolution Flow**: Detailing ProcessTracker's exponential backoff, MissingOrderResolution checks, and the final `AUTO_CANCELLED` + Email alert fallback.
4. **Interview Story**: A polished STAR-format story you can use.
5. **Director-Level Follow-Up Questions**: Focus on distributed transactions, eventually consistent systems, idempotency, and the trade-off of alerting merchants vs auto-refunding.

## Review Requested
Please review this plan. Upon your approval, I will modify the existing `magento_deep_dive.md` artifact to include this extensive new section.
