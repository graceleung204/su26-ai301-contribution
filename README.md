# Contribution [#]: [Wrap BC billing save/update multi-DAO writes in transactional services]

**Contribution Number:** [1]  
**Student:** [Grace Leung]  
**Issue:** [[GitHub issue link](https://github.com/carlos-emr/carlos/issues/2149)]  
**Status:** [Phase III] [IN PROGRESS]

---

## Why I Chose This Issue

I am interested in this issue because it is related to Java and Javascript and backend service layer refactorization. I have experiences working on backend development and payments related services, so this issue seems to be more align with my expertise. 

I understand that currently the billing save/update actions are still in the Struts action which is in the controller layer, but should be moved to transactional layer for better peforamnces. This is mainly a refactoring and writing regression tests to make sure the changes work and is still compatible. 

---

## Understanding the Issue

### Problem Description

BC billing save/update actions are currently performing multiple DAO writes directly inside Struts action classes. This creates a risk where a failure in any later DAO call can leave the system in a partially persisted state (e.g., billing rows saved but related appointment or master records not updated).

The current PR improved validation and removed unsafe logging, but did not address the underlying architectural issue of missing transactional boundaries across these multi-step persistence operations.

### Expected Behavior

All billing-related persistence operations (billing rows, billing master records, appointment updates/archival, WCB linkage, and receipt redirect prerequisites) should be executed within a single transactional service boundary.

If any step fails, the entire operation should be rolled back to prevent partial data persistence and ensure atomicity.

Controller (Struts Action) code should only handle:

Request validation
Authorization
Delegation to service layer
Response handling / redirects

### Current Behavior

BillingSaveBilling2Action and BillingUpdateBilling2Action directly invoke multiple DAO operations sequentially.
No unified transaction boundary exists across these operations.
If a failure occurs after initial DAO writes succeed, partial data may be committed.
Error handling is inconsistent and does not guarantee rollback of earlier writes.

### Affected Components

BillingSaveBilling2Action
BillingUpdateBilling2Action
Billing DAO layer (billing rows, billing master, appointment, WCB linkage)
Receipt/redirect prerequisite logic tied to billing save/update flow

---

## Reproduction Process

### Environment Setup

The CARLOS EMR development environment was set up using the DevContainer workflow.

Key setup steps:

Installed Docker Desktop and VS Code Dev Containers extension
Cloned repository and opened project in VS Code
Reopened project inside DevContainer (Reopen in Container)
Waited for initial build to complete (Maven + DB initialization)
Verified application running at http://localhost:8080
Logged in using dev credentials:
Username: carlosdoc
Password: carlos2026
PIN: 2026



### Steps to Reproduce

Although full deterministic reproduction requires simulating DAO failure mid-transaction, the issue can be observed under partial failure conditions in the billing flow:

Step 1: Start CARLOS EMR in DevContainer environment
Step 2: Log in to the application
Step 3: Navigate to billing workflow:
- Create or update a billing entry via BillingSaveBilling2Action or BillingUpdateBilling2Action
- Modify execution flow (or simulate failure condition):
- Force a DAO exception after initial billing row insert (e.g., DB constraint violation, network interruption, or injected failure in later DAO call such as appointment update or WCB linkage)
- Submit billing save/update request

Observed result:

- Initial billing records are persisted
- Subsequent DAO operation fails
- Related entities (billing master / appointment / WCB linkage) are not consistently updated
- System remains in partially committed state

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** 
- **My findings:**
No transactional boundary wraps the full billing save/update flow
Each DAO call commits independently instead of participating in a single atomic transaction
Failure in later stage does not rollback earlier writes
Confirms need for service-layer transaction refactor as described in PR scope

---

## Solution Approach

### Analysis

Based on the PR review comments and initial code inspection, the likely root cause is that BillingSaveBilling2Action and BillingUpdateBilling2Action perform multiple persistence operations directly from the Struts action layer.

These actions appear to execute several DAO writes sequentially, including billing row persistence, billing master updates, appointment updates, WCB linkage operations, and receipt-related processing. Because these operations are not wrapped in a single transactional service boundary, a failure in a later DAO call may leave earlier writes committed, resulting in partial persistence and inconsistent billing data.

The issue is primarily architectural rather than a validation or security problem. Business logic and persistence orchestration currently reside in the controller layer, making transaction management difficult and failure behavior harder to test.Based on the PR review comments and initial code inspection, the likely root cause is that BillingSaveBilling2Action and BillingUpdateBilling2Action perform multiple persistence operations directly from the Struts action layer.

These actions appear to execute several DAO writes sequentially, including billing row persistence, billing master updates, appointment updates, WCB linkage operations, and receipt-related processing. Because these operations are not wrapped in a single transactional service boundary, a failure in a later DAO call may leave earlier writes committed, resulting in partial persistence and inconsistent billing data.

The issue is primarily architectural rather than a validation or security problem. Business logic and persistence orchestration currently reside in the controller layer, making transaction management difficult and failure behavior harder to test.

### Proposed Solution

Introduce a dedicated service-layer transaction boundary for billing save and update workflows.

The Struts actions should be responsible only for request validation, authorization checks, and response handling. All billing-related persistence operations should be delegated to a transactional service method that executes the workflow atomically.

This approach ensures that:

- All billing-related updates succeed together or fail together.
- Partial persistence is prevented through rollback behavior.
- Business logic becomes easier to test independently of the web layer.
- Future billing enhancements can be implemented within a consistent service architecture.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** The billing save and update flows currently perform multiple DAO writes directly within Struts actions. If one operation succeeds and a later operation fails, the system may be left in a partially updated state.

The goal is to move persistence orchestration into a transactional service layer while preserving existing billing behavior.

**Match:** Similar patterns likely already exist elsewhere in the application where:

- Service classes coordinate multiple DAO operations.
- Spring-managed transactions are used to ensure atomic updates.
- Controllers delegate business operations to services rather than interacting with DAOs directly.

Before implementation, identify existing transactional service implementations that can be used as a reference.

**Plan:** [Step-by-step implementation plan]
1. Inspect BillingSaveBilling2Action and document all DAO operations performed during save workflows
2. Inspect BillingUpdateBilling2Action and document all DAO operations performed during update workflows
3. Identify any existing billing service classes or transactional service patterns in the codebase.
4. Create or extend a billing service responsible for coordinating the full save workflow.
5. Create or extend a billing service responsible for coordinating the full update workflow.
6. Move DAO orchestration logic from the Struts actions into service methods.
7. Apply transactional boundaries to the service methods.
8. Refactor actions to delegate to the service layer.
9. Add regression tests covering rollback behavior when failures occur during later persistence steps.
10. Verify existing billing functionality continues to behave as before.

**Implement:** TBD

**Review:** 
- No DAO writes remain directly in Struts actions.
- Transaction boundaries exist at the service layer.
- Existing billing behavior is preserved.
- Rollback behavior is tested.
- New code follows existing project conventions.
- No unnecessary functional changes are introduced.
- Logging and error handling remain consistent with project standards.
- Tests pass locally.

**Evaluate:** 

1. Execute existing billing save workflow and confirm successful persistence.
2. Execute existing billing update workflow and confirm successful persistence.
3. Simulate failure after the first DAO write and verify rollback occurs.
4. Simulate failure during appointment update processing and verify rollback occurs.
5. Simulate failure during WCB linkage processing and verify rollback occurs.
6. Run relevant unit and integration tests.
7. Perform manual regression testing on billing-related user flows.
8. Confirm no partial billing data remains after simulated failures.

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: Verify billing save succeeds when all DAO operations complete successfully.
- [ ] Test case 2: Verify transaction rollback occurs when a failure happens after the initial billing record is saved.
- [ ] Test case 3: Verify billing update succeeds and all related records are updated within the same transaction

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week 1 Progress

1. Set up the CARLOS dev environment and got the application running locally.
2. Reviewed the PR comments and started looking through the billing save/update flow.
3. Identified that BillingSaveBilling2Action and BillingUpdateBilling2Action perform multiple DAO operations directly.
4. Started tracing the code to understand where transaction boundaries should be introduced.

Challenges: The billing flow touches several related operations, so it took some time to follow the full execution path.


### Code Changes

- **Files modified:**
  BillingSaveBilling2Action
  BillingUpdateBilling2Action
  Billing service classes
  Billing-related tests
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
