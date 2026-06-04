# AI Coding Standards — Lucky's Infrastructure
<!-- Filename: ai_coding_standards.md -->
<!-- Version: 4 -->
<!-- Date: 2026-05-19T21:12:00Z -->
<!-- Author: Claude Sonnet 4.6 -->
<!-- Purpose: Coding and operational standards for all agents on this host.
     Identity, tone, and collaboration rules are in AGENTS.md, not here.
     Applies to all Linux servers and cross-platform work unless a section
     states otherwise. -->

---

## 1. File Headers

Every code file, script, or configuration file must include a header block at the top. The header uses comment syntax appropriate to the file type.

**Required fields:** Filename (full absolute path), Version (incrementing integer — every change, including header-only changes, bumps the version), Date (ISO 8601 with timezone, queried live: `date -u +%Y-%m-%dT%H:%M:%SZ`), Author (agent version string, queried live — see AGENTS.md §Agent-Specific Configuration), Purpose (max 3 lines), Usage (invocation example).

### Syntax by file type

**Bash, Python, Ruby, YAML, TOML (`#`):**
```bash
# Filename: /absolute/path/to/example.sh
# Version: 1
# Date: 2026-05-19T21:12:00Z
# Author: Claude Sonnet 4.6 1.2.3
# Purpose: Does the thing.
# Usage: bash /absolute/path/to/example.sh --flag value
```

**JavaScript, TypeScript, Rust, C/C++, Go (`//`):**
```javascript
// Filename: /absolute/path/to/app.js
// Version: 1
// Date: 2026-05-19T21:12:00Z
// Author: Claude Sonnet 4.6 1.2.3
// Purpose: Client-side logic for X.
// Usage: node app.js
```

**HTML, Markdown, XML (`<!-- -->`):**
```markdown
<!-- Filename: /absolute/path/to/README.md -->
<!-- Version: 1 -->
<!-- Date: 2026-05-19T21:12:00Z -->
<!-- Author: Claude Sonnet 4.6 1.2.3 -->
<!-- Purpose: Project documentation. -->
<!-- Usage: View in Markdown renderer -->
```

**INI, some config formats (`;`):**
```ini
; Filename: /absolute/path/to/app.ini
; Version: 1
; Date: 2026-05-19T21:12:00Z
; Author: Claude Sonnet 4.6 1.2.3
; Purpose: Application configuration.
; Usage: Read by application at startup
```

**SQL (`--`):**
```sql
-- Filename: /absolute/path/to/schema.sql
-- Version: 1
-- Date: 2026-05-19T21:12:00Z
-- Author: Claude Sonnet 4.6 1.2.3
-- Purpose: Database schema.
-- Usage: psql -f schema.sql dbname
```

**Strict JSON (no comment support — use metadata object):**
```json
{
  "_metadata": {
    "filename": "/absolute/path/to/data.json",
    "version": 1,
    "date": "2026-05-19T21:12:00Z",
    "author": "Claude Sonnet 4.6 1.2.3",
    "purpose": "API payload for service X"
  },
  "actual_data": "goes_here"
}
```

Use whichever comment syntax is canonical for the language. The above are examples, not an exhaustive list. Never use `#` comments in `.md`, `.json`, `.xml`, or `.html` files.

---

## 2. GitHub CLI Preference

`gh` is preferred over plain `git` for any operation that touches GitHub's API or platform features.

Use `gh` for: `gh pr create/list/view`, `gh issue create/list/view`, `gh repo clone/create/view/sync`, `gh release create/list`.

Use plain `git` for local operations: `git commit`, `git branch`, `git checkout`, `git merge`, `git rebase`, `git log`, `git diff`, `git status`.

For push/pull, both `gh repo sync` and `git push`/`git pull` are acceptable.

---

## 3. Critical File Edit Protocol

Applies to all files intended for machine consumption: config files (nginx, systemd, etc.), system prompts, scripts, code files, structured data (JSON, YAML, TOML, XML).

Does **not** apply to human-readable content (Markdown docs, prose, text files).

**Mandatory workflow:**

```bash
# 1. Backup with absolute path and timestamp
cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.backup.20260519-211200

# 2. Make edits

# 3. Read back the edited file completely
cat /etc/nginx/nginx.conf

# 4. Generate diff
diff /etc/nginx/nginx.conf.backup.20260519-211200 /etc/nginx/nginx.conf

# 5. Verify diff matches intent; iterate if not

# 6. Delete backup only after successful verification
rm /etc/nginx/nginx.conf.backup.20260519-211200
```

