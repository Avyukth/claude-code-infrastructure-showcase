---
name: c4-architecture
description: Comprehensive C4 Model architecture documentation and diagram generation for software systems. Use when creating system architecture diagrams, documenting software structure, designing microservices, explaining system context, containers, components, or code organization. Covers C4 diagrams (Context, Container, Component, Code), PlantUML and Mermaid generation, architecture documentation, system design patterns, microservices architecture, API boundaries, data flow modeling, deployment views, and integration patterns. Supports both greenfield design and brownfield documentation.
version: 1.0
---

# C4 System Architecture

## Purpose

This skill provides comprehensive guidance for creating, documenting, and maintaining software architecture using the C4 Model (Context, Container, Component, Code). It enables systematic architecture documentation through visual diagrams and structured descriptions, supporting both new system design and documentation of existing systems.

The C4 Model provides a hierarchical approach to architecture diagrams, making complex systems understandable to different audiences—from executives to developers.

## When to Use This Skill

Use this skill when you need to:
- **Create architecture diagrams** for new or existing systems
- **Document system structure** with C4 Context, Container, Component, or Code diagrams
- **Design microservices architecture** and define service boundaries
- **Explain system architecture** to stakeholders at different technical levels
- **Plan system migrations** or refactoring with before/after architecture
- **Generate PlantUML or Mermaid diagrams** from architecture descriptions
- **Establish API boundaries** and integration points between systems
- **Model data flow** through distributed systems
- **Create deployment views** showing infrastructure and runtime organization
- **Onboard new team members** with visual system overviews

## The C4 Model: Four Levels of Abstraction

The C4 Model uses four hierarchical levels, each serving different audiences and purposes:

### Level 1: System Context Diagram

**Purpose**: Show how your system fits into the world
**Audience**: Everyone (technical and non-technical)
**Scope**: System and its external dependencies

**Shows**:
- Your system as a single box
- External systems and actors that interact with it
- Relationships and key interactions

**Example Use Cases**:
- Executive presentations
- Stakeholder alignment
- System boundary definition
- External dependency mapping

### Level 2: Container Diagram

**Purpose**: Show the high-level technology choices
**Audience**: Technical stakeholders, architects, developers
**Scope**: Applications and data stores within your system

**Shows**:
- Web applications, mobile apps, desktop applications
- Databases, file systems, message queues
- Technology stack for each container
- Inter-container communication

**Example Use Cases**:
- Technical architecture reviews
- Technology stack documentation
- Deployment planning
- Team responsibility mapping

### Level 3: Component Diagram

**Purpose**: Show internal structure of a container
**Audience**: Developers, architects
**Scope**: Components within a single container

**Shows**:
- Key components/modules
- Their responsibilities
- Dependencies between components
- Technology/framework details

**Example Use Cases**:
- Code organization planning
- Detailed design documentation
- Refactoring planning
- Developer onboarding

### Level 4: Code Diagram

**Purpose**: Show implementation details
**Audience**: Developers
**Scope**: Class/interface level details

**Shows**:
- Classes, interfaces, database tables
- Methods and properties
- Implementation relationships

**Example Use Cases**:
- Implementation guidance
- Code review context
- Detailed technical documentation
- IDE-generated diagrams often sufficient

**Note**: Level 4 is often optional—tools like IDEs can auto-generate these diagrams from code.

## Quick Start: Create a C4 Diagram in 5 Steps

### Step 1: Choose Your Level

Ask yourself:
- **Who is the audience?** → Determines abstraction level
- **What are we documenting?** → Determines diagram type
- **How detailed do we need to be?** → Determines which levels to create

**Decision Guide**:
```
Executive/stakeholder presentation → Level 1 (Context)
Technical overview/deployment planning → Level 2 (Container)
Detailed design/code organization → Level 3 (Component)
Implementation specifics → Level 4 (Code) - often skip
```

### Step 2: Identify Elements

