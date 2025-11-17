---
name: prd-creation
description: Comprehensive Product Requirements Document (PRD) creation skill for software products and features. Use when creating PRDs, product specifications, feature documentation, requirements analysis, product planning, stakeholder alignment, MVP definition, agile user stories, technical specifications, success metrics, release planning, or product roadmaps. Supports incremental/agile (lean MVP-focused) and granular/waterfall (detailed specification) approaches with iterative refinement, phased rollouts, user personas, acceptance criteria, and evidence-based validation. Integrates Atlassian, ProductPlan, Aha.io, Figma, and industry best practices for 2024.
version: 1.0
---

# Product Requirements Document (PRD) Creation

## Purpose

This skill provides a systematic framework for creating professional Product Requirements Documents that align cross-functional teams, guide development, and ensure products address user needs and business goals. It integrates meta-skills principles—**adaptability** through iterative refinement, **clear communication** for stakeholder alignment, **collaboration** via structured feedback loops, and **self-management** for handling evolving requirements.

The skill transforms product ideas into comprehensive, actionable PRDs optimized for both agile startups (lean, MVP-focused) and enterprise teams (detailed, compliance-oriented).

## When to Use This Skill

Use this skill when you need to:
- **Create new PRDs** from product ideas or feature requests
- **Document product specifications** for development teams
- **Define MVPs** with phased rollout strategies
- **Align stakeholders** on product vision and scope
- **Plan releases** with clear success criteria
- **Write user stories** with acceptance criteria
- **Specify technical requirements** for engineering
- **Establish success metrics** and measurement strategies
- **Conduct requirements analysis** for complex features
- **Update existing PRDs** based on new learnings or feedback

## Core Principles

### 1. Problem-First Thinking

**Principle**: Start with the user problem, not the solution.

- Articulate the problem in 1-2 clear sentences
- Validate the problem is worth solving (user impact + business value)
- Ask "why" to ensure addressing root cause, not symptoms
- Document evidence: user research, data, customer quotes

**Example**:
```
❌ BAD: "We need to build a mobile app"
✅ GOOD: "75% of our users access the platform on mobile, but our web app
         isn't responsive, causing 40% cart abandonment on phones"
```

### 2. Incremental Value Delivery

**Principle**: Build and ship in phases, learning between each release.

- Define MVP as "minimum that delivers measurable value"
- Plan 2-3 phases ahead (requirements will evolve)
- Build-Measure-Learn feedback loops at every stage
- Version control with clear changelog

### 3. Evidence-Based Decision Making

**Principle**: Ground all decisions in data, research, or user feedback.

- Support requirements with customer evidence
- Set measurable targets for every goal
- Define how success will be validated post-launch
- Update based on learnings, not opinions

### 4. Collaborative Documentation

**Principle**: PRDs are living documents created with, not for, stakeholders.

- Involve engineering, design, marketing, legal early
- Use drafts and placeholders (TBD) to gather input
- Regular reviews and updates based on feedback
- Clear ownership but shared responsibility

### 5. Clarity Over Completeness

**Principle**: Be specific where it matters; link details elsewhere.

- "p95 load time < 2s" not "fast performance"
- Keep main PRD to 5-10 pages, link supporting docs
- Use visuals (mockups, flows, diagrams) liberally
- Write from user perspective, not engineering's

## Quick Start: Create Your First PRD

### Step 1: Gather Initial Context

**Required Inputs**:
- Product idea or feature request (1-2 sentences)
- Target user segments
- Business objectives
- Known constraints (timeline, budget, technical limits)

**Optional Inputs**:
- Existing user research
- Competitive analysis
- Success metrics requirements

### Step 2: Choose Your Approach

**Incremental (Agile/MVP)**:
- Fast-moving teams, high uncertainty
- 1-5 page PRDs, living documents
- Focus: Core value, fast learning
- **Use when**: Startups, new products, 2-4 week sprints

**Granular (Detailed)**:
- Complex systems, regulated industries
- 10-20 page comprehensive specs
- Focus: Complete coverage, compliance
- **Use when**: Enterprise, security requirements, external clients

### Step 3: Complete the PRD Sections

Follow the [PRD Template](resources/prd-template.md) with 8 core sections:

1. **Executive Summary** - Problem, solution, impact (1 page)
2. **User Personas & Stories** - Who + What + Why (1-2 pages)
3. **Functional Requirements** - Features with priorities (2-4 pages)
4. **Non-Functional Requirements** - Performance, security, scale (1-2 pages)
5. **Design & UX** - Flows, mockups, interactions (2-3 pages)
6. **Technical Specifications** - Architecture, APIs, data (2-3 pages)
7. **Success Metrics** - KPIs with targets (1 page)
8. **Release Plan** - Phases, timeline, risks (1-2 pages)

### Step 4: Validate with Checklist

Use the [Quality Checklist](resources/quality-checklist.md) to verify:
- [ ] Problem statement clear and evidence-based
- [ ] All requirements have priorities (P0/P1/P2)
- [ ] User stories include acceptance criteria
- [ ] 3-5 measurable KPIs defined
- [ ] Technical feasibility validated with engineering
- [ ] Edge cases and error handling documented
- [ ] Release criteria and success thresholds clear