Never trust tool success messages without verification. Read the file back. AIs frequently write empty files or files with no changes.

---

## 4. Path Requirements

**No tilde (`~`) anywhere, ever.** Shell expansion of `~` can fail or expand to the wrong user when combined with `sudo`, `su`, or SSH one-liners.

Every path must be fully qualified: `/home/shamrock/file.txt`, not `~/file.txt` or `file.txt`.

Relative paths are only acceptable within a single command sequence where `pwd` has been explicitly verified immediately prior.

**Per-command user/path verification** — since AI typically executes commands as SSH one-liners with zero state retention, include inline verification when user identity or current directory matters:

```bash
[[ $(whoami) == "shamrock" ]] && [[ $(pwd) == "/opt/myproject" ]] && git pull
```

Use inline verification for: git operations, package installations, file modifications, any command where wrong user or wrong directory would cause damage. Not required for commands that explicitly specify absolute paths for all arguments, or read-only operations where location doesn't matter.

**Commands must never include `sudo`** — Lucky runs as root or a user with passwordless sudo when needed. Do not prepend `sudo` to commands in scripts or runbooks.

---

## 5. Filesystem Context Verification Before Writes

Before any write, create, delete, move, chmod, chown, or symlink operation, verify which filesystem you are operating in. This prevents operating in the wrong context (sandbox vs. target server).

Fast check — examine `pwd` output or hostname. If the filesystem does not match the expected target for the operation, stop and ask before proceeding.

Not required for read-only operations or when absolute paths make the target unambiguous.

---

## 6. Code Style and Structure

Use language-appropriate standard libraries. Minimize external dependencies unless justified by complexity or time savings.

Variable names must be clear and descriptive. Complex logic gets inline comments explaining the *why*, not the *what*. Error handling and logging must be included.

Do not use frameworks unless the project specifically requires them.

**For MVP / proof-of-concept work:** single file over multiple modules unless complexity demands separation. Refactoring later is acceptable. "Good enough" is often the correct answer. Optimization is premature unless performance is a stated requirement.

Commands must use absolute paths in all cases and must be actual executable commands — not pseudocode.

---

## 7. Documentation Format

Use `.md` (Markdown) for all documentation. This applies to both internal docs where Lucky is the primary audience and external/ecosystem documentation.

Previously `.rst` was used for internal docs but this was dropped due to tooling friction. `.md` is now the universal default.

---

## 8. Directory Entry Protocol

When entering any directory to begin work, check for and read (in order): `AGENTS.md`, `CLAUDE.md`, `README.md`. If any of these exist, read them before proceeding. Directives in these files take precedence as local rules and supplement or override general standards for that workspace.

**Scope limit:** only read files in the current working directory and its subdirectories. Never traverse up (`..`) or across to other directory trees unless explicitly instructed. If an `AGENTS.md` references a file in another project tree, ask before following that reference.

---

## 9. File Operation Verification

After any file modification:

- Files < 100 lines: read the entire file back; verify expected changes are present and no unexpected changes appeared.
- Files 100–1,000 lines: read only the relevant sections containing the change.
- Files > 1,000 lines: use grep/search to confirm the change exists at the correct location.
- Sequential edits to the same file: verify after the first edit and after the last edit; skip intermediate verifications unless there is reason to suspect failure.

Verification format: `Verified: [file] now contains [expected change].` or `Verification failed: [file] unchanged. Trying different approach.`

If verification reveals unexpected state (intended change is present but unintended changes also appeared): **stop immediately, make no additional changes, alert Lucky with specifics.**

---

## 10. API and Technical Claims

For any technical claim about APIs, system behavior, or tool capabilities: cite the official source (vendor docs, RFC), verify currency, and link to documentation. Never make API behavior claims without verification. If documentation is unavailable or unclear, state uncertainty explicitly.

---

## 11. Software Package Quality Thresholds

Before presenting any package, library, or tool as an option, apply the following filter.

**Minimum viable:** 10+ GitHub stars AND last commit within last 3 months, no critical security warnings, dependencies not 2+ years stale without activity.

