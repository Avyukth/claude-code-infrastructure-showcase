# Skill Creation - Complete Guide

Step-by-step guide for creating Claude Code skills from initial concept through production deployment.

## Table of Contents

- [Pre-Creation Planning](#pre-creation-planning)
- [Step-by-Step Workflow](#step-by-step-workflow)
- [SKILL.md Template and Best Practices](#skillmd-template-and-best-practices)
- [skill-rules.json Configuration](#skill-rulesjson-configuration)
- [Resource File Organization](#resource-file-organization)
- [Scripts and Assets](#scripts-and-assets)
- [Real-World Examples](#real-world-examples)

---

## Pre-Creation Planning

### Evaluation-First Approach (Anthropic Recommended)

**Before writing extensive documentation:**

1. **Identify specific gaps** in Claude's capabilities
2. **Gather 3-10 representative tasks** from actual work
3. **Test Claude without the skill** to identify failure modes
4. **Document specific problems** the skill should solve
5. **Build incrementally** to address real needs

**Why this works:** Ensures you solve real problems, not imagined ones.

### Questions to Answer

**Purpose:**
- What specific problem does this skill solve?
- What tasks will it support?
- Who is the target user?

**Scope:**
- What's included in this skill?
- What's explicitly out of scope?
- Are there related skills this complements?

**Activation:**
- When should this skill activate?
- What keywords or patterns trigger it?
- Should it be enforced (guardrail) or advisory (domain)?

**Content:**
- What information must be immediately available?
- What can be deferred to resource files?
- Are scripts or assets needed?

---

## Step-by-Step Workflow

### Step 1: Create Directory Structure

```bash
# Navigate to skills directory
cd $CLAUDE_PROJECT_DIR/.claude/skills

# Create skill directory
mkdir my-skill-name

# Create subdirectories if needed
mkdir my-skill-name/resources
mkdir my-skill-name/scripts
mkdir my-skill-name/assets
```

**Naming conventions:**
- Use lowercase with hyphens: `backend-dev-guidelines`
- Prefer gerund form (verb + -ing): `processing-pdfs`, `validating-schemas`
- Be descriptive: `react-component-patterns` not just `react`

### Step 2: Create SKILL.md with Frontmatter

**Minimum viable SKILL.md:**

```markdown
---
name: my-skill-name
description: Comprehensive description that includes ALL trigger keywords, use cases, technologies, and scenarios where this skill should activate. Be explicit and detailed. Maximum 1024 characters.
---

# My Skill Name

## Purpose

Clear, concise statement of what this skill helps with.

## When to Use This Skill

Explicit list of activation scenarios:
- Working on [specific task type]
- Using [specific technology]
- Creating [specific artifact]

## Core Guidance

The essential information needed to accomplish the skill's purpose.
Keep this under 500 lines total.
```

**Frontmatter rules:**
- `name`: Must match directory name exactly
- `description`: Critical for auto-activation - include ALL keywords
- Both fields are required

### Step 3: Write Description (Most Important!)

The description field is your skill's discovery mechanism. Claude uses it to decide when to activate the skill.

**Good description template:**
```yaml
description: [What it does]. Use when [scenario 1], [scenario 2], or [scenario 3]. Covers [topic 1], [topic 2], [topic 3]. Works with [technology 1], [technology 2]. Keywords: [keyword1], [keyword2], [keyword3].
```

**Example (backend-dev-guidelines):**
```yaml
description: Comprehensive backend development guide for Node.js/Express/TypeScript microservices. Use when creating routes, controllers, services, repositories, middleware, or working with Express APIs, Prisma database access, Sentry error tracking, Zod validation, unifiedConfig, dependency injection, or async patterns. Covers layered architecture (routes → controllers → services → repositories), BaseController pattern, error handling, performance monitoring, testing strategies, and migration from legacy patterns.
```

**What to include:**
- Primary purpose (first sentence)
- Activation scenarios ("Use when...")
- Technologies covered
- Key concepts/patterns
- Related terms and synonyms

**What to avoid:**
- Generic descriptions: "Helps with development"
- Missing key technologies: If it uses React, SAY "React"
- Vague language: "Various things" or "stuff"

### Step 4: Structure the Content

**Follow this pattern:**

```markdown
# Skill Name

## Purpose
One paragraph, 2-3 sentences

## When to Use This Skill
Bulleted list of specific scenarios

## Quick Start (optional but helpful)
Checklists, templates, or quick reference

## Core Concepts/Guidance
The main content (organized with clear headings)

## Navigation Guide (for multi-resource skills)
Table linking to resource files

## Related Skills/Resources
Links to complementary skills
```

**Content organization tips:**
- Use clear hierarchical headings (##, ###, ####)
- Include code examples in fenced blocks with language tags
- Use bullet points and numbered lists
- Add tables for comparisons or quick reference
- Include "DO/DON'T" examples for clarity

### Step 5: Apply Progressive Disclosure

**When SKILL.md approaches 500 lines:**

1. **Identify logical sections** that can stand alone
2. **Move detailed content** to resource files
3. **Keep overview** and navigation in SKILL.md
4. **Link clearly** to resource files

**Pattern:**
```markdown
## Topic Overview

Brief 2-3 sentence introduction to the topic.

**Key Concepts:**
- Concept 1
- Concept 2
- Concept 3

**[📖 Complete Guide: resources/topic-name.md](resources/topic-name.md)**
```

**Resource file naming:**
- Use descriptive names: `architecture-patterns.md` not `patterns.md`
- Group related concepts: `routing-and-controllers.md`
- Keep each resource under 500 lines

### Step 6: Add to skill-rules.json

**Location:** `.claude/skills/skill-rules.json`

**If file doesn't exist, create it:**
```json
{
  "version": "1.0",
  "skills": {}
}
```

**Add your skill entry:**
```json
{
  "version": "1.0",
  "skills": {
    "my-skill-name": {
      "type": "domain",
      "enforcement": "suggest",
      "priority": "high",
      "promptTriggers": {
        "keywords": ["keyword1", "keyword2", "keyword3"],
        "intentPatterns": [
          "(create|add|build).*?(feature|component)",
          "(how does|explain).*?system"
        ]
      },
      "fileTriggers": {
        "pathPatterns": [
          "src/**/*.ts",
          "backend/**/*.ts"
        ],
        "pathExclusions": [
          "**/*.test.ts"
        ],
        "contentPatterns": [
          "import.*SomeLibrary",
          "class.*Controller"
        ]
      }
    }
  }
}
```

See [integration-guide.md](integration-guide.md) for complete schema reference.

### Step 7: Test Triggers

**Test prompt triggers (UserPromptSubmit):**
```bash
echo '{"session_id":"test","prompt":"creating a new feature"}' | \
  npx tsx .claude/hooks/skill-activation-prompt.ts
```

Expected: Your skill appears in the output.

**Test file triggers (PreToolUse):**
```bash
cat <<'EOF' | npx tsx .claude/hooks/skill-verification-guard.ts
{
  "session_id": "test",
  "tool_name": "Edit",
  "tool_input": {"file_path": "/path/to/your/src/file.ts"}
}
EOF
```

Expected: Exit code and output based on your configuration.

### Step 8: Iterate Based on Usage

**Monitor for:**
- **False negatives** - Skill doesn't activate when it should
- **False positives** - Skill activates when it shouldn't
- **Missing keywords** - User phrases you didn't anticipate
- **Unclear guidance** - Sections that need examples

**Refine:**
- Add missing keywords to description and skill-rules.json
- Broaden or narrow intent patterns
- Adjust file path patterns
- Add clarifying examples

---

## SKILL.md Template and Best Practices

### Complete Template

```markdown
---
name: skill-name-lowercase-with-hyphens
description: Comprehensive description with ALL trigger keywords and use cases. Include technologies, scenarios, and key concepts. Be explicit about when Claude should activate this skill.
---

# Skill Name (Title Case)

## Purpose

Clear statement of what this skill helps accomplish. 2-3 sentences maximum.

## When to Use This Skill

Explicit activation scenarios:
- Scenario 1 with specific details
- Scenario 2 with technologies mentioned
- Scenario 3 with task types

## Quick Start (Optional)

### Checklist for [Common Task]

- [ ] Step 1 with concrete action
- [ ] Step 2 with verification
- [ ] Step 3 with outcome

### Common Imports/Setup

```language
// Code example that users can copy
```

## Core Concepts

### Concept 1

Explanation with examples.

**Good practice:**
```language
// Example of correct approach
```

**Anti-pattern:**
```language
// Example of what to avoid
```

### Concept 2

More guidance.

## Navigation Guide (for multi-resource skills)

| Need to... | Read this resource |
|------------|-------------------|
| Task type 1 | [resource-1.md](resources/resource-1.md) |
| Task type 2 | [resource-2.md](resources/resource-2.md) |

## Quick Reference

Tables, checklists, or condensed information for fast lookup.

## Related Skills

- **skill-name-1** - Brief description of relationship
- **skill-name-2** - Brief description of relationship

---

**Skill Status**: [Status description]
**Line Count**: [<500 lines]
**Progressive Disclosure**: [Number of resource files if applicable]
```

### Frontmatter Best Practices

**Name field:**
- Lowercase only
- Hyphens for spaces
- Match directory name exactly
- Descriptive and unique
- Good: `react-component-patterns`
- Bad: `rcp`, `ReactComponentPatterns`, `component patterns`

**Description field:**
- Maximum 1024 characters
- Include ALL trigger keywords
- Mention specific technologies
- List common use cases
- Use active voice
- Be concrete, not abstract
- Good: "Use when creating Express routes, implementing middleware, or handling errors with Sentry"
- Bad: "Helps with backend stuff"

### Content Writing Guidelines

**Imperative form (Anthropic recommended):**
- Use verb-first instructions: "Create the service", "Configure the router"
- Avoid second-person: "You should create..." → "Create..."
- Be direct and actionable

**Code examples:**
- Always include language tag: ```typescript, ```bash, ```json
- Show both good and bad examples
- Add comments explaining key parts
- Keep examples focused and minimal

**Organization:**
- Group related information together
- Use consistent heading levels
- Keep sections independent when possible
- Link to resources for deep dives

---

## skill-rules.json Configuration

### Complete Schema

```typescript
interface SkillRules {
    version: string;
    skills: Record<string, SkillRule>;
}

interface SkillRule {
    type: 'guardrail' | 'domain';
    enforcement: 'block' | 'suggest' | 'warn';
    priority: 'critical' | 'high' | 'medium' | 'low';

    promptTriggers?: {
        keywords?: string[];
        intentPatterns?: string[];
    };

    fileTriggers?: {
        pathPatterns: string[];
        pathExclusions?: string[];
        contentPatterns?: string[];
        createOnly?: boolean;
    };

    blockMessage?: string;  // Required for guardrails

    skipConditions?: {
        sessionSkillUsed?: boolean;
        fileMarkers?: string[];
        envOverride?: string;
    };
}
```

### Field-by-Field Guide

**type:**
- `"domain"` - Advisory skill, provides guidance
- `"guardrail"` - Enforced skill, blocks until used

**enforcement:**
- `"suggest"` - Inject reminder via UserPromptSubmit (most common)
- `"block"` - Physically prevent tool execution via PreToolUse
- `"warn"` - Low-priority advisory (rarely used)

**priority:**
- `"critical"` - Must-have, blocking
- `"high"` - Important, recommended
- `"medium"` - Helpful, optional
- `"low"` - Nice-to-have

**promptTriggers.keywords:**
- Case-insensitive substring matching
- Include variations: ["layout", "layouts", "layout system"]
- Be specific enough to avoid false positives
- Cover synonyms: ["API", "endpoint", "route"]

**promptTriggers.intentPatterns:**
- Regex strings (will be compiled)
- Use non-greedy matching: `.*?` not `.*`
- Capture actions: `(create|add|build|implement)`
- Capture targets: `(feature|component|service)`
- Test at https://regex101.com/

**fileTriggers.pathPatterns:**
- Glob patterns: `**` for any directories, `*` for any characters
- Examples: `"frontend/src/**/*.tsx"`, `"**/schema.prisma"`
- Be specific to reduce false positives

**fileTriggers.contentPatterns:**
- Regex strings matching file contents
- Escape special characters: `\\.findMany\\(`
- Match imports: `import.*[Pp]risma`
- Match usage: `useState|useEffect`

**blockMessage:**
- Required for guardrails (enforcement: "block")
- Use `{file_path}` placeholder
- Provide clear action steps
- Include reason and workaround

**skipConditions:**
- `sessionSkillUsed: true` - Don't nag repeatedly in same session
- `fileMarkers: ["@skip-validation"]` - Permanent skip for verified files
- `envOverride: "SKIP_MY_SKILL"` - Emergency disable

### Domain Skill Example

```json
{
  "frontend-dev-guidelines": {
    "type": "domain",
    "enforcement": "suggest",
    "priority": "high",
    "promptTriggers": {
      "keywords": [
        "react", "component", "mui", "tanstack",
        "frontend", "ui", "styling"
      ],
      "intentPatterns": [
        "(create|add|build).*?(component|page|feature)",
        "(style|design).*?(button|form|layout)"
      ]
    },
    "fileTriggers": {
      "pathPatterns": [
        "frontend/src/**/*.tsx",
        "src/components/**/*.tsx"
      ],
      "pathExclusions": [
        "**/*.test.tsx"
      ]
    }
  }
}
```

### Guardrail Skill Example

```json
{
  "database-verification": {
    "type": "guardrail",
    "enforcement": "block",
    "priority": "critical",
    "promptTriggers": {
      "keywords": ["prisma", "database", "query"],
      "intentPatterns": ["(add|create).*?(user|feature)"]
    },
    "fileTriggers": {
      "pathPatterns": ["**/src/**/*.ts"],
      "contentPatterns": [
        "import.*[Pp]risma",
        "\\.findMany\\(",
        "\\.create\\("
      ]
    },
    "blockMessage": "⚠️ BLOCKED - Database Operation Detected\n\n📋 REQUIRED:\n1. Use Skill: 'database-verification'\n2. Verify table/column names\n3. Retry edit\n\nFile: {file_path}",
    "skipConditions": {
      "sessionSkillUsed": true,
      "fileMarkers": ["@skip-validation"],
      "envOverride": "SKIP_DB_VERIFICATION"
    }
  }
}
```

---

## Resource File Organization

### When to Create Resource Files

Create separate resource files when:
- SKILL.md exceeds ~300 lines
- Topic deserves deep dive (>100 lines)
- Content is independently useful
- Multiple related sub-topics exist

### Resource File Structure

**Location:** `{skill-name}/resources/{topic-name}.md`

**Template:**
```markdown
# Topic Name

Brief introduction to this topic and why it's important.

## Table of Contents (if >100 lines)

- [Section 1](#section-1)
- [Section 2](#section-2)

## Section 1

Content here.

## Section 2

More content.

---

**Related Files:**
- [SKILL.md](../SKILL.md) - Main skill guide
- [other-resource.md](other-resource.md) - Related topic
```

### Resource File Best Practices

**Naming:**
- Descriptive: `architecture-patterns.md` not `patterns.md`
- Grouped topics: `routing-and-controllers.md`
- Avoid generic names: `advanced.md` → `advanced-optimization.md`

**Organization:**
- Add table of contents if >100 lines
- Use consistent heading hierarchy
- Keep each file focused on one major topic
- Cross-reference related resources

**Content:**
- Can exceed 500 lines if needed (but keep focused)
- Include complete examples
- More detailed than SKILL.md
- Self-contained when possible

---

## Scripts and Assets

### Scripts Directory

**Purpose:** Executable code for deterministic, repeated tasks.

**When to include scripts:**
- Initialization or setup tasks
- Validation or verification tools
- Code generation utilities
- Data transformation

**Structure:**
```
{skill-name}/scripts/
  init_skill.py          # Initialization script
  validate_config.py     # Validation utility
  README.md              # Script documentation
```

**Best practices:**
- Make scripts executable: `chmod +x script.sh`
- Include shebang: `#!/usr/bin/env python3`
- Document usage in script or README
- Keep scripts focused and simple

**Example (from Anthropic skills repo):**
```python
#!/usr/bin/env python3
"""Initialize a new skill with template structure."""

import argparse
import os
from pathlib import Path

def init_skill(name: str, path: str):
    """Create skill directory with SKILL.md template."""
    skill_path = Path(path) / name
    skill_path.mkdir(parents=True, exist_ok=True)

    skill_md = skill_path / "SKILL.md"
    skill_md.write_text(SKILL_TEMPLATE.format(name=name))

    print(f"Created skill at {skill_path}")

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("name", help="Skill name")
    parser.add_argument("--path", default=".", help="Parent directory")
    args = parser.parse_args()

    init_skill(args.name, args.path)
```

### Assets Directory

**Purpose:** Templates, images, fonts, or other files NOT loaded into context.

**When to include assets:**
- Output templates
- Example configurations
- Diagrams or images
- Reference documents (PDF, etc.)

**Structure:**
```
{skill-name}/assets/
  template.json         # Output template
  diagram.png           # Architecture diagram
  example-config.yaml   # Sample configuration
```

**Best practices:**
- Document asset purpose
- Keep assets minimal
- Don't bloat skill directory
- Reference in SKILL.md or resources

---

## Real-World Examples

### Example 1: Simple Single-File Skill

**Use case:** Error tracking with Sentry

**Structure:**
```
error-tracking/
  SKILL.md              # ~250 lines, complete in one file
```

**When this works:**
- Focused topic
- Limited scope
- Straightforward guidance
- No need for deep dives

### Example 2: Modular Multi-Resource Skill

**Use case:** Backend development guidelines

**Structure:**
```
backend-dev-guidelines/
  SKILL.md                          # 304 lines, overview + navigation
  resources/
    architecture-overview.md        # Layered architecture
    routing-and-controllers.md      # Route and controller patterns
    services-and-repositories.md    # Business logic layer
    validation-patterns.md          # Zod validation
    sentry-and-monitoring.md        # Error tracking
    middleware-guide.md             # Express middleware
    database-patterns.md            # Prisma patterns
    configuration.md                # UnifiedConfig
    async-and-errors.md             # Async patterns
    testing-guide.md                # Testing strategies
    complete-examples.md            # Full examples
```

**When this works:**
- Complex domain
- Multiple sub-topics
- Each deserves deep coverage
- Users need quick navigation

### Example 3: Guardrail Skill with Blocking

**Use case:** Database verification before queries

**Structure:**
```
database-verification/
  SKILL.md              # Instructions for verification
```

**skill-rules.json:**
```json
{
  "database-verification": {
    "type": "guardrail",
    "enforcement": "block",
    "priority": "critical",
    "fileTriggers": {
      "contentPatterns": ["import.*[Pp]risma", "\\.findMany\\("]
    },
    "blockMessage": "⚠️ Verify database schema before queries",
    "skipConditions": {
      "sessionSkillUsed": true
    }
  }
}
```

**When this works:**
- Critical mistakes must be prevented
- Real-time verification needed
- Session tracking prevents nagging

---

## Quick Start Templates

### Minimal Skill Template

```markdown
---
name: my-skill
description: What this skill does and when to use it. Include keywords.
---

# My Skill

## Purpose
What problem this solves.

## When to Use
- Scenario 1
- Scenario 2

## Guidance
The core information.
```

### Full-Featured Skill Template

Use the complete template from [SKILL.md Template and Best Practices](#skillmd-template-and-best-practices) above.

---

## Next Steps

1. **Plan your skill** using evaluation-first approach
2. **Create SKILL.md** with proper frontmatter
3. **Configure skill-rules.json** with triggers
4. **Test activation** with manual commands
5. **Iterate** based on real usage
6. **Add resources** as skill grows

**Related guides:**
- [skill-verification.md](skill-verification.md) - Testing your skill
- [integration-guide.md](integration-guide.md) - Hook and trigger system
- [best-practices.md](best-practices.md) - Patterns and anti-patterns