**For Context Diagram**:
- [ ] Your system (one box)
- [ ] External users/actors (people icons)
- [ ] External systems (boxes)
- [ ] Relationships (lines with descriptions)

**For Container Diagram**:
- [ ] Web applications
- [ ] Mobile applications
- [ ] Backend services/APIs
- [ ] Databases
- [ ] Message queues, caches, etc.
- [ ] Technology for each container

**For Component Diagram**:
- [ ] Major components/modules within a container
- [ ] Component responsibilities
- [ ] Dependencies between components
- [ ] Key interfaces

### Step 3: Define Relationships

For each connection between elements:
- [ ] What is the nature of the relationship?
- [ ] What protocol or technology is used?
- [ ] What data flows through this connection?
- [ ] Is it synchronous or asynchronous?

**Example Relationship Descriptions**:
```
✓ GOOD:
"Reads/writes customer data via HTTPS REST API"
"Sends order events via RabbitMQ"
"Queries product catalog using GraphQL"

✗ AVOID:
"Uses" (too vague)
"Connects to" (no protocol info)
"Talks to" (unclear)
```

### Step 4: Choose Diagram Format

**PlantUML** (Recommended for detailed diagrams)
- Rich formatting options
- Better for complex systems
- IDE and GitHub support
- Can be version-controlled

**Mermaid** (Recommended for simplicity)
- Native GitHub/GitLab rendering
- Simpler syntax
- Great for documentation
- Limited styling options

**Structurizr DSL** (For comprehensive workspace)
- Full C4 Model support
- Multiple views from single model
- Cloud hosting option
- Steeper learning curve

### Step 5: Generate and Refine

1. **Create initial diagram** using templates (see resources)
2. **Review with stakeholders** for accuracy
3. **Iterate based on feedback**
4. **Document assumptions** and decisions
5. **Version control** the diagram source

## Core Principles

### 1. Audience-Appropriate Abstraction

**Principle**: Match diagram level to audience technical expertise.

- **C-level, product managers** → Context diagrams
- **Tech leads, architects** → Context + Container diagrams
- **Development teams** → Container + Component diagrams
- **Individual developers** → Component + Code (if needed)

### 2. Consistent Notation

**Principle**: Use standard C4 notation and styling.

**Element Types**:
- **Person**: External user or actor (stick figure icon)
- **Software System**: Your system or external systems (box)
- **Container**: Application or data store (box with technology)
- **Component**: Module or service within container (box)

**Relationship Types**:
- Solid line: Synchronous interaction
- Dashed line: Asynchronous interaction
- Arrow direction: Direction of dependency

**Colors** (standard C4 palette):
- Person: `#08427B`
- System (internal): `#1168BD`
- System (external): `#999999`
- Container: `#438DD5`
- Component: `#85BBF0`

### 3. Clear Relationship Descriptions

**Principle**: Every relationship must describe protocol and purpose.

**Template**: `[Verb] [what] using [protocol/technology]`

**Examples**:
```
✓ "Reads customer data using REST API (HTTPS)"
✓ "Sends order events using RabbitMQ (AMQP)"
✓ "Queries products using GraphQL over HTTPS"
✓ "Stores user sessions in Redis"
```

### 4. Technology Transparency

**Principle**: Include relevant technology choices in diagrams.

**For Containers**, specify:
- Programming language/framework
- Runtime environment
- Database type
- Communication protocols

**Example**:
```
Container: API Gateway
Technology: Node.js, Express
Description: Routes requests to backend services
```

### 5. Supplementary Documentation

**Principle**: Diagrams show structure; documentation explains decisions.

**Accompany diagrams with**:
- Architecture Decision Records (ADRs)
- Deployment documentation
- Data flow descriptions
- Security considerations
- Scalability notes

## Navigation Guide

