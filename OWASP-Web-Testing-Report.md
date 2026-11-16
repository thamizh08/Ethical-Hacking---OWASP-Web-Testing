# OWASP Web Testing Lab — Security Assessment Report

## 1. Assessment Overview

- **Application:** OWASP Juice Shop
- **Version:** 20.4.0-SNAPSHOT
- **Target:** http://127.0.0.1:3000
- **Environment:** Locally hosted Docker lab
- **Reverse Proxy:** Nginx 1.31.8
- **Testing Scope:** Authentication, access control, input handling, and security headers
- **Testing Type:** Manual, non-destructive testing

## 2. Findings Summary

| ID | Finding | Severity | Status |
|---|---|---|---|
| WEB-01 | Database error disclosure through product search | Medium (provisional) | Reproduced |
| WEB-02 | JWT remains usable after logout | Medium (provisional) | Reproduced |
| OBS-01 | JWT stored in browser Local Storage without `exp` claim | Informational | Observed |
| OBS-02 | Several security headers absent from observed response | Informational | Observed |
| OBS-03 | Wildcard CORS response header | Informational | Requires policy review |

## 3. Detailed Findings

### WEB-01: Database Error Disclosure Through Product Search

**Endpoint:** `/rest/products/search`

**Method:** GET

**Evidence:**
- Normal search input returned HTTP 200 with product data.
- Special-character input `test<>"'` returned HTTP 500.
- Response exposed an SQLite database error.

**Potential Impact:**
Internal database implementation details are disclosed. The error may indicate unsafe handling of user-controlled input.

**Likely Root Cause:**
Potentially unsafe query construction or insufficient input handling. Source-code inspection is required to confirm the exact cause.

**Recommended Remediation:**
- Use parameterized database queries.
- Validate and safely handle search input.
- Return generic errors to clients.
- Keep detailed database errors in server-side logs.

**Limitation:** Successful SQL injection or data extraction was not demonstrated.

### WEB-02: JWT Remains Usable After Logout

**Endpoint:** `/rest/basket/1`

**Method:** GET

**Evidence:**
- Logged in using a dedicated lab account.
- Copied the JWT locally.
- Logged out through the application.
- Reused the old JWT against the basket endpoint.
- Received HTTP 200 with basket data.
- Repeated the test and obtained HTTP 200 again.

**Potential Impact:**
A stolen token may remain usable after logout, allowing continued access to protected resources.

**Likely Root Cause:**
The server appears not to revoke previously issued tokens when the user logs out.

**Recommended Remediation:**
- Implement server-side token revocation.
- Use short-lived access tokens.
- Implement refresh-token rotation and revocation.
- Review logout behavior and session invalidation.

### OBS-01: JWT Stored in Local Storage

**Evidence:**
The authentication token was visible in browser Local Storage. The decoded JWT contained `iat`, `date`, and `bid` fields, but no `exp` claim.

**Potential Impact:**
JavaScript executing in the same origin could access the token. A successful XSS attack could therefore expose it.

**Recommended Remediation:**
Evaluate secure, HttpOnly cookie-based session storage and appropriate token expiration and revocation.

**Limitation:** Missing `exp` alone does not prove indefinite token validity. The post-logout token reuse test is documented separately as WEB-02.

### OBS-02: Security Header Observations

The observed HTTP response included:
- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: SAMEORIGIN`

The following headers were not present in the captured response:
- `Content-Security-Policy`
- `Permissions-Policy`
- `Referrer-Policy`

HSTS was not present; the tested target used HTTP.

**Recommendation:** Review the application's security-header policy and configure appropriate headers at the correct application or reverse-proxy layer.

### OBS-03: Wildcard CORS

**Observed header:** `Access-Control-Allow-Origin: *`

This is an observation, not a confirmed vulnerability. Review whether public cross-origin access is intended for the exposed resources.

## 4. Authentication and Access Control Tests

| Test | Result |
|---|---|
| Invalid login | HTTP 401 with generic error |
| Successful login | HTTP 200; token returned |
| Unauthenticated basket request | HTTP 401 |
| Old token after logout | HTTP 200 with basket data |

## 5. Input Handling Tests

| Input | Result |
|---|---|
| `apple123` | HTTP 200, empty results |
| `apple` | HTTP 200, product data |
| `test<>"'` | HTTP 500, SQLite error |

## 6. Conclusion

The local assessment identified two reproducible security concerns: database error disclosure during product search and continued acceptance of a JWT after logout.

Other tests confirmed that invalid login attempts and unauthenticated basket requests were rejected. Additional security-header and CORS observations require policy review.

The assessment was limited to the authorized local OWASP Juice Shop environment. No external systems were tested.

**Security note:** No reusable JWT, password, or personal account information is included in this report.