# Integration Guide - Claude Code Ecosystem

Complete guide for integrating skills with the Claude Code ecosystem including hooks, agents, commands, and the auto-activation system.

## Table of Contents

- [Hook System Architecture](#hook-system-architecture)
- [skill-rules.json Complete Schema](#skill-rulesjson-complete-schema)
- [Trigger Types Deep Dive](#trigger-types-deep-dive)
- [Session State Management](#session-state-management)
- [settings.json Configuration](#settingsjson-configuration)
- [Integration with Agents](#integration-with-agents)
- [Integration with Commands](#integration-with-commands)
- [Project Structure Adaptation](#project-structure-adaptation)

---

## Hook System Architecture

### The Two-Hook Auto-Activation System

Claude Code skills activate automatically through a two-hook architecture:

**1. UserPromptSubmit Hook** (Suggestion System)
- **File:** `.claude/hooks/skill-activation-prompt.ts` + `.sh` wrapper
- **Trigger:** BEFORE Claude sees user's prompt
- **Purpose:** Suggest relevant skills based on keywords + intent patterns
- **Method:** Injects formatted reminder as context (stdout → Claude's input)
- **Use Cases:** Topic-based skills, implicit work detection

**2. PreToolUse Hook** (Enforcement System)
- **File:** `.claude/hooks/skill-verification-guard.ts` + `.sh` wrapper
- **Trigger:** BEFORE Claude executes Edit/Write tools
- **Purpose:** Enforce critical guardrails, block risky operations
- **Method:** Exit code 2 + stderr → blocks tool, message to Claude
- **Use Cases:** Critical validations, data integrity, security

### Hook Execution Flow

**UserPromptSubmit:**
```
User submits prompt
    ↓
Hook registered in settings.json executes
    ↓
skill-activation-prompt.sh runs
    ↓
npx tsx skill-activation-prompt.ts
    ↓
Reads stdin JSON: {"prompt": "user's question"}
    ↓
Loads .claude/skills/skill-rules.json
    ↓
Matches keywords + intent patterns against prompt
    ↓
Groups matches by priority (critical → high → medium)
    ↓
Outputs formatted skill suggestion to stdout
    ↓
stdout becomes Claude's context (injected before user prompt)
    ↓
Claude sees: [skill suggestions] + user's original prompt
```

**PreToolUse:**
```
Claude calls Edit/Write tool
    ↓
Hook registered with matcher "Edit|Write" executes
    ↓
skill-verification-guard.sh runs
    ↓
npx tsx skill-verification-guard.ts
    ↓
Reads stdin JSON: {"tool_name": "Edit", "tool_input": {...}}
    ↓
Loads skill-rules.json
    ↓
Checks file path patterns (glob matching)
    ↓
Reads file content if exists (for content patterns)
    ↓
Checks session state (skill already used this session?)
    ↓
Checks skip conditions (file markers, env vars)
    ↓
IF BLOCKED:
  - Updates session state
  - Outputs block message to stderr
  - Exits with code 2
  - Tool execution PREVENTED
  - Claude sees stderr message
ELSE:
  - Exits with code 0
  - Tool executes normally
```

### Exit Code Behavior (CRITICAL)

| Exit Code | Hook Type | stdout | stderr | Tool Execution | Claude Sees |
|-----------|-----------|--------|--------|----------------|-------------|
| 0 | UserPromptSubmit | → Context | → User only | N/A | stdout content |
| 0 | PreToolUse | → User only | → User only | **Proceeds** | Nothing |
| 2 | PreToolUse | → User only | → **CLAUDE** | **BLOCKED** | stderr content |
| Other | Any | → User only | → User only | Blocked | Nothing |

**Why exit code 2 matters:**
- ONLY way to send message to Claude from PreToolUse
- stderr content "fed back to Claude automatically"
- Enables enforcement of critical guardrails
- Tool execution prevented until compliance

---

## skill-rules.json Complete Schema

### File Location and Structure

**Path:** `.claude/skills/skill-rules.json`

**Complete schema:**
```typescript
interface SkillRules {
    version: string;  // Currently "1.0"
    skills: Record<string, SkillRule>;
}

interface SkillRule {
    // Core settings
    type: 'guardrail' | 'domain';
    enforcement: 'block' | 'suggest' | 'warn';
    priority: 'critical' | 'high' | 'medium' | 'low';

    // Prompt-based triggers (UserPromptSubmit)
    promptTriggers?: {
        keywords?: string[];          // Case-insensitive substring matching
        intentPatterns?: string[];    // Regex patterns for intent detection
    };

    // File-based triggers (PreToolUse)
    fileTriggers?: {
        pathPatterns: string[];       // Glob patterns - REQUIRED if fileTriggers present
        pathExclusions?: string[];    // Glob patterns to exclude
        contentPatterns?: string[];   // Regex patterns for file content
        createOnly?: boolean;         // Only trigger on file creation, not edits
    };

    // Guardrail settings
    blockMessage?: string;  // REQUIRED for enforcement: "block"
                           // Use {file_path} placeholder

    // Skip conditions
    skipConditions?: {
        sessionSkillUsed?: boolean;   // Skip if skill used this session
        fileMarkers?: string[];       // Skip if file contains marker comment
        envOverride?: string;         // Environment variable to disable skill
    };
}
```

### Field Reference

**type:**
- `"guardrail"` - Critical enforcement, prevents mistakes
- `"domain"` - Advisory guidance, best practices

**enforcement:**
- `"block"` - Exit code 2 from PreToolUse, physically prevents tool execution
- `"suggest"` - Inject reminder via UserPromptSubmit, advisory only
- `"warn"` - Low-priority suggestion (rarely used)

**priority:**
- `"critical"` - Absolutely must-have, blocking
- `"high"` - Strongly recommended
- `"medium"` - Helpful optional guidance
- `"low"` - Nice-to-have suggestions

**promptTriggers.keywords:**
- Array of strings
- Case-insensitive substring matching
- Include variations: `["layout", "layouts", "layout system"]`
- Cover synonyms: `["API", "endpoint", "route"]`

**promptTriggers.intentPatterns:**
- Array of regex strings (compiled with 'i' flag for case-insensitive)
- Non-greedy matching recommended: `.*?` not `.*`
- Common pattern: `"(verb1|verb2|verb3).*?(noun1|noun2)"`
- Test at https://regex101.com/ with ECMAScript flavor

**fileTriggers.pathPatterns:**
- Glob patterns: `**` = any directories, `*` = any characters
- Examples: `"frontend/src/**/*.tsx"`, `"**/schema.prisma"`
- Be specific to reduce false positives

**fileTriggers.contentPatterns:**
- Regex patterns matching file contents
- Case-insensitive matching
- Escape special characters: `\\.findMany\\(` not `.findMany(`
- Match imports: `"import.*[Pp]risma"`

### Complete Examples

**Domain skill (advisory):**
```json
{
  "frontend-dev-guidelines": {
    "type": "domain",
    "enforcement": "suggest",
    "priority": "high",
    "promptTriggers": {
      "keywords": [
        "react", "component", "mui", "frontend",
        "styling", "tanstack", "routing"
      ],
      "intentPatterns": [
        "(create|add|build).*?(component|page|feature)",
        "(style|design).*?(button|form|layout)",
        "(how does|how do|explain).*?(react|component|routing)"
      ]
    },
    "fileTriggers": {
      "pathPatterns": [
        "frontend/src/**/*.tsx",
        "src/components/**/*.tsx"
      ],
      "pathExclusions": [
        "**/*.test.tsx",
        "**/*.spec.tsx"
      ]
    }
  }
}
```

**Guardrail skill (enforced):**
```json
{
  "database-verification": {
    "type": "guardrail",
    "enforcement": "block",
    "priority": "critical",
    "promptTriggers": {
      "keywords": ["prisma", "database", "table", "column", "schema"],
      "intentPatterns": [
        "(add|create|implement).*?(user|login|auth|tracking)",
        "(modify|update|change).*?(table|column|schema)"
      ]
    },
    "fileTriggers": {
      "pathPatterns": [
        "database/src/**/*.ts",
        "*/src/services/**/*.ts"
      ],
      "pathExclusions": [
        "**/*.test.ts"
      ],
      "contentPatterns": [
        "import.*[Pp]risma",
        "PrismaService",
        "\\.findMany\\(",
        "\\.create\\(",
        "\\.update\\(",
        "\\.delete\\("
      ]
    },
    "blockMessage": "⚠️ BLOCKED - Database Operation Detected\n\n📋 REQUIRED ACTION:\n1. Use Skill tool: 'database-verification'\n2. Verify ALL table and column names\n3. Check schema with DESCRIBE\n4. Then retry this edit\n\nReason: Prevent column name errors\nFile: {file_path}\n\n💡 TIP: Add '// @skip-validation' to skip future checks",
    "skipConditions": {
      "sessionSkillUsed": true,
      "fileMarkers": ["@skip-validation"],
      "envOverride": "SKIP_DB_VERIFICATION"
    }
  }
}
```

---

## Trigger Types Deep Dive

### 1. Keywords (Explicit Matching)

**How it works:** Case-insensitive substring search in user's prompt.

**Best practices:**
- Include common variations
- Cover full terms and abbreviations
- Add synonyms
- Be specific enough to avoid false matches

**Examples:**
```json
"keywords": [
  "layout",           // Matches: "layout", "layouts", "grid layout"
  "react",           // Matches: "react", "React", "REACT"
  "mui",             // Matches: "MUI", "material-ui"
  "component",       // Matches: "component", "components"
  "api",             // Matches: "API", "api", "APIs"
  "endpoint"         // Matches: "endpoint", "endpoints"
]
```

### 2. Intent Patterns (Implicit Detection)

**How it works:** Regex pattern matching to detect user's intention.

**Pattern structure:**
```
(action_verbs).*?(target_nouns)
```

**Common action verbs:**
- Create/Build: `(create|add|build|implement|make)`
- Modify: `(update|modify|change|edit|refactor)`
- Debug: `(fix|debug|troubleshoot|resolve)`
- Query: `(how does|how do|explain|what is|describe)`

**Best practices:**
- Use non-greedy matching: `.*?` not `.*`
- Test patterns at https://regex101.com/
- Start broad, refine to reduce false positives
- Group related actions: `(create|add|build)`

**Examples:**
```json
"intentPatterns": [
  "(create|add|build).*?(feature|endpoint|route|service)",
  "(how does|how do|explain).*?(layout|system|workflow)",
  "(modify|update|change).*?(table|column|schema)",
  "(debug|fix|troubleshoot).*?(error|bug|issue)"
]
```

### 3. File Path Patterns (Location-Based)

**How it works:** Glob pattern matching against file path being edited.

**Glob syntax:**
- `**` - Match any number of directories (including zero)
- `*` - Match any characters within a directory/filename
- `?` - Match single character

**Best practices:**
- Be specific: `src/api/**/*.ts` not `**/*.ts`
- Use exclusions for tests: `"pathExclusions": ["**/*.test.ts"]`
- Match multiple related paths

**Examples:**
```json
"pathPatterns": [
  "frontend/src/**/*.tsx",        // All React files in frontend/src
  "backend/src/controllers/**",   // All controller files
  "**/schema.prisma",             // Prisma schema anywhere
  "src/services/**/*.ts"          // All service files
],
"pathExclusions": [
  "**/*.test.ts",
  "**/*.test.tsx",
  "**/*.spec.ts"
]
```

### 4. Content Patterns (Technology Detection)

**How it works:** Regex matching against file contents.

**When to use:**
- Detect specific imports: `import.*Prisma`
- Match class patterns: `export class.*Controller`
- Find method calls: `\\.findMany\\(`
- Identify hooks: `useState|useEffect`

**Best practices:**
- Escape regex special characters: `\\.` not `.`
- Use character classes for case-insensitive: `[Pp]risma`
- Match actual usage, not comments/strings
- Keep patterns specific

**Examples:**
```json
"contentPatterns": [
  "import.*[Pp]risma",                    // Prisma imports
  "PrismaService",                        // Service class usage
  "prisma\\.",                            // prisma.method calls
  "\\.findMany\\(|\\.create\\(",         // Specific Prisma methods
  "export.*React\\.FC",                   // React components
  "useState|useEffect|useMemo",           // React hooks
  "router\\.|app\\.(get|post|put)"       // Express routes
]
```

---

## Session State Management

### Purpose

Prevent repeated nagging - once a skill is used (or blocked on), don't keep blocking in the same session.

### State File

**Location:** `.claude/hooks/state/skills-used-{session_id}.json`

**Structure:**
```json
{
  "skills_used": [
    "database-verification",
    "error-tracking"
  ],
  "files_verified": []
}
```

### How It Works

1. **First violation:**
   - Hook blocks with exit code 2
   - Updates session state: adds skill to `skills_used` array
   - Claude sees block message, uses skill

2. **Subsequent edits (same session):**
   - Hook checks session state
   - Finds skill in `skills_used` array
   - Exits with code 0 (allow)
   - No message to Claude

3. **Different session:**
   - New session ID = new state file
   - Hook blocks again

### Configuration

Enable session tracking in skill-rules.json:
```json
"skipConditions": {
  "sessionSkillUsed": true
}
```

### Manual State Management

**View session state:**
```bash
cat .claude/hooks/state/skills-used-*.json
```

**Clear session state (force re-trigger):**
```bash
rm .claude/hooks/state/skills-used-{session-id}.json
```

---

## settings.json Configuration

### Hook Registration

**Location:** `.claude/settings.json`

**UserPromptSubmit hook:**
```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/skill-activation-prompt.sh"
          }
        ]
      }
    ]
  }
}
```

**PreToolUse hook:**
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/skill-verification-guard.sh"
          }
        ]
      }
    ]
  }
}
```

**PostToolUse hook (optional):**
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|MultiEdit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/post-tool-use-tracker.sh"
          }
        ]
      }
    ]
  }
}
```

### Complete settings.json Example

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/skill-activation-prompt.sh"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/skill-verification-guard.sh"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|MultiEdit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/post-tool-use-tracker.sh"
          }
        ]
      }
    ]
  }
}
```

---

## Integration with Agents

### How Skills Complement Agents

**Skills** - Domain knowledge loaded into Claude's context
**Agents** - Specialized task executors that run autonomously

**Synergy:**
- Agents can reference skills for domain expertise
- Skills guide agents toward best practices
- Agents execute complex workflows using skill patterns

### Example: Code Architecture Reviewer + Backend Skill

**Workflow:**
1. User: "Review this backend code"
2. Skill activates: backend-dev-guidelines
3. Agent launches: code-architecture-reviewer
4. Agent references skill patterns during review
5. Agent produces review aligned with skill standards

### Agent-Skill Communication

Agents can explicitly load skills:
```markdown
In the agent instructions:

