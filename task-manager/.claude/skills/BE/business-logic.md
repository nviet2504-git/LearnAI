---
name: business-logic-engineering
description: >
  Engineer business logic by translating requirements into executable flows, edge cases, rules, and diagrams. Use this skill when analyzing requirements, defining business rules, designing workflows, writing pseudocode, identifying edge cases, creating decision trees, mapping process flows, generating sequence diagrams, or modeling any kind of business logic, domain rules, validation logic, state machines, approval workflows, transaction flows, or system interactions. Also applies when the user mentions requirement analysis, process design, system behavior, rule engines, or needs to document how a system should behave.
---

# Business Logic Engineering

Business logic is the core value-creating code that transforms data according to domain rules. It sits between the presentation layer and data access, determining what can happen, when, and under what conditions. Strong business logic engineering means translating fuzzy requirements into precise, testable, maintainable rules and flows.

## Core Workflow

### 1. Requirements → Domain Understanding

**Extract the "what" and "why" behind each requirement:**
- Identify actors (who initiates, who approves, who receives)
- Clarify triggers (events, time-based, user-initiated, system-initiated)
- Define success criteria and failure states
- Surface implicit assumptions ("obvious" behaviors are often the source of bugs)

**Ask clarifying questions:**
- What happens if X occurs during Y?
- Who can override this rule?
- What's the rollback/compensation behavior?
- Are there time windows, quotas, or rate limits?

### 2. Identify Edge Cases Early

Edge cases aren't exceptions—they're requirements that weren't explicitly stated. Hunt for them systematically:

- **Boundary conditions**: zero, negative, max values, empty collections
- **Timing issues**: concurrent requests, race conditions, out-of-order events
- **State transitions**: invalid states, missing prerequisites, circular dependencies
- **External dependencies**: service down, timeout, partial failure, stale data
- **Permissions and context**: user lacks permission mid-flow, session expires, data changes between steps

Document edge cases alongside happy paths. Each edge case becomes either a validation rule, an error handler, or a compensation step.

### 3. Define Business Rules Explicitly

Rules are the atoms of business logic. State them as testable propositions:

**Good rule statements:**
- "A transaction above $500 requires manager approval"
- "Refunds are allowed within 30 days if the item is unopened"
- "Users can edit their own posts within 15 minutes of creation"

