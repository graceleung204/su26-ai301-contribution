# Contribution [#]: [Wrap BC billing save/update multi-DAO writes in transactional services]

**Contribution Number:** [1]  
**Student:** [Grace Leung]  
**Issue:** [[GitHub issue link](https://github.com/carlos-emr/carlos/issues/2149)]  
**Status:** [Phase I] [Complete]

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

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
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
