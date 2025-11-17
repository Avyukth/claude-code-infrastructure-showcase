# Product Requirements Document (PRD) Creation Skill

## Overview

This skill provides a comprehensive, evidence-based framework for creating professional Product Requirements Documents (PRDs) that align cross-functional teams, guide development, and ensure products address user needs and business goals. It integrates **meta-skills principles** (adaptability, clear communication, collaboration, self-management) with industry best practices from leading product management organizations.

## Verification Status

✅ **Verified against meta-skill conventions**
✅ **Research-based** - Incorporates findings from 10+ authoritative sources (2024)
✅ **Dual-approach** - Supports both agile/MVP and enterprise/granular methodologies

### Compliance Checklist

- ✅ **YAML Frontmatter**: Complete with comprehensive description and trigger keywords
- ✅ **500-Line Rule**: SKILL.md = 422 lines (16% under limit)
- ✅ **Progressive Disclosure**: 1 comprehensive template resource file
- ✅ **Evidence-Based**: Grounded in research from Atlassian, ProductPlan, Aha.io, Figma, Type.ai
- ✅ **Structure**: Purpose, When to Use, Core Principles, Quick Start, Process, Navigation, Checklists
- ✅ **Auto-Activation**: skill-rules.json with 24 keywords and 7 intent patterns
- ✅ **Success Metrics**: Measurable targets for PRD effectiveness
- ✅ **Meta-Skills Integration**: Adaptability, iteration, collaboration, evidence-based validation

## File Structure

```
prd/
├── SKILL.md                          # 422 lines - Main PRD creation guidance
├── skill-rules.json                  # Auto-activation configuration
├── README.md                         # This file
└── resources/
    └── prd-template.md               # Comprehensive PRD template
```

## What's Included

### Main Skill (SKILL.md)

**Core Content**:
- **5 Core Principles**: Problem-first thinking, incremental value delivery, evidence-based decisions, collaboration, clarity over completeness
- **4-Phase Creation Process**: Problem Alignment → Requirements Development → Solution Design → Planning & Validation
- **Quick Start Guide**: 5 steps from idea to validated PRD
- **Dual Approach**: Incremental (agile/MVP) vs. Granular (detailed/waterfall) methodologies
- **Best Practices**: 10 DOs and 10 DON'Ts based on research
- **Quality Checklist**: Comprehensive validation criteria
- **Success Metrics**: 6 measurable targets for PRD effectiveness

**Research-Based**:
Incorporates findings from:
1. Atlassian - Agile product management and requirements best practices
2. ProductPlan - PRD templates and glossary
3. Aha.io - Requirements management frameworks
4. Type.ai - Modern PRD writing with AI integration (2024)
5. Shortcut.com - Component analysis and benefits
6. HelpLook - SaaS PRD guide (2024)
7. Zeda.io - Product requirement fundamentals
8. Jama Software - Effective PRD writing
9. LaunchNotes - Agile requirement documents
10. Planio - Lean PRD approaches

### Resource File (prd-template.md)

**Comprehensive Template** with 8 core sections:
1. **Executive Summary** - Problem, solution, impact, timeline
2. **User Personas & Stories** - Detailed personas with user stories in standard format
3. **Functional Requirements** - Prioritized features with acceptance criteria
4. **Non-Functional Requirements** - Performance, security, scalability, accessibility
5. **Design & UX** - User flows, wireframes, interaction specifications
6. **Technical Specifications** - Architecture, APIs, data models, integrations
7. **Success Metrics & KPIs** - Measurement plan with instrumentation details
8. **Release Plan** - Phased rollout, timeline, risks, dependencies

**Features**:
- Copy-paste ready sections with placeholders
- Example content and formatting guidance
- Tables for structured data (requirements, risks, stakeholders)
- Code snippets for API specs and data models
- Checklists for release criteria and validation

### Auto-Activation (skill-rules.json)

**Trigger Keywords** (24 total):
- Primary: prd, product requirements document, product specification
- Functional: user stories, acceptance criteria, functional requirements
- Planning: mvp definition, product roadmap, release planning
- Metrics: success metrics, kpis, product goals