**Bad rule statements (too vague):**
- "Large transactions need approval" (what's large?)
- "Recent posts are editable" (how recent?)

Each rule should map to a policy, validator, or guard clause in code. Group related rules into policies (e.g., `RefundEligibilityPolicy`, `TransactionApprovalPolicy`).

### 4. Model Flows with Pseudocode

Pseudocode bridges requirements and implementation. Write it at a level where logic is clear but syntax doesn't matter.

**Structure:**
```
BEGIN ProcessRefundRequest
  INPUT refund_request, user
  
  IF NOT user.has_permission("request_refund"):
    RETURN Error("Unauthorized")
  
  SET order = FETCH Order BY refund_request.order_id
  IF order IS NULL:
    RETURN Error("Order not found")
  
  SET eligibility = CALL CheckRefundEligibility(order, refund_request)
  IF NOT eligibility.is_eligible:
    RETURN Error(eligibility.reason)
  
  IF order.amount > 500:
    SET approval = CALL RequestManagerApproval(refund_request)
    IF NOT approval.approved:
      RETURN Error("Approval denied")
  
  SET refund = CALL IssueRefund(order, refund_request.amount)
  CALL NotifyUser(user, refund)
  
  RETURN Success(refund)
END
```

**Key elements:**
- Clear entry/exit points
- Explicit error paths (failures are first-class)
- Named sub-procedures (extract complexity into named operations)
- Guard clauses early (fail fast on invalid state)

Pseudocode should be reviewable by non-engineers. If a product manager can't follow it, simplify.

### 5. Visualize with Sequence Diagrams

Sequence diagrams show **who talks to whom, when, and in what order**. Use them when:
- Multiple services/actors interact
- Timing and order matter
- Async or event-driven flows
- Clarifying API contracts

**Anatomy:**
- **Lifelines**: actors, services, systems (vertical dashed lines)
- **Messages**: synchronous calls (solid arrow), async (open arrow), returns (dashed arrow)
- **Activation boxes**: show when an object is actively processing
- **Alt/Opt/Loop frames**: represent conditionals, optional steps, repetition

**Example (text notation for tools like Mermaid, PlantUML, SequenceDiagram.org):**
```
User -> API: POST /refund
API -> AuthService: validate_token(token)
AuthService --> API: user
API -> OrderService: get_order(order_id)
OrderService --> API: order
API -> RefundPolicy: check_eligibility(order)
RefundPolicy --> API: eligible
alt amount > 500
  API -> ApprovalService: request_approval(refund)
  ApprovalService --> API: approval_result
end
API -> PaymentGateway: issue_refund(amount)
PaymentGateway --> API: refund_confirmation
API -> NotificationService: notify_user(user, refund)
API --> User: 200 OK {refund}
```

Focus on the main flow first, then layer in alternatives and error paths. Don't clutter—create separate diagrams for complex branches.

## Workflow Patterns

Common patterns in business logic:

**Sequential (task chaining):** Step A → Step B → Step C. Output of one is input to the next. Use when order matters and steps are dependent.

**Parallel (fan-out/fan-in):** Execute multiple tasks concurrently, then synchronize. Use for independent operations (e.g., fetch user, fetch order, fetch inventory in parallel).

**Conditional (branching):** Choose path based on data. Use exclusive choice (XOR) when exactly one path is taken, or conditional execution when paths are optional.

**Compensation (rollback):** When a multi-step flow fails partway, undo completed steps. Each step defines a compensating action (e.g., cancel reservation, reverse charge, release lock).

**Event-driven:** Triggered by external events rather than direct calls. Use when loose coupling is needed or when reacting to state changes across services.

**Approval/human-in-the-loop:** Pause workflow, wait for human decision, resume. Model as async: request approval, wait for event, continue or abort.

Recognize these patterns in requirements. They map cleanly to code structures and make flows easier to reason about.

## Separation of Concerns

Business logic should be isolated from infrastructure:

- **Business logic**: pure domain rules, policies, state transitions. No knowledge of HTTP, databases, or frameworks.
- **Application logic**: orchestrates use cases, coordinates business logic with infrastructure (repositories, gateways, event publishers).
- **Infrastructure**: HTTP handlers, ORM, message queues, external APIs.

This separation makes business logic testable without spinning up servers or databases. Write tests against pure functions and policies, not against frameworks.

## Validation Strategy

Validation happens at multiple layers:

1. **Input validation**: shape, type, format (e.g., email format, required fields). Fail fast.
2. **Business rule validation**: domain constraints (e.g., "refund amount can't exceed order total"). Part of business logic.
3. **State validation**: ensure preconditions are met (e.g., "order must be in 'completed' state to refund").

Don't mix these. Input validation is a guard at the boundary. Business rules live in domain policies. State validation is part of state machines or aggregates.

## State Machines for Complex Flows

When an entity has multiple states and specific allowed transitions, model it explicitly:

```
States: Draft, Submitted, Approved, Rejected, Completed
Transitions:
  Draft → Submitted (on submit)
  Submitted → Approved (on manager approval)
  Submitted → Rejected (on manager denial)
  Approved → Completed (on fulfillment)
  Rejected → Draft (on resubmit)
```

State machines prevent invalid transitions (e.g., can't go from Draft to Completed). They make business rules explicit and eliminate a class of bugs.

## Testing Business Logic

Business logic is the highest-value code to test. Focus on:

- **Happy path**: core flow works as expected
- **Edge cases**: boundary values, empty inputs, concurrent access
- **Business rule violations**: each rule should have a test that violates it
- **State transitions**: valid and invalid transitions
- **Compensation**: rollback works when mid-flow failures occur

Use table-driven tests for rules with many cases. Each row is a scenario: inputs, expected outcome, reason.

## Common Pitfalls

**Implicit rules:** Requirements say "process the order" but don't specify what happens if inventory is insufficient. Make implicit rules explicit.

**Leaking infrastructure into domain:** Business logic that knows about HTTP status codes, database transactions, or JSON serialization is harder to test and change.

**Anemic domain models:** All logic in service classes, entities are just data bags. Push behavior into domain objects where it belongs.

**Overly generic logic:** "One module equals one generic logic" sounds good but leads to complex, hard-to-maintain god classes. Balance reusability with clarity.

**Ignoring failure paths:** Pseudocode and diagrams that only show success are incomplete. Model retries, timeouts, rollbacks, and errors explicitly.

## Tools and Notations

- **Pseudocode**: Plain text, structured with BEGIN/END, IF/ELSE, FOR/WHILE, CALL/RETURN
- **Sequence diagrams**: Mermaid, PlantUML, SequenceDiagram.org, Eraser.io, Creately
- **State diagrams**: For state machines and lifecycle modeling
- **Decision tables**: For complex rule matrices (multiple conditions → multiple outcomes)
- **BPMN**: For business process modeling (more formal than pseudocode, less technical than code)

Choose the tool that matches your audience. Engineers prefer pseudocode and sequence diagrams. Business stakeholders prefer BPMN and decision tables.

## Workflow

1. **Gather requirements** → extract actors, triggers, rules, edge cases
2. **Define rules explicitly** → testable, unambiguous statements
3. **Identify edge cases** → boundaries, timing, state, dependencies
4. **Write pseudocode** → logic flow with error handling
5. **Draw sequence diagrams** → multi-actor interactions, timing, async flows
6. **Model state machines** (if needed) → valid states and transitions
7. **Review with stakeholders** → validate understanding before coding
8. **Implement** → translate pseudocode into domain objects, policies, services
9. **Test** → happy path, edge cases, rule violations, state transitions

This workflow catches ambiguity and gaps early, when they're cheap to fix. Code becomes the formalization of already-validated logic, not the first place you discover requirements are incomplete.