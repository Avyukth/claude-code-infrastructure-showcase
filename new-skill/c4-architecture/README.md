# C4 System Architecture Skill

## Overview

This skill provides comprehensive guidance for creating, documenting, and maintaining software architecture using the **C4 Model** (Context, Container, Component, Code). It enables systematic architecture documentation through visual diagrams and structured descriptions, supporting both greenfield design and brownfield documentation.

## Verification Status

✅ **Verified against meta-skill conventions**

### Compliance Checklist

- ✅ **YAML Frontmatter**: Complete with name, description, and version
- ✅ **500-Line Rule**: SKILL.md = 370 lines (26% under limit)
- ✅ **Progressive Disclosure**: 3 resource files for detailed content
- ✅ **Rich Description**: All trigger keywords included
- ✅ **Structure**: Purpose, When to Use, Core Guidance, Navigation, Checklist
- ✅ **Auto-Activation**: skill-rules.json with comprehensive triggers
- ✅ **Success Metrics**: Measurable targets defined
- ✅ **Quick Reference**: Checklists for all diagram types

## File Structure

```
c4-architecture/
├── SKILL.md                          # 370 lines - Main skill guidance
├── skill-rules.json                  # Auto-activation configuration
├── README.md                         # This file
└── resources/
    ├── plantuml-templates.md         # 612 lines - PlantUML C4 templates
    ├── mermaid-templates.md          # 565 lines - Mermaid C4 templates
    └── complete-examples.md          # 455 lines - Production examples
```

## What's Included

### Main Skill (SKILL.md)

- **C4 Model Overview**: All 4 levels (Context, Container, Component, Code)
- **Quick Start**: 5-step process for creating diagrams
- **Core Principles**: 5 key principles for effective diagrams
- **Best Practices**: DO/DON'T guidelines
- **Quick Reference Checklists**: For each diagram type
- **Success Metrics**: Measurable quality targets
- **Navigation Guide**: Links to resource files

### Resource Files

1. **plantuml-templates.md** - Complete PlantUML templates
   - Templates for all 4 C4 levels
   - Advanced styling and layout control
   - Rendering and CI/CD integration
   - Complete working examples

2. **mermaid-templates.md** - Mermaid diagram templates
   - Native GitHub/GitLab rendering
   - Simpler syntax for documentation-first approach
   - Container and Context diagram templates
   - Styling and customization options

3. **complete-examples.md** - Production-ready examples
   - E-Commerce Platform (all levels)
   - Healthcare Management System
   - Real-Time Analytics Platform
   - Demonstrates best practices and patterns

### Auto-Activation (skill-rules.json)

**Trigger Keywords**:
- c4, c4 model, architecture diagram, system architecture
- plantuml, mermaid, structurizr
- context diagram, container diagram, component diagram
- microservices architecture, system design

**Intent Patterns**:
- `(create|generate|build).*?(architecture|diagram)`
- `(document|explain).*?(system|architecture)`
- `C4.*(context|container|component|code)`

**File Triggers**:
- `**/*.puml`, `**/*.mmd` - Diagram files
- `**/architecture/**/*` - Architecture directories
- `**/ADR-*.md` - Architecture Decision Records
- Content patterns: `@startuml`, `C4Context`, etc.

## Installation

### Option 1: Move to .claude/skills/ (Recommended)

```bash
# From the new-skill directory
mv c4-architecture /path/to/your/project/.claude/skills/
```

### Option 2: Add to Existing skill-rules.json

If you already have a `.claude/skills/skill-rules.json`, merge the contents:

```bash
# View the skill-rules.json entry
cat c4-architecture/skill-rules.json

# Manually merge into your existing .claude/skills/skill-rules.json
```

## Usage

Once installed, the skill will automatically activate when:

1. **You mention C4 or architecture keywords** in prompts
2. **You edit diagram files** (.puml, .mmd)
3. **You work in architecture directories**
4. **You create/edit ADRs** (Architecture Decision Records)

### Manual Activation

You can also manually reference the skill:

```
"Following the C4 architecture skill, create a system context diagram for our e-commerce platform"
```

## Examples

### Create a Context Diagram

```
"Create a C4 context diagram for an e-commerce platform with customers, sellers,
and integrations with Stripe, SendGrid, and FedEx"
```

### Generate PlantUML

```
"Generate PlantUML code for a container diagram showing a microservices architecture
with API gateway, user service, order service, and their databases"
```

### Document Existing System

```
"Help me document our existing healthcare system using C4 diagrams. We have a web app,
mobile app, API gateway, and several backend services"
```

## Success Metrics

According to the skill, well-created C4 diagrams should achieve:

| Metric | Target |
|--------|--------|
| Comprehension | Stakeholders understand in <10 min |
| Accuracy | 100% match to actual/planned system |
| Currency | Updated within 1 sprint of changes |
| Usefulness | Referenced in PRs and decisions |

## Meta-Skill Compliance

### Follows Meta-Skill Best Practices

✅ **500-Line Rule**: Main file under 500 lines
✅ **Progressive Disclosure**: Detailed content in resource files
✅ **Rich Description**: Comprehensive trigger keywords
✅ **Imperative Language**: Action-oriented instructions
✅ **DO/DON'T Examples**: Clear patterns and anti-patterns
✅ **Checklists**: Step-by-step actionable items
✅ **Navigation Table**: Clear resource file organization
✅ **Success Metrics**: Measurable quality targets

### Skill Type

- **Type**: Domain skill (advisory)
- **Enforcement**: Suggest (not blocking)
- **Priority**: High
- **Use Case**: Creating and documenting software architecture

## Integration with Other Skills

Works well with:

- **backend-dev-guidelines** - Implementing the architecture
- **frontend-dev-guidelines** - Building frontend containers
- **rust-skills** - Implementing services in Rust
- **meta-skill** - General skill development patterns

## Maintenance

### Future Enhancements

Potential additions (as separate resource files):

1. **best-practices.md** - Extended best practices
2. **microservices-patterns.md** - Microservices-specific guidance
3. **integration-patterns.md** - Integration and data flow patterns
4. **deployment-views.md** - Infrastructure and deployment diagrams
5. **troubleshooting.md** - Common issues and solutions

### Updating

When updating the skill:

1. Keep SKILL.md under 500 lines
2. Move detailed content to resource files
3. Update skill-rules.json if adding new keywords
4. Test triggers with real prompts
5. Verify line counts remain compliant

## Credits

Created following the meta-skill conventions from the Claude Code Infrastructure Showcase.

**Version**: 1.0
**Status**: Production-ready
**Compliance**: Meta-skill verified ✅

---

## Quick Links

- [Main Skill](SKILL.md) - Start here
- [PlantUML Templates](resources/plantuml-templates.md) - PlantUML diagrams
- [Mermaid Templates](resources/mermaid-templates.md) - Mermaid diagrams
- [Complete Examples](resources/complete-examples.md) - Real-world examples
- [Auto-Activation Config](skill-rules.json) - Trigger configuration
