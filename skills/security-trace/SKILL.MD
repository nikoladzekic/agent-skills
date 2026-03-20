---
name: security-trace
description: >
  Trace data flow through source code from user-controlled sources to dangerous sinks, then
  generate comprehensive parsers/PoC payloads based on the gadgets found. Use this skill
  whenever the user wants to audit a specific feature in source code, trace taint flow,
  find injection points, or turn a code-level finding into a working exploit or test harness.
  Triggers include: "trace this feature", "find the sink for X", "what are the sources for Y",
  "build a parser for this vuln", "trace the data flow", "audit this code path", "find gadgets
  for X", "generate payloads based on what you found", "source to sink analysis". Always use
  this skill for any substantive code auditing task — don't freehand it.
---

# Security Trace Skill

Given a feature name or entry point, systematically trace data from user-controlled **sources**
to dangerous **sinks**, document intermediate **gadgets** (transformations, sanitizers bypassed,
logic branches), and produce a **parser** — a structured output that maps the full attack surface
and generates targeted test payloads.

---

## Input

User provides:
- Feature name or description (e.g. "the webhook delivery system", "file import", "search API")
- Source code location (local path, GitHub URL, or already-cloned repo)
- Optionally: a specific suspected vulnerability class to focus on

If the code isn't already local, fetch it first (`git clone` or `web_fetch`).

---

## Concepts

