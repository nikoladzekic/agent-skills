---
name: security-setup
description: >
  Set up a target application locally for security research by following its README and
  preferring Docker-based environments. Use this skill whenever the user wants to run a
  target app locally, spin up a test instance, or get a debuggable environment going.
  Triggers include: "set up this target", "get this app running locally", "dockerize this",
  "spin up a test instance", "I want to run this app for research", "set up the environment
  for this program", "help me install this target". Always prefer Docker. If no Dockerfile
  exists, synthesize one from the README's manual setup steps. Output a working environment
  the user can `docker compose up` and a SETUP.md explaining what was done and how to use it.
---

# Security Setup Skill

Stand up a locally-running, debuggable instance of a target application for security research.
Docker is strongly preferred — it keeps the researcher's machine clean, enables easy resets,
and allows instrumentation (proxying, debug flags, extra logging) without polluting the host.

---

## Input

User provides one or more of:
- GitHub repository URL
- A README or setup instructions (pasted or linked)
- A Docker Hub image name
- A zip/tarball of the source

If nothing is provided, ask for the repository URL before proceeding.

---

## Workflow

### Step 1 — Fetch and Read the Repository

1. Clone or fetch the repository:
   ```bash
   git clone [URL] /home/claude/target
   ```
2. Read the README (and any `docs/` setup guides):
   - `README.md`, `CONTRIBUTING.md`, `docs/development.md`, `docs/setup.md`
3. Inventory the repo structure:
   - Languages and frameworks used
   - Existing Docker/compose files
   - Database requirements
   - External service dependencies (Redis, S3, SMTP, OAuth provider)
   - Required environment variables
   - Seed data / fixture scripts

### Step 2 — Choose Setup Path

**Path A — Docker/Compose already exists**

If `Dockerfile` + `docker-compose.yml` (or `compose.yaml`) are present:
1. Read them carefully — note any gaps (missing env vars, unset secrets, hardcoded paths)
2. Create a `.env` file with sensible research defaults (weak secrets are fine — this is local)
3. Try `docker compose up --build`
4. Fix any build errors iteratively
5. Verify the app is reachable at its configured port

**Path B — No Docker, but manual setup exists in README**

Translate the manual steps into Docker:
1. Choose a base image matching the runtime (e.g., `node:20-alpine`, `python:3.12-slim`, `openjdk:21-jdk-slim`)
2. Write a `Dockerfile` that mirrors the README steps:
   - Install system deps
   - Copy source
   - Install app deps (`npm install`, `pip install -r requirements.txt`, `bundle install`, etc.)
   - Run migrations if applicable
   - Expose the right port
   - Set `CMD` / `ENTRYPOINT`
3. Write a `docker-compose.yml` that wires up any required services (Postgres, Redis, etc.)
4. Create `.env.research` with all required variables pre-filled with safe defaults
5. Build and verify

**Path C — No source, only a Docker Hub image**

1. Pull the image and inspect it: `docker inspect [image]`
2. Identify required env vars from docs/image description
3. Write a `docker-compose.yml` wrapping the image
4. Add any required sidecars (DB, cache)
5. Run and verify

### Step 3 — Research Instrumentation

After the app is running, apply research-friendly modifications:

**Proxy support**
- Add `HTTP_PROXY=http://host.docker.internal:8080` and `HTTPS_PROXY=...` to the env file
  so all outbound traffic can be intercepted by Burp/ZAP running on the host
- If the app does TLS verification internally, note this and provide the command to disable it
  or inject a custom CA cert

**Debug logging**
- Set log levels to maximum verbosity where configurable (e.g., `LOG_LEVEL=debug`, `DEBUG=*`)
- If the app has a debug mode flag, enable it

**Seed data / test accounts**
- Run any fixture/seed scripts the repo provides
- If none exist, note what the user needs to create manually (admin account, test org, etc.)
- Default credentials used: document them in SETUP.md

**Port exposure**
- Expose all relevant ports in compose (app, any internal admin ports, DB port for direct querying)
- Note if any ports are only available internally

**Easy reset**
- Confirm `docker compose down -v && docker compose up --build` produces a clean state

### Step 4 — Verify

Run basic smoke tests:
```bash
# App responds
curl -si http://localhost:[PORT]/ | head -5

# Check for tech stack headers
curl -si http://localhost:[PORT]/ | grep -i "x-powered-by\|server\|x-frame\|content-security"

# GraphQL endpoint (if applicable)
curl -s http://localhost:[PORT]/graphql -d '{"query":"{__typename}"}' -H 'Content-Type: application/json'
```

Document what you find.

### Step 5 — Write SETUP.md

````markdown
# Local Research Setup — [App Name]

## Prerequisites
- Docker Desktop (or Docker Engine + Compose plugin)
- [Any other host requirements]

## Quick Start
```bash
git clone [URL]
cd [dir]
cp .env.research .env
docker compose up --build
```
App runs at: `http://localhost:[PORT]`

## Default Credentials
| Role | Username | Password |
|------|----------|----------|
| Admin | ... | ... |
| User | ... | ... |

## Proxy Setup (Burp/ZAP)
Set your proxy to `127.0.0.1:8080`, then:
```bash
# Traffic is already routed via HTTP_PROXY in .env
# For HTTPS, import Burp's CA cert into the container:
[exact command if needed]
```

## Reset to Clean State
```bash
docker compose down -v && docker compose up --build
```

## Environment Variables
| Variable | Value | Notes |
|----------|-------|-------|
| ... | ... | ... |

## Architecture Notes
[Brief description of services, how they communicate, what DB is used, etc.]

## Research Notes
[Anything unusual about the setup, known quirks, endpoints to know about]
````

---

## Output

Files to save in `/home/claude/target/` (or working dir):
- `Dockerfile` (if synthesized)
- `docker-compose.yml` (if synthesized or modified)
- `.env.research` (pre-filled research defaults)
- `SETUP.md`

Present all created/modified files via `present_files`. Print the Quick Start block inline.

---

## Failure Handling

If the build fails after 2–3 fix iterations, stop and write a `SETUP_BLOCKED.md` that documents:
- What was attempted
- The exact error
- What manual steps the user should take to unblock
- Which step to resume from

Don't spin forever. A partial setup with clear documentation of the blocker is better than silence.