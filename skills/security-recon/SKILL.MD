---
name: security-recon
description: >
  Perform an initial security reconnaissance pass over a bug bounty or research target program.
  Use this skill whenever the user provides a program URL, HackerOne/Bugcrowd scope, or a target
  application and wants a structured security-focused overview. Triggers include: "do a recon pass",
  "initial review of this program", "what should I look at in this target", "create OVERVIEW.md",
  "analyze this program for bugs", "enumerate attack surface". Outputs a structured OVERVIEW.md
  with per-feature descriptions and associated vulnerability hypotheses. Always use this skill
  when starting work on a new target — don't wing it from memory.
---

# Security Recon Skill

Produce a structured `OVERVIEW.md` for a target program from the eyes of a senior security researcher.
Goal: capture every feature surface in 1–2 sentences, then map each feature to its most likely
vulnerability class(es), creating a prioritized attack map before touching any code.

---

## Input

The user will provide one or more of:
- A program URL (HackerOne, Bugcrowd, self-hosted VDP, or direct app URL)
- A scope definition (in-scope domains/endpoints)
- A product description or README
- Already-fetched docs/scope text pasted inline

If none of these are provided, ask the user for the target before proceeding.

---

## Workflow

### Step 1 — Collect Program Documentation

Fetch all available program documentation. In order of preference:
1. HackerOne program page (`https://hackerone.com/[program]/policy`) — fetch scope, out-of-scope, and feature descriptions
2. Bugcrowd program brief
3. Target's own `README.md`, `/docs`, or public API docs
4. Marketing/feature pages of the live app (e.g. `/features`, `/pricing`, `/api`)

Use `web_fetch` or `web_search` to pull these down. If authentication is required and you can't access the content, note the gap and proceed with what's available.

Extract:
- In-scope assets (domains, endpoints, mobile apps, APIs)
- Out-of-scope items (note them but don't analyze)
- Feature list (user-facing and internal)
- Technology hints (language, framework, cloud provider, auth method)

### Step 2 — Feature Enumeration

For each distinct feature or surface area identified, write:
- **1–2 sentence functional description** — what it does from a user perspective
- **Likely tech stack** — inferred from docs, job listings, or HTTP headers if accessible

Group features into categories:
- Authentication & Session Management
- User Input & Data Submission (forms, uploads, search, filters)
- API Endpoints (REST/GraphQL/RPC)
- File Handling (upload, download, export, import)
- Webhooks & Integrations (outbound HTTP, OAuth flows, third-party connectors)
- Admin / Internal Tooling
- Real-time Features (WebSockets, SSE, long-polling)
- Template / Rendering Engine (email templates, PDF generation, report builder)
- Scripting / Eval surfaces (custom formulas, Lua/JS sandbox, scheduled jobs)
- Infrastructure-Facing (SSRF vectors, metadata access, DNS rebinding candidates)

### Step 3 — Vulnerability Hypothesis Mapping

For each feature, assign one or more vulnerability hypotheses using the mapping table below.
Be specific — don't just write "XSS"; write "stored XSS via markdown renderer if sanitization
is applied client-side only."

**Core mapping table** (extend as appropriate):

| Feature Type | Primary Vuln Hypotheses |
|---|---|
| Rich text / markdown input | Stored XSS, HTML injection, mXSS via parser differential |
| File upload | Stored XSS (SVG/HTML), path traversal on filename, RCE via polyglot, SSRF if URL fetch |
| URL / domain input | SSRF, open redirect, DNS rebinding |
| PDF / screenshot generation | SSRF via headless browser, local file read via `file://` |
| Template engine (user-controlled) | SSTI — check Jinja2, Twig, Pebble, Handlebars, Smarty |
| GraphQL API | Introspection leak, batch query abuse, IDOR via node IDs, field-level auth bypass |
| REST API | IDOR, mass assignment, HTTP verb confusion, broken object-level auth |
| OAuth / SSO | State fixation, redirect_uri bypass, token leakage via Referer |
| Webhooks (outbound) | SSRF, credential leak in headers, replay attacks |
| Admin panel | Broken access control, privilege escalation, auth bypass via role confusion |
| Scripting / formula eval | RCE, sandbox escape, XXE if XML-based |
| Import (CSV/XML/JSON) | XXE, CSV injection, deserialization |
| WebSocket | Authentication not re-checked after upgrade, message injection, DoS |
| Password reset | Token predictability, host header injection, token reuse |
| Search / filter params | SQLi, NoSQLi, LDAP injection, ReDoS |
| Export (PDF/Excel/HTML) | Stored XSS in export, formula injection (Excel), local file inclusion |
| Email rendering | Header injection, phishing amplification |
| Multi-tenancy / org isolation | Tenant isolation bypass, cross-org IDOR |

### Step 4 — Prioritization

Score each feature on rough exploitability × impact:
- **P0 — Investigate First**: RCE candidates, auth bypass, SQLi, SSRF with internal network access
- **P1 — High Value**: Stored XSS with wide blast radius, IDOR chains, privilege escalation
- **P2 — Worth Checking**: Reflected XSS, open redirect, info disclosure
- **P3 — Low Signal**: Self-XSS, rate limiting, minor info leaks

### Step 5 — Write OVERVIEW.md

Output a single `OVERVIEW.md` file with the structure below.

---

## OVERVIEW.md Template

```markdown
# Security Overview — [Program Name]

**Date**: [date]  
**Scope**: [summarize in-scope assets in one line]  
**Stack Hints**: [language/framework/infra observations]  
**Out of Scope** (skipped): [brief list]

---

## Attack Surface Summary

[2–3 sentence exec summary of the most interesting areas and why]

---

## Feature Map

### [Category Name]

#### [Feature Name]
**What it does**: [1–2 sentences]  
**Stack hint**: [if known]  
**Vuln hypotheses**:
- [Specific hypothesis 1 — be precise, not generic]
- [Specific hypothesis 2]

**Priority**: P[0–3] — [one-line rationale]

---
[repeat for each feature]

---

## Quick Win Candidates

Ordered list of the 3–5 highest-priority things to look at first, with the specific test
that would confirm or rule out the hypothesis fast.

| # | Feature | Hypothesis | Fast Test |
|---|---------|------------|-----------|
| 1 | ... | ... | ... |

---

## Technology & Recon Notes

- [Any stack clues from HTTP headers, JS bundles, job listings, etc.]
- [Known CVEs or public writeups for this tech stack — link if found]
- [Relevant HackerOne disclosed reports for this program or similar ones]
```

---

## Output

Save the completed file as `OVERVIEW.md` in the working directory and present it to the user
via `present_files`. Also print the "Quick Win Candidates" table inline in the conversation
so the user sees the priorities immediately without opening the file.

---

## Quality Checks Before Saving

- Every feature has at least one specific (not generic) vuln hypothesis
- Priority ratings are assigned to all features
- Quick Win table has ≥3 entries with concrete fast-test descriptions
- Out-of-scope items are listed but not analyzed
- No placeholders left unfilled