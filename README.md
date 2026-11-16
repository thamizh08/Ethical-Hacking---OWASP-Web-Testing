# OWASP Web Testing Lab — Security Assessment

A manual, non-destructive web application security assessment conducted against a locally hosted instance of **OWASP Juice Shop**. This repository documents the lab setup, testing methodology, reproducible security findings, and remediation recommendations.

---

## 📋 Table of Contents
- [Lab Environment & Scope](#-lab-environment--scope)
- [Findings Summary](#-findings-summary)
- [Key Vulnerabilities & Observations](#-key-vulnerabilities--observations)
- [Test Matrix](#-test-matrix)
- [Remediation Roadmap](#-remediation-roadmap)
- [Disclaimer & Security Note](#-disclaimer--security-note)

---

## 🔬 Lab Environment & Scope

| Parameter | Details |
| :--- | :--- |
| **Target Application** | OWASP Juice Shop (`v20.4.0-SNAPSHOT`) |
| **Target URL** | `http://127.0.0.1:3000` |
| **Infrastructure** | Locally hosted Docker container behind **Nginx 1.31.8** |
| **Testing Methodology** | Manual, non-destructive black/grey-box testing |
| **In-Scope Areas** | Authentication, Access Control, Input Handling, Security Headers |

---

## 📊 Findings Summary

| ID | Finding | Severity | Status |
| :--- | :--- | :--- | :--- |
| **WEB-01** | Database error disclosure through product search | Medium (Provisional) | Reproduced |
| **WEB-02** | JWT remains usable after logout | Medium (Provisional) | Reproduced |
| **OBS-01** | JWT stored in browser Local Storage without `exp` claim | Informational | Observed |
| **OBS-02** | Several security headers absent from observed response | Informational | Observed |
| **OBS-03** | Wildcard CORS response header (`*`) | Informational | Policy Review |

---

## 🔍 Key Vulnerabilities & Observations

### [WEB-01] Database Error Disclosure Through Product Search
* **Endpoint:** `GET /rest/products/search`
* **Observation:** Passing special characters (`test<>"'`) triggered an `HTTP 500` response exposing raw SQLite error details.
* **Impact:** Reveals internal database architecture and indicates potentially unsafe query construction. *(Note: Full SQLi data extraction was not demonstrated).*
* **Fix:** Implement parameterized queries, sanitize search inputs, and suppress verbose stack/DB errors in client responses.

### [WEB-02] JWT Remains Usable After Logout
* **Endpoint:** `GET /rest/basket/1`
* **Observation:** After logging out of the web application, reusing a previously captured JWT still returned `HTTP 200` along with sensitive basket data.
* **Impact:** Compromised or stolen tokens remain valid indefinitely post-logout, allowing unauthorized session persistence.
* **Fix:** Implement server-side token revocation/blocklisting upon logout and enforce short-lived access tokens with refresh token rotation.

### [OBS-01] JWT Storage & Missing Expiration (`exp`)
* **Observation:** Tokens are stored in `localStorage` and lack a standard `exp` (expiration) claim in the payload (containing only `iat`, `date`, and `bid`).
* **Impact:** Susceptible to theft via Cross-Site Scripting (XSS).
* **Fix:** Migrate session tokens to `HttpOnly`, `Secure`, `SameSite` cookies and enforce strict token lifetimes.

### [OBS-02] Missing HTTP Security Headers
* **Present:** `X-Content-Type-Options: nosniff`, `X-Frame-Options: SAMEORIGIN`
* **Missing:** `Content-Security-Policy`, `Permissions-Policy`, `Referrer-Policy` *(HSTS omitted due to local HTTP setup)*.
* **Fix:** Configure missing defense-in-depth headers at the Nginx reverse-proxy or application middleware layer.

### [OBS-03] Wildcard CORS Policy
* **Observation:** Responses include `Access-Control-Allow-Origin: *`.
* **Fix:** Restrict cross-origin resource sharing to explicitly trusted domains if public API access is not strictly required.

---

## 🧪 Test Matrix

### Authentication & Access Control
| Test Case | Expected / Observed Result |
| :--- | :--- |
| Invalid login credentials | `HTTP 401` with generic error message |
| Valid login credentials | `HTTP 200` with JWT issued |
| Unauthenticated basket access | `HTTP 401` Unauthorized |
| Authenticated basket access (post-logout JWT) | `HTTP 200` with basket payload (**Finding WEB-02**) |

### Input Handling (`/rest/products/search`)
| Payload | Result |
| :--- | :--- |
| `apple` | `HTTP 200` — Valid product JSON returned |
| `apple123` | `HTTP 200` — Empty result set |
| `test<>"'` | `HTTP 500` — SQLite syntax error disclosed (**Finding WEB-01**) |

---

## 🛠 Remediation Roadmap

1. **Input Sanitization & Error Handling:** Refactor search endpoints to use parameterized ORM queries and configure global exception handlers to return generic HTTP 500 messages.
2. **Session Lifecycle Management:** Introduce a token invalidation mechanism (e.g., Redis blocklist or token versioning) triggered on user logout.
3. **Hardening & Headers:** Move JWT storage out of `localStorage` into `HttpOnly` cookies and tighten Nginx header/CORS configurations.

---

## ⚠️ Disclaimer & Security Note

* **Authorized Testing Only:** All testing was performed strictly within an isolated, local Docker lab environment running OWASP Juice Shop. No external or production systems were targeted.
* **Sanitized Report:** All reusable JWTs, credentials, and personal identifiers have been excluded from this documentation.
