# Security Policy

## Introduction
This document defines the security posture for the blog application built with Laravel 11. It establishes mandatory security practices for contributors, outlines the project's security boundaries, and provides a placeholder system for unresolved decisions. All developers working on this repository must adhere to these rules.

## Reporting a Vulnerability
If you discover a security vulnerability, please **do not** open a public issue. Instead, email the maintainers at `[TODO: replace with actual security contact email]`. We aim to acknowledge reports within 48 hours and provide a fix timeline within 5 business days. We follow coordinated disclosure and request that you not publicly disclose the vulnerability until a patch is released.

## Required Security Inputs
The following items must be addressed before any PR that touches the relevant layer is merged:
- **Input validation rules** for every user-facing endpoint (Form Request classes).
- **Authorization checks** (Gates/Policies) for all sensitive operations.
- **Data sanitization strategy** for stored HTML (blog post content) – currently using HTMLPurifier.
- **File upload validation**: MIME type whitelist, size caps, malware scanning plan (optional but documented).
- **Secrets management plan**: All credentials must be injected via environment variables; never committed.
- **GDPR compliance**: See the [GDPR Compliance Checklist](#gdpr-compliance-checklist) below.
- **Rate limiting configuration**: Endpoints must have throttle definitions (auth, API, uploads).

## Security Rules
1. **HTTP Security Headers** – All responses must include:
    - `Strict-Transport-Security` (HSTS) with max-age ≥ 1 year.
    - `Content-Security-Policy` (CSP) with a strict policy that allows only necessary sources (scripts, images, styles).
    - `X-Content-Type-Options: nosniff`
    - `X-Frame-Options: DENY` (or `SAMEORIGIN` if embedding necessary)
    - `Referrer-Policy: strict-origin-when-cross-origin`
    - Implement via middleware.
2. **Authentication & Session Security**:
    - Use `secure` and `httpOnly` cookies for session tokens (Laravel’s default).
    - `sameSite` attribute set to `Lax` or `Strict` for web routes.
    - API tokens (Sanctum) must be treated as bearer tokens; never stored in `localStorage` if used in SPA.
    - Password hashing must use bcrypt with cost ≥ 12 (Laravel default).
3. **Authorization** – Every action that modifies data must be gated by a Policy (e.g., `PostPolicy`). Default to deny.
4. **Input Validation & Sanitization**:
    - All incoming requests pass through Laravel Form Requests; no manual validation in controllers.
    - Blog post content (rich HTML) must be sanitized through a well-maintained library (HTMLPurifier) before storage. No raw HTML is echoed without sanitization.
    - All output in Blade templates is automatically escaped. If using Inertia/Vue/React, treat all data as untrusted and escape appropriately; avoid `v-html` or `dangerouslySetInnerHTML` unless sanitized on the server.
5. **File Uploads**:
    - Validate MIME type by content inspection, not just extension.
    - Store uploaded files outside the web root, served via an application route or CDN.
    - Images processed with Intervention Image; resize/re-encode to remove potential embedded threats.
    - Limit upload size (10 MB per image) and total per request.
6. **Database Security**:
    - Use parameterized queries (Eloquent does this automatically).
    - No raw SQL unless absolutely necessary and then with bindings.
    - Enable mass assignment protection (`protected $guarded` or `$fillable` on models).
7. **Secrets Management** – All secrets (DB credentials, app keys, API keys) must be stored in `.env` (excluded from git via `.gitignore`). Use `.env.example` with dummy values. In production, use environment variables or a vault service.
8. **Logging & Error Handling**:
    - Never expose stack traces in production (`APP_DEBUG=false`).
    - Log authentication events (login, logout, failed attempts) and sensitive operations (post publish, user deletion).
    - Sanitize logs to remove passwords, tokens, and personal data.
    - Use structured logging for monitoring.
9. **Dependency Management** – Regularly update Composer and NPM dependencies; use `composer audit` and `npm audit` in CI. Pin known vulnerable versions with immediate patches.
10. **CI/CD Pipeline** – Include security checks: static analysis (Larastan), code style (Pint), unit/feature tests, and OWASP Dependency Check. Never expose secrets in build logs.

## Prompt Placeholders To Resolve
This section maps security-related prompts to concrete actions or references. Unresolved items are marked `UNRESOLVED`.

### {{MANICODE_CODE_QUALITY_PROMPT}}
**Resolved as:** Maintain low cyclomatic complexity and low cognitive complexity. Follow the separation of concerns already established (Controllers → Services → Repositories). All new code must pass Larastan level 8 and have appropriate unit tests. Use PSR-12 coding standards enforced by Laravel Pint.

### {{MANICODE_API_SECURITY_PROMPT}}
**Resolved as:** All REST API endpoints must adhere to [OWASP REST Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html). Implement proper authentication (Sanctum tokens), authorization, input validation, rate limiting, and content-type headers. Never expose sensitive data in URLs; use HTTPS exclusively.

### {{MANICODE_BACKEND_FRAMEWORK_PROMPT}}
**Resolved as:** Use Laravel’s built-in security features: CSRF protection for web routes, XSS filtering (Blade auto-escapes), SQL injection prevention via Eloquent, mass assignment protection, and validation rules. Refer to the [Laravel 11 Security Documentation](https://laravel.com/docs/11.x/security) for additional guidelines. Ensure all controllers and services leverage these features.

### {{MANICODE_FRONTEND_FRAMEWORK_PROMPT}}
**UNRESOLVED** – The frontend stack is not yet fixed (Blade/Livewire/Alpine.js or Inertia.js with Vue 3/React). Regardless of the final choice, the fundamental rule is: **assume all data rendered in the frontend is untrusted**. For Blade, this is automatic; for Inertia/Vue/React, follow the respective framework's security best practices (e.g., avoid `v-html`, sanitize content). This placeholder will be updated with a specific prompt file once the frontend technology is chosen. For now, contributors should review:
- Vue: [Security Best Practices](https://vuejs.org/guide/best-practices/security.html)
- React: [React Security](https://react.dev/reference/react-dom/components/common#dangerously-setting-the-inner-html)

### {{MANICODE_AUTH_PROMPT}}
**Resolved as:** Implement at least **AAL2** (multi-factor authentication) for administrative and author accounts. Consider passkey support as a future enhancement using WebAuthn (Laravel packages available). Currently, Laravel Sanctum provides token-based authentication; session-based auth uses `auth` middleware with password reset and throttling. Password complexity rules must enforce minimum 12 characters (per NIST SP 800-63B), at least one uppercase, one number, one special character.

### {{MANICODE_DEPLOYMENT_PROMPT}}
**UNRESOLVED** – Deployment platform is not yet specified. Candidate technologies include Nginx, PHP-FPM, Docker, and S3-compatible storage, but no cloud provider (AWS, Azure, GCP, etc.) has been confirmed. Once decided, update this section with the relevant security practices (e.g., Azure Security Benchmarks, AWS Well-Architected Framework). For now, ensure:
- Environment isolation (development/staging/production).
- Secrets injected via environment variables.
- HTTPS/TLS enforced at the load balancer or web server.
- Firewall rules restrict database access.
- Regular OS and runtime patching.

## Placeholder Import Summary
| Placeholder | Status | Resolution/Note |
|-------------|--------|-----------------|
| `{{MANICODE_CODE_QUALITY_PROMPT}}` | Resolved | Low complexity, separation of concerns, Larastan/Pint. |
| `{{MANICODE_API_SECURITY_PROMPT}}` | Resolved | OWASP REST Security Cheat Sheet linked. |
| `{{MANICODE_BACKEND_FRAMEWORK_PROMPT}}` | Resolved | Laravel 11 security features and docs. |
| `{{MANICODE_FRONTEND_FRAMEWORK_PROMPT}}` | UNRESOLVED | Frontend stack pending; placeholder with general rule. |
| `{{MANICODE_AUTH_PROMPT}}` | Resolved | AAL2, passkey considered, password complexity (min 12 chars). |
| `{{MANICODE_DEPLOYMENT_PROMPT}}` | UNRESOLVED | Deployment platform not confirmed; candidate technologies listed. |

## GDPR Compliance Checklist
- [ ] User consent mechanism (cookie banner) implemented.
- [ ] Data export endpoint for registered users (JSON/CSV of their posts, profile, comments).
- [ ] Account deletion endpoint that permanently removes or anonymizes personal data within 30 days.
- [ ] Privacy policy link in footer.
- [ ] Logs scrubbed of IP addresses and personal identifiers after 30 days.
- [ ] Data Processing Agreement (DPA) with any third-party services (CDN, storage) in place.

## Security Testing
All new features must include feature tests for authentication, authorization, and input validation. See also [Security Rules](#security-rules) §10 for CI/CD pipeline requirements.
- Run `composer audit` and `npm audit` on every PR.
- Periodically run OWASP ZAP baseline scan against the staging environment.
- Penetration testing prior to major releases.

---

**Document Version**: 1.1
**Last Updated**: 2026-06-04
**Approved By**: [TODO: stakeholder input needed]
**Next Review Date**: [TODO: stakeholder input needed]