# Learning Journal

One entry per task, problem or drill. The template is in `docs/milestones/M00-engineering-standards.md` §5.11.
Predictions are written **before** running a break test.

---

## 2026-11-18 · M00-T01 · Initialise the repository

**Problem I hit:**
_(fill in)_

**Why it happened:**
_(fill in)_

**What I changed:**
- `git init -b main`, remote `origin` → `github.com/Its-Nitin/nexusflow`
- `.gitattributes` (`* text=auto eol=lf`, CRLF only for `*.cmd`/`*.bat`, binaries marked) added **before the first commit**
- `.gitignore` (secrets, Maven, IDEs, OS, logs, frontend, Python)
- Minimal `README.md`
- Repo settings: squash merge only, delete branches after merge
- Ruleset `protect-main`: no direct pushes, no force pushes, no deletion, PR required (0 approvals)

**Break test: direct push to `main` after the ruleset is active**
- Prediction (written before testing): _(fill in)_
- Actual result: _(fill in)_

**Trade-off / what it costs:**
_(fill in)_

**Self-check: why have branch protection when you're the only contributor?**
_(fill in)_

**Open questions:**
- [ ] Add the `ci` workflow as a required status check in the ruleset once CI exists (M00-T10).
