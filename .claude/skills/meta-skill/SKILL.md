---
name: skill-builder-guide
description: Comprehensive meta-guide for creating, verifying, and integrating Claude Code skills. Use when building new skills, understanding skill architecture, configuring auto-activation triggers, implementing progressive disclosure patterns, integrating with hooks and agents, verifying skill functionality, troubleshooting activation issues, or establishing skill development workflows. Covers complete skill lifecycle from conception through deployment including YAML frontmatter, trigger systems (keywords, intent patterns, file paths, content patterns), skill-rules.json configuration, 500-line rule, modular resource patterns, testing strategies, and Claude Code ecosystem integration.
---

# Skill Builder Guide

## Purpose

This meta-skill provides a comprehensive framework for creating, verifying, and maintaining high-quality skills in Claude Code. It synthesizes production-tested patterns from real-world usage with official Anthropic best practices to guide developers in building modular, auto-activating skills that integrate seamlessly with the Claude Code ecosystem.

Think of this as your complete reference for skill development—from initial concept through production deployment.

## When to Use This Skill

This skill automatically activates when you:
- Create new skills or modify existing ones
- Configure skill auto-activation systems
- Design trigger patterns (keywords, intents, file paths)
- Implement progressive disclosure with resource files
- Integrate skills with hooks, agents, or commands
- Debug skill activation issues
- Establish skill development workflows
- Understand the 500-line rule and modular patterns
- Verify and test skill functionality

## Core Concepts

### What is a Skill?

A skill is a modular knowledge package that transforms Claude from a general-purpose assistant into a domain-specific expert. Skills provide:

1. **Specialized workflows** - Multi-step procedures for specific domains
2. **Tool integrations** - Instructions for working with APIs and file formats
3. **Domain expertise** - Project-specific knowledge and business logic
4. **Bundled resources** - Scripts, references, and assets for complex tasks

### Anatomy of a Modern Skill

**Required:**
- **SKILL.md** with YAML frontmatter (name, description) and markdown content

**Optional (for complex skills):**
- **resources/** directory with topic-specific reference files
- **scripts/** directory with executable utilities
- **assets/** directory with templates, images, or other files

### The Three-Tier Loading Model

Skills use progressive disclosure to manage context efficiently:

1. **Metadata** (always loaded, ~100 words) - Name and description in YAML frontmatter
2. **SKILL.md body** (loaded when triggered, <500 lines) - Core guidance and navigation
3. **Bundled resources** (loaded as needed) - Detailed references and deep dives

This pattern keeps skills under context limits while providing comprehensive coverage.

### Auto-Activation System

**The breakthrough feature** - skills that activate automatically when needed:

```
User Prompt/File Edit
    ↓
Hooks execute (UserPromptSubmit/PreToolUse)
    ↓
skill-rules.json triggers checked
    ↓
Matching skills suggested/enforced
    ↓
Claude loads relevant skills
```

**Components:**
- **skill-rules.json** - Central configuration defining all triggers
- **Hooks** - TypeScript/Bash scripts that run on events
- **Session tracking** - Prevents repeated nagging
- **Skip conditions** - User escape hatches

## Quick Start: Create Your First Skill

### Step 1: Understand Through Examples (Evaluation-First)

Before writing anything, gather 3+ concrete examples:
- What tasks will this skill support?
- What questions should it answer?
- When should it activate?

**Best Practice:** Build evaluations BEFORE extensive documentation.

### Step 2: Plan the Structure

Decide what belongs in the skill:
- Core guidance → SKILL.md
- Detailed procedures → resource files
- Reusable code → scripts
- Templates/assets → assets directory

### Step 3: Create the Skill File

**Location:** `.claude/skills/{skill-name}/SKILL.md`

**Template:**
```markdown
---
name: my-skill-name
description: Comprehensive description including ALL trigger keywords, use cases, and when Claude should activate this skill. Be explicit about technologies, patterns, and scenarios. Max 1024 characters.
---

# My Skill Name

## Purpose
Clear statement of what this skill helps with

## When to Use This Skill
Explicit list of activation scenarios

## Core Guidance
The essential information (under 500 lines)
```

### Step 4: Configure Auto-Activation

Add entry to `.claude/skills/skill-rules.json`:

```json
{
  "my-skill-name": {
    "type": "domain",
    "enforcement": "suggest",
    "priority": "high",
    "promptTriggers": {
      "keywords": ["keyword1", "keyword2"],
      "intentPatterns": ["(create|add).*?something"]
    },
    "fileTriggers": {
      "pathPatterns": ["src/**/*.ts"],
      "contentPatterns": ["import.*Something"]
    }
  }
}
```

### Step 5: Test and Iterate

Test triggers manually:
```bash
# Test prompt triggers
echo '{"prompt":"test prompt with keyword"}' | \
  npx tsx .claude/hooks/skill-activation-prompt.ts

