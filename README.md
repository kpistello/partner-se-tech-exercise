# Partner SE Technical Exercise: Submission

This fork contains my completed exercise: the todo app's container moved to Chainguard Containers, Python dependencies configured to install from Chainguard Libraries only, and the Docker Compose setup hardened, with `docker compose up --build` still the only command a developer needs.

**Full write-up, including what changed, why, tradeoffs, and next steps: [WRITEUP.md](WRITEUP.md)**

| | Before | After |
|---|---|---|
| Known vulnerabilities | 373 (61 high) | **0** |
| App image size | 353 MB | **163 MB** (−54%) |
| Runs as | root | **non-root** |
| Shell and extra tools in image | yes | **none** |
| Starts on a fresh build | **no** (crashes) | **yes** |

## Run the hardened version

```bash
export CHAINGUARD_PYTHON_IDENTITY_ID=...
export CHAINGUARD_PYTHON_TOKEN=...
docker compose up --build
```

Then open http://localhost:8000.

**Note:** the Chainguard Libraries credentials provided for this exercise returned HTTP 401 throughout, so the "after" numbers were measured with packages from public PyPI. The submission still defaults to Chainguard Libraries only. See [Chainguard Libraries access](WRITEUP.md#chainguard-libraries-access) for the diagnosis. To test the rest of the setup without Libraries access:

```bash
PIP_INDEX_URL=https://pypi.org/simple/ docker compose up --build
```

---

*The original exercise instructions follow.*

## Original exercise: Partner SE Technical Exercise Starter Repo

This repository contains a small Flask + Postgres todo application intended for use as a Partner SE technical exercise.

### Scenario

A customer has a small Python application that runs locally with Docker Compose. It works, but the application container and dependency setup are not aligned with their supply chain and container hardening goals.

The exercise for the candidate is to improve this application by:

- migrating the application container to Chainguard Containers
- configuring Python dependency installation to use Chainguard Libraries only
- preserving a simple local developer workflow with Docker Compose
- making practical, high-impact improvements without over-engineering the solution

### Timebox

- Expected: 90 minutes
- Max: 2 hours

### Starter repo notes

This starter intentionally includes several issues for the candidate to evaluate and improve. Examples include container image choice, build structure, dependency installation, runtime posture, and local orchestration behavior.

### Run locally

```bash
docker compose up --build
```

Then open <http://localhost:8000>.

### ⚠️ Chainguard Libraries Access

**Configuring Python dependency installation to use Chainguard Libraries is a required part of this exercise.**

If you have not been provided with a Chainguard Libraries token and identity ID, **please request one before starting**. You will need these credentials to complete the Libraries portion of the exercise. Do not submit without attempting this step — if access cannot be arranged, document what you would have done and why.

### Candidate deliverables

Candidates should submit a link to their forked repository containing:

1. Updated code and configuration
2. A write-up **committed to the repository** (Markdown preferred) describing:
   - what they changed
   - why they changed it
   - what risks or issues were reduced
   - tradeoffs they made
   - what they would do next in a real customer engagement
3. A brief note on AI tool usage — specifically: which tools you used, where in the exercise you used them, and how you distinguished AI-suggested changes from your own judgment. There is no penalty for using AI; transparency about how you used it is what matters.

### Interviewer notes

A strong solution will usually:

- use appropriate Chainguard Python image variants
- separate build and runtime concerns sensibly
- remove unnecessary packages and dependencies
- avoid running as root
- keep `docker compose up --build` working
- clearly explain prioritization and tradeoffs