### Step 5: Iterate Based on Feedback

- Share draft with stakeholders (use TBD placeholders)
- Incorporate feedback from eng, design, legal, support
- Update as development uncovers new insights
- Version control changes with changelog

## The PRD Creation Process

### Phase 1: Problem Alignment

**1.1 Define the Problem Statement**
- What user problem are we solving? (1-2 sentences)
- Why is this problem worth solving?
- What evidence do we have? (research data, customer quotes)
- What happens if we don't solve it?

**1.2 Identify Target Users**
- Create 2-4 user personas with:
  - Demographics and context
  - Goals and motivations
  - Pain points and current workarounds
  - Success criteria from their perspective

**1.3 Establish Success Criteria**
- Define 3-5 quantifiable KPIs
- Set target values with measurement methodology
- Include user sentiment and qualitative goals
- Specify post-launch validation process

### Phase 2: Requirements Development

**2.1 Gather Inputs**
- User research (interviews, surveys, usage data)
- Market analysis (competitors, trends, demand)
- Stakeholder feedback (eng, design, legal, support)
- Technical assessment (feasibility, dependencies)

**2.2 Define Functional Requirements**

Use user story format:
```markdown
As a [user type], I want to [action], so that I can [goal]

Priority: P0 (Must-have) | P1 (Should-have) | P2 (Nice-to-have)

Acceptance Criteria:
- [ ] Specific, testable condition 1
- [ ] Specific, testable condition 2
- [ ] Specific, testable condition 3

Dependencies: [Prerequisites or related features]
```

**2.3 Define Non-Functional Requirements**
- Performance (load times, response times, uptime targets)
- Security (auth, encryption, compliance: GDPR, WCAG 2.1 AA)
- Scalability (concurrent users, data volume, growth projections)
- Reliability (error handling, failover, disaster recovery)
- Usability (accessibility, localization, browser support)

### Phase 3: Solution Design

**3.1 Create User Flows**

Document for each major feature:
- **Happy Path**: Standard user journey
- **Alternative Paths**: Secondary ways to accomplish tasks
- **Error States**: What happens when operations fail
- **Empty States**: First-time user experience
- **Edge Cases**: Boundary conditions, unusual inputs

**3.2 Technical Specifications**
- System requirements (OS, browsers, hardware)
- Architecture (high-level design, components)
- APIs (endpoints, formats, rate limits)
- Data models (schema, relationships, retention)
- Integrations (third-party services, webhooks)

**3.3 Design Guidelines**
- Link to Figma/design files
- UI/UX principles and patterns
- Visual specifications
- Interaction patterns (hover, animations, gestures)
- Accessibility considerations

### Phase 4: Planning and Validation

**4.1 Phased Rollout Strategy**

**Incremental Approach**:
```
Phase 1 (MVP): Core features solving primary problem
- Feature A, Feature B
- Target: Week 4
- Success: [Specific metrics]

Phase 2: Important value additions
- Feature C, Feature D
- Target: Week 8
- Success: [Specific metrics]

Phase 3: Optimization and nice-to-haves
- Feature E, Feature F
- Target: Week 12
- Success: [Specific metrics]
```

**4.2 Risk Assessment**
- Identify potential risks (technical, market, timeline)
- Assess likelihood and impact (High/Medium/Low)
- Define mitigation strategies
- Document assumptions and dependencies

**4.3 Release Criteria**
- Functionality thresholds (which features complete)
- Usability benchmarks (testing results, scores)
- Reliability standards (uptime, performance)
- Compliance requirements (legal, security sign-offs)
- Support readiness (docs, training, escalation)

## Navigation Guide

| Need to...                                  | Read this resource                                            |
|---------------------------------------------|---------------------------------------------------------------|
| Use the complete PRD template               | [PRD Template](resources/prd-template.md)                     |
| See step-by-step creation guide             | [PRD Creation Workflow](resources/prd-creation-workflow.md)   |
| Validate PRD quality                        | [Quality Checklist](resources/quality-checklist.md)           |
| Learn agile vs detailed approaches          | [Agile vs Granular PRDs](resources/agile-vs-granular.md)      |
| View complete example PRD                   | [Example: Recipe Platform](resources/example-recipe-prd.md)   |

## Incremental vs. Granular Approaches

### Incremental (Agile/MVP) PRD

**When to Use**: Startups, new products, high uncertainty, 2-4 week sprints

**Characteristics**:
- Length: 1-5 pages (lean documentation)
- Focus: "Just enough" context for alignment
- Prioritization: Ruthless MVP focus (MoSCoW method)
- Flexibility: Living document updated each sprint
- Timeline: Phased releases, 4-8 week cycles

**Best Practices**:
- Keep to essentials, link details progressively
- Define MVP as "minimum that delivers value"
- Build-Measure-Learn feedback loops
- Validate assumptions early with prototypes
- Plan only 2-3 phases ahead

### Granular (Detailed Specification) PRD