# Test file triggers
cat <<'EOF' | npx tsx .claude/hooks/skill-verification-guard.ts
{"tool_name":"Edit","tool_input":{"file_path":"src/test.ts"}}
EOF
```

Refine based on actual usage patterns.

## Navigation Guide

| Need to... | Read this resource |
|------------|-------------------|
| Create a new skill step-by-step | [skill-creation.md](resources/skill-creation.md) |
| Verify and test skills | [skill-verification.md](resources/skill-verification.md) |
| Integrate with Claude Code ecosystem | [integration-guide.md](resources/integration-guide.md) |
| Learn patterns and anti-patterns | [best-practices.md](resources/best-practices.md) |
| Advanced topics (hooks, progressive disclosure) | [advanced-topics.md](resources/advanced-topics.md) |
| Troubleshoot issues | [troubleshooting.md](resources/troubleshooting.md) |
| See complete examples | [complete-examples.md](resources/complete-examples.md) |

## Key Principles (From Both Sources)

### From Production Experience (70% Weight)

1. **500-Line Rule** - Keep SKILL.md under 500 lines; use resource files for details
2. **Progressive Disclosure** - Load information only as needed
3. **Auto-Activation Focus** - Configure triggers so skills suggest themselves
4. **Session Tracking** - Prevent repeated nagging with state management
5. **Modular Design** - Separate concerns across multiple files

### From Anthropic Best Practices (30% Weight)

6. **Evaluation-First** - Build evaluations before extensive documentation
7. **Deterministic Testing** - Use concrete, repeatable test cases
8. **Rich Descriptions** - Include all trigger keywords in frontmatter
9. **Imperative Instructions** - Use verb-first, action-oriented language
10. **Context Efficiency** - Information lives in ONE place only

## Skill Types

### Domain Skills (Advisory)
- **Type:** `"domain"`
- **Enforcement:** `"suggest"`
- **Purpose:** Provide comprehensive guidance for specific areas
- **Examples:** backend-dev-guidelines, frontend-dev-guidelines
- **When to use:** Complex systems requiring deep knowledge

### Guardrail Skills (Enforced)
- **Type:** `"guardrail"`
- **Enforcement:** `"block"`
- **Purpose:** Prevent critical mistakes through PreToolUse hooks
- **Examples:** database-verification, security-checks
- **When to use:** Mistakes that cause runtime errors or data issues

## Trigger System Overview

### Four Trigger Types

1. **Keywords** (Explicit) - Case-insensitive substring matching in prompts
2. **Intent Patterns** (Implicit) - Regex detecting user actions/questions
3. **File Paths** (Location) - Glob patterns matching file locations
4. **Content Patterns** (Technology) - Regex matching file contents

### Configuration Example

```json
{
  "promptTriggers": {
    "keywords": ["layout", "grid", "component"],
    "intentPatterns": [
      "(create|add|build).*?(feature|component)",
      "(how does|explain).*?system"
    ]
  },
  "fileTriggers": {
    "pathPatterns": ["frontend/src/**/*.tsx"],
    "pathExclusions": ["**/*.test.tsx"],
    "contentPatterns": ["import.*React", "useState|useEffect"]
  }
}
```

See [integration-guide.md](resources/integration-guide.md) for complete trigger reference.

## The 500-Line Rule in Practice

**Problem:** Large skills hit context limits and slow down responses.

**Solution:** Modular structure with progressive disclosure.

**Pattern:**
```
my-skill/
  SKILL.md                    # <500 lines - overview + navigation
  resources/
    architecture.md           # <500 lines - deep dive topic 1
    patterns.md               # <500 lines - deep dive topic 2
    examples.md               # <500 lines - complete examples
