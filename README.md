# Claude Skills

A collection of [Claude Code](https://docs.claude.com/en/docs/claude-code) skills I use regularly. Each skill is a self-contained slash command that teaches Claude how to perform a specific, repeatable task — code review, bug tracking, and feature logging.

A **skill** is just a directory containing a `SKILL.md` file with YAML frontmatter (`name`, `description`, `user-invocable`). When installed, Claude Code exposes each one as a slash command (e.g. `/fix-bug`) that you can invoke directly.

---

## Installation

Skills can be installed for **all projects** (personal) or **a single project**.

### Option A — Personal (available in every project)

Personal skills live in `~/.claude/skills/`. Clone this repo and copy each skill directory into it:

```bash
# Clone the repo somewhere convenient
git clone https://github.com/Crypto69/claude-skills.git
cd claude-skills

# Copy every skill into your personal skills directory
mkdir -p ~/.claude/skills
cp -R add-bug add-feature fix-bug list-bugs review-my-code ~/.claude/skills/
```

Prefer to keep them updatable with a `git pull`? Symlink instead of copying:

```bash
for skill in add-bug add-feature fix-bug list-bugs review-my-code; do
  ln -s "$(pwd)/$skill" ~/.claude/skills/$skill
done
```

### Option B — Per project (checked into one repo)

Drop the skill directories under that project's `.claude/skills/` folder so they travel with the repo and are shared with your team:

```bash
mkdir -p .claude/skills
cp -R /path/to/claude-skills/{add-bug,add-feature,fix-bug,list-bugs,review-my-code} .claude/skills/
```

### Verify

Start (or restart) Claude Code and type `/` — the installed skills appear in the slash-command list. You can also run any of them directly, e.g. `/review-my-code`.

> **Configure before use:** The bug- and feature-tracking skills contain placeholders (`<area-1>`, `<TIMEZONE>`, test commands) that you should replace with your project's values first. See [Configuring the skills](#configuring-the-skills-for-your-project) below.

---

## Configuring the skills for your project

The tracking skills ship with **placeholders** — written as `<TOKEN>` — that you replace with your own project's values before using them. Nothing in these skills is tied to any particular repo, language, or framework. Each skill repeats its own placeholders under a **Configuration** heading at the bottom; the table below is the shared reference.

| Placeholder | Appears in | Replace with | Example |
|-------------|-----------|--------------|---------|
| `<area-1> / <area-2> / …` | `add-bug`, `list-bugs`, `fix-bug`, `add-feature` | The modules / sub-projects in your repo, used to tag bugs & features | `frontend / backend / api / mobile / infra` |
| `<TIMEZONE>` | `add-bug`, `fix-bug`, `add-feature` | The timezone used for timestamps | `UTC`, `America/New_York`, `Europe/London` |
| `<test command …>` | `fix-bug` | How to run tests for each area | `cd frontend && npm test`, `pytest backend/` |

Keep the **area list identical across all four tracking skills** so tags and filters line up. `review-my-code` has no placeholders — its checklist is framework-agnostic and works on any stack as-is.

### Or let Claude fill in the areas for you

Don't want to pick the areas by hand? Drop this into Claude Code from your project root and let it propose them in plan mode first:

```text
Analyse this repo and identify the major areas of the codebase (top-level
modules, services, or apps that a bug or feature would naturally belong to).
Then replace the `<area-1> / <area-2> / …` placeholder in the add-bug,
list-bugs, fix-bug, and add-feature skills with that list, using the same
list in all four skills so tags and filters stay consistent. Do this in
plan mode so I can approve the area list before any files are edited.
```

You can extend the same prompt to fill in `<TIMEZONE>` and the `<test command …>` mapping in `fix-bug` — Claude can infer the test commands from the project's `package.json`, `pyproject.toml`, `Makefile`, etc.

### Example: a Python API + React web app

Say your repo has a Python API and a React frontend, with two areas, `api` and `web`. You'd edit each tracking skill to:

- set the area list to `api / web`
- set `<TIMEZONE>` to `UTC`
- in `fix-bug`, map the test commands: `api` → `pytest`, `web` → `cd web && npm test`

After that, `/add-bug Login returns 500 on an expired token` logs a bug you tag `api`, `/list-bugs api` filters to it, and `/fix-bug 1` runs `pytest` to verify the fix.

---

## The skills

### `/review-my-code` — Senior code review

A **read-only** critical review of your uncommitted changes, performed as if by a senior engineer reviewing a pull request. It gathers the diff (`git status` + `git diff HEAD`), reads surrounding context, and reports real problems — no rubber-stamping, no filler.

It checks for:

- **Bugs & correctness** — off-by-ones, null access, race conditions, bad async/`await`, broken error handling
- **Security** — OWASP top 10, leaked secrets, missing auth, unsafe HTML injection, missing row-level security on new tables
- **Performance** — unnecessary recomputation/re-rendering, N+1 queries, unbounded fetches, leaked subscriptions/timers
- **Duplication & reusability** — logic that should be a shared utility, service, or component
- **YAGNI / KISS** — speculative abstractions and over-engineering
- **Tests & architecture** — missing regression tests, logic in the wrong layer, broken shared interfaces

Findings are grouped by severity (Critical / Warning / Suggestion) with file:line references. It never edits files — you decide what to act on. The checklist is framework-agnostic, so it works on any stack out of the box.

**Usage:**

```text
/review-my-code              # review all uncommitted changes
/review-my-code src/auth.ts  # scope the review to specific files
```

---

### Bug & feature tracking workflow

Four skills that share a lightweight, Markdown-based tracker — no external tools required. Bugs live in `docs/bugs.md` and features in `docs/features.md`, each entry carrying an ID, status, severity/priority, and area.

```
/add-bug ──▶ /list-bugs ──▶ /fix-bug        (docs/bugs.md)
/add-feature                                 (docs/features.md)
```

> These four skills use placeholders for the area list, timezone, and test commands — see [Configuring the skills](#configuring-the-skills-for-your-project) before first use.

#### `/add-bug` — Log a bug

Appends a structured bug entry to `docs/bugs.md`. It auto-assigns the next `BUG-NNN` id, timestamps it (in your configured timezone), and infers severity (Critical / High / Medium / Low) and area from your description when you don't specify them. Bugs start at status `🔴 Reported`.

**Usage:**

```text
/add-bug Checkout fails when the payment webhook times out
/add-bug                 # prompts you for details if none are given
```

#### `/list-bugs` — List & filter bugs

Prints a summary table of bugs in `docs/bugs.md` with a count breakdown. With no arguments it shows all **open** bugs (everything except Fixed / Won't Fix). Filters by status, severity, and area combine with AND logic.

**Usage:**

```text
/list-bugs                  # all open bugs
/list-bugs critical         # only critical severity
/list-bugs reported api     # reported bugs in a given area
/list-bugs all              # everything, including Fixed & Won't Fix
```

#### `/fix-bug` — Investigate, fix & verify a bug

Picks a bug from `docs/bugs.md` (by number, or it presents the open bugs in priority order for you to choose), investigates the root cause, implements the **minimal** fix, runs the relevant test suite, and updates the bug to `🟢 Fixed` with root-cause and changed-files notes. It flags risky payment/auth changes before applying.

**Usage:**

```text
/fix-bug            # lists open bugs in priority order to choose from
/fix-bug 3          # fix a specific bug by number
/fix-bug BUG-003    # …or by full id
```

#### `/add-feature` — Log a feature request

Appends a structured feature entry to `docs/features.md`. Auto-assigns the next `FEAT-NNN` id, timestamps it (in your configured timezone), and infers priority (High / Medium / Low / Nice-to-have) and area. New features start at status `Backlog`. It deliberately **does not** start building the feature — it asks first.

**Usage:**

```text
/add-feature Add a dark mode toggle to the settings page
/add-feature             # prompts you for details if none are given
```

---

## Repository layout

```
claude-skills/
├── add-bug/SKILL.md        # log a bug to docs/bugs.md
├── add-feature/SKILL.md    # log a feature to docs/features.md
├── fix-bug/SKILL.md        # investigate, fix & verify a tracked bug
├── list-bugs/SKILL.md      # list & filter tracked bugs
├── review-my-code/SKILL.md # read-only senior code review
└── README.md
```

## License

MIT — use, adapt, and share freely.