**Source**: Any location where attacker-controlled data enters the application.  
**Sink**: Any location where data is used in a dangerous way (executed, rendered, passed to OS, etc.).  
**Gadget**: Any transformation, branch, or logic between source and sink that affects exploitability —
sanitizers, type coercions, string operations, encoding/decoding, permission checks that can be
bypassed, conditional branches controlled by attacker input.  
**Parser** (in this skill's context): A structured document + payload set that operationalizes
the trace — turning the code-level finding into a test harness or PoC.

---

## Workflow

### Step 1 — Orient

1. Identify the feature's entry point(s) in the codebase:
   - Search for route definitions, controller methods, GraphQL resolvers, RPC handlers
   - Commands: `grep -rn "route\|@app\.\|Router\|resolver\|handler" --include="*.py" .`
   - For GraphQL: find schema definitions and resolver mappings
2. List all HTTP parameters, headers, body fields, or other inputs the feature accepts
3. Note the language/framework — this determines which sink patterns to look for

### Step 2 — Source Enumeration

Enumerate all sources of attacker-controlled input reaching this feature:

**Common source patterns by layer:**

| Layer | Sources to check |
|---|---|
| HTTP | Query params, path params, body (JSON/form/multipart), headers (Host, X-Forwarded-*, Content-Type), cookies |
| GraphQL | Argument values, input type fields, directives, operation name |
| File | Filename, file content, MIME type, metadata fields |
| Env/Config | User-editable config files, environment vars set by user |
| Stored | DB fields written by attacker in a prior request (for stored vulns) |
| Headers | Forwarded-For, Origin, Referer, Authorization, custom headers |

For each source, note:
- Parameter name
- Type constraints enforced (if any)
- Where in code it's first read
- Whether it's validated at the boundary or passed raw

### Step 3 — Sink Identification

Search the code path for dangerous sink patterns. Use language-specific grep patterns:

**Universal patterns to grep:**
```bash
# Execution sinks
grep -rn "eval\|exec\|spawn\|popen\|system\|subprocess\|Runtime.exec\|ProcessBuilder" .

# Template/render sinks
grep -rn "render\|template\|jinja\|mustache\|handlebars\|ejs\|erb\|twig\|smarty" .

# DB sinks (look for raw query construction)
grep -rn "query\|execute\|raw\|format.*SELECT\|f\"SELECT\|f'SELECT\|\${\|string.*sql" .

# File sinks
grep -rn "open(\|write(\|readFile\|writeFile\|path.join\|os.path.join\|File(" .

# HTTP/SSRF sinks
grep -rn "requests.get\|fetch(\|http.get\|urllib\|axios\|curl\|HttpClient\|WebClient" .

# Reflection sinks (deserialization, dynamic dispatch)
grep -rn "pickle\|marshal\|yaml.load\|unserialize\|ObjectMapper\|readObject\|__import__" .

# HTML/JS sinks
grep -rn "innerHTML\|outerHTML\|document.write\|eval(\|dangerouslySetInner" .
```

For each sink found on the code path:
- Record file, line number, sink function
- Note what variable feeds into it
- Trace that variable backwards toward the source

### Step 4 — Gadget Tracing (Source → Sink Path)

Walk the data flow between each source–sink pair. Document every transformation:

For each gadget (transformation step), record:
```
GADGET: [file:line]
  Input var:  [variable name and type]
  Operation:  [what happens — sanitize, encode, cast, branch, etc.]
  Output var: [resulting variable]
  Bypass potential: [yes/no — and how]
  Notes: [anything interesting]
```

**High-value gadget patterns to look for:**

| Gadget Pattern | Why It Matters |
|---|---|
| `htmlspecialchars` / `escape()` on wrong context | Bypassed if output context is JS, attribute without quotes, or CSS |
| Path normalization missing | `../` traversal if `realpath` not called before open |
| `allowlist` check before transformation | If transformation happens after check, bypass possible |
| Type cast then use | `int(x)` then back to string in SQL — integer overflow or truncation |
| Partial sanitizer | Only strips `<script>` but not other event handlers |
| Trust boundary via stored data | Stored input sanitized at write time but used unsanitized at read time |
| `isAdmin` check on wrong object | Checks role on requester but operates on target object |
| Redirect after auth check | Auth check returns response, but code continues executing |
| Second-order sink | Input stored, later rendered/executed in different context |

### Step 5 — Build the Trace Graph

For each exploitable path, produce a trace block:

```
=== TRACE: [Vuln Class] via [Feature Name] ===

SOURCE
  Location : [file:line]
  Parameter: [name]
  Reaches  : [first gadget or directly to sink]

GADGETS (in order)
  1. [file:line] — [operation] — bypass: [yes/no — reasoning]
  2. [file:line] — [operation] — bypass: [yes/no — reasoning]
  ...

SINK
  Location : [file:line]
  Function : [dangerous function]
  Input    : [variable name feeding the sink]
  Exploitable under: [conditions — e.g. "when Content-Type is text/html", "when admin=true"]

IMPACT
  [What an attacker can achieve if exploited]

CONFIDENCE: [High / Medium / Low]
  [Reasoning — did you trace every step, or are there gaps?]
```

### Step 6 — Generate Parsers / Test Payloads

For each confirmed or high-confidence trace, generate a **parser**: a targeted payload set
and test procedure that operationalizes the finding.

#### Parser structure:

```markdown
## Parser: [Vuln Class] — [Feature]

### Endpoint
[METHOD] [path]
Content-Type: [type]

### Minimal Reproduction Request
[Raw HTTP or curl command — fully parameterized]

### Payload Variants
[A table or list of payloads to try in order, from safe detection to impact escalation]

| # | Payload | Purpose | Expected indicator of success |
|---|---------|---------|-------------------------------|
| 1 | [canary/detection] | Confirm injection point | [what to look for] |
| 2 | [escalation] | Measure impact | [what to look for] |
| 3 | [full PoC if confident] | Confirm exploitability | [what to look for] |

### Bypass Variants
[Encoding tricks, case variations, alternative syntax — derived from gadgets found]

### How to Confirm (Blind)
[If no visible output — OOB callback instructions, timing, side-channel]

### Scope of Impact
[What other users/data are affected — self-only vs stored vs systemic]
```

#### Payload generation rules by vuln class:

**XSS**
- Start with `<img src=x onerror=alert(1)>` for detection
- If filtered: try SVG, event handler variants, mXSS sequences
- If markdown: `[text](javascript:alert(1))`, `![x](x "onmouseover=alert(1)")"`
- Always include a stored vs reflected distinction if applicable

**SSRF**
- Use a Burp Collaborator or interactserver.io URL for blind detection
- Variants: `http://`, `https://`, `file://`, `dict://`, `gopher://`
- Internal targets to probe: `169.254.169.254` (AWS metadata), `localhost`, `0.0.0.0`

**SQLi**
- Detection: `'`, `''`, `' OR '1'='1`, time-based `'; WAITFOR DELAY '0:0:5'--`
- Adapt to DB type based on stack hints from recon

**SSTI**
- Detection: `{{7*7}}`, `${7*7}`, `<%= 7*7 %>`, `#{7*7}`, `*{7*7}`
- Escalation depends on engine (Jinja2 vs Twig vs Pebble vs Mako)

**Path Traversal**
- `../../../etc/passwd`, URL-encoded variants, double-encoded, null byte
- Windows: `..\`, `..%5C`

**RCE via deserialization**
- Use ysoserial (Java), phpggc (PHP), pickle-gen (Python) with OOB callback first
- Don't detonate blind — confirm via sleep/DNS first

**GraphQL-specific**
- Introspection: `{"query":"{__schema{types{name}}}"}`
- Batch: array of queries `[{...},{...}]`
- Alias-based batching for rate limit bypass
- Field suggestion probing for non-introspectable endpoints

---

### Step 7 — Write TRACE_[FEATURE].md

Save all traces and parsers to `TRACE_[FEATURE_NAME].md`. Structure:

```markdown
# Trace Report — [Feature Name]

**Date**: [date]  
**Target**: [app name]  
**Feature**: [description]  
**Analyst**: Claude (security-trace skill)

---

## Summary

[2–3 sentences: what was found, confidence level, highest-impact path]

---

## Sources Identified
[Table of all sources found]

## Sinks Identified
[Table of all sinks found on the code path]

## Traces

[One trace block per source→sink path, as described in Step 5]

---

## Parsers

[One parser block per trace, as described in Step 6]

---

## Dead Ends

[Paths that looked promising but were ruled out — and why. This is important for
not re-doing work in follow-up sessions.]

---

## Recommended Next Steps

[What to manually verify, what to set up in Burp, what to run with the local instance]
```

---

## Output

Save `TRACE_[FEATURE].md` and present via `present_files`. Print the summary and the first
parser inline so the user can act immediately.

---

## Notes on Confidence

Never over-claim. If you couldn't follow the full path (e.g. compiled dependency, external
service, dynamic dispatch you can't resolve statically), mark confidence as **Medium** or **Low**
and explain the gap. A precise Medium-confidence finding with a clear testing procedure is
more valuable than an overconfident High that wastes the researcher's time.