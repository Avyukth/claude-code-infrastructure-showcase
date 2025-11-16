# Troubleshooting - Debugging Skill Activation Issues

Comprehensive guide for diagnosing and solving skill activation problems.

## Table of Contents

- [Skill Not Activating](#skill-not-activating)
- [False Positives](#false-positives)
- [False Negatives](#false-negatives)
- [Hook Execution Failures](#hook-execution-failures)
- [Performance Issues](#performance-issues)
- [Configuration Errors](#configuration-errors)

---

## Skill Not Activating

### Symptom: Skill Never Suggests

**Check 1: Hook installed and executable**
```bash
ls -la .claude/hooks/skill-activation-prompt.sh
# Should show -rwxr-xr-x (executable)

# If not executable:
chmod +x .claude/hooks/skill-activation-prompt.sh
```

**Check 2: Hook registered in settings.json**
```bash
cat .claude/settings.json | jq '.hooks.UserPromptSubmit'
# Should show hook entry
```

**Check 3: skill-rules.json exists and is valid**
```bash
cat .claude/skills/skill-rules.json | jq .
# Should parse without errors
```

**Check 4: Skill name matches**
```bash
# In SKILL.md frontmatter
grep "^name:" .claude/skills/my-skill/SKILL.md

# In skill-rules.json
jq '.skills | keys' .claude/skills/skill-rules.json

# Names must match EXACTLY
```

**Check 5: Keywords match prompt**
```bash
# Test manually
echo '{"prompt":"your test prompt"}' | \
  npx tsx .claude/hooks/skill-activation-prompt.ts

# Expected: Skill should appear in output
```

### Symptom: File Editing Doesn't Trigger

**Check 1: PreToolUse hook installed**
```bash
ls -la .claude/hooks/skill-verification-guard.sh
chmod +x .claude/hooks/skill-verification-guard.sh
```

**Check 2: File path matches pattern**
```bash
# Test the pattern
cd $CLAUDE_PROJECT_DIR
find . -path "./src/api/**/*.ts" | head -5

# Should match your actual file structure
```

**Check 3: Content pattern matches**
```bash
# Check if file contains pattern
grep -i "prisma" path/to/file.ts

# Should find matches if pattern is import.*[Pp]risma
```

**Check 4: Not in exclusions**
```bash
# Check if file matches exclusion pattern
# If editing src/api/user.test.ts
# And pathExclusions includes "**/*.test.ts"
# Then it won't trigger (by design)
```

---

## False Positives

### Symptom: Skill Activates When It Shouldn't

**Problem 1: Keywords too generic**
```json
// Bad
"keywords": ["user", "create", "system"]

// Matches: "user manual", "create directory", "file system"
```

**Solution:**
```json
// Good
"keywords": ["user authentication", "create backend", "workflow system"]
```

**Problem 2: Intent patterns too broad**
```json
// Bad
"intentPatterns": ["(create)"]

// Matches everything with "create"
```

**Solution:**
```json
// Good
"intentPatterns": ["(create|add).*?(backend|api|service)"]
```

**Problem 3: File paths too broad**
```json
// Bad
"pathPatterns": ["**/*.ts"]

// Triggers on ALL TypeScript files
```

**Solution:**
```json
// Good
"pathPatterns": [
  "src/api/**/*.ts",
  "backend/services/**/*.ts"
],
"pathExclusions": ["**/*.test.ts"]
```

---

## False Negatives

### Symptom: Should Activate But Doesn't

**Cause 1: Missing keyword variation**
```json
// Only has singular
"keywords": ["component"]

// User prompt: "How do components work?"
// Doesn't match "components" (plural)
```

**Solution:**
```json
"keywords": ["component", "components", "React component"]
```

**Cause 2: Intent pattern too specific**
```json
// Too specific
"intentPatterns": ["(create).*?(backend).*?(service)"]

// User prompt: "create a service" (missing "backend")
// Doesn't match
```

**Solution:**
```json
"intentPatterns": [
  "(create|add).*?(service|backend)",
  "backend.*?service"
]
```

**Cause 3: Typo in pattern**
```json
// Typo: "compnent"
"keywords": ["compnent"]

// Should be:
"keywords": ["component"]
```

---

## Hook Execution Failures

### Symptom: Hook Doesn't Run At All

**Check 1: Dependencies installed**
```bash
cd .claude/hooks
ls node_modules/
# Should show packages

# If missing:
npm install
```

**Check 2: TypeScript compiles**
```bash
cd .claude/hooks
npx tsc --noEmit skill-activation-prompt.ts
# Should show no errors
```

**Check 3: Correct shebang**
```bash
head -1 .claude/hooks/skill-activation-prompt.sh
# Should be: #!/bin/bash
```

**Check 4: npx/tsx available**
```bash
which npx
npx tsx --version
# Should show version
```

---

## Performance Issues

### Symptom: Hooks Are Slow (>100ms)

**Measure first:**
```bash
time echo '{"prompt":"test"}' | \
  npx tsx .claude/hooks/skill-activation-prompt.ts
```

**If slow, diagnose:**

**Check 1: Too many patterns**
```bash
# Count patterns
jq '[.skills[] | .promptTriggers.keywords // [] | length] | add' \
  .claude/skills/skill-rules.json

# If >50, consider reducing
```

**Check 2: Complex regex**
```json
// Slow
"(create|add|build|implement|make|construct|develop|generate).*?(feature|endpoint|route|service|controller|component|UI|page|modal|dialog|form)"

// Faster
"(create|add|build).*?(feature|endpoint|component)"
```

**Check 3: Broad file paths**
```json
// Slow (checks all files)
"pathPatterns": ["**/*.ts"]

// Faster (specific directories)
"pathPatterns": ["src/api/**/*.ts"]
```

---

## Configuration Errors

### JSON Syntax Errors

**Symptom:** Hook fails silently or with parse error.

**Check:**
```bash
jq . .claude/skills/skill-rules.json
# Shows syntax error if invalid
```

**Common errors:**
```json
// Trailing comma (invalid JSON)
{
  "keywords": ["one", "two",]
}

// Single quotes (invalid JSON)
{
  'keywords': ['one']
}

// Missing quotes on key
{
  keywords: ["one"]
}
```

### Regex Pattern Errors

**Symptom:** Pattern doesn't match as expected.

**Test pattern:**
1. Copy pattern from skill-rules.json
2. Go to https://regex101.com/
3. Select "ECMAScript (JavaScript)" flavor
4. Test against sample prompts
5. Fix pattern, update skill-rules.json

---

## Diagnostic Commands

**Test prompt trigger:**
```bash
echo '{"session_id":"test","prompt":"YOUR PROMPT HERE"}' | \
  npx tsx .claude/hooks/skill-activation-prompt.ts
```

**Test file trigger:**
```bash
cat <<'EOF' | npx tsx .claude/hooks/skill-verification-guard.ts
{"session_id":"test","tool_name":"Edit","tool_input":{"file_path":"/full/path/to/file.ts"}}
EOF
echo "Exit code: $?"
```

**Validate JSON:**
```bash
jq . .claude/skills/skill-rules.json
```

**Check hook permissions:**
```bash
ls -la .claude/hooks/*.sh
```

**View session state:**
```bash
cat .claude/hooks/state/skills-used-*.json
```

---

**Related files:**
- [skill-verification.md](skill-verification.md) - Testing methods
- [integration-guide.md](integration-guide.md) - Hook system details
- [best-practices.md](best-practices.md) - Avoiding common pitfalls
