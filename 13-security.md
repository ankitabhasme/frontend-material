# Frontend Security

> XSS, CSRF, CORS, CSP, auth, token storage, and supply-chain — the threats a frontend owns and how to defend in depth.

## XSS (Cross-Site Scripting)
- **What:** attacker injects script that runs in a victim's browser in your origin's context (steals tokens, hijacks sessions).
- **Types:** stored, reflected, DOM-based.
- **React defense:** JSX **auto-escapes** interpolated values → strong default protection.
- **Danger zones:** `dangerouslySetInnerHTML` (sanitize with **DOMPurify**), injecting into `href`/`src` (`javascript:` URLs), `eval`, setting raw HTML, third-party scripts.
- **Defense in depth:** **Content Security Policy (CSP)** to restrict script sources and block inline scripts; sanitize/validate all input; encode on output for the correct context.

## CSRF (Cross-Site Request Forgery)
- **What:** a malicious site makes the victim's browser send an authenticated request to your app using their cookies.
- **Defenses:** **SameSite cookies** (`Lax`/`Strict`), **anti-CSRF tokens** (synchronizer/double-submit), checking `Origin`/`Referer`, and preferring non-cookie auth (Authorization header) for APIs.

## Cookies & Token Storage
- **httpOnly** — JS can't read it → mitigates XSS token theft. **Secure** — HTTPS only. **SameSite** — CSRF mitigation.
- **localStorage tokens are XSS-vulnerable** (readable by any script). Preferred: **httpOnly, Secure, SameSite cookies** for session/refresh tokens; keep short-lived access tokens in memory if needed.
- **JWT** — stateless; hard to revoke → keep short-lived + refresh tokens + rotation; validate signature/expiry server-side. Never trust client-side claims for authorization.

## CORS
- Browser same-origin policy blocks cross-origin reads by default. **CORS** is the server opting in via `Access-Control-Allow-Origin` etc. Common misconception: CORS is *server-controlled* and is **not** itself a feature that protects your API — it relaxes SOP. Don't use `*` with credentials. Preflight (`OPTIONS`) for non-simple requests.

## Content Security Policy (CSP)
- HTTP header (`Content-Security-Policy`) restricting where scripts/styles/images/connects can load from → strong XSS mitigation.
- Prefer **nonces/hashes** over `'unsafe-inline'`; avoid `'unsafe-eval'`. Use `report-uri`/`report-to` to monitor. Also `frame-ancestors` (clickjacking), `upgrade-insecure-requests`.

## Other headers & clickjacking
- `X-Frame-Options` / CSP `frame-ancestors` (clickjacking), `Strict-Transport-Security` (HSTS), `X-Content-Type-Options: nosniff`, `Referrer-Policy`, `Permissions-Policy`.
- Subresource Integrity (**SRI**) for third-party scripts.

## Auth patterns
- OAuth 2.0 / OIDC, **PKCE** for public (SPA/mobile) clients, avoid the implicit flow. Silent refresh, token rotation. Enforce **authorization on the server** — never rely on hiding UI.

## Supply chain
- `npm audit`, lockfiles + integrity hashes, `npm ci`, dependency pinning, Dependabot/Renovate, minimize deps, verify provenance, watch for typosquatting and compromised packages. Sandbox untrusted third-party scripts.

## Other frontend risks
- **Sensitive data exposure** — never put secrets in client bundles; anything shipped to the browser is public.
- **Open redirects**, **prototype pollution**, **ReDoS**, insecure `postMessage` (always check `origin`), leaking data via `iframe`/referrer.

---

### Interview Questions — Security

**Q1. How does React protect against XSS, and where can it still happen?**
> JSX auto-escapes interpolated text, neutralizing most injection. It can still happen via `dangerouslySetInnerHTML` (sanitize with DOMPurify), unsafe URLs (`javascript:` in href/src), non-escaped sinks, and third-party scripts. Add CSP as defense in depth.

**Q2. Where should you store auth tokens in an SPA?**
> Prefer httpOnly, Secure, SameSite cookies so XSS can't read them; pair with CSRF defenses (SameSite + tokens). localStorage is convenient but XSS-exposed. If using in-memory access tokens, keep them short-lived with refresh rotation. Never trust the client for authorization.

**Q3. Explain CORS — is it a security mechanism for my API?**
> CORS relaxes the browser's same-origin policy so a server can opt in to cross-origin reads. It's enforced by the browser and controlled by the server, but it does **not** protect your API from non-browser clients — server-side authz/validation still must. Don't combine `*` with credentials.

**Q4. CSRF vs XSS?**
> XSS runs attacker script in your origin (injection). CSRF tricks the browser into sending authenticated requests using existing cookies (no script needed). XSS defenses: escaping/sanitization + CSP. CSRF defenses: SameSite cookies, anti-CSRF tokens, Origin checks.

**Q5. What does a good CSP look like?**
> Restrict `default-src`/`script-src` to trusted origins, use nonces/hashes instead of `'unsafe-inline'`, forbid `'unsafe-eval'`, set `frame-ancestors 'none'`, `upgrade-insecure-requests`, and enable reporting to detect violations before enforcing strictly.

**Q6. How do you secure the dependency supply chain?**
> Commit lockfiles with integrity hashes, use `npm ci`, run `npm audit`/Snyk in CI, automate updates (Dependabot/Renovate), pin and minimize dependencies, watch for typosquatting, and use SRI for third-party scripts.
