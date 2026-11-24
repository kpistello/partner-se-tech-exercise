# Write-up: Hardening the Todo App with Chainguard

## Summary

The customer's app worked on paper but was built on a general-purpose Debian image, ran as root with Flask's debug server, pulled unpinned packages from public PyPI, and exposed its database. I moved the app container to Chainguard's minimal Python images with a two-stage build, configured dependency installation to use Chainguard Libraries only, removed everything the app doesn't need, and tightened the Docker Compose setup, while keeping `docker compose up --build` as the single command a developer runs.

Along the way, baseline testing showed that **the starter app no longer starts at all** on a fresh build, because an unpinned dependency changed its behavior upstream. That is now fixed by naming the database driver explicitly, and prevented from recurring by locking every package, including indirect ones, to an exact version.

### Before and after

| Measurement | Before | After |
|---|---|---|
| Known vulnerabilities (total) | **373** (61 high) | **0** |
| ...in the OS layer | 357 (57 high) | 0 |
| ...in Python packages | 16 (4 high) | 0 |
| App image size | 353 MB (85.2 MB compressed) | 163 MB (40.3 MB compressed), **−54%** |
| Runs as | root (`uid=0`) | non-root (`65532`) |
| Shell and extra tools in image | `sh`, `curl`, `vim`, `nc`, `pip` | none |
| Starts on a fresh build | **No**, crashes on startup | Yes, both containers healthy |
| Web server | Flask dev server with `debug=True` | gunicorn |
| Database port | published on all interfaces (`0.0.0.0:5432`) | not published (internal only) |
| App port | all interfaces | localhost only |
| Python | 3.11 (Debian slim) | 3.15.2 (Chainguard) |
| Dependency pinning | direct packages only | every package, including indirect ones |