| Need to...                                  | Read this resource                                            |
|---------------------------------------------|---------------------------------------------------------------|
| Create PlantUML C4 diagrams                 | [PlantUML C4 Templates](resources/plantuml-templates.md)      |
| Create Mermaid C4 diagrams                  | [Mermaid C4 Templates](resources/mermaid-templates.md)        |
| See complete system examples                | [Complete C4 Examples](resources/complete-examples.md)        |

## Best Practices Summary

### DO:
✅ Start with Context diagram for any new documentation
✅ Use standard C4 notation and color scheme
✅ Include technology stack in Container diagrams
✅ Describe relationships with protocol and purpose
✅ Version control diagram source files
✅ Keep diagrams up-to-date with system changes
✅ Create separate diagrams for current and future state
✅ Use consistent naming across all diagram levels
✅ Supplement diagrams with ADRs and documentation
✅ Review diagrams with team regularly

### DON'T:
❌ Mix abstraction levels in a single diagram
❌ Include implementation details in Context diagrams
❌ Use vague relationship descriptions ("uses", "connects to")
❌ Create diagrams without stakeholder input
❌ Let diagrams become stale or outdated
❌ Over-complicate Context diagrams with too many elements
❌ Forget to show external dependencies
❌ Use non-standard notation without explanation
❌ Create diagrams in isolation from the team
❌ Skip Level 2 (Container) - it's the most useful level

## Quick Reference Checklist

### Creating a Context Diagram
- [ ] Identified your system (single box with clear name)
- [ ] Listed all external users/actors
- [ ] Listed all external systems
- [ ] Described each relationship with protocol
- [ ] Added high-level system description
- [ ] Verified audience-appropriate level

### Creating a Container Diagram
- [ ] Listed all applications (web, mobile, desktop)
- [ ] Listed all backend services/APIs
- [ ] Listed all databases with types
- [ ] Listed message queues, caches, storage
- [ ] Specified technology stack for each container
- [ ] Described inter-container communication
- [ ] Verified all protocols specified

### Creating a Component Diagram
- [ ] Identified major components within container
- [ ] Defined component responsibilities
- [ ] Mapped dependencies between components
- [ ] Documented external dependencies
- [ ] Specified key interfaces/APIs
- [ ] Verified appropriate granularity

### Diagram Quality
- [ ] Used standard C4 notation
- [ ] Applied standard color scheme
- [ ] No vague relationship descriptions
- [ ] Consistent naming across levels
- [ ] Version controlled diagram source
- [ ] Reviewed with stakeholders
- [ ] Documented alongside ADRs

## Success Metrics

A well-created C4 diagram set should achieve:

| Metric                        | Target                                                    |
|-------------------------------|-----------------------------------------------------------|
| **Comprehension**             | Stakeholders understand system in <10 minutes             |
| **Accuracy**                  | Diagrams match actual/planned system 100%                 |
| **Completeness**              | All major elements and relationships documented           |
| **Currency**                  | Diagrams updated within 1 sprint of system changes        |
| **Accessibility**             | Diagrams in version control and easily found              |
| **Usefulness**                | Referenced in PRs, onboarding, and decisions              |
| **Clarity**                   | No ambiguous relationships or unclear elements            |
| **Consistency**               | Standard notation used throughout                         |

## Related Skills

- **backend-dev-guidelines** - Implementing the architecture in Node.js/Express
- **frontend-dev-guidelines** - Implementing frontend containers
- **rust-skills** - Implementing services in Rust
- **meta-skill** - General skill development and architecture patterns

---

**Skill Status**: Production-ready comprehensive C4 architecture guide
**Line Count**: <350 lines following 500-line rule
**Coverage**: Complete C4 Model (Context, Container, Component, Code levels)
**Diagram Formats**: PlantUML, Mermaid, Structurizr DSL

**Next Steps**:
- For PlantUML templates: [PlantUML C4 Templates](resources/plantuml-templates.md)
- For Mermaid templates: [Mermaid C4 Templates](resources/mermaid-templates.md)
- For complete examples: [Complete C4 Examples](resources/complete-examples.md)
