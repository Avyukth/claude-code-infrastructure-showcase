# Skill Verification - Testing and Validation Guide

Comprehensive guide for testing, validating, and ensuring quality of Claude Code skills.

## Table of Contents

- [Testing Philosophy](#testing-philosophy)
- [Manual Testing Methods](#manual-testing-methods)
- [Automated Evaluation Framework](#automated-evaluation-framework)
- [Test-Driven Development for Skills](#test-driven-development-for-skills)
- [Verification Workflows](#verification-workflows)
- [Performance Testing](#performance-testing)
- [Regression Testing](#regression-testing)
- [Quality Checklist](#quality-checklist)

---

## Testing Philosophy

### Evaluation-First Approach

**Core principle from Anthropic:** Build evaluations BEFORE extensive documentation.

**Process:**
1. Identify 3-10 representative tasks from real work
2. Test Claude on these tasks WITHOUT the skill
3. Document specific failure modes
4. Build skill incrementally to address failures
5. Re-test to verify improvement
6. Iterate until success rate is acceptable

**Why it works:**
- Ensures skill solves actual problems
- Prevents over-documentation of unused features
- Provides objective success metrics
- Enables data-driven refinement

### Success Metrics

A well-tested skill should demonstrate:
- ✅ **90%+ activation rate** when needed (test with manual prompts)
- ✅ **<5% false positive rate** (doesn't activate incorrectly)
- ✅ **<100ms UserPromptSubmit** execution time
- ✅ **<200ms PreToolUse** execution time
- ✅ **100% JSON validity** in skill-rules.json
- ✅ **Clear, actionable guidance** verified with users
- ✅ **Working code examples** tested in actual environment

---

## Manual Testing Methods

### Test 1: Hook Execution Verification

**Purpose:** Verify hooks are installed and executable.

```bash
# Check hook files exist
ls -la $CLAUDE_PROJECT_DIR/.claude/hooks/*.sh
ls -la $CLAUDE_PROJECT_DIR/.claude/hooks/*.ts

# Verify executable permission (should show -rwxr-xr-x)
ls -l $CLAUDE_PROJECT_DIR/.claude/hooks/skill-activation-prompt.sh

# Make executable if needed
chmod +x $CLAUDE_PROJECT_DIR/.claude/hooks/*.sh
```

**Expected:** Files exist with execute permissions.

### Test 2: UserPromptSubmit Trigger Testing

**Purpose:** Verify prompt-based triggers activate correctly.

**Test keywords:**
```bash
echo '{"session_id":"test","prompt":"help with backend development"}' | \
  npx tsx $CLAUDE_PROJECT_DIR/.claude/hooks/skill-activation-prompt.ts
```

**Expected output:** Skill suggestion appears if keywords match.

**Test intent patterns:**
```bash
echo '{"session_id":"test","prompt":"create a new user feature"}' | \
  npx tsx $CLAUDE_PROJECT_DIR/.claude/hooks/skill-activation-prompt.ts
```

**Expected:** Intent pattern `(create).*?(feature)` matches.

**Test variations:**
```bash
# Test multiple phrasings
for prompt in \
  "building a component" \
  "add new endpoint" \
  "implement authentication" \
  "how does routing work"
do
  echo "Testing: $prompt"
  echo "{\"prompt\":\"$prompt\"}" | \
    npx tsx .claude/hooks/skill-activation-prompt.ts | \
    grep -q "RECOMMENDED SKILLS" && echo "✓ Activated" || echo "✗ Not activated"
done
```

### Test 3: PreToolUse File Trigger Testing

**Purpose:** Verify file-based triggers work correctly.

**Test path patterns:**
```bash
cat <<'EOF' | npx tsx $CLAUDE_PROJECT_DIR/.claude/hooks/skill-verification-guard.ts
{
  "session_id": "test",
  "tool_name": "Edit",
  "tool_input": {
    "file_path": "/path/to/your/src/component.tsx"
  }
}
EOF
echo "Exit code: $?"
```

**Expected:**
- Exit code 0 = allowed (no block)
- Exit code 2 = blocked (stderr message shown)

**Test content patterns:**
```bash
# Create test file with matching content
echo "import { PrismaService } from './prisma'" > /tmp/test-file.ts

# Test against this file
cat <<EOF | npx tsx .claude/hooks/skill-verification-guard.ts
{
  "tool_name": "Edit",
  "tool_input": {"file_path": "/tmp/test-file.ts"}
}
EOF
```

**Expected:** Content pattern `import.*[Pp]risma` matches.

### Test 4: JSON Syntax Validation

**Purpose:** Ensure skill-rules.json is valid JSON.

```bash
# Validate JSON syntax
cat $CLAUDE_PROJECT_DIR/.claude/skills/skill-rules.json | jq .

# Check specific skill entry
cat $CLAUDE_PROJECT_DIR/.claude/skills/skill-rules.json | jq '.skills["my-skill-name"]'

# Pretty-print for review
jq . $CLAUDE_PROJECT_DIR/.claude/skills/skill-rules.json
```

**Expected:** No errors, clean JSON output.

### Test 5: Regex Pattern Validation

**Purpose:** Verify regex patterns are valid and match correctly.

**Using regex101.com:**
1. Copy intent pattern from skill-rules.json
2. Paste into https://regex101.com/
3. Select "ECMAScript (JavaScript)" flavor
4. Test against sample prompts
5. Verify matches and no errors

**Using command line:**
```bash
# Test regex pattern (requires grep with -P or use node)
node -e "
const pattern = /(create|add).*?(feature|component)/i;
const tests = [
  'create a new feature',
  'add component to page',
  'building something'
];
tests.forEach(t => {
  console.log(t, pattern.test(t) ? '✓' : '✗');
});
"
```

### Test 6: Glob Pattern Validation

**Purpose:** Verify file path patterns match intended files.

```bash
# Test glob patterns using find
cd $CLAUDE_PROJECT_DIR

# Pattern: frontend/src/**/*.tsx
find frontend/src -name "*.tsx" | head -5

# Pattern: **/schema.prisma
find . -name "schema.prisma"

# Pattern with exclusions
find src -name "*.ts" ! -path "*/test/*" ! -name "*.test.ts"
```

**Expected:** Patterns match intended files, exclusions work correctly.

### Test 7: Real-World Activation Test

**Purpose:** Verify skill activates in actual Claude Code session.

**Process:**
1. Start Claude Code session
2. Type prompt that should trigger skill
3. Observe if skill is suggested
4. Edit file that should trigger skill
5. Observe if skill activates or blocks

**Example prompts to test:**
- "Help me create a new backend service"
- "How do I style this React component?"
- "Add error tracking to this function"

---

## Automated Evaluation Framework

### Evaluation Set Structure

Create a JSON file with test cases:

```json
{
  "skill_name": "backend-dev-guidelines",
  "evaluations": [
    {
      "id": "eval-001",
      "prompt": "create a new Express route",
      "expected_activation": true,
      "category": "route-creation"
    },
    {
      "id": "eval-002",
      "prompt": "update README file",
      "expected_activation": false,
      "category": "false-positive-check"
    },
    {
      "id": "eval-003",
      "file_path": "src/controllers/UserController.ts",
      "file_content": "export class UserController extends BaseController",
      "expected_activation": true,
      "category": "file-trigger"
    }
  ]
}
```

### Evaluation Script

**Purpose:** Automate testing of multiple scenarios.

```python
#!/usr/bin/env python3
"""Automated skill evaluation script."""

import json
import subprocess
import sys
from pathlib import Path

def test_prompt_trigger(prompt: str) -> bool:
    """Test if prompt triggers skill activation."""
    input_json = json.dumps({"session_id": "eval", "prompt": prompt})
    result = subprocess.run(
        ["npx", "tsx", ".claude/hooks/skill-activation-prompt.ts"],
        input=input_json,
        capture_output=True,
        text=True
    )
    return "RECOMMENDED SKILLS" in result.stdout

def test_file_trigger(file_path: str) -> tuple[bool, int]:
    """Test if file editing triggers skill."""
    input_json = json.dumps({
        "session_id": "eval",
        "tool_name": "Edit",
        "tool_input": {"file_path": file_path}
    })
    result = subprocess.run(
        ["npx", "tsx", ".claude/hooks/skill-verification-guard.ts"],
        input=input_json,
        capture_output=True,
        text=True
    )
    return result.returncode == 2, result.returncode

def run_evaluations(eval_file: str):
    """Run all evaluations and report results."""
    with open(eval_file) as f:
        data = json.load(f)

    results = {"passed": 0, "failed": 0, "total": 0}

    for eval_case in data["evaluations"]:
        results["total"] += 1
        eval_id = eval_case["id"]

        if "prompt" in eval_case:
            activated = test_prompt_trigger(eval_case["prompt"])
            expected = eval_case["expected_activation"]

            if activated == expected:
                print(f"✓ {eval_id}: PASS")
                results["passed"] += 1
            else:
                print(f"✗ {eval_id}: FAIL (expected {expected}, got {activated})")
                results["failed"] += 1

        elif "file_path" in eval_case:
            blocked, exit_code = test_file_trigger(eval_case["file_path"])
            expected = eval_case["expected_activation"]

            if blocked == expected:
                print(f"✓ {eval_id}: PASS")
                results["passed"] += 1
            else:
                print(f"✗ {eval_id}: FAIL (expected {expected}, got {blocked})")
                results["failed"] += 1

    print(f"\n{'='*50}")
    print(f"Results: {results['passed']}/{results['total']} passed")
    print(f"Success rate: {100*results['passed']/results['total']:.1f}%")

    return results["failed"] == 0

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Usage: ./evaluate_skill.py <eval-file.json>")
        sys.exit(1)

    success = run_evaluations(sys.argv[1])
    sys.exit(0 if success else 1)
```

**Usage:**
```bash
# Run evaluations
./evaluate_skill.py skill-evaluations.json

# Run in CI/CD
./evaluate_skill.py skill-evaluations.json || exit 1
```

---

## Test-Driven Development for Skills

### TDD Cycle for Skills

**Recommended by Anthropic for skills with verifiable outputs.**

**Process:**

1. **Write evaluations first**
   - Define 3-10 test cases
   - Specify expected input/output pairs
   - Document success criteria

2. **Run tests (expect failures)**
   - Test Claude without the skill
   - Document specific failures
   - Identify gaps in capability

3. **Create minimal skill**
   - Write just enough to pass first test
   - Focus on one scenario at a time
   - Don't over-document

4. **Run tests again**
   - Verify improvement
   - Identify remaining failures
   - Iterate until all pass

5. **Refactor and expand**
   - Add resource files if needed
   - Improve trigger patterns
   - Add more test cases

### Example TDD Workflow

**Step 1: Define test cases**
```json
{
  "test_cases": [
    {
      "input": "create Express route for user login",
      "expected": "skill activates, provides BaseController pattern"
    },
    {
      "input": "add error handling to service",
      "expected": "skill activates, shows Sentry integration"
    }
  ]
}
```

**Step 2: Test without skill (baseline)**
```bash
# Document Claude's response without skill
# Save to baseline.txt
```

**Step 3: Create minimal skill**
```markdown
---
name: backend-dev
description: Express route and error handling patterns
---

# Backend Development

## Routes
Use BaseController pattern...

## Error Handling
Integrate Sentry...
```

**Step 4: Test with skill**
```bash
# Activate skill and test same inputs
# Compare to baseline
```

**Step 5: Iterate**
- Add missing keywords
- Refine guidance
- Add examples
- Re-test until 90%+ success

---

## Verification Workflows

### Plan-Validate-Execute Pattern

**Purpose:** Catch errors early by validating plans before execution.

**Workflow:**

1. **Analyze** - Claude examines the task
2. **Create plan** - Claude writes structured plan to file
3. **Validate plan** - Script checks plan for issues
4. **Execute** - Claude implements the plan
5. **Verify** - Claude provides evidence of success

**Example validation script:**
```python
#!/usr/bin/env python3
"""Validate skill development plan."""

import json
import sys

def validate_plan(plan_file: str) -> bool:
    """Check plan completeness."""
    with open(plan_file) as f:
        plan = json.load(f)

    required_fields = ["skill_name", "description", "triggers", "tests"]
    missing = [f for f in required_fields if f not in plan]

    if missing:
        print(f"❌ Missing required fields: {missing}")
        return False

    if len(plan["description"]) > 1024:
        print(f"❌ Description too long: {len(plan['description'])} > 1024")
        return False

    if not plan["triggers"].get("keywords"):
        print("❌ No keywords defined")
        return False

    print("✅ Plan is valid")
    return True

if __name__ == "__main__":
    success = validate_plan(sys.argv[1])
    sys.exit(0 if success else 1)
```

**Usage in skill creation:**
1. Ask Claude to create plan: "Create a plan for the new skill in plan.json"
2. Validate: `./validate_plan.py plan.json`
3. If valid, proceed: "Implement the plan"
4. If invalid, fix: "Fix the plan based on validation errors"

### Verification-Before-Completion

**Purpose:** Require concrete evidence that skill works.

**Pattern:**
```markdown
Before marking this task complete:
1. Run test command: `npx tsx .claude/hooks/skill-activation-prompt.ts`
2. Capture output showing skill activation
3. Verify trigger patterns match
4. Confirm no false positives in test cases
5. Only then mark as complete
```

**Benefits:**
- Prevents premature completion
- Ensures testing actually happens
- Provides audit trail
- Catches issues before deployment

---

## Performance Testing

### Measuring Hook Execution Time

**UserPromptSubmit (target: <100ms):**
```bash
time echo '{"prompt":"test prompt"}' | \
  npx tsx .claude/hooks/skill-activation-prompt.ts
```

**PreToolUse (target: <200ms):**
```bash
time cat <<'EOF' | npx tsx .claude/hooks/skill-verification-guard.ts
{"tool_name":"Edit","tool_input":{"file_path":"src/test.ts"}}
EOF
```

**Interpreting results:**
```
real    0m0.085s  # ✅ Total time <100ms - GOOD
user    0m0.072s
sys     0m0.013s

real    0m0.450s  # ❌ Total time >200ms - NEEDS OPTIMIZATION
user    0m0.380s
sys     0m0.070s
```

### Performance Optimization Strategies

**If hooks are slow:**

1. **Reduce pattern count**
   - Combine similar patterns
   - Remove redundant patterns
   - Use more specific patterns (fewer files to check)

2. **Simplify regex**
   - Avoid complex alternations: `(a|b|c|d|e|f)` → `(a|b)`
   - Use anchors when possible: `^import` faster than `import`
   - Avoid nested groups

3. **Optimize file operations**
   - Narrow path patterns: `src/specific/**` not `**/*`
   - Reduce content pattern usage
   - Skip binary files

4. **Cache compiled regex** (future enhancement)
   - Compile once at load time
   - Reuse compiled patterns
   - Watch for config changes, reload only when needed

### Load Testing

**Test with multiple scenarios:**
```bash
#!/bin/bash
# Load test with 100 prompts

for i in {1..100}; do
  time echo '{"prompt":"test '$i'"}' | \
    npx tsx .claude/hooks/skill-activation-prompt.ts > /dev/null
done 2>&1 | grep real | awk '{total+=$2; count++} END {print "Average:", total/count}'
```

---

## Regression Testing

### Maintaining Test Suite

**Create regression test suite:**
```
skill-tests/
  backend-dev-guidelines/
    prompt-triggers.json       # Prompt test cases
    file-triggers.json         # File test cases
    expected-outputs.json      # Expected results
  frontend-dev-guidelines/
    ...
```

**Test before commits:**
```bash
# Pre-commit hook
#!/bin/bash
./.claude/skills/test-all-skills.sh || {
  echo "❌ Skill tests failed - commit aborted"
  exit 1
}
```

### Version Control Integration

**Track changes to triggers:**
```bash
# Show changes to skill-rules.json
git diff .claude/skills/skill-rules.json

# Review trigger pattern changes
git log -p .claude/skills/skill-rules.json
```

**Test after updates:**
```bash
# After updating skill-rules.json
./evaluate-skills.py all-evaluations.json

# Compare with previous version
git checkout HEAD~1 -- .claude/skills/skill-rules.json
./evaluate-skills.py all-evaluations.json > old-results.txt
git checkout HEAD -- .claude/skills/skill-rules.json
./evaluate-skills.py all-evaluations.json > new-results.txt
diff old-results.txt new-results.txt
```

---

## Quality Checklist

### Pre-Deployment Checklist

**Skill File:**
- [ ] SKILL.md has valid YAML frontmatter
- [ ] Name matches directory name exactly
- [ ] Description includes ALL trigger keywords
- [ ] Description is under 1024 characters
- [ ] Main file is under 500 lines
- [ ] Code examples are tested and working
- [ ] Headings are properly hierarchical
- [ ] Links to resources are correct

**skill-rules.json:**
- [ ] Valid JSON syntax (tested with `jq`)
- [ ] Skill name matches SKILL.md
- [ ] Type is appropriate (domain vs guardrail)
- [ ] Enforcement level matches intent
- [ ] Priority is set correctly
- [ ] At least 2 trigger mechanisms configured
- [ ] Regex patterns tested on regex101.com
- [ ] Glob patterns tested with `find`
- [ ] Block message exists if enforcement is "block"

**Triggers:**
- [ ] Keywords cover common variations
- [ ] Intent patterns tested with 5+ prompts
- [ ] File path patterns match actual structure
- [ ] Content patterns tested against real files
- [ ] False positive rate <5%
- [ ] False negative rate <10%
- [ ] Activation rate >90% for intended scenarios

**Performance:**
- [ ] UserPromptSubmit executes in <100ms
- [ ] PreToolUse executes in <200ms
- [ ] No unnecessary file reads
- [ ] Regex patterns are optimized

**Testing:**
- [ ] Manual trigger tests passed
- [ ] Automated evaluation suite exists
- [ ] All test cases pass
- [ ] Real-world activation verified
- [ ] Edge cases tested

**Integration:**
- [ ] Hooks are installed and executable
- [ ] settings.json configured correctly
- [ ] No hardcoded paths to other projects
- [ ] Works in target environment

### Post-Deployment Monitoring

**Track metrics:**
- Actual activation rate
- False positive reports
- False negative reports
- Execution time in production
- User feedback

**Iterate based on data:**
- Add missing keywords discovered in use
- Refine patterns causing false positives
- Optimize slow operations
- Expand guidance for common questions

---

## Testing Tools and Scripts

### Quick Test Script

```bash
#!/bin/bash
# test-skill.sh - Quick skill testing script

SKILL_NAME=$1
HOOKS_DIR=".claude/hooks"

if [ -z "$SKILL_NAME" ]; then
  echo "Usage: ./test-skill.sh <skill-name>"
  exit 1
fi

echo "Testing skill: $SKILL_NAME"
echo ""

# Test 1: Check skill file exists
if [ -f ".claude/skills/$SKILL_NAME/SKILL.md" ]; then
  echo "✓ SKILL.md exists"
else
  echo "✗ SKILL.md not found"
  exit 1
fi

# Test 2: Validate JSON
if jq -e ".skills[\"$SKILL_NAME\"]" .claude/skills/skill-rules.json > /dev/null; then
  echo "✓ skill-rules.json entry exists"
else
  echo "✗ skill-rules.json entry not found"
  exit 1
fi

# Test 3: Test prompt trigger
RESULT=$(echo "{\"prompt\":\"testing $SKILL_NAME\"}" | \
  npx tsx $HOOKS_DIR/skill-activation-prompt.ts)
if echo "$RESULT" | grep -q "$SKILL_NAME"; then
  echo "✓ Prompt trigger works"
else
  echo "⚠ Prompt trigger may not work"
fi

echo ""
echo "Basic tests complete"
```

**Usage:**
```bash
chmod +x test-skill.sh
./test-skill.sh backend-dev-guidelines
```

---

## Next Steps

1. **Start with manual testing** - Verify basic functionality
2. **Create evaluation set** - Build 3-10 test cases
3. **Automate testing** - Use evaluation script
4. **Monitor in production** - Track actual metrics
5. **Iterate** - Refine based on data

**Related guides:**
- [skill-creation.md](skill-creation.md) - Creating skills
- [integration-guide.md](integration-guide.md) - Hook system details
- [troubleshooting.md](troubleshooting.md) - Debugging activation issues
- [best-practices.md](best-practices.md) - Optimization patterns