*Scanned with Trivy 0.76.2 on November 24, 2026. The "after" image was built with packages from public PyPI because Chainguard Libraries access was blocked at the time (see [Chainguard Libraries access](#chainguard-libraries-access)).*

## How to run

```bash
# Chainguard Libraries credentials, read from your shell and passed to the build as secrets
export CHAINGUARD_PYTHON_IDENTITY_ID=...
export CHAINGUARD_PYTHON_TOKEN=...

docker compose up --build
```

Then open http://localhost:8000.

Database settings have local-development defaults. To override them, copy `.env.example` to `.env` (which is git-ignored).

For testing only, while Chainguard Libraries access is unavailable, the package source can be overridden:

```bash
PIP_INDEX_URL=https://pypi.org/simple/ docker compose up --build
```

This override exists because real customers usually point builds at a repository manager (Artifactory, Nexus, Cloudsmith) that proxies Chainguard Libraries. The default remains Chainguard Libraries only, with no PyPI fallback.

## What I changed and why

### Why this matters to the business

The technical changes below add up to outcomes a customer's leadership, security team, and engineers each care about:

- **Less time lost to vulnerability triage.** Every scanner finding has to be investigated, fixed, or formally accepted as a risk, usually by engineers who would rather be shipping features. Going from 373 findings to 0 removes that work for this service and keeps security reviews focused on issues that matter. Staying near zero depends on keeping base images current, which is why digest-update automation is on the next-steps list.
- **Easier compliance and faster security reviews.** Frameworks such as SOC 2, PCI DSS, and FedRAMP expect known vulnerabilities to be fixed within set timeframes, and enterprise buyers increasingly ask vendors for clean scans and an account of where their software comes from. A minimal image with no known vulnerabilities, built from Chainguard-built packages, makes those conversations shorter and audits less painful.
- **Lower breach risk, and less damage if one happens.** Public package registries are a recurring attack route: hijacked maintainer accounts, look-alike package names, and tampered releases. Using only Chainguard-built packages narrows that exposure. And if an attacker does get in, a non-root container with no shell, no tools, and a read-only filesystem gives them very little to work with.
- **Reliable builds and fewer surprise outages.** The starter app broke without anyone touching its code, because an upstream package changed. Locked dependencies and pinned images mean the same inputs produce the same result every time, and upgrades happen as deliberate, reviewed changes rather than surprises on deploy day.
- **Faster, cheaper delivery.** A 54% smaller image means quicker pulls in CI and at deploy time, plus less storage and network transfer across every environment that runs it.
- **Low cost to adopt.** The application code needed only two small edits, and developers still run a single command. Teams gain these benefits without a rewrite or a new workflow to learn, which is what makes the pattern practical to repeat across a customer's other services.

### 1. Container image (`app/Dockerfile`)

- **Moved to Chainguard Python images with a two-stage build.** Packages are installed in `cgr.dev/chainguard/python:latest-dev`, which has a shell and pip. Only the finished virtual environment and app files are copied into `cgr.dev/chainguard/python:latest`, which has no shell and no package manager. Build tools never reach the image that runs.
- **Runs as Chainguard's non-root user (65532).** App code and packages stay root-owned, so the running app can read but not modify them.
- **Removed `curl`, `vim`, and `netcat`.** The app used none of them, and they are useful to an attacker who gets in.
- **Removed pip from the shipped image.** It's only needed to install packages, and it carried a bundled copy of a vulnerable setuptools (see [issues found](#issues-found-through-due-diligence-and-fixed)).
- **Pinned both base images by digest.** Every build uses exactly the same images rather than whatever `latest` points to that day.
- **Added `.dockerignore`** so stray files, and especially a `.env` file, can't end up in the build.

### 2. Dependencies (`app/requirements.in`, `app/requirements.txt`)

- **Configured installation to use Chainguard Libraries only** (`https://libraries.cgr.dev/python/simple/`), with no fallback to public PyPI. If a package isn't available from Chainguard, the build fails rather than quietly pulling it from elsewhere.
- **Credentials are handled as BuildKit secrets.** They're written to a temporary in-memory file only for the install step, so they never land in an image layer, the build history, or the repository.
- **Removed unused packages:** `requests` (which also had known vulnerabilities) and `humanize`. Neither is imported anywhere.
- **Locked every package to an exact version, including indirect ones.** `requirements.in` lists the five packages the app actually needs, and `requirements.txt` is the full locked list generated from it.
- **Replaced `psycopg2-binary` with `pg8000`** (see [tradeoffs](#tradeoffs)).
- **Upgraded gunicorn** from 21.2.0, which has known vulnerabilities, to 27.0.1, and actually uses it.

### 3. Application runtime (`app/app.py`)

- **Removed `debug=True`.** Flask's debugger allows code execution by anyone who can reach the page.
- **Replaced the dev server with gunicorn**, started directly by the image. `entrypoint.sh` was removed, since it needed a shell the new image doesn't have.
- **Removed the hardcoded database credentials** from the code. The database address now comes only from the environment and names its driver explicitly.

### 4. Local orchestration (`docker-compose.yml`)

- **The app waits for the database to be ready** (a health check plus `depends_on: condition: service_healthy`) rather than just started.
- **Removed the published database port.** Only the app can reach Postgres, over Compose's private network.
- **Bound the app to localhost** (`127.0.0.1:8000`).
- **Locked down the app container at runtime:** read-only filesystem (with a small temporary `/tmp`), all Linux capabilities dropped, and `no-new-privileges`.
- **Added health checks and automatic restarts** for both services. The app's health check uses Python, since there's no shell or curl in the image.
- **Removed the `./app:/app` volume mount**, which replaced the hardened image's contents with the local folder at runtime.
- **Moved credentials to variables** with local-development defaults and optional `.env` overrides, and removed `SECRET_KEY`, which the app never used.
- **Pinned the Postgres image by digest** and removed the obsolete `version` attribute.

## Risks reduced

| Risk | Before | After |
|---|---|---|
| Known vulnerabilities in the shipped image | 373, including 61 high | 0 at time of scan |
| Remote code execution through Flask's debugger | debug mode on | dev server and debug mode gone |
| Impact of a compromise | root, with a shell and attacker tooling available | non-root, no shell, no tools, read-only filesystem, no capabilities |
| Tampered or unexpected upstream packages | anything from public PyPI, mostly unpinned | Chainguard-built packages only, every version locked |
| Builds silently changing | indirect dependencies drifted and broke the app | every package locked; base images pinned by digest |
| Credential leakage | DB password in code and Compose; no protection for a `.env` file | no credentials in code; build secrets never stored; `.env` git-ignored and excluded from builds |
| Database exposed beyond the app | port published on all interfaces | not published |
| Flaky startup | app could start before the database was ready | app waits for a healthy database |

## Issues found through due diligence and fixed

Measuring before changing, and re-testing after each change, turned up several problems that wouldn't have been visible from reading the code alone.

**1. The starter app no longer starts.** On a fresh build, the app crashed with `ModuleNotFoundError: No module named 'psycopg'`. The cause: SQLAlchemy wasn't pinned, and SQLAlchemy 2.1.0 (released September 24, 2026) changed the default PostgreSQL driver for `postgresql://` addresses from `psycopg2` to `psycopg` (v3). The app's code hadn't changed; an indirect dependency drifted upstream. **Fix:** every package is now locked to an exact version, and the database address names its driver explicitly (`postgresql+pg8000://`). This is the clearest argument in the exercise for reproducible, pinned dependencies.

**2. The exposed database port wasn't theoretical.** While the original app ran in GitHub Codespaces, Codespaces detected the published Postgres port and offered to forward it, including an option to make it public. With the default `postgres/postgres` credentials, that would have put the database on the internet. **Fix:** the database port is no longer published.

**3. A hidden vulnerable package.** After the migration, the scan still showed 3 findings in setuptools 70.3.0, even though setuptools isn't a dependency and no setuptools files exist in the image. The source was pip, which bundles part of setuptools 70.3.0 internally. **Fix:** pip is removed from the shipped virtual environment, since the running app never installs packages. Result: 0 findings. Context worth noting: one of the three was Windows-only, and the high-severity one was in a legacy download feature the app never calls. Even so, removing an unneeded package beats arguing about exploitability.

**4. gunicorn 27 tries to write to the home directory.** Testing showed that gunicorn 27 creates a "control socket" file in the user's home directory at startup by default, which fails in a read-only container. **Fix:** disabled with `--no-control-socket`, since the feature isn't needed.

**5. Authentication failures were confusing.** When Chainguard Libraries rejected the credentials, pip tried to prompt for a username in a non-interactive build and crashed with an `EOFError` that looked like a Dockerfile bug. **Fix:** pip runs with `--no-input`, and the build logs whether credentials were provided, without revealing them. One remaining limitation: pip then reports "no matching distribution" for both a rejected login and a genuinely missing package, so I confirmed the cause with a direct request to the index (HTTP 401).

**6. The startup race, verified.** On a completely fresh start (database volume deleted), the app waited about 6 seconds for Postgres to finish initializing before starting. In the original setup it would have tried to connect immediately.

## Chainguard Libraries access

**How it's designed to work.** The Dockerfile defaults to `https://libraries.cgr.dev/python/simple/` with no PyPI fallback. Compose passes the identity ID and token from the shell environment into the build as BuildKit secrets, and the build writes them to a temporary in-memory `netrc` file used only by pip. The build log confirms credentials arrive this way ("credentials: provided via build secrets").

**What happened.** Every request to `libraries.cgr.dev` returned **HTTP 401**, both from the build and from direct requests. I ruled out the likely causes on my side:

- The token isn't expired (valid until December 23, 2026).
- The ID and token have the expected lengths and format: a single `/` in the ID, and no stray characters.
- It isn't a network problem. Requests ran from GitHub's infrastructure, and a 401 is a deliberate response from Chainguard's server.

The server's 401 carried no reason. I reported it to the exercise owner, who issued the credentials, in my submission email, with the request trace ID (`a9c8c169c75a5b380e9bf5761cbce646`, 16:11 UTC on November 24) so Chainguard can see why the request was refused. The likely cause is that this identity is missing the Python libraries pull role, or that the ID and token aren't a matching pair.

**How I validated everything else.** I temporarily built with packages from public PyPI to verify the image, hardening, and Compose changes end to end. The committed default remains Chainguard Libraries only.

**What remains once access is restored:**

1. Build with the default configuration and confirm every locked package is available from Chainguard Libraries. The less common ones (`pg8000`, `asn1crypto`, `scramp`) are the ones to watch. If any are missing, the options are a different driver or asking Chainguard to add them.
2. Regenerate `requirements.txt` against Chainguard Libraries **with hashes**, and install with `--require-hashes`. This has to be done against Chainguard's index, because Chainguard rebuilds packages from source, so their hashes differ from PyPI's.
3. Re-run the vulnerability scan on the image built from Chainguard packages.

## Tradeoffs

- **pg8000 instead of psycopg.** Chainguard's runtime image deliberately lacks `libpq`, the system library that psycopg2 and psycopg need. `psycopg2-binary` works around that by bundling prebuilt system libraries, which conflicts with the rebuilt-from-source model. pg8000 is pure Python and needs nothing extra. The cost is lower performance, which doesn't matter for this app but would be worth benchmarking for a high-traffic service.
- **One gunicorn worker with four threads.** The app creates its tables at startup. Multiple worker processes would each try to do that at once against a fresh database and could collide. One worker with threads avoids that without adding a migration tool. The proper long-term fix is migrations (below).
- **Digest pinning vs. automatic updates.** Pinned images give reproducible builds, but they won't pick up Chainguard's frequent security rebuilds until someone updates the digest. That's acceptable only with automated update pull requests (below).
- **Kept the official `postgres:15` image.** The exercise focuses on the application container. Moving to Chainguard's Postgres image is worthwhile, but it likely means a major-version change, which requires migrating existing developers' data volumes. I'd plan that rather than slip it in.
- **Default local database credentials remain in Compose.** I chose to keep `docker compose up --build` working with no setup step over requiring a `.env` file. The risk is low, since the database isn't published and is reachable only by the app, and `.env` can override the defaults.
- **No hot-reload developer mode.** Developers rebuild to see code changes. I kept the hardened configuration as the default rather than add an automatically loaded override file that would quietly re-publish the database and re-add the source mount.
- **Hashes deferred.** Exact versions are locked now, but hash verification has to wait for Chainguard Libraries access.

## What I'd do next in a real customer engagement

1. **Restore Libraries access and finish the dependency work:** confirm package availability, regenerate the lock file with hashes, and enforce `--require-hashes`.
2. **Route packages through the customer's repository manager.** Configure Chainguard Libraries as an upstream in Artifactory, Nexus, or Cloudsmith, so developers and CI use one internal endpoint instead of each holding a Chainguard token.
3. **Use short-lived credentials in CI.** Replace long-lived pull tokens with an OIDC-based assumable identity, and fail the build on new high or critical findings.
4. **Automate digest updates** with digestabot or Renovate, so Chainguard's security rebuilds arrive as reviewed, tested pull requests.
5. **Verify what runs.** Check Chainguard's image signatures and SBOMs with cosign, and enforce signed images with an admission policy at deploy time.
6. **Move the database to Chainguard's Postgres image** with a planned data migration.
7. **Application-level follow-ups** outside this exercise's scope: replace `db.create_all()` with real migrations (for example, Alembic), add CSRF protection to the forms, and clear deprecation warnings (`datetime.utcnow`, the legacy `Query.get`).
8. **Agree on success measures with the customer**, such as vulnerability counts over time, time spent triaging scanner findings, and time to roll out a fixed base image.

## How a partner would repeat this

This exercise took one service through the full path. A partner (reseller, consultancy, or systems integrator) can turn the same path into a repeatable offering across a customer's portfolio.

### The playbook

1. **Baseline first.** Scan the current image and record its size, the user it runs as, the tools inside it, and whether it builds and starts cleanly from scratch. Note the scanner version and date. This gives the customer a before/after story, and it often uncovers problems nobody knew about, like the broken startup here.
2. **Two-stage build on Chainguard images.** Install in the `-dev` variant and ship only the result on the minimal runtime image, running as non-root. Pin both images by digest.
3. **Dependencies from Chainguard Libraries.** Pass credentials as build secrets, allow no fallback to public registries, and lock every package, including indirect ones, with hashes generated against Chainguard's index.
4. **Remove what the app doesn't need.** OS tools, unused packages, and the package installer itself have no place in the shipped image.
5. **Harden the runtime.** Use a production server, a read-only filesystem, dropped capabilities, `no-new-privileges`, and health checks, and don't publish internal services like databases.
6. **Re-measure with the same scanner** and present the before/after to the customer.

### Known issues to plan for

These came up in this exercise and are likely to recur across other Python services:

- **Database drivers that need system libraries.** `psycopg2` and `psycopg` need `libpq`, which the minimal runtime image doesn't include. Choose a pure-Python driver, or deliberately bring the library across from the build stage.
- **Scanners flag pip's bundled setuptools.** Remove pip from the shipped image.
- **Hashes from PyPI won't match.** Chainguard rebuilds packages from source, so lock files must be generated against Chainguard's index.
- **No shell in the runtime image.** Shell-script entrypoints and `curl`-based health checks have to be replaced.
- **Apps that write files at startup** (here, gunicorn 27's control socket) fail on a read-only filesystem. Test with the hardened settings, not just the new image.
- **Unpinned indirect dependencies** can break a build without anyone touching the code. Expect the baseline to surface these.
- **Package availability.** Confirm early that every dependency exists in Chainguard Libraries. Because builds fail closed, a missing package blocks the build rather than quietly pulling from PyPI.
- **Compose override files load automatically.** A `docker-compose.override.yml` added for developer convenience can quietly re-publish ports or remount source code and undo the hardening.

### Where the partner adds value beyond the migration

- **Assessment:** baseline the customer's services and prioritize by exposure and business importance.
- **Platform setup:** Chainguard Libraries as an upstream in the customer's repository manager, short-lived CI credentials, automated digest updates, and signature and policy enforcement at deploy time.
- **Migration at scale:** turn the playbook into "golden" Dockerfile and Compose templates per language, so each additional service is faster than the last.
- **Ongoing service:** keep images current, report on vulnerability trends, and supply evidence for audits and customer security reviews.

## AI tool usage

**Tools used:** Claude (Anthropic), through claude.ai, throughout the exercise.

**Where I used it:**

- Understanding the exercise and planning the approach.
- Setting up my environment: checking my Mac for tools and moving to GitHub Codespaces when the Docker Desktop download was too slow on my connection.
- Reviewing the starter repo to identify likely problems.
- Drafting the Dockerfile, Compose file, and requirements changes.
- Troubleshooting: the startup crash, the Chainguard Libraries 401, the setuptools finding, and git remotes.
- Drafting this write-up from my notes.

**How I separated AI suggestions from my own judgment:**

- **I verified every claim by running it myself.** Every number in the before/after table comes from commands I ran on the original and the updated images. I treated the starter-repo review as a list of hypotheses and confirmed each one by testing.
- **AI was wrong in places, and testing caught it.** Claude predicted the startup crash was a timing problem; the logs showed it was the SQLAlchemy driver change. Claude once misread a failed local scan as a clean result and later corrected itself. Its suggestion that the token's audience would reveal the token type turned out not to help. Each of these was resolved by checking actual output rather than accepting the explanation.
- **Decisions were mine.** Claude recommended pg8000 and a configurable package source for testing. I evaluated and accepted both, and I chose to keep the hardened setup as the default over a dev-convenience override.
- **Other sources.** The idea to pin base images by digest came from reviewing another candidate's public fork of this exercise. I implemented it in my own way and tested it.
- **AI-run tests.** Claude tested the proposed package set on Python 3.15 against Postgres before I applied the changes. That's how the gunicorn control-socket issue surfaced early. I then confirmed everything in my own builds and scans.
- **Cleanup.** Claude reviewed the files after I pushed them and flagged leftover commented-out code from my copy-pasting, including an old hardcoded password and the old `debug=True` line. I removed it and re-tested before pushing again.
