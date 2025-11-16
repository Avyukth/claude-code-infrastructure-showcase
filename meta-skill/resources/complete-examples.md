# Complete Examples - Full Working Skills

Real-world skill examples demonstrating best practices.

## Table of Contents

- [Example 1: Simple Single-File Skill](#example-1-simple-single-file-skill)
- [Example 2: Modular Multi-Resource Skill](#example-2-modular-multi-resource-skill)
- [Example 3: Guardrail Skill with Blocking](#example-3-guardrail-skill-with-blocking)
- [Example 4: Adapting for Different Tech Stacks](#example-4-adapting-for-different-tech-stacks)

---

## Example 1: Simple Single-File Skill

**Use case:** Simple advisory skill for API testing.

### Directory Structure

```
route-tester/
  SKILL.md
```

### SKILL.md

```markdown
---
name: route-tester
description: Testing authenticated API routes with JWT cookies. Use when testing routes, endpoints, APIs, debugging auth issues, verifying route functionality, or validating API responses. Covers JWT cookie authentication, route testing patterns, curl commands, and debugging workflows.
---

# Route Tester

## Purpose

Guide for testing authenticated API routes using JWT cookie authentication.

## When to Use This Skill

- Testing new routes or endpoints
- Debugging authentication issues
- Verifying API functionality
- Validating route responses

## Quick Start: Test a Route

### Step 1: Get Auth Cookie

```bash
# Login and capture cookie
curl -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password"}' \
  -c cookies.txt
```

### Step 2: Test Route

```bash
# Use cookie for authenticated request
curl http://localhost:3000/api/users/me \
  -b cookies.txt
```

### Step 3: Verify Response

Check for:
- [ ] Correct HTTP status code (200, 201, etc.)
- [ ] Expected JSON structure
- [ ] Proper data values
- [ ] No error messages

## Common Patterns

**Test POST route:**
```bash
curl -X POST http://localhost:3000/api/resource \
  -H "Content-Type: application/json" \
  -b cookies.txt \
  -d '{"field":"value"}'
```

**Test with query params:**
```bash
curl "http://localhost:3000/api/resource?filter=active" \
  -b cookies.txt
```

**Debug with verbose:**
```bash
curl -v http://localhost:3000/api/resource \
  -b cookies.txt
```

## Related Skills

- **backend-dev-guidelines** - Route and controller patterns
- **error-tracking** - Debugging errors in routes

---

**Skill Status**: Simple single-file skill
**Line Count**: ~100 lines
```

### skill-rules.json Entry

```json
{
  "route-tester": {
    "type": "domain",
    "enforcement": "suggest",
    "priority": "medium",
    "promptTriggers": {
      "keywords": [
        "route", "endpoint", "API", "test route",
        "curl", "authentication", "auth cookie"
      ],
      "intentPatterns": [
        "(test|debug|verify).*?(route|endpoint|API)",
        "how do I test.*?(route|API|endpoint)"
      ]
    }
  }
}
```

**Why this works:**
- Focused scope (just route testing)
- Under 500 lines easily
- Clear, actionable guidance
- Minimal but sufficient

---

## Example 2: Modular Multi-Resource Skill

**Use case:** Comprehensive backend development guidelines.

### Directory Structure

```
backend-dev-guidelines/
  SKILL.md                          # 304 lines - main guide
  resources/
    architecture-overview.md        # Layered architecture details
    routing-and-controllers.md      # Route and controller patterns
    services-and-repositories.md    # Business logic layer
    validation-patterns.md          # Zod validation
    sentry-and-monitoring.md        # Error tracking
    middleware-guide.md             # Express middleware
    database-patterns.md            # Prisma patterns
    configuration.md                # Config management
    async-and-errors.md             # Async/error handling
    testing-guide.md                # Testing strategies
    complete-examples.md            # Full examples
```

### SKILL.md (Condensed)

```markdown
---
name: backend-dev-guidelines
description: Express/Prisma/TypeScript backend patterns. Use when creating routes, controllers, services, repositories, middleware, working with APIs, databases, error tracking. Covers layered architecture, BaseController, Sentry, Zod validation, dependency injection.
---

# Backend Development Guidelines

## Purpose

Establish consistency across Express/Prisma/TypeScript microservices.

## When to Use

- Creating routes, controllers, services
- Database operations with Prisma
- Error tracking with Sentry
- Input validation with Zod
- Backend testing

## Quick Start Checklist

- [ ] Route definition (delegate to controller)
- [ ] Controller (extends BaseController)
- [ ] Service (business logic with DI)
- [ ] Repository (database access if complex)
- [ ] Validation (Zod schema)
- [ ] Sentry integration
- [ ] Tests

## Architecture Overview

```
Request → Routes → Controllers → Services → Repositories → Database
```

Each layer has ONE responsibility.

See [architecture-overview.md](resources/architecture-overview.md) for details.

## Core Principles

1. Routes only route, controllers control
2. All controllers extend BaseController
3. All errors to Sentry
4. Use unifiedConfig, never process.env
5. Validate input with Zod
6. Repository pattern for data access
7. Comprehensive testing required

## Navigation Guide

| Need to... | Read this |
|------------|-----------|
| Understand architecture | [architecture-overview.md](resources/architecture-overview.md) |
| Create routes/controllers | [routing-and-controllers.md](resources/routing-and-controllers.md) |
| Organize business logic | [services-and-repositories.md](resources/services-and-repositories.md) |
| Validate input | [validation-patterns.md](resources/validation-patterns.md) |
| Add error tracking | [sentry-and-monitoring.md](resources/sentry-and-monitoring.md) |
| Create middleware | [middleware-guide.md](resources/middleware-guide.md) |
| Database access | [database-patterns.md](resources/database-patterns.md) |
| Manage config | [configuration.md](resources/configuration.md) |
| Handle async/errors | [async-and-errors.md](resources/async-and-errors.md) |
| Write tests | [testing-guide.md](resources/testing-guide.md) |
| See examples | [complete-examples.md](resources/complete-examples.md) |

---

**Skill Status**: Complete modular skill
**Line Count**: 304 lines main + 11 resources
**Progressive Disclosure**: Load resources as needed
```

### skill-rules.json Entry

```json
{
  "backend-dev-guidelines": {
    "type": "domain",
    "enforcement": "suggest",
    "priority": "high",
    "promptTriggers": {
      "keywords": [
        "backend", "express", "prisma", "controller",
        "service", "repository", "route", "endpoint",
        "middleware", "validation", "sentry"
      ],
      "intentPatterns": [
        "(create|add|build).*?(route|endpoint|controller|service)",
        "(how does|explain).*?(backend|architecture|layered)",
        "backend.*?(pattern|best practice)"
      ]
    },
    "fileTriggers": {
      "pathPatterns": [
        "backend/src/**/*.ts",
        "*/src/controllers/**/*.ts",
        "*/src/services/**/*.ts"
      ],
      "pathExclusions": [
        "**/*.test.ts"
      ]
    }
  }
}
```

**Why this works:**
- Main file under 500 lines
- 11 focused resource files
- Progressive loading
- Clear navigation
- Comprehensive coverage

---

## Example 3: Guardrail Skill with Blocking

**Use case:** Prevent database errors by enforcing schema verification.

### SKILL.md

```markdown
---
name: database-verification
description: Verify database schema before Prisma queries. Use when working with Prisma, database queries, table names, column names, or schema changes. Prevents runtime errors from incorrect column/table references.
---

# Database Verification

## Purpose

Enforce verification of table and column names before database operations to prevent runtime errors.

## When This Activates

**Automatically blocks** when editing files that contain Prisma usage.

## Required Workflow

When blocked:

1. **Verify Schema**
   ```bash
   # Check table structure
   npx prisma studio
   # or
   psql -d database_name -c "\d table_name"
   ```

2. **Confirm Column Names**
   - Match EXACTLY to schema
   - Check for typos
   - Verify relationships

3. **Retry Edit**
   - Once verified, retry the edit
   - Skill won't block again this session

## Skip Conditions

**Add comment to skip:**
```typescript
// @skip-validation
import { PrismaService } from './prisma';
// This file has been manually verified
```

**Environment override:**
```bash
export SKIP_DB_VERIFICATION=true
```

## Common Verification Commands

```bash
# List all tables
npx prisma studio

# Show table structure
\d table_name  # PostgreSQL
DESCRIBE table_name;  # MySQL

# Check Prisma schema
cat prisma/schema.prisma | grep -A 10 "model TableName"
```

---

**Skill Status**: Guardrail (blocking)
**Enforcement**: PreToolUse hook
```

### skill-rules.json Entry

```json
{
  "database-verification": {
    "type": "guardrail",
    "enforcement": "block",
    "priority": "critical",
    "promptTriggers": {
      "keywords": ["prisma", "database", "table", "column", "schema"],
      "intentPatterns": [
        "(add|create|modify).*?(user|table|column|field)",
        "database.*?(query|change|update)"
      ]
    },
    "fileTriggers": {
      "pathPatterns": [
        "**/src/**/*.ts"
      ],
      "pathExclusions": [
        "**/*.test.ts"
      ],
      "contentPatterns": [
        "import.*[Pp]risma",
        "PrismaService",
        "\\.findMany\\(|\\.create\\(|\\.update\\(|\\.delete\\("
      ]
    },
    "blockMessage": "⚠️ BLOCKED - Database Operation Detected\n\n📋 REQUIRED ACTION:\n1. Use Skill tool: 'database-verification'\n2. Verify ALL table and column names against schema\n3. Check database structure with DESCRIBE commands\n4. Then retry this edit\n\nReason: Prevent column name errors in Prisma queries\nFile: {file_path}\n\n💡 TIP: Add '// @skip-validation' comment to skip future checks",
    "skipConditions": {
      "sessionSkillUsed": true,
      "fileMarkers": ["@skip-validation"],
      "envOverride": "SKIP_DB_VERIFICATION"
    }
  }
}
```

**Why this works:**
- Critical enforcement prevents errors
- Session tracking prevents nagging
- Clear actionable message
- Multiple skip options

---

## Example 4: Adapting for Different Tech Stacks

**Scenario:** User has Vue.js instead of React, wants frontend guidelines.

### Original (React)

```yaml
name: frontend-dev-guidelines
description: React/TypeScript/MUI patterns. Use when creating React components, styling with MUI, using hooks...
```

### Adapted (Vue)

```yaml
name: vue-dev-guidelines
description: Vue 3/TypeScript/Vuetify patterns. Use when creating Vue components, styling with Vuetify, using composables, Pinia stores, Vue Router...
```

### Key Changes

**SKILL.md:**
- Replace: `React.FC` → `defineComponent`
- Replace: `useState/useEffect` → `ref/onMounted`
- Replace: `MUI components` → `Vuetify components`
- Replace: `TanStack Query` → `Vue Query` or `composables`
- Keep: File organization, performance patterns, TypeScript standards

**skill-rules.json:**
```json
{
  "vue-dev-guidelines": {
    "type": "domain",
    "enforcement": "suggest",
    "priority": "high",
    "promptTriggers": {
      "keywords": [
        "vue", "component", "vuetify", "composable",
        "pinia", "vue router", "frontend"
      ],
      "intentPatterns": [
        "(create|add).*?(component|page|view)",
        "(style).*?(button|form)",
        "vue.*?(pattern|best practice)"
      ]
    },
    "fileTriggers": {
      "pathPatterns": [
        "src/**/*.vue",
        "src/components/**/*.vue"
      ],
      "pathExclusions": [
        "**/*.spec.vue"
      ]
    }
  }
}
```

**What transfers:**
- ✅ File organization principles
- ✅ Performance optimization strategies
- ✅ TypeScript best practices
- ✅ Testing approaches
- ✅ Lazy loading concepts

**What doesn't transfer:**
- ❌ React-specific hooks
- ❌ MUI component examples
- ❌ TanStack Query patterns

---

**Related files:**
- [skill-creation.md](skill-creation.md) - Creating these patterns
- [best-practices.md](best-practices.md) - Why these work
- [integration-guide.md](integration-guide.md) - Hook configuration
