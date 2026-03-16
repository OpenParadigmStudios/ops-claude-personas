---
name: check-api-contract
description: Verify API endpoints match schemas, frontend expectations, and spec documentation
argument-hint: [domain]
allowed-tools: [Read, Glob, Grep]
context: fork
agent: Explore
---

# Check API Contract

Cross-reference API route definitions against validation schemas, frontend API calls, test assertions, and spec documentation to find mismatches.

**Arguments**: `$ARGUMENTS` — optional domain filter (e.g., `characters`, `proposals`). If omitted, checks all API endpoints.

---

## Workflow

### Step 1: Discover Route Definitions

Search the project's API routes directory for endpoint definitions. For each endpoint, extract:
- HTTP method (GET, POST, PUT, PATCH, DELETE)
- Path (including path parameters)
- Request body schema (if any)
- Response schema
- Status codes (success and error)
- Auth requirements
- Query parameters

### Step 2: Discover Validation Schemas

Search the project's schemas/DTOs directory. For each schema, extract:
- Field names and types
- Required vs. optional fields
- Validation constraints (min/max, patterns, enums)
- Nested object shapes

### Step 3: Discover Frontend API Calls

Search the frontend source for API calls (fetch, axios, API client methods). For each call, extract:
- Target endpoint (method + path)
- Request payload shape
- Expected response shape
- Error handling expectations

### Step 4: Discover Test Assertions

Search the test directory for API test cases. For each test, extract:
- Endpoint being tested
- Request payload
- Expected status code
- Expected response shape/values

### Step 5: Load Spec Documentation

Read relevant spec documents:
- Domain specs that describe API endpoints
- `spec/glossary.md` — for term/field name consistency

### Step 6: Cross-Reference

Build a matrix of each endpoint against all sources:

**Route ↔ Schema**
- Does the route reference the correct request/response schemas?
- Do schema field names match route parameter names?
- Are all route parameters validated by a schema?

**Route ↔ Frontend**
- Does the frontend call the correct method and path?
- Does the frontend send the expected request shape?
- Does the frontend handle all documented status codes?
- Are there frontend calls to endpoints that don't exist?

**Route ↔ Tests**
- Is every endpoint tested?
- Do tests assert the correct status codes?
- Do tests verify response shapes match schemas?
- Are error cases tested?

**Route ↔ Spec**
- Does the spec document all implemented endpoints?
- Do field names in the spec match the schema and code?
- Are behavioral claims in the spec tested?
- Are there spec-documented endpoints that aren't implemented?

### Step 7: Report

```markdown
## API Contract Check: [scope]

### Endpoints Analyzed
| Method | Path | Schema | Frontend | Tests | Spec |
|--------|------|--------|----------|-------|------|
| GET | /api/characters | CharacterList | characters.js | test_characters.py | characters.md |

### Mismatches Found

#### Schema Mismatches
- **[endpoint]**: [Route expects X, schema defines Y]

#### Frontend Mismatches
- **[endpoint]**: [Frontend sends/expects X, route provides Y]

#### Test Gaps
- **[endpoint]**: [Not tested / missing error case tests]

#### Spec Divergences
- **[endpoint]**: [Spec says X, implementation does Y]

### Orphans
- **Undocumented endpoints**: [Routes with no spec entry]
- **Dead frontend calls**: [Frontend calls to non-existent endpoints]
- **Stale tests**: [Tests for endpoints that no longer exist]

### Coverage Summary
| Metric | Count |
|--------|-------|
| Total endpoints | X |
| Schema-validated | X |
| Frontend-consumed | X |
| Test-covered | X |
| Spec-documented | X |
| Fully aligned | X |

### Priority Fixes
1. [Highest risk mismatch]
2. [Second priority]
3. [Third priority]
```

---

## Rules

- This is a read-only analysis skill — report mismatches, don't fix them
- A missing frontend call isn't always a bug (some endpoints are internal/admin-only) — note it but don't flag as critical
- Focus on shape mismatches (wrong fields, wrong types) over style differences (naming conventions)
- If the project has no frontend, skip Steps 3 and the frontend columns
- If the project has an OpenAPI/Swagger spec, use it as the primary contract source