**Preferred:** 50+ stars, active community (PRs, issues, discussions), regular releases, good docs, visible CI/CD.

Never present: packages with < 10 stars and no commits in 3+ months, archived or explicitly deprecated repos, packages with known critical vulnerabilities.

If the ecosystem is sparse: say so clearly. If only low-quality options exist, describe the problems (stars, last commit, known issues) and ask whether to proceed or build custom. Do not hide the shit options — describe them accurately.

**Abandonment assumption:** if a tool lacks a public changelog OR visible issue history, assume it is dead until proven otherwise. The burden of proof is on the tool to demonstrate active maintenance.

Abandonment signals: no changelog/release notes, issues tab disabled or empty, last release > 18 months ago with no commits, maintainer non-responsive to critical bugs, vulnerable dependencies with no response.

### Discovery methodology by tool class

- **Python/Node packages:** start at PyPI/npm, verify on GitHub (stars, commits, issues), augment with Libraries.io and OpenSSF Scorecard. Never rely solely on registry metadata.
- **CLI tools:** start on GitHub (most CLI tools are open source), verify via package managers (apt, cargo), cross-check Repology.
- **MCP servers:** start at MCP Registry and Glama directory, verify on GitHub. Many MCP entries are abandonware — apply star/commit filters aggressively.
- **Infrastructure tools (databases, servers, orchestration):** start at upstream project + GitHub, verify via OS vendor documentation (Debian packages, etc.).

### Post-discovery quality checks (universal)

Last commit ≤ 6 months for active projects; ≤ 12 months acceptable for mature/stable. Check issue response ratio, CI/CD presence, license clarity.

For open source: star count trend matters more than absolute number; bus factor ≥ 2; dependency hygiene maintained.

---

## 12. Dependency Tree Assessment

When evaluating NPM or PyPI packages, check: dependency freshness (`npm outdated`, `pip list --outdated`), deprecated or abandoned transitive deps, security warnings (`npm audit`, `pip-audit`), known vulnerabilities.

Red flags requiring disclosure: dependencies > 2 years old without activity, "no longer supported" warnings, memory leak warnings, deprecated core dependencies, critical CVEs.

When reporting: state how old the dependencies are, explain severity (security risk vs. merely old), assess whether this signals poor maintenance, estimate impact on reliability.

Do not describe packages as "enterprise-grade" or "production-ready" when the dependency tree shows abandonment.

---

## 13. Build vs. Buy

Suggest existing solutions when multiple mature options exist (>50 stars, active maintenance, recent commits). Suggest building custom when all existing solutions fail requirements, scope is simple, and maintenance burden is acceptable.

When all existing options are broken or inadequate: acknowledge it clearly, support the build decision, never pressure Lucky to use broken solutions.

Present 2–3 alternatives with pros/cons rather than trying multiple workarounds sequentially. If good options exist, don't mention bad ones. If only bad options exist, describe them accurately and ask whether to proceed or build custom.

---

## 14. Language and Tool Selection

No dogmatic preferences. Match tool to task and environment:

- Python: quick scripts, data processing, AI/ML, automation
- Rust: performance-critical, systems programming, native APIs
- JavaScript/TypeScript: web development, Node.js servers
- Bash: Unix scripting, pipeline automation

---

## 15. Architecture Preferences

Single file over multiple modules for MVP/proof-of-concept. Refactor later, not prematurely. Fewer, longer config files beat many small config files. File-based persistence for simple projects (no database unless needed). Append-only logs (immutable, easy to debug). Efficiency over completeness.

---

## 16. Python Environment Management

`uv` is the mandated Python environment tool for all new Python projects. It is a drop-in replacement for pip + venv, written in Rust, 10–100× faster.

Usage pattern: `uv --directory <project_path> run <script.py>` — automatically creates the environment, installs deps, and runs the script. Compatible with `requirements.txt` and `pyproject.toml`.

Always verify the active Python executable before assuming PATH: `which python` (Unix). Do not assume the system-wide PATH applies to the current shell session.

---

## 17. Code Anti-Patterns — Never Do These

**Never over-document obvious code:**
- BAD: `x = 5  # Set x to 5`
- GOOD: `max_retries = 5  # API rate limit recovery`

**Never offer unsolicited learning resources:**
- BAD: "Here are some tutorials on..."
- GOOD: Show the code.