When reviewing backend code:
1. Use the Skill tool to load 'backend-dev-guidelines'
2. Apply the layered architecture principles
3. Check for BaseController usage
4. Verify error handling with Sentry
```

---

## Integration with Commands

### Slash Commands + Skills

**Commands** trigger workflows, **skills** provide knowledge.

**Example: /dev-docs + backend-dev-guidelines**

```markdown
/dev-docs command:
1. Analyzes current task
2. Activates backend-dev-guidelines skill (via triggers)
3. Generates documentation using skill patterns
4. Outputs structured dev docs
```

### Creating Skill-Aware Commands

**Pattern:**
```markdown
---
name: custom-command
---

# Custom Command

When executed:
1. Use Skill tool to load relevant skill
2. Apply skill principles to the task
3. Execute command workflow
4. Verify output matches skill standards
```

---

## Project Structure Adaptation

### Adapting Skills to Your Project

**Problem:** Example skills use specific paths (frontend/, backend/, etc.)

**Solution:** Update pathPatterns in skill-rules.json.

### Common Project Structures

**Monorepo with workspaces:**
```json
"pathPatterns": [
  "packages/*/src/**/*.ts",
  "apps/*/src/**/*.tsx"
]
```

**Nx monorepo:**
```json
"pathPatterns": [
  "apps/api/src/**/*.ts",
  "libs/*/src/**/*.ts"
]
```

**Simple structure:**
```json
"pathPatterns": [
  "src/**/*.ts"
]
```

**Multi-service (like showcase):**
```json
"pathPatterns": [
  "blog-api/src/**/*.ts",
  "auth-service/src/**/*.ts"
]
```

### Customization Checklist

When integrating a skill:
- [ ] Update pathPatterns for your project structure
- [ ] Update contentPatterns if using different libraries
- [ ] Test triggers with actual file paths
- [ ] Verify no hardcoded paths in SKILL.md
- [ ] Update examples to match your tech stack

---

**Related files:**
- [skill-creation.md](skill-creation.md) - Creating skills
- [skill-verification.md](skill-verification.md) - Testing triggers
- [best-practices.md](best-practices.md) - Pattern optimization
- [troubleshooting.md](troubleshooting.md) - Debugging activation