**Intent Patterns** (7 regex patterns):
- `(create|write|draft).*?(prd|product requirements)`
- `(document|specify|define).*?(product|feature)`
- `write.*?(user stories|acceptance criteria)`
- `(plan|planning).*?(product|mvp|release)`
- `(define|establish).*?(success metrics|kpis)`

**File Triggers**:
- Path patterns: `**/PRD*.md`, `**/requirements/**/*.md`, `**/product-specs/**/*.md`
- Content patterns: "Product Requirements Document", "## Executive Summary", "Priority: P0"

## Installation

### Option 1: Move to .claude/skills/ (Recommended)

```bash
# From the new-skill directory
mv prd /path/to/your/project/.claude/skills/
```

### Option 2: Merge skill-rules.json

If you have existing `.claude/skills/skill-rules.json`:

```bash
# View the PRD skill entry
cat prd/skill-rules.json

# Manually merge into your existing .claude/skills/skill-rules.json
```

## Usage

Once installed, the skill automatically activates when:

1. **You mention PRD keywords**: "create a prd", "product requirements", "user stories"
2. **You express PRD intent**: "I need to document this feature", "let's plan the MVP"
3. **You edit PRD files**: Working on files matching `**/PRD*.md` or in `requirements/` folders
4. **Content contains PRD elements**: Files with "Executive Summary", "Acceptance Criteria", etc.

### Manual Activation

Reference the skill explicitly:

```
"Following the prd-creation skill, help me create a comprehensive PRD for a mobile recipe
sharing platform with AI-powered ingredient substitution suggestions"
```

## Example Usage

### Example 1: Create MVP PRD

**Input**:
```
Create a PRD for a mobile water intake tracking app targeting health-conscious adults
aged 25-45. Focus on MVP with simple logging, reminders, and progress tracking.

Constraints: 8-week timeline, React Native, team of 3
Business goal: 100K users in 6 months with 40% 30-day retention
```

**Output**: Complete incremental PRD with:
- Lean executive summary
- 2-3 user personas
- MVP features prioritized (P0/P1/P2)
- User stories with acceptance criteria
- 3-phase rollout plan (MVP → Enhancement → Monetization)
- Success metrics with targets

### Example 2: Enterprise Feature Specification

**Input**:
```
Create a detailed PRD for adding Single Sign-On (SSO) to our enterprise SaaS platform.
Must support SAML 2.0 and OIDC, integrate with existing user management, and be SOC 2 compliant.

Constraints: 16-week timeline, must not disrupt existing users, comprehensive testing required
Business goal: Enable enterprise sales for 50+ user organizations
```

**Output**: Granular PRD with:
- Comprehensive executive summary with compliance context
- Detailed technical specifications (SAML flows, data models)
- Non-functional requirements (security, performance benchmarks)
- Complete user flows for all SSO scenarios (happy path, errors, edge cases)
- Release criteria with SOC 2 compliance checklist

## Success Metrics

According to the skill, well-created PRDs should achieve:

| Metric | Target |
|--------|--------|
| Stakeholder Alignment | 100% approval before development |
| Development Clarity | <5% engineering questions during build |
| Scope Stability | <10% requirement changes mid-development |
| On-Time Delivery | 90%+ milestones within 1 week of target |
| Post-Launch Success | 80%+ of KPIs meet success thresholds |
| Team Satisfaction | 4+ out of 5 PRD clarity rating |

## Meta-Skills Integration

This skill embodies meta-skills principles:

### Adaptability
- **Dual methodologies**: Supports both agile (lean) and waterfall (detailed) approaches
- **Iterative refinement**: Encourages updating PRDs based on learnings, not treating as "done"
- **Flexible templates**: Adapt sections based on product type and team needs

### Clear Communication
- **Structured format**: 8 standard sections for consistency
- **Audience-appropriate**: Different detail levels for executives vs. engineers
- **Visual aids**: Emphasis on mockups, flows, diagrams for clarity

