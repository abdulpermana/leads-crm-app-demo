# Security Review: `feature/lead-search`

## Executive Summary
A security review was conducted on the diff between `main` and `feature/lead-search` (`server.ts`).
The diff introduces a new search API endpoint (`GET /api/leads/search`). Three security issues were identified, ranging from **Critical** to **Low** severity.

---

## Findings

### 1. SQL Injection Vulnerability
* **Severity**: Critical
* **OWASP Category**: A03:2021 – Injection
* **File**: [`server.ts`](file:///home/abdulpermana_dev/leads-crm-app-demo/server.ts#L238)
* **Description**:
  The search endpoint constructs SQL queries by directly concatenating user-supplied input (`req.query.q`) into the SQL string:
  ```typescript
  const sql = `SELECT * FROM leads WHERE name LIKE '%${query}%' OR company LIKE '%${query}%' OR email LIKE '%${query}%'`;
  const results = db.prepare(sql).all();
  ```
  Unsanitized user input concatenated into a SQL statement allows attackers to inject arbitrary SQL fragments, potentially bypassing search boundaries, executing unauthorized database operations, or extracting data.

---

### 2. Internal Error Details Exposure (Information Disclosure)
* **Severity**: Low / Medium
* **OWASP Category**: A05:2021 – Security Misconfiguration
* **File**: [`server.ts`](file:///home/abdulpermana_dev/leads-crm-app-demo/server.ts#L248-L251)
* **Description**:
  In the `catch` block, internal error message details are directly returned to the client in the HTTP response:
  ```typescript
  res.status(500).json({
    error: "Search failed",
    details: error.message,
  });
  ```
  Exposing raw error details (`error.message`) to external callers leaks database structural details, column names, or driver implementation details that can assist an attacker in reconnaissance.

---

### 3. Sensitive Data & Raw Input Logging
* **Severity**: Low
* **OWASP Category**: A09:2021 – Security Logging and Monitoring Failures
* **File**: [`server.ts`](file:///home/abdulpermana_dev/leads-crm-app-demo/server.ts#L240-L244)
* **Description**:
  Raw user inputs and constructed SQL strings are logged directly to standard output:
  ```typescript
  console.log(`[search] Executing query: ${sql}`);
  console.log(`[search] Found ${results.length} results for query: ${query}`);
  ```
  Logging raw user query strings (which may include names, company details, or email addresses) can expose Personally Identifiable Information (PII) in application logs or log management platforms.

---

## Plan to Fix

### Step 1: Parameterize the SQL Query
Use parameterized query placeholders (`?`) provided by `better-sqlite3` / SQLite driver rather than string template literals.

```typescript
const searchPattern = `%${query}%`;
const sql = `SELECT * FROM leads WHERE name LIKE ? OR company LIKE ? OR email LIKE ?`;
const results = db.prepare(sql).all(searchPattern, searchPattern, searchPattern);
```

### Step 2: Sanitize API Error Responses
Remove the `details` field from the production error response to prevent leaking internal database error messages. Log internal errors server-side securely instead of sending them to the client.

```typescript
res.status(500).json({
  error: "Search failed",
});
```

### Step 3: Clean Up / Redact Verbose Console Logging
Remove or sanitize `console.log` statements that output raw SQL strings or user search terms.

---

## Verification Plan
Once the fixes are applied:
1. Verify search functionality works as expected with standard text inputs (e.g., matching name, company, email).
2. Test input containing SQL special characters (e.g. `'`, `"`, `--`, `OR 1=1`) to confirm parameterized binding prevents injection.
3. Confirm HTTP 500 error responses do not leak raw exception messages.