```

**Benefits:**
- Stay under context limits
- Load only relevant information
- Faster responses
- Better organization

**Real Example:** backend-dev-guidelines has 12 resource files covering routing, controllers, services, repositories, validation, error handling, testing, and more - all accessible via progressive loading.

## Quick Reference Checklist

### Creating a New Skill
- [ ] Gather 3+ concrete use cases
- [ ] Create SKILL.md with proper frontmatter
- [ ] Write description with ALL trigger keywords
- [ ] Keep main file under 500 lines
- [ ] Create resource files for detailed topics
- [ ] Add entry to skill-rules.json
- [ ] Configure triggers (keywords, intents, paths)
- [ ] Test with manual commands
- [ ] Verify activation in real usage
- [ ] Iterate based on false positives/negatives

### Skill-Rules.json Entry
- [ ] Unique skill name matching SKILL.md
- [ ] Appropriate type (domain vs guardrail)
- [ ] Correct enforcement level
- [ ] Meaningful priority setting
- [ ] At least 2 trigger mechanisms
- [ ] Tested intent patterns (regex101.com)
- [ ] Validated glob patterns
- [ ] Skip conditions configured
- [ ] Valid JSON syntax (test with `jq`)

### Integration
- [ ] skill-activation-prompt hook installed
- [ ] Hook registered in settings.json
- [ ] Hook dependencies installed (npm)
- [ ] Hook executable (chmod +x)
- [ ] File paths match project structure
- [ ] No hardcoded paths to other projects

## Resource Files

### [skill-creation.md](resources/skill-creation.md)
Complete step-by-step guide for building skills from scratch:
- Detailed workflow from concept to deployment
- Templates for SKILL.md and skill-rules.json
- Frontmatter best practices
- Resource file organization
- Script and asset management

### [skill-verification.md](resources/skill-verification.md)
Testing and validation strategies:
- Manual testing with npx tsx
- Automated evaluation frameworks
- Test-driven development workflow
- Verification-before-completion pattern
- Coverage and regression testing

### [integration-guide.md](resources/integration-guide.md)
Claude Code ecosystem integration:
- Hook system deep dive
- skill-rules.json complete schema
- Trigger types and patterns
- Session state management
- settings.json configuration

### [best-practices.md](resources/best-practices.md)
Synthesized patterns and anti-patterns:
- Progressive disclosure strategies
- Trigger pattern library
- Common pitfalls and solutions
- Performance optimization
- Maintenance and versioning

### [advanced-topics.md](resources/advanced-topics.md)
Advanced concepts and future enhancements:
- Custom hook development
- Dynamic rule updates
- Skill dependencies
- Analytics and monitoring
- Multi-language support

### [troubleshooting.md](resources/troubleshooting.md)
Comprehensive debugging guide:
- Skill not activating
- False positives/negatives
- Hook execution failures
- Performance issues
- Common configuration errors

### [complete-examples.md](resources/complete-examples.md)
Full working examples:
- Simple domain skill
- Complex guardrail skill
- Multi-resource modular skill
- Tech-specific skill adaptation
- Migration from monolithic to modular

## Philosophy: Evaluation-First Development

**Traditional approach (less effective):**
1. Write extensive documentation
2. Hope it covers the right scenarios
3. Discover gaps during use
4. Refactor documentation

**Evaluation-first approach (Anthropic recommended):**
1. Identify 3-10 representative tasks
2. Test Claude on these tasks WITHOUT the skill
3. Identify specific failure modes
4. Build skill incrementally to address gaps
5. Verify improvement with evaluations

**Why it works:** Ensures skills solve real problems, not imagined ones.

## Integration with Broader Ecosystem

Skills work best when integrated with:

**Hooks** - Auto-activation system
- skill-activation-prompt (UserPromptSubmit)
- skill-verification-guard (PreToolUse)
- post-tool-use-tracker (PostToolUse)

**Agents** - Specialized task executors
- Code architecture reviewers reference skills
- Refactor planners use skill patterns
- Documentation generators follow skill standards

**Commands** - Quick workflows
- /dev-docs leverages skill knowledge
- Custom commands can trigger skills

**Dev Docs** - Persistent context
- Skills complement session-level docs
- Dev docs survive context resets
- Skills provide the "how", docs provide the "what"

## Getting Started: Recommended Path

### For First-Time Skill Creators
1. Read [skill-creation.md](resources/skill-creation.md) completely
2. Study one example from [complete-examples.md](resources/complete-examples.md)
3. Create a simple domain skill for your project
4. Test and iterate
5. Expand with resource files as needed

### For Experienced Developers
1. Review [best-practices.md](resources/best-practices.md) for patterns
2. Use templates from [skill-creation.md](resources/skill-creation.md)
3. Configure advanced triggers via [integration-guide.md](resources/integration-guide.md)
4. Implement guardrails if needed
5. Set up monitoring and analytics

### For Teams
1. Establish skill creation standards
2. Use evaluation-first approach
3. Create shared skill library
4. Document project-specific patterns
5. Version control skill-rules.json

## Success Metrics

A well-designed skill should:
- ✅ Activate automatically 90%+ of the time when needed
- ✅ Have <5% false positive rate
- ✅ Stay under 500 lines per file
- ✅ Load in <100ms (UserPromptSubmit) or <200ms (PreToolUse)
- ✅ Provide clear, actionable guidance
- ✅ Include working code examples
- ✅ Reference external resources efficiently
- ✅ Integrate with hooks and agents

## Related Skills

- **skill-developer** (from showcase) - Original meta-skill this builds upon
- **backend-dev-guidelines** - Example of modular domain skill
- **frontend-dev-guidelines** - Another modular skill example
- **error-tracking** - Simpler single-file skill example

---

**Skill Status**: Complete comprehensive meta-guide
**Line Count**: <500 lines following 500-line rule
**Progressive Disclosure**: 7 resource files for deep dives
**Coverage**: Full skill lifecycle from creation to production

**Next Steps**: Read [skill-creation.md](resources/skill-creation.md) to create your first skill, or [integration-guide.md](resources/integration-guide.md) to set up auto-activation.
