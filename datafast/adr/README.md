# Architecture Decision Records (ADRs)

## Overview
This directory contains Architecture Decision Records (ADRs) for the DataFa.st project. ADRs document significant architectural decisions, their context, alternatives considered, and consequences.

## ADR Index

| ADR | Title | Status | Date |
|-----|-------|--------|------|
| [ADR-001](ADR-001-time-series-database.md) | Time-Series Database Selection | Accepted | 2025-11-18 |
| [ADR-002](ADR-002-tracking-script-architecture.md) | Tracking Script Architecture | Accepted | 2025-11-18 |
| [ADR-003](ADR-003-frontend-framework.md) | Frontend Framework Selection | Accepted | 2025-11-18 |
| [ADR-004](ADR-004-revenue-attribution-model.md) | Revenue Attribution Model | Accepted | 2025-11-18 |
| [ADR-005](ADR-005-authentication-strategy.md) | Authentication Strategy | Accepted | 2025-11-18 |
| [ADR-006](ADR-006-api-design.md) | API Design Approach | Accepted | 2025-11-18 |

## ADR Template

When creating a new ADR, use the following template:

```markdown
# ADR-XXX: [Title]

**Date:** YYYY-MM-DD
**Status:** Proposed | Accepted | Deprecated | Superseded
**Deciders:** [List of people involved]
**Technical Story:** [Link to user story or ticket]

## Context and Problem Statement

[Describe the context and problem statement]

## Decision Drivers

- [Driver 1]
- [Driver 2]
- ...

## Considered Options

1. [Option 1]
2. [Option 2]
3. [Option 3]

## Decision Outcome

**Chosen option:** "[Option X]"

**Rationale:** [Why this option was chosen]

### Positive Consequences

- [Positive 1]
- [Positive 2]

### Negative Consequences

- [Negative 1]
- [Negative 2]

## Pros and Cons of the Options

### [Option 1]

**Pros:**
- [Pro 1]
- [Pro 2]

**Cons:**
- [Con 1]
- [Con 2]

### [Option 2]

**Pros:**
- [Pro 1]

**Cons:**
- [Con 1]

## Links

- [Link to related documentation]
- [Link to external resources]
```

## Status Definitions

- **Proposed:** Decision is being considered
- **Accepted:** Decision has been approved and is being implemented
- **Deprecated:** Decision is no longer valid but kept for historical reference
- **Superseded:** Decision has been replaced by a newer ADR (link to new ADR)
