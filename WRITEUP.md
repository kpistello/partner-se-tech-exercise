# Write-up: Hardening the Todo App with Chainguard

## Summary

The customer's app ran on a general-purpose Debian image, as root, with Flask's debug server, unpinned packages from public PyPI, and an exposed database. I moved it to Chainguard's minimal Python images with a two-stage build, configured dependencies to install from Chainguard Libraries only, removed everything the app doesn't need, and hardened the Compose setup, while keeping `docker compose up --build` as the single command a developer runs.

Baseline testing also showed that **the starter app no longer starts at all** on a fresh build, because an unpinned dependency changed upstream. That is now fixed by naming the database driver explicitly, and prevented from recurring by locking every package, including indirect ones, to an exact version.

### Before and after

| Measurement | Before | After |
|---|---|---|
| Known vulnerabilities | **373** (61 high) | **0** |
| ...in the OS layer | 357 (57 high) | 0 |
| ...in Python packages | 16 (4 high) | 0 |
| App image size | 353 MB (85.2 MB compressed) | 163 MB (40.3 MB compressed), **−54%** |
| Runs as | root | non-root (`65532`) |
| Shell and extra tools | `sh`, `curl`, `vim`, `nc`, `pip` | none |
| Starts on a fresh build | **No**, crashes | Yes, both containers healthy |
| Web server | Flask dev server with `debug=True` | gunicorn |
| Database port | published on all interfaces | not published |
| App port | all interfaces | localhost only |
| Dependency pinning | direct packages only | every package |