**Never hedge with false uncertainty:**
- BAD: "I think this might possibly work..."
- GOOD: "This will work." or "This might fail if X; fallback is Y."

**Never suggest scope creep:**
- BAD: "We could also add a web UI, metrics dashboard..."
- GOOD: Stick to stated scope unless asked.

**Never try workarounds without permission when installations fail.**

**Never trust tool success messages without verification (reading files back).**

**Never guess at file paths or executable locations without verification.**

**Never present abandonware (< 10 stars, no commits in 3+ months) as a viable option without a clear disclaimer.**

**Never make API behavior claims without citing official documentation.**

---

## 18. Execution Permission Boundaries

The following always require explicit permission before proceeding:

- Changing approach after a failure (approach A failed → proposing approach B)
- Installing packages with workaround flags: `--ignore-scripts`, `--force`, `--legacy-peer-deps`, `pip install --break-system-packages`, etc.
- Modifying system-level configurations
- Installing system packages via `apt` or equivalent
- Making irreversible changes to data
- Editing third-party source code (patching vendor packages)
- Hardcoding file paths as workarounds
- Setting environment variables as workarounds instead of fixing configuration
- Implementing temporary solutions that updates will wipe out
- Any workaround that does not address the root cause
- Architectural or approach changes from an agreed plan

When an approach fails: stop, determine if the error is transient (retry up to 5×, max 1–2 minutes), then perform root cause analysis, state "Approach A failed. Root cause: [analysis]", propose approach B with tradeoffs, and ask "Proceed with approach B?" — then wait for explicit yes/no.

**Root cause philosophy:** fix root causes, never hack around problems. Hacks (hardcoded paths, workaround flags, editing vendor source, env-var workarounds, temporary fixes) are only acceptable with explicit permission when no clean solution exists.

---

## 19. Long-Running Tasks and Failure Handling

For tasks running longer than 2 minutes: provide a non-blocking status update every 2 minutes. Keep updates brief — what's happening, what's next.

After 5 failed attempts at a task: pause execution. Provide a status summary of what was tried and what failed. Ask for guidance before continuing.

**Session-level RCA:** when the same type of error occurs 3+ times in a session, stop fixing individual instances and do root cause analysis on the recurring pattern. Example: "PATH resolution has failed 3 times. Root cause appears to be [X]. Should we fix the underlying issue?"

When a task is taking too long: pause and suggest deferring. "We've tried N approaches over M minutes without success. Should we defer and research alternatives?"

Do **not** periodically summarize progress to Lucky during long sessions — he knows where things stand.

---

## 20. Handling Frustration, Uncertainty, and Conflict

When Lucky is frustrated: focus on solutions, not empathy. Match his directness. "Understood. Let's work fast." — not "I'm sorry you're frustrated..."

When information is ambiguous: state assumptions explicitly, proceed with educated guess when not critical, offer to course-correct. Do not block on ambiguity unless the decision is irreversible.

When uncertain about technical facts: state uncertainty clearly, look it up, do not guess at critical details.

When prompt or directive sections conflict: stop and explain the ambiguity to Lucky so he can resolve it. Do not pick one side silently.

When discovering a successful approach not documented in these standards: note it explicitly and suggest adding it.

---

## 21. Discovery Safety

**"Find" means find.** When the task is to locate, discover, or investigate, never create, generate, overwrite, or modify anything. Report findings and stop. Ask before taking action.

**Read-only investigation is the default** for any diagnostic or discovery task unless Lucky explicitly authorizes modifications.

---

## 22. Completion Signals

When a task is complete: `Completed [X]. Ready for next step.`

When information is needed:
```
Need: [specific item]. Cannot proceed without this.
Options: [A, B, C]
Recommend: [X]
Proceed with which?
```

When an error is found: `Error in previous code: [specific fix]. Updated version: [code].`

When asked for thoughts: `Opinion: [clear stance with reasoning].`

---

## 23. Core Principles

Lucky and the agent are equals working on technical projects together.

Lucky values speed and pragmatism over perfection.

Code speaks louder than explanations.

When in doubt, implement and iterate.

This is Lucky's project — support his vision, don't impose yours.

When Lucky says "proceed," proceed confidently. Ask permission before pivoting approaches, using workaround flags, or making architectural changes.

There is great delight in working with team members who kick ass and take names.
