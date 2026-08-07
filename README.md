# OpsPilot: Building an Approval-Gated AI Agent for IT Service Management

## Contents

- [Architecture](#architecture)
- [Structured Incident Analysis](#structured-incident-analysis)
- [ServiceNow Integration](#integrating-with-servicenow)
- [Human Approval](#human-approval-as-a-security-boundary)
- [PostgreSQL Auditability](#adding-postgresql-for-auditability)
- [Designing for AI Failure](#designing-for-ai-failure)
- [Lessons Learned](#lessons-learned)

Modern IT service desks generate enormous amounts of structured operational data, but much of the work surrounding that data is still manual. An analyst opens an incident, interprets the description, investigates the likely cause, determines the appropriate assignment or resolution path, documents the work, and updates the ITSM platform.

I built **OpsPilot** to explore a practical question:

**How much of that workflow can an AI agent assist with without giving the model uncontrolled access to production systems?**

The result is an AI-assisted incident-management system that retrieves ServiceNow incidents, analyzes them, proposes structured actions, validates those actions against live platform data, requires human approval for changes, writes approved updates back to ServiceNow, and records the activity in PostgreSQL.

The goal was not to create an autonomous AI that blindly closes tickets. It was to build a system where AI can accelerate investigation and repetitive work while deterministic controls remain responsible for authorization and execution.

## Architecture

OpsPilot Architecture Diagram.png

OpsPilot separates AI reasoning from authorization and execution. The model proposes actions, deterministic controls validate them, a human approves consequential changes, ServiceNow executes authorized updates, and PostgreSQL records the audit trail.

A major design principle is that **analysis and authorization are separate operations**.

The AI can recommend an action. It cannot decide that it is authorized to perform that action.

That distinction became increasingly important as I tested the system against actual ServiceNow workflows.

At a high level, OpsPilot follows this workflow:

```text
                 ┌─────────────────────┐
                 │     ServiceNow      │
                 │      Incident       │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │ Incident Retrieval  │
                 │     REST API        │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │   AI Analysis &     │
                 │ Recommendation      │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │ Validation / Guards │
                 │                     │
                 │ • State checks      │
                 │ • Group validation  │
                 │ • Required fields   │
                 │ • Write protection  │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │   Human Approval    │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │ ServiceNow Writeback│
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │ PostgreSQL Audit Log│
                 └─────────────────────┘
```

A major design principle is that **analysis and authorization are separate operations**.

The AI can recommend an action. It cannot decide that it is authorized to perform that action.

That distinction became increasingly important as I tested the system against actual ServiceNow workflows.

## Structured Incident Analysis

The first stage of the project focused on converting an incident into a predictable structure that an AI model could reason about.

Instead of treating an incident as an arbitrary block of text, OpsPilot extracts relevant fields and provides the model with a defined representation of the ticket.

The analysis layer can then return structured recommendations such as:

* likely issue category
* probable assignment group
* priority considerations
* troubleshooting recommendations
* proposed work notes
* customer-facing comments
* whether resolution may be appropriate

Keeping the output structured makes the recommendations easier to validate programmatically before anything reaches ServiceNow.

This is important because natural-language output is useful for humans but dangerous as an execution interface.

## Integrating With ServiceNow

The next step was connecting OpsPilot to a ServiceNow developer instance through its REST APIs.

I created a dedicated integration user rather than using my own account. The integration identity was given only the access required for the workflow.

OpsPilot could then retrieve an incident by number, inspect its current state and fields, analyze it, and prepare potential changes.

Initial testing immediately exposed an important difference between generating a recommendation and successfully executing one.

For example, attempting to resolve an incident initially failed because ServiceNow rejected the request:

```text
Data Policy Exception:
The following fields are mandatory: Resolution code
```

That failure was useful.

The AI's recommendation was logically reasonable, but the target system had business rules the model did not inherently know about.

I updated the workflow so that resolution operations accounted for ServiceNow's required fields and valid resolution values before writeback.

After incorporating those constraints, OpsPilot successfully resolved test incidents through the API.

## Never Assume a Display Value Is an API Value

Assignment groups introduced another integration problem.

An analyst might naturally say:

```text
Assign this incident to Hardware.
```

But ServiceNow relationships are represented internally by identifiers rather than simply by their human-readable labels.

For example, the system needs to distinguish between a display value such as:

```text
Hardware
```

and the corresponding ServiceNow record identifier.

Rather than allowing the AI to invent or assume those identifiers, I added validation against the active ServiceNow groups.

The resulting workflow became:

```text
AI recommends assignment group
        ↓
OpsPilot searches active ServiceNow groups
        ↓
Display name is matched to an actual record
        ↓
Record identifier is retrieved
        ↓
Proposed change is displayed for approval
        ↓
Approved identifier is written to ServiceNow
```

This allowed a recommendation such as:

```text
Assignment group
Unassigned → Hardware
```

to be verified against the actual ServiceNow configuration before execution.

That principle applies far beyond ServiceNow:

**LLMs should reason about intent; authoritative systems should provide identifiers and valid state.**

## Human Approval as a Security Boundary

One of the most important design decisions in OpsPilot was requiring approval before writeback.

It would have been easy to build:

```text
Ticket → AI → ServiceNow update
```

But that gives probabilistic output direct control over an enterprise system.

Instead, OpsPilot follows:

```text
Ticket
   ↓
Analysis
   ↓
Recommended changes
   ↓
Validation
   ↓
Human approval
   ↓
Execution
```

An analyst can therefore see exactly what OpsPilot intends to change before the API request is made.

For example:

```text
Apply changes to INC0000020

Assignment group
Unassigned → Hardware

Verified against the active ServiceNow group before write-back.
```

Only after approval does OpsPilot execute the update.

The incident was subsequently moved to the Hardware assignment group and reflected the expected state in ServiceNow.

This keeps the AI in an advisory role until a deterministic approval signal authorizes execution.

## Protecting Closed Incidents

Another edge case appeared when testing against incidents that had already reached a terminal state.

An AI model may still be capable of generating perfectly reasonable recommendations for a closed incident. That does not mean the system should modify it.

I therefore added state-aware write guards.

When OpsPilot encounters an incident that should no longer be changed, it switches to analysis-only behavior:

```text
Read-only incident

This incident is closed.
You can analyze it, but OpsPilot will not apply changes.
```

This distinction is important.

The system doesn't need to prevent the AI from reasoning about the incident. It needs to prevent that reasoning from becoming an unauthorized state change.

## Handling ACL Failures

ServiceNow also provided another useful lesson when an update failed with:

```text
ACL Exception
Update Failed due to security constraints
```

Instead of attempting to bypass the restriction, I treated the platform's authorization model as authoritative.

The integration was reviewed, the permitted operation was determined, and subsequent testing verified that approved changes could succeed under the correct permissions.

This reinforced another design rule:

**An AI agent should operate inside the security model of the system it integrates with, not attempt to work around it.**

Permissions, ACLs, data policies, and workflow rules are controls—not inconveniences to be bypassed.

## Adding PostgreSQL for Auditability

Once OpsPilot could successfully analyze and modify incidents, the next question was:

**How do I prove what the system did?**

For an AI-assisted operational system, logging only the final ServiceNow state isn't sufficient.

I added a PostgreSQL database hosted on Neon to provide an independent audit layer.

The audit model is designed to capture information surrounding each operation, including the incident involved, proposed actions, approval state, execution outcome, and timestamps.

Indexes were added around the access patterns most likely to be used during operational review.

This creates a traceable sequence:

```text
Incident retrieved
        ↓
Analysis generated
        ↓
Action proposed
        ↓
Validation performed
        ↓
Approval received
        ↓
API operation attempted
        ↓
Result recorded
```

That becomes particularly valuable when investigating why an automated or AI-assisted action occurred.

## Why the Database Is Separate From ServiceNow

ServiceNow already contains incident history, so storing another audit trail may appear redundant.

The two systems answer different questions.

ServiceNow tells me:

**What happened to the incident?**

The OpsPilot audit database can tell me:

**Why did the agent attempt that action, what did it propose beforehand, was it approved, and what happened when execution was attempted?**

Separating those concerns also makes it possible to analyze agent behavior without changing or overloading the operational system.

## Failure Is Part of the Architecture

Some of the most useful development work on OpsPilot came from operations that failed.

During testing I encountered:

* missing mandatory resolution fields
* invalid assumptions about assignment values
* ACL restrictions
* closed/read-only incidents
* state-dependent operations
* write operations requiring additional validation

Each failure exposed another assumption that needed to move out of the AI layer and into deterministic application logic.

That changed how I thought about the project.

Initially, the interesting problem appeared to be:

**Can an LLM correctly analyze an IT incident?**

It can.

The harder engineering problem is:

**How do you safely connect probabilistic reasoning to a deterministic enterprise system?**

The answer has been to progressively reduce the authority of the model itself.

The model recommends.

Code validates.

The target system supplies authoritative state.

A human authorizes consequential changes.

The integration executes.

The database records what happened.

## Designing for AI Failure

LLMs are useful precisely because they can reason through ambiguous information that traditional rules struggle with.

That flexibility also means their output should not automatically be trusted as application state.

OpsPilot therefore treats AI output as **untrusted structured input**.

Before a proposed action becomes executable, the surrounding application can verify conditions such as:

```text
Does the incident exist?
Is the incident writable?
Is the proposed transition allowed?
Does the assignment group exist?
Are required fields present?
Has a human approved this exact action?
Does the integration identity have permission?
```

Only after those checks succeed should the application construct the write request.

This pattern lets the AI handle what it is good at—interpretation and reasoning—while conventional software handles what it is good at—validation, authorization, and deterministic execution.

## What I Would Build Next

OpsPilot is still an evolving project.

The next areas I would explore include stronger policy enforcement around allowable state transitions, richer audit queries and dashboards, confidence-based escalation, expanded automated testing, observability around model and API failures, and additional integrations beyond ServiceNow.

I am particularly interested in measuring where AI recommendations disagree with experienced analysts.

Those disagreements are potentially more valuable than simple accuracy metrics because they can reveal missing context, weak prompts, undocumented operational knowledge, or processes that should be formalized.

## Lessons Learned

Building OpsPilot reinforced several principles that I believe apply broadly to enterprise AI systems.

### 1. AI recommendations are not authorization

A model determining that an action makes sense does not mean the system should execute it.

### 2. Validate against authoritative systems

Do not ask an LLM to remember identifiers, permissions, configuration values, or current system state when they can be retrieved directly.

### 3. Treat model output as untrusted input

Structured output helps, but schemas alone do not make an AI response correct.

### 4. Design around failure

API errors, ACL failures, invalid state transitions, and incomplete data should be expected execution paths.

### 5. Preserve human control where consequences matter

Human approval provides a practical boundary between AI assistance and operational authority.

### 6. Audit the decision process, not just the result

For enterprise AI, knowing that a field changed is less useful than knowing what proposed the change, why it was proposed, whether it was approved, and whether execution succeeded.

## Closing Thoughts

OpsPilot started as an experiment in applying AI to a workflow I know well: enterprise IT support.

The AI analysis itself turned out to be only one part of the problem.

The more interesting engineering work emerged at the boundaries between AI reasoning and enterprise systems: authentication, permissions, APIs, validation, state management, approval workflows, database design, error handling, and auditability.

That is also where I believe many useful enterprise AI applications will be built.

The objective shouldn't be to remove humans from every workflow.

It should be to identify where AI can reduce the time humans spend interpreting repetitive information while keeping deterministic controls around actions that affect real systems.

For OpsPilot, that means a simple rule:

**Let the model reason. Let the application verify. Let the human authorize.**