**When to Use**: Complex systems, regulated industries, enterprise, external clients

**Characteristics**:
- Length: 10-20 pages (comprehensive docs)
- Focus: Detailed specs reducing ambiguity
- Edge Cases: Extensive error handling coverage
- Technical Depth: Complete architecture and data models
- Compliance: Audit trails and regulatory docs

**Best Practices**:
- Document all user flows (happy, alternative, error, edge)
- Include if-then logic for conditional behaviors
- Create decision trees for complex workflows
- Collaborate with QA early for test scenarios
- Specify performance with measurable targets
- Use visual aids (flowcharts, diagrams, state machines)

## Best Practices Summary

### DO:
✅ Start with user problem, not solution
✅ Support requirements with evidence (data, research)
✅ Use specific metrics ("p95 < 2s" not "fast")
✅ Write from user perspective
✅ Include visual aids (mockups, flows, diagrams)
✅ Involve stakeholders early and often
✅ Keep main PRD to 5-10 pages, link details
✅ Update based on learnings (living document)
✅ Define every goal with a measurable metric
✅ Version control with clear changelog

### DON'T:
❌ Write 20+ page PRDs no one reads
❌ Spec every detail upfront (iterate instead)
❌ Ignore user research (ground in evidence)
❌ Work in isolation (collaborate cross-functionally)
❌ Favor solution before exploring alternatives
❌ Feature overload (ruthless prioritization)
❌ Missing edge cases (work with QA early)
❌ Treat PRD as "done" (update with learnings)
❌ Confuse purpose with solution (what/why not how)
❌ Skip success metrics (define from day one)

## Quick Reference Checklist

### Problem & Strategy
- [ ] Problem statement clear (1-2 sentences)
- [ ] Evidence supports problem (research, data, quotes)
- [ ] Root cause identified (not just symptoms)
- [ ] Strategic alignment documented
- [ ] Target users defined with personas

### Requirements
- [ ] All requirements have priorities (P0/P1/P2)
- [ ] User stories in standard format
- [ ] Acceptance criteria specific and testable
- [ ] Edge cases and error handling documented
- [ ] Out-of-scope items explicitly listed
- [ ] Dependencies mapped and owned

### Technical
- [ ] Technical constraints identified
- [ ] Architecture validated with engineering
- [ ] Performance targets realistic and measurable
- [ ] Security and compliance needs specified
- [ ] Integration points documented

### Success & Planning
- [ ] 3-5 quantifiable KPIs with targets
- [ ] Measurement methodology specified
- [ ] Timeline realistic with milestones
- [ ] Risks assessed with mitigations
- [ ] Release criteria clear

### Document Quality
- [ ] All 8 core sections present
- [ ] Visual aids included (mockups, flows)
- [ ] Technical jargon explained/avoided
- [ ] Easy to navigate with clear structure
- [ ] Links to supporting materials

## Success Metrics

A well-created PRD should achieve:

| Metric                        | Target                                                    |
|-------------------------------|-----------------------------------------------------------|
| **Stakeholder Alignment**     | 100% of key stakeholders approve before development       |
| **Development Clarity**       | <5% of engineering questions during build                 |
| **Scope Stability**           | <10% requirement changes mid-development                  |
| **On-Time Delivery**          | 90%+ of milestones hit within 1 week of target            |
| **Post-Launch Success**       | 80%+ of defined KPIs meet success thresholds              |
| **Team Satisfaction**         | 4+ out of 5 rating from eng/design/PM on PRD clarity      |

## Common Patterns

### Pattern 1: MVP + Phases
**Use For**: New products, agile teams, high uncertainty
**Structure**: Lean 1-5 page PRD with MVP focus, 2-3 planned phases
**Example**: Mobile app with core features → social → monetization

### Pattern 2: Enterprise Feature Addition
**Use For**: Adding to existing complex systems
**Structure**: Detailed 10-15 page PRD with integration specs
**Example**: Adding SSO to enterprise SaaS platform

### Pattern 3: Regulated Product
**Use For**: Healthcare, finance, government products
**Structure**: Comprehensive 15-20 page PRD with compliance section
**Example**: HIPAA-compliant patient portal

### Pattern 4: Quick Win Feature
**Use For**: Small improvements, optimizations
**Structure**: One-pager combining all sections
**Example**: Adding dark mode toggle to existing app

## Related Skills

- **backend-dev-guidelines** - Implementing the technical specifications
- **frontend-dev-guidelines** - Building the UI/UX requirements
- **c4-architecture** - Creating system architecture diagrams
- **meta-skill** - Understanding skill creation and iteration principles

---

**Skill Status**: Production-ready comprehensive PRD creation framework
**Line Count**: <500 lines following meta-skill guidelines
**Coverage**: Complete PRD lifecycle from idea to validated document
**Approaches**: Both incremental (agile/MVP) and granular (detailed) supported

**Next Steps**:
- For templates: [PRD Template](resources/prd-template.md)
- For step-by-step guide: [PRD Creation Workflow](resources/prd-creation-workflow.md)
- For examples: [Example: Recipe Platform](resources/example-recipe-prd.md)