### Collaboration
- **Stakeholder involvement**: Explicit checkpoints for eng, design, legal input
- **TBD placeholders**: Gather input rather than spec in isolation
- **Living documents**: Continuous updates as team discusses and learns

### Self-Management
- **Quality checklists**: Built-in validation for completeness
- **Success metrics**: Define measurement from day one
- **Risk assessment**: Proactive identification and mitigation planning

### Evidence-Based Validation
- **Problem validation**: Require evidence (research, data, quotes)
- **Measurable targets**: Every goal needs a metric
- **Post-launch tracking**: Instrumentation plan for validation

## Common Patterns

### Pattern 1: Startup MVP PRD
**Structure**: 3-5 page lean PRD
**Focus**: Core value proposition, MVP features, rapid iteration
**Timeline**: 4-8 weeks to launch
**Example**: Mobile app, SaaS tool, consumer product

### Pattern 2: Enterprise Feature Addition
**Structure**: 10-15 page detailed specification
**Focus**: Integration points, security, compliance, comprehensive testing
**Timeline**: 12-20 weeks
**Example**: SSO, advanced reporting, API integrations

### Pattern 3: Regulated Product
**Structure**: 15-20 page comprehensive with compliance section
**Focus**: Audit trails, security architecture, regulatory requirements
**Timeline**: 16-24 weeks with extensive validation
**Example**: Healthcare (HIPAA), finance (SOC 2), government

### Pattern 4: Quick Win Feature
**Structure**: 1-2 page one-pager
**Focus**: Single improvement, fast ship
**Timeline**: 1-2 weeks
**Example**: UI refinement, small optimization, config change

## Research Sources

This skill is grounded in evidence from leading product management resources (2024):

1. **Atlassian** - Agile product management and requirements
   - Focus: Agile methodologies, collaboration, iteration

2. **ProductPlan** - PRD templates and frameworks
   - Focus: Structure, best practices, stakeholder alignment

3. **Aha.io** - Requirements management
   - Focus: Roadmapping, prioritization, release planning

4. **Type.ai** - Modern PRD writing with AI (2024)
   - Focus: Efficiency, AI integration, contemporary practices

5. **Figma** - Product requirements for design-led teams
   - Focus: UX specifications, design collaboration

6. **Shortcut.com** - Components and benefits analysis
   - Focus: Developer-friendly requirements, technical clarity

7. **HelpLook** - SaaS PRD guide (2024)
   - Focus: SaaS-specific considerations, modern practices

8. **LaunchNotes** - Agile requirement documents
   - Focus: User stories, sprint planning, agile workflows

9. **Planio** - Lean PRD approaches
   - Focus: Minimal viable documentation, efficiency

10. **Jama Software** - Effective PRD writing
    - Focus: Traceability, requirements quality

## Future Enhancements

Potential additions (as separate resource files):

1. **prd-creation-workflow.md** - Detailed step-by-step process
2. **quality-checklist.md** - Comprehensive validation checklist
3. **agile-vs-granular.md** - Deep dive on methodology selection
4. **example-recipe-prd.md** - Complete worked example
5. **industry-specific-prds.md** - Healthcare, finance, e-commerce variations

## Maintenance

When updating the skill:

1. Keep SKILL.md under 500 lines
2. Move detailed examples to resource files
3. Update skill-rules.json if adding new keywords
4. Test triggers with real prompts
5. Incorporate new industry best practices (annual review)
6. Add case studies and examples as PRDs are created

## Credits

**Version**: 1.0
**Status**: Production-ready
**Compliance**: Meta-skill verified ✅
**Research Date**: November 2024
**Sources**: 10+ authoritative product management resources

---

## Quick Links

- [Main Skill](SKILL.md) - Start here for PRD creation guidance
- [PRD Template](resources/prd-template.md) - Copy-paste ready comprehensive template
- [Auto-Activation Config](skill-rules.json) - Trigger configuration

**Created following meta-skill conventions from the Claude Code Infrastructure Showcase**
