# Security Review: `feature/lead-search`

## Executive Summary
A security review was performed on the `feature/lead-search` branch against `main`, applying the `security-and-hardening` and `code-review-and-quality` agent skills.

The review evaluated the newly introduced API endpoint `GET /api/leads/search` in [server.ts](file:///home/abdulpermana_dev/leads-crm-app-demo/server.ts) across OWASP Top 10 categories, threat vectors (STRIDE), and safe coding practices.

Three primary security issues were identified:
1. **SQL Injection** (Critical)
2. **Internal Error Details Exposure / Information Disclosure** (Medium)
3. **Sensitive Data & PII Logging** (Low)

---

## Threat Model & Findings

### 1. SQL Injection Vulnerability
* **Severity**: Critical
* **OWASP Category**: A03:2021 – Injection
* **File**: [server.ts](file:///home/abdulpermana_dev/leads-crm-app-demo/server.ts)
* **Threat Vector**: Tampering / Elevation of Privilege
* **Description**:
  The search route directly concatenates user input (`req.query.q`) into a SQL string:
  ```typescript
  const sql = `SELECT * FROM leads WHERE name LIKE '%${query}%' OR company LIKE '%${query}%' OR email LIKE '%${query}%'`;
  const results = db.prepare(sql).all();
  ```
  Unsanitized string interpolation in database queries allows an attacker to manipulate SQL control flow, potentially accessing unauthorized data or executing arbitrary SQL commands.

---

### 2. Internal Error Message Exposure (Information Disclosure)
* **Severity**: Medium
* **OWASP Category**: A05:2021 – Security Misconfiguration
* **File**: [server.ts](file:///home/abdulpermana_dev/leads-crm-app-demo/server.ts)
* **Threat Vector**: Information Disclosure
* **Description**:
  The route returns raw exception messages (`error.message`) directly in the HTTP 500 response payload:
  ```typescript
  res.status(500).json({
    error: "Search failed",
    details: error.message,
  });
  ```
  Exposing internal database error messages leaks database architecture details, table schemas, or query structure to untrusted clients, aiding attacker reconnaissance.

---

### 3. Raw User Query & PII Logging
* **Severity**: Low
* **OWASP Category**: A09:2021 – Security Logging and Monitoring Failures
* **File**: [server.ts](file:///home/abdulpermana_dev/leads-crm-app-demo/server.ts)
* **Threat Vector**: Information Disclosure
* **Description**:
  Raw search query parameters and formatted SQL statements containing user queries are written to standard log output:
  ```typescript
  console.log(`[search] Executing query: ${sql}`);
  console.log(`[search] Found ${results.length} results for query: ${query}`);
  ```
  Logging unredacted search queries can persist sensitive search queries or Personally Identifiable Information (PII) into plain-text system logs or log management platforms.

---

## Remediation Plan (Fix Strategy)

### Step 1: Parameterize Database Queries
Replace template string concatenation with parameterized SQL bindings (`?` placeholders) supported by `better-sqlite3`.

```typescript
const searchPattern = `%${query}%`;
const sql = `SELECT * FROM leads WHERE name LIKE ? OR company LIKE ? OR email LIKE ?`;
const results = db.prepare(sql).all(searchPattern, searchPattern, searchPattern);
```

### Step 2: Sanitize HTTP Error Responses
Sanitize response payloads on server errors by omitting internal error messages (`details`). Log errors server-side via `console.error` for internal diagnostics.

```typescript
console.error('[search] Error executing search:', error);
res.status(500).json({
  error: "Search failed"
});
```

### Step 3: Redact Raw Input Logging
Remove or redact `console.log` statements that output raw SQL strings or user search terms.

---

## Verification Plan

1. **Functional Test**: Perform valid search requests to ensure matching leads (by name, company, or email) are returned accurately.
2. **SQL Injection Mitigation Test**: Pass inputs containing SQL control characters (e.g. `' OR '1'='1`, `'; DROP TABLE leads; --`) to verify parameterized binding prevents SQL payload execution.
3. **Error Handling Test**: Trigger a simulated query failure to verify HTTP 500 responses return generic error messages without leaking internal exception details.