*Scanned with Trivy 0.76.2 on November 24, 2026. The "after" image used packages from public PyPI because Chainguard Libraries access was blocked (see [Chainguard Libraries access](#chainguard-libraries-access)).*

## How to run

```bash
export CHAINGUARD_PYTHON_IDENTITY_ID=...
export CHAINGUARD_PYTHON_TOKEN=...
docker compose up --build
```

Then open http://localhost:8000. Database settings have local defaults and can be overridden by copying `.env.example` to `.env` (git-ignored).

For testing without Chainguard Libraries access only: `PIP_INDEX_URL=https://pypi.org/simple/ docker compose up --build`. The override exists because customers usually point builds at a repository manager that proxies Chainguard Libraries. The default remains Chainguard Libraries only, with no PyPI fallback.

## Working environment

I completed this exercise on a train between London and Edinburgh, over a mobile hotspot, where downloading Docker Desktop was estimated at 20 hours. Rather than wait, I switched to **GitHub Codespaces**, a cloud development environment with Docker preinstalled that runs in the browser, so all the downloading and building happened in GitHub's infrastructure instead of over my connection.

It also proved a useful test in its own right. The full workflow (Chainguard images, build secrets, and `docker compose up --build`) works unchanged in a cloud development environment, which matters for customers standardizing on them. Codespaces also surfaced the database port risk when it offered to make Postgres public (see [issues found](#issues-found-through-due-diligence-and-fixed)). One quirk: Codespaces secrets didn't reach the environment as expected, so I entered the Chainguard credentials per session with hidden input, never writing them to a file or shell history.

## What I changed and why

### Why this matters to the business

- **Less time lost to vulnerability triage.** Every scanner finding has to be investigated, fixed, or formally accepted. Going from 373 to 0 gives that engineering time back.
- **Easier compliance and faster security reviews.** Frameworks such as SOC 2, PCI DSS, and FedRAMP expect known vulnerabilities to be fixed within set timeframes, and enterprise buyers ask for clean scans. Starting from zero makes both easier.
- **Lower breach risk, and less damage if one happens.** Trusted packages narrow the supply chain attack surface, and a locked-down container leaves an attacker little to work with.
- **Reliable builds.** The starter broke without anyone touching its code. Locked inputs mean the same build every time, with upgrades as deliberate decisions.
- **Faster, cheaper delivery.** A 54% smaller image means quicker pulls in CI and deployment, and less storage and transfer.
- **Low cost to adopt.** Two small code edits and the same one-command workflow, so the pattern is practical to repeat across other services.

### 1. Container image (`app/Dockerfile`)

- **Chainguard Python images with a two-stage build.** Packages install in `python:latest-dev`; only the finished app is copied into the minimal `python:latest`, which has no shell or package manager.

  *Why it matters:* the unused Debian system accounted for 357 of the 373 findings. Not shipping it removes that patching work at the source.
- **Runs as non-root (65532), with app files root-owned.**

  *Why it matters:* an attacker who exploits the app gets its permissions. Without root, and unable to change the app's own files, they have far less to work with.
- **Removed `curl`, `vim`, and `netcat`.**

  *Why it matters:* downloading tools, editing files, and opening a connection back are an attacker's usual first moves after breaking in. These were a ready-made toolkit.
- **Removed pip from the shipped image.**

  *Why it matters:* a running app never needs to install packages, and pip's bundled setuptools was the source of the last 3 findings (see [issues found](#issues-found-through-due-diligence-and-fixed)).
- **Pinned both base images by digest.**

  *Why it matters:* `latest` changes over time. A digest guarantees every developer, CI run, and deployment uses the identical image, and updates arrive as reviewable changes.
- **Added `.dockerignore`.**

  *Why it matters:* it keeps files like `.env` out of the build, so real passwords can't end up in a shared image.

### 2. Dependencies (`app/requirements.in`, `app/requirements.txt`)

- **Chainguard Libraries only** (`https://libraries.cgr.dev/python/simple/`), with no fallback to public PyPI.

  *Why it matters:* public registries are a recurring attack route (look-alike names, hijacked accounts, tampered releases). Chainguard rebuilds packages from source, and with no fallback, a missing package stops the build instead of silently coming from PyPI.
- **Credentials passed as BuildKit secrets**, used only for the install step.

  *Why it matters:* build arguments, environment variables, or credentials typed into the Dockerfile leave the token readable in the image or repository. With secrets, it never persists.
- **Removed unused `requests` (which had known vulnerabilities) and `humanize`.**

  *Why it matters:* unused code still has to be scanned and patched, and it's still available to an attacker. Pure cost, no benefit.
- **Locked every package to an exact version**, with `requirements.in` listing the five direct dependencies and `requirements.txt` the full locked list.

  *Why it matters:* this is what fixed the broken app, and it makes next month's build identical to today's.
- **Replaced `psycopg2-binary` with `pg8000`.**

  *Why it matters:* `psycopg2-binary` bundles prebuilt system libraries that sit outside both the minimal image and Chainguard's rebuilt-from-source model. pg8000 is pure Python (see [tradeoffs](#tradeoffs)).
- **Upgraded gunicorn** from 21.2.0, which has known vulnerabilities, to 27.0.1, and actually used it.

  *Why it matters:* the web server faces every request, so its vulnerabilities are among the most reachable in the app.

### 3. Application runtime (`app/app.py`)

- **Removed `debug=True`.**

  *Why it matters:* debug mode turns error pages into a Python console that runs code on the server, guarded only by a PIN. Flask's documentation says never to use it in production.
- **Replaced the dev server with gunicorn** and removed `entrypoint.sh`, which needed a shell.

  *Why it matters:* Flask's built-in server isn't built for real traffic. gunicorn handles concurrent requests and restarts failed workers, so local behavior matches production.
- **Removed hardcoded database credentials**; the address comes from the environment and names its driver.

  *Why it matters:* credentials in code spread to every copy of the repository and can't be rotated without a code change. The explicit driver prevents another silent switch.

### 4. Local orchestration (`docker-compose.yml`)

- **The app waits for a healthy database** (`depends_on: condition: service_healthy`).

  *Why it matters:* otherwise startup is a race the app can lose, and flaky local setups push developers toward insecure workarounds.
- **Removed the published database port.**

  *Why it matters:* the database had a guessable password, and Codespaces even offered to make the port public. Now only the app can reach it.
- **Bound the app to localhost** (`127.0.0.1:8000`).

  *Why it matters:* by default, a published port is reachable from any network the laptop is on, such as office Wi-Fi or a coffee shop.
- **Runtime lockdown:** read-only filesystem (with a small `/tmp`), all Linux capabilities dropped, and `no-new-privileges`.

  *Why it matters:* each closes a common step in turning a bug into a full compromise: modifying files, using special permissions, or gaining more rights.
- **Health checks and automatic restarts** for both services, with the app's check written in Python since there's no shell.

  *Why it matters:* "running" isn't "working." These let Docker, and later production orchestration, detect and recover from failures.
- **Removed the `./app:/app` volume mount.**

  *Why it matters:* it replaced the built image's files at runtime, so what ran wasn't what was scanned and hardened.
- **Credentials as variables** with local defaults and `.env` overrides; removed the unused `SECRET_KEY`.

  *Why it matters:* configuration stays out of tracked files, and the unused setting implied a protection that didn't exist.
- **Pinned Postgres by digest** and removed the obsolete `version` attribute.

  *Why it matters:* the same reproducibility for the database, and no constant warning teaching people to ignore output.

## Risks reduced

| Risk | Before | After |
|---|---|---|
| Known vulnerabilities in the image | 373, including 61 high | 0 at time of scan |
| Code execution through Flask's debugger | debug mode on | dev server and debugger gone |
| Impact of a compromise | root, shell, attacker tooling | non-root, no shell or tools, read-only, no capabilities |
| Tampered or unexpected packages | public PyPI, mostly unpinned | Chainguard-built only, every version locked |
| Builds silently changing | indirect dependency drift broke the app | packages locked, images pinned by digest |
| Credential leakage | password in code and Compose | none in code; build secrets never stored; `.env` excluded |
| Database exposure | published on all interfaces | not published |
| Flaky startup | app could start before the database | waits for a healthy database |

## Issues found through due diligence and fixed

Measuring before changing, and re-testing after each change, surfaced problems that reading the code wouldn't have.

1. **The starter app no longer starts.** It crashed with `ModuleNotFoundError: No module named 'psycopg'`. SQLAlchemy wasn't pinned, and SQLAlchemy 2.1.0 (released September 24, 2026) changed the default driver for `postgresql://` addresses to `psycopg` (v3). **Fix:** every package locked, and the driver named explicitly (`postgresql+pg8000://`).
2. **The exposed database port wasn't theoretical.** Running the original app in GitHub Codespaces, Codespaces offered to make Postgres's port public, with the default `postgres/postgres` login. **Fix:** port no longer published.
3. **A hidden vulnerable package.** The migrated image still showed 3 findings in setuptools 70.3.0, although no setuptools files existed. The source was pip, which bundles part of setuptools. **Fix:** pip removed from the shipped image, giving 0 findings. One finding was Windows-only and the high one in a legacy feature the app never calls, but removing the package beat arguing about exploitability.
4. **gunicorn 27 writes to the home directory at startup** (a "control socket"), which fails on a read-only filesystem. **Fix:** disabled with `--no-control-socket`.
5. **Authentication failures looked like Dockerfile bugs.** On a 401, pip tried to prompt for a username and crashed with `EOFError`. **Fix:** `--no-input`, plus a log line saying whether credentials were provided, without revealing them. Because pip then reports a rejected login and a missing package the same way, I confirmed the cause with a direct request (HTTP 401).
6. **The startup race, verified fixed.** From a fresh database, the app waited about 6 seconds for Postgres before starting.

## Chainguard Libraries access

**Design.** The build defaults to `https://libraries.cgr.dev/python/simple/` with no fallback. Compose passes the identity ID and token from the shell into the build as secrets, written to a temporary in-memory `netrc` used only by pip. The build log confirms they arrive ("credentials: provided via build secrets").

**What happened.** Every request returned **HTTP 401**, from the build and directly. I ruled out expiry (valid until December 23, 2026), formatting (expected lengths, a single `/` in the ID), and the network (requests came from GitHub's infrastructure, and a 401 is a deliberate server response). The 401 carried no reason. I reported it to the exercise owner, who issued the credentials, in my submission email, with the request trace ID (`a9c8c169c75a5b380e9bf5761cbce646`, 16:11 UTC on November 24). The likely cause is a missing Python libraries pull role, or an ID and token that aren't a matching pair.

**Validation.** I built with public PyPI only to test everything else end to end. The committed default remains Chainguard Libraries only.

**Remaining once access works:** confirm every locked package is available from Chainguard Libraries (`pg8000`, `asn1crypto`, and `scramp` are the ones to watch), regenerate `requirements.txt` with hashes against Chainguard's index (rebuilt packages have different hashes than PyPI's), install with `--require-hashes`, and re-scan.

## Tradeoffs

- **pg8000 instead of psycopg.** The minimal runtime lacks `libpq`, which psycopg needs. pg8000 is pure Python, at the cost of performance that doesn't matter here but would for a high-traffic service.
- **One gunicorn worker with four threads.** Multiple workers would each try to create the tables on a fresh database and could collide. The long-term fix is migrations.
- **Digest pinning vs. automatic updates.** Pinned images don't pick up Chainguard's security rebuilds until updated, which is only acceptable with automated update pull requests.
- **Kept the official `postgres:15` image.** The exercise targets the app container, and moving to Chainguard's Postgres likely means a major-version upgrade that needs a planned data migration.
- **Default local database credentials in Compose.** I kept `docker compose up --build` working with no setup step. The risk is low because the database is unpublished, and `.env` can override them.
- **No hot-reload developer mode.** Developers rebuild to see changes, rather than an auto-loaded override file that would quietly undo the hardening.
- **Hashes deferred** until Chainguard Libraries access works.

## What I'd do next in a real customer engagement

1. **Finish the dependency work** once access is restored: availability check, hashed lock file, `--require-hashes`.
2. **Route packages through the customer's repository manager** (Artifactory, Nexus, or Cloudsmith) with Chainguard Libraries as an upstream, so developers and CI don't each hold tokens.
3. **Use short-lived credentials in CI** (an OIDC assumable identity), and fail builds on new high or critical findings.
4. **Automate digest updates** with digestabot or Renovate.
5. **Verify what runs:** check Chainguard's image signatures and SBOMs with cosign, and enforce signed images at deploy time.
6. **Move the database to Chainguard's Postgres** with a planned migration.
7. **App follow-ups:** real migrations (for example, Alembic) instead of `db.create_all()`, CSRF protection on the forms, and clearing deprecation warnings.
8. **Agree on success measures:** vulnerability counts over time, triage time, and time to roll out a fixed base image.

## How a partner would repeat this

A partner (reseller, consultancy, or systems integrator) can turn this into a repeatable offering across a customer's services.

**The playbook:**

1. **Baseline:** scan, size, user, tools, and a clean build from scratch, with scanner version and date.
2. **Two-stage build** on Chainguard images, non-root, pinned by digest.
3. **Chainguard Libraries** via build secrets, no public fallback, everything locked with hashes from Chainguard's index.
4. **Remove what the app doesn't need**, including the package installer.
5. **Harden the runtime:** production server, read-only filesystem, dropped capabilities, health checks, no published internal services.
6. **Re-measure** with the same scanner and present the before/after.

**Known issues to plan for** (details in [issues found](#issues-found-through-due-diligence-and-fixed)): database drivers needing system libraries; pip's bundled setuptools; PyPI hashes not matching Chainguard's; no shell for entrypoints or health checks; apps writing files at startup on a read-only filesystem; unpinned indirect dependencies; confirming package availability early; and Compose override files that load automatically and can undo hardening.

**Where the partner adds value:** assessing and prioritizing services; platform setup (repository manager, CI credentials, digest automation, policy enforcement); golden Dockerfile and Compose templates so each service is faster than the last; and ongoing updates, reporting, and audit evidence.

## AI tool usage

**Tool:** Claude (Anthropic), through claude.ai, throughout the exercise.

**Where:** understanding the exercise and planning; setting up my environment (including moving to GitHub Codespaces when the Docker Desktop download was too slow); reviewing the starter repo; drafting the Dockerfile, Compose, and requirements changes; troubleshooting the crash, the 401, the setuptools finding, and git; and drafting this write-up from my notes.

**How I separated AI suggestions from my own judgment:**

- **I verified by running.** Every number here comes from commands I ran on the original and updated images. I treated the starter-repo review as hypotheses and confirmed each by testing.
- **AI was wrong in places, and testing caught it.** Claude predicted the crash was a timing problem; the logs showed the driver change. It once misread a failed local scan as clean, and a suggestion to read the token's audience didn't help diagnose the 401. Each was resolved by checking actual output.
- **Decisions were mine.** Claude recommended pg8000 and a configurable package source for testing; I evaluated and accepted both. I kept the hardened setup as the default rather than add a dev-convenience override.
- **AI-run tests.** Claude tested the package set on Python 3.15 against Postgres before I applied the changes, which surfaced the gunicorn issue early. I confirmed everything in my own builds.
- **Cleanup.** Claude reviewed my pushed files and flagged leftover commented-out code from copy-pasting, including an old hardcoded password and `debug=True`. I removed it and re-tested.
