# Best Practices - Patterns and Anti-Patterns

Synthesized wisdom from production experience and Anthropic recommendations.

## Table of Contents

- [Progressive Disclosure Strategies](#progressive-disclosure-strategies)
- [Trigger Pattern Library](#trigger-pattern-library)
- [Common Pitfalls and Solutions](#common-pitfalls-and-solutions)
- [Performance Optimization](#performance-optimization)
- [Maintenance and Versioning](#maintenance-and-versioning)
- [Writing Effective Documentation](#writing-effective-documentation)

---

## Progressive Disclosure Strategies

### The 500-Line Rule (Production-Tested)

**Problem:** Large skills (>500 lines) cause:
- Context limit issues
- Slow responses
- Information overload
- Difficult maintenance

**Solution:** Modular structure with progressive loading.

### Three-Tier Pattern

**Tier 1: Metadata** (always loaded)
```yaml
---
name: skill-name
description: Comprehensive description with ALL triggers (max 1024 chars)
---
```
~100 words, always in context.

**Tier 2: SKILL.md** (loaded when activated)
```markdown
# Skill Name

## Purpose (50 words)
## When to Use (100 words)
## Quick Reference (200 words)
## Navigation to Resources (100 words)
```
~450 lines max, loaded on activation.

**Tier 3: Resource Files** (loaded as needed)
```
resources/
  topic-1.md    # Loaded only when Claude needs deep dive on topic 1
  topic-2.md    # Loaded only when needed
```
<500 lines each, loaded progressively.

### When to Split Content

**Keep in SKILL.md:**
- Overview and purpose
- When to use (activation scenarios)
- Quick start checklist
- Common patterns (condensed)
- Navigation guide

**Move to resource files:**
- Detailed explanations (>100 lines on one topic)
- Complete examples
- Advanced topics
- Edge cases and troubleshooting
- Comprehensive API references

### Real Example: backend-dev-guidelines

**Main file (304 lines):**
- Architecture overview (diagram + 50 words)
- Core principles (7 rules, 1 sentence each)
- Quick start checklist
- Navigation table to 11 resource files

**Resource files (each <500 lines):**
- architecture-overview.md - Detailed layered architecture
- routing-and-controllers.md - Complete route/controller patterns
- services-and-repositories.md - Service layer deep dive
- validation-patterns.md - Zod validation examples
- ... 7 more specialized files

**Result:** User loads only what they need, stays under context limits.

---

## Trigger Pattern Library

### Keyword Patterns (70% Weight - Primary Source)

**Generic Technology Keywords:**
```json
["react", "typescript", "node", "express", "prisma"]
```

**Specific Features:**
```json
["layout", "grid", "workflow", "authentication", "validation"]
```

**Development Actions:**
```json
["create", "build", "implement", "refactor", "debug"]
```

**Best Practices:**
- Include both singular and plural: `["component", "components"]`
- Cover abbreviations: `["API", "api"]`
- Add common variations: `["route", "routes", "routing", "router"]`

### Intent Pattern Library

**Feature Creation:**
```regex
(create|add|build|implement).*?(feature|endpoint|route|service|component)
```

**Explanations:**
```regex
(how does|how do|explain|what is|describe|tell me about).*?
```

**Modifications:**
```regex
(modify|update|change|refactor|improve).*?(table|column|component|service)
```

**Debugging:**
```regex
(fix|debug|troubleshoot|resolve).*?(error|bug|issue|problem)
```

**Database Work:**
```regex
(add|create|modify).*?(user|table|column|schema|migration)
database.*?(change|update|query)
```

**UI Work:**
```regex
(style|design|create).*?(button|form|modal|dialog|layout)
(add|update).*?(styling|theme|css)
```

### File Path Pattern Library

**Frontend:**
```json
[
  "frontend/src/**/*.tsx",
  "src/components/**/*.tsx",
  "app/**/*.tsx"
]
```

**Backend:**
```json
[
  "backend/src/**/*.ts",
  "server/**/*.ts",
  "api/**/*.ts"
]
```

**Database:**
```json
[
  "**/schema.prisma",
  "**/migrations/**/*.sql",
  "database/**/*.ts"
]
```

**Exclusions (always include):**
```json
"pathExclusions": [
  "**/*.test.ts",
  "**/*.test.tsx",
  "**/*.spec.ts",
  "**/node_modules/**"
]
```

### Content Pattern Library

**Prisma:**
```json
[
  "import.*[Pp]risma",
  "PrismaService",
  "prisma\\.",
  "\\.findMany\\(|\\.create\\(|\\.update\\("
]
```

**Express:**
```json
[
  "express\\.Router",
  "app\\.(get|post|put|delete|patch)",
  "router\\."
]
```

**React:**
```json
[
  "export.*React\\.FC",
  "useState|useEffect|useMemo|useCallback",
  "import.*from 'react'"
]
```

---

## Common Pitfalls and Solutions

### Pitfall 1: Vague Descriptions

**Bad:**
```yaml
description: Helps with backend development stuff and things
```

**Good:**
```yaml
description: Express/Prisma/TypeScript backend patterns. Use when creating routes, controllers, services, or working with database queries, error tracking with Sentry, and Zod validation. Covers layered architecture, dependency injection, async patterns.
```

**Why:** Description is THE discovery mechanism. Include all keywords.

### Pitfall 2: Too-Generic Keywords

**Bad:**
```json
"keywords": ["system", "code", "work", "create"]
```
False positives: "file system", "create directory"

**Good:**
```json
"keywords": [
  "backend service",
  "express routes",
  "prisma queries",
  "database schema"
]
```

### Pitfall 3: Over-Broad Intent Patterns

**Bad:**
```json
"intentPatterns": ["(create)"]
```
Matches EVERYTHING with "create"

**Good:**
```json
"intentPatterns": [
  "(create|add).*?(backend|api|service|endpoint)"
]
```
Context makes it specific

### Pitfall 4: Missing Test Files Exclusion

**Bad:**
```json
"pathPatterns": ["src/**/*.ts"]
```
Triggers on test files too!

**Good:**
```json
"pathPatterns": ["src/**/*.ts"],
"pathExclusions": ["**/*.test.ts", "**/*.spec.ts"]
```

### Pitfall 5: Forgetting Progressive Disclosure

**Bad:** 1200-line SKILL.md file

**Good:**
```
SKILL.md (450 lines)
resources/
  topic-1.md (400 lines)
  topic-2.md (380 lines)
```

### Pitfall 6: No Session Tracking

**Bad:**
```json
{
  "enforcement": "block"
  // No skipConditions
}
```
Blocks EVERY edit forever!

**Good:**
```json
{
  "enforcement": "block",
  "skipConditions": {
    "sessionSkillUsed": true
  }
}
```
Blocks once per session.

---

## Performance Optimization

### Measurement

**Target metrics:**
- UserPromptSubmit: <100ms
- PreToolUse: <200ms

**Measure:**
```bash
time echo '{"prompt":"test"}' | npx tsx .claude/hooks/skill-activation-prompt.ts
```

### Optimization Strategies

**1. Reduce pattern count**
```json
// Bad: 50 keywords, 20 intent patterns
"keywords": [...50 items...],
"intentPatterns": [...20 patterns...]

// Good: 15 carefully chosen keywords, 5 focused patterns
"keywords": ["react", "component", "mui", "frontend", ...],
"intentPatterns": [
  "(create|add).*?(component|page)",
  "(style).*?(button|form)"
]
```

**2. Simplify regex**
```json
// Bad: Complex alternation
"(create|add|build|implement|make|construct|develop).*?(feature|endpoint|route|service|controller|component|UI|page|modal)"

// Good: Essential only
"(create|add|build).*?(feature|endpoint|component)"
```

**3. Narrow file paths**
```json
// Bad: Checks ALL TypeScript files
"pathPatterns": ["**/*.ts"]

// Good: Specific directories only
"pathPatterns": ["src/api/**/*.ts", "src/services/**/*.ts"]
```

**4. Minimize content patterns**
```json
// Only use when necessary
"contentPatterns": [
  "import.*Prisma"  // Just check imports, not all usage
]
```

---

## Maintenance and Versioning

### Tracking Changes

**Use version control:**
```bash
# Track skill-rules.json changes
git log -p .claude/skills/skill-rules.json

# Review trigger changes before commit
git diff .claude/skills/skill-rules.json
```

**Document changes:**
```markdown
## Changelog

### 2025-01-15
- Added "component" keyword for better React detection
- Refined intent pattern to reduce false positives
- Excluded test files from path patterns

### 2025-01-10
- Initial release
```

### Versioning Strategy

**Option 1: In SKILL.md**
```markdown
---
**Skill Version**: 1.2.0
**Last Updated**: 2025-01-15
**Changelog**: See bottom of file
---
```

**Option 2: Separate CHANGELOG.md**
```
skill-name/
  SKILL.md
  CHANGELOG.md
  resources/
```

### Regression Testing

**Before deploying changes:**
```bash
# Save old version
cp skill-rules.json skill-rules.json.backup

# Make changes
# ...

# Test new version
./test-all-evaluations.sh

# Compare results
diff old-results.txt new-results.txt

# If worse, revert
mv skill-rules.json.backup skill-rules.json
```

---

## Writing Effective Documentation

### Anthropic-Recommended Patterns (30% Weight)

**Use imperative form:**
```markdown
✅ Create the service file
✅ Configure the router
✅ Add error handling

❌ You should create the service file
❌ The developer needs to configure
```

**Be concise and objective:**
```markdown
✅ This pattern prevents race conditions

❌ This is an absolutely amazing pattern that will totally prevent all race conditions forever!
```

**One concept per section:**
```markdown
✅ ## Error Handling
    (just error handling content)

❌ ## Error Handling and Logging and Monitoring
    (too broad)
```

### Production-Tested Patterns (70% Weight)

**Use checklists for workflows:**
```markdown
## New Feature Checklist

- [ ] Create route
- [ ] Add controller
- [ ] Implement service
- [ ] Write tests
```

**Show DO/DON'T examples:**
```typescript
// ✅ DO: Use BaseController
export class UserController extends BaseController {
  // ...
}

// ❌ DON'T: Direct response handling
router.post('/users', async (req, res) => {
  res.json(data);  // Missing error handling
});
```

**Include navigation tables:**
```markdown
| Need to... | Read this |
|------------|-----------|
| Task 1 | [file-1.md](file-1.md) |
| Task 2 | [file-2.md](file-2.md) |
```

**Add quick reference sections:**
```markdown
## Quick Reference: HTTP Status Codes

| Code | Use Case |
|------|----------|
| 200  | Success  |
| 400  | Bad Request |
| 500  | Server Error |
```

---

## Summary: Essential Best Practices

**Creating Skills:**
1. ✅ Evaluation-first approach (test before documenting)
2. ✅ Rich descriptions with ALL keywords
3. ✅ Progressive disclosure (500-line rule)
4. ✅ Multiple trigger types (keywords + intents + paths)
5. ✅ Session tracking for guardrails

**Triggers:**
6. ✅ Specific keywords, avoid generic terms
7. ✅ Contextual intent patterns
8. ✅ Narrow file path patterns
9. ✅ Exclude test files
10. ✅ Test patterns before deployment

**Performance:**
11. ✅ Keep pattern count reasonable (<20)
12. ✅ Simplify complex regex
13. ✅ Measure execution time
14. ✅ Optimize if >100ms (UserPrompt) or >200ms (PreTool)

**Documentation:**
15. ✅ Imperative instructions
16. ✅ DO/DON'T examples
17. ✅ Checklists and tables
18. ✅ Clear navigation

**Maintenance:**
19. ✅ Version control skill-rules.json
20. ✅ Test before deploying changes
21. ✅ Monitor false positive/negative rates
22. ✅ Iterate based on usage data

---

**Related files:**
- [skill-creation.md](skill-creation.md) - Step-by-step creation guide
- [skill-verification.md](skill-verification.md) - Testing strategies
- [integration-guide.md](integration-guide.md) - Hook system details
- [troubleshooting.md](troubleshooting.md) - Solving activation issues
