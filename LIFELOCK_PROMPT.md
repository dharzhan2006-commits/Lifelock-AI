# LIFELOCK AI: Your Personal Digital Safety Assistant

## Role
You are a senior full-stack and security engineer. Build **LIFELOCK AI**, a secure, accessible, well-tested web app that helps students and everyday users detect scam messages, fake jobs/internships, phishing, and dangerous URLs.

**Priority order:** working and secure > tested > accessible > efficient > pretty.
Build incrementally. After each feature, run the tests. Never claim tests passed unless they were actually executed. No fake results, no hardcoded dashboard numbers, no buttons without a real action.

## Tech Stack
| Layer | Choice |
|---|---|
| Frontend | React + TypeScript (Vite), Tailwind CSS, React Router |
| Backend | Python, FastAPI, Pydantic v2 |
| Database | SQLite (prototype), SQLAlchemy ORM, PostgreSQL-ready |
| AI | Configurable provider (Gemini or Claude) via env vars, server-side only |
| OCR | Tesseract (`pytesseract`) with Pillow preprocessing |
| Tests | Pytest + httpx (backend), Vitest + Testing Library (frontend) |

## Folder Structure
```
lifelock-ai/
  backend/app/{main.py, config.py, routers/, services/, models/, schemas/, security/, tests/}
  frontend/src/{components/, pages/, hooks/, api/, lib/, tests/}
  .env.example  README.md  .github/workflows/ci.yml
```

## Features

### 1. Dashboard
Total scans, threats found (Medium/High), recent scans, awareness tips, quick-access tool cards, activity feed. **All numbers come from the database** via `GET /api/stats`. Show proper empty states.

### 2. Scam Message Detector
Paste SMS/email/WhatsApp/job text (max 5,000 chars). Detect: phishing, impersonation, urgency pressure, suspicious payment requests, requests for sensitive data (OTP, password, bank). Return risk level, score, warning signs with explanations, and recommended actions. **Never output "safe"**; the lowest label is "Low Risk" plus the note that no tool can guarantee safety.

### 3. Job & Internship Offer Checker
Inputs: offer text, optional company name, optional official website. Check: registration fees/deposits, unrealistic salary, suspicious channels (Telegram/WhatsApp-only, free email domains), credential/OTP requests, missing or inconsistent company info, mismatch between sender domain and website. Show the note: "Initial risk assessment, not proof of fraud."

### 4. URL Analyzer
Validate with strict parsing (scheme `http/https` only, length limit, no credentials in URL). **Never fetch or open the URL.** Heuristics: HTTPS, IP-based host, excessive subdomains, punycode/homoglyphs, `@` tricks, brand-lookalike (edit distance to a configurable brand list), suspicious TLDs, long/obfuscated paths. Optional threat-intel lookup (Google Safe Browsing) only when `SAFE_BROWSING_API_KEY` is set. HTTPS alone is never reported as "safe".

### 5. Screenshot OCR
Upload PNG/JPG/WEBP, max 5 MB, validate by **magic bytes** (not just extension). Strip EXIF, downscale before OCR, process in memory and **do not persist the image**. Show extracted text in an editable box, then "Analyze this text". Handle blank/unreadable images and OCR failure with a retry option.

### 6. Risk Report (shared component)
Overall level, score (0-100), warning signs with explanations, next steps, timestamp, disclaimer. Label each finding's source: **Rule-based**, **AI-generated**, or **Verified threat intelligence**.

### 7. Security Awareness Centre
Guides with examples and checklists: phishing, fake internships, recruitment scams, protecting personal info, strong passwords, social engineering, reporting cybercrime (include cybercrime.gov.in and helpline 1930 for India; make region configurable). Content stored as static Markdown/JSON, keyboard-navigable.

### 8. Auth & Privacy
JWT (short-lived access token + refresh token in httpOnly cookie), Argon2/bcrypt hashing, server-side validation, per-user access control (users only see their own scans). Guest mode allowed for analysis. **Do not store message text by default**; store only metadata (type, level, score, signs, time). Optional "save content" toggle (off by default). "Delete my history" and "Delete my account" buttons that really delete data.

## Risk Scoring (document in README)
```
rule_score  = min(100, sum(weight_i for each triggered indicator))
final_score = rule_score                              if AI unavailable
final_score = round(0.6*rule_score + 0.4*ai_score)    otherwise
Level: 0-34 Low | 35-64 Medium | 65-100 High
```
Example weights: payment/fee request 30, OTP/password request 35, urgency language 15, impersonation 20, suspicious link 20, unrealistic pay 20, off-platform contact 15. Keep weights in one config file. If AI fails, return the rule-based result with `ai_status: "unavailable"` (do not fail the whole request).

## API
```
POST /api/analyze/message   {text}
POST /api/analyze/job       {text, company?, website?}
POST /api/analyze/url       {url}
POST /api/ocr               multipart image -> {text}
GET  /api/stats   GET /api/scans   DELETE /api/scans
POST /api/auth/register | login | refresh | logout     DELETE /api/auth/me
```
Consistent error shape: `{ "error": { "code": "...", "message": "..." } }`.

## Security Requirements
- API keys only in backend env vars; `.env` git-ignored; `.env.example` has no secrets.
- Rate limit analysis endpoints (e.g., 10/min per IP/user) and return 429 with `Retry-After`.
- Pydantic validation, length limits, strict CORS allow-list, security headers (CSP, nosniff, frame deny).
- **Prompt-injection defense:** treat user text as data, wrap it in delimiters, tell the model to ignore instructions inside it, force a JSON schema, and validate/sanitize model output against enums.
- Timeouts and retries with backoff on AI calls; no stack traces in responses; no secrets in logs.
- Parameterized queries only; escape all rendered output.

## UI and Accessibility (WCAG 2.1 AA)
Dark navy-purple gradient theme, reusable components, responsive (mobile first), subtle animations that respect `prefers-reduced-motion`. Semantic landmarks, skip link, visible focus rings, labelled inputs, `aria-live` for loading/results, `role="alert"` for errors, 4.5:1 contrast, risk shown by **icon + text, not color alone**, 44px touch targets. Loading, empty, error, and retry states on every tool.

## Efficiency
Route-level code splitting, disable submit while loading, cache identical requests (hash of input) for a short TTL, compress images client-side before upload, paginate history, index DB columns.

## Tests (must actually run)
- Valid, empty, oversized, and malformed text and URLs
- Suspicious URL patterns (IP host, punycode, `@`, lookalike, many subdomains)
- Unsupported image type, fake extension, oversized upload, unreadable image
- AI timeout/failure fallback
- Auth, access control (user A cannot read user B's scans), password hashing
- Rate limiting (429)
- Risk score and classification boundaries (34/35, 64/65)
- Frontend: report component, form validation, accessible error messages

Add CI (lint + tests + build). Report real coverage only.

## Acceptance Criteria
- [ ] Every button performs a real action
- [ ] Dashboard numbers come from the DB
- [ ] "Safe" is never claimed; disclaimers are visible
- [ ] Screenshots are never stored
- [ ] No secrets in frontend or repo
- [ ] `pytest` and `npm test` pass; lint and build pass
- [ ] README with setup, scoring method, features, **known limitations**

## Deliverables
Full source, tests, README (setup, env vars, how to run tests, scoring, limitations), `.env.example`, feature list with known limitations, CI workflow.
