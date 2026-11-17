# Peribolos: GitHub Org Member, Team & Permission Management

This repository manages **org membership, teams, and team permissions** for
organizations in the [NeoNephos GitHub Enterprise](https://github.com/enterprises/neonephos)
as code, using [Peribolos](https://docs.prow.k8s.io/docs/components/cli-tools/peribolos/)
— the same tool the Kubernetes project uses to manage its org at scale.

The model matches the migration guideline: **the enterprise owns the guardrails
(the automation, safety rails, and default policies), while each project
self-administers its own membership** via pull requests.

---

## How it works

```
orgs/<org>/org.yaml   ──PR──▶  review + CODEOWNERS approval  ──merge──▶  apply on GitHub
  (desired state)              (plan posted as PR comment)              (peribolos --confirm)
```

1. Each org has one file: `orgs/<org>/org.yaml` — the **single source of truth**
   for that org's settings, members, admins, teams, and team membership.
2. A PR that edits it triggers the `peribolos` workflow in **dry-run** mode — a
   `peribolos reconcile` check that mutates nothing; a green check is the signal.
3. A CODEOWNER approves. On merge to `main`, the same workflow runs in **apply**
   mode (`CONFIRM=true`)
   `peribolos --confirm` and reconciles GitHub to match the file.

> **Authoritative membership:** removing a user from `members`/`admins` in the
> YAML **removes them from the org** on apply. The `--maximum-removal-delta`
> safety rail (default 0.25) blocks any run that would delete >25% of
> memberships, guarding against typos.

---

## Repository layout

```
orgs/
  neonephos/
    org.yaml            # desired state for one org
Taskfile.yml             # validate / dump / plan / apply tasks
.taskrc.yml              # enables the env-precedence experiment
admin/
  github-app/           # GitHub App creation guide
.github/workflows/
  peribolos.yml         # PR = dry-run; push to main = apply
CODEOWNERS              # who approves what
```

**Local prerequisites:** [go-task](https://taskfile.dev) (`task`), Docker, and a
`GITHUB_TOKEN` with org admin scope. `GITHUB_TOKEN` is a global Task variable
that defaults to your shell environment (`export GITHUB_TOKEN=...`) and can be
overridden inline (`task reconcile ORG=<org> GITHUB_TOKEN=...`); `.taskrc.yml` enables
Task's [env-precedence experiment](https://taskfile.dev/docs/experiments/env-precedence)
so an overridden value wins over a stale OS value. CI installs Task
automatically.

---

## One-time setup (enterprise admin)

### 1. Create the GitHub App

Peribolos authenticates as a **GitHub App** (preferred over a PAT: not tied to a
person, short-lived tokens, least privilege).

1. Create an App (at the enterprise or a central org). Permissions:
   - **Organization → Members: Read and write** (`members`)
   - **Organization → Administration: Read and write**
     (`organization_administration`)

   Both are **Organization** permissions. The *Repository* permission also
   called "Administration" is a different scope and does **not** allow
   `PATCH /orgs/{org}`: with only that one, `--fix-org` fails the apply with
   `403 Resource not accessible by integration`. Verify with
   `gh api orgs/<org>/installations --jq '.installations[].permissions'` —
   `organization_administration: write` must be listed.
2. Generate a private key (PEM).
3. **Install the App on each managed org.**
4. Store the credentials as repo (or org) secrets in this repository:
   - `PERIBOLOS_APP_ID`
   - `PERIBOLOS_APP_PRIVATE_KEY` (the PEM contents)

### 2. Protect `main`

- Require pull request reviews.
- Enable **Require review from Code Owners**.
- (Optional) Create a `peribolos-apply` **Environment** with the enterprise
  admins as **required reviewers** — this adds a manual approval gate before any
  mutation runs, on top of PR review.

### 3. Fix the CODEOWNERS placeholders

Replace `@neonephos/enterprise-admins` and the per-org owners in `CODEOWNERS`
with real teams/users.

---

## Onboarding a new org

1. **Seed from the live org** (captures current state so the first apply is a
   no-op):

   ```bash
   export GITHUB_TOKEN=...   # token/App token with org admin scope
   task dump ORG=<org> > orgs/<org>/org.yaml
   ```

2. **Trim** the dumped file to the org settings block, `admins`, `members`, and
   `teams` (drop the top-level `repos:` block — peribolos runs without
   `--fix-repos`). Keep every team's `repos:` block exactly as dumped: with
   `--fix-team-repos` on, a grant missing from the file is **revoked** on
   apply. Review the member/admin/team lists.
3. Add a `CODEOWNERS` line for `orgs/<org>/`. The workflow discovers orgs from
   `orgs/*/org.yaml` automatically — no workflow edit needed.
4. Open a PR. Confirm the `peribolos` check passes and its job log shows
   **no destructive changes** (a freshly-dumped config should be a near no-op).
5. Merge. The `peribolos` workflow reconciles in apply mode.

---

## Everyday operations

### Add/remove a member or change a team (any contributor)

Edit `orgs/<org>/org.yaml` in a PR:

```yaml
orgs:
  my-org:
    members:
      - alice        # add a member
    teams:
      reviewers:
        members:
          - bob      # add bob to the reviewers team
        repos:
          my-repo: write
```

The plan check must pass; a CODEOWNER approves; merge applies it.

### Run a dry-run locally

```bash
export GITHUB_TOKEN=...
task reconcile ORG=<org>
```

### Apply locally (rarely needed — CI does this)

```bash
# Dry-run (default: CONFIRM=false — safe, mutates nothing)
task reconcile ORG=<org>

# Actually mutate GitHub
task reconcile ORG=<org> CONFIRM=true
```

---

## Safety rails (configured in the Taskfile)

| Flag | Default | Purpose |
| ------ | --------- | --------- |
| `--confirm` | `CONFIRM=false` | No mutations unless `CONFIRM=true`. Local `task reconcile` is a dry-run by default; the `peribolos` workflow passes `CONFIRM=true` on push to `main` (PRs stay dry-run). |
| `--maximum-removal-delta` | `0.25` | Refuse runs deleting >25% of memberships (typo guard) |
| `--min-admins` | `2` | Refuse a config with fewer than 2 admins (lockout guard) |
| `--require-self` | `false` | If true, the bot must be an admin to apply |

Tune these via `task VAR=value ...` or environment variables (see `Taskfile.yml`).

---

## Config schema reference

`org.yaml` follows the Peribolos schema. Common fields:

```yaml
orgs:
  <org-name>:
    # org settings (omit any field to leave GitHub's current value untouched)
    default_repository_permission: read      # read|write|admin|none
    members_can_create_repositories: false

    admins:   [login1, login2]   # org owners (authoritative list)
    members:  [login3]           # org members (authoritative list)

    teams:
      <team-name>:
        description: ...
        privacy: closed          # closed (org-visible) | secret
        previously: [old-name]   # rename an existing team
        maintainers: [login1]
        members:     [login3]
        repos:                   # permission this team gets per repo
          repo-a: admin          # read|triage|write|maintain|admin
          repo-b: read
```

Full schema: <https://docs.prow.k8s.io/docs/components/cli-tools/peribolos/>

---

## Notes / limitations

- This setup manages **org settings, org membership (admins/members), teams,
  team membership, and team repo permissions**. It does **not** manage
  repositories themselves (`--fix-repos` off), direct repo collaborators
  (`--fix-collaborators` off), branch protection, or rulesets. For
  centrally-enforced repo policy, pair it with
  [`github/safe-settings`](https://github.com/github/safe-settings) later.
- `default_repository_permission: none` is deliberate: org membership grants no
  access to private repos. Access is explicit — team `repos:` blocks here, or
  direct collaborators on the repo. Repos holding personal data (e.g.
  `contactlist`) depend on this.
- Internal repos are unaffected: GitHub gives them a **minimum visibility of
  read for all enterprise members, even when the base permission is `none`**.
  `tac`, `Community-Management` and `social-media` therefore need no team or
  grant to stay readable org-wide.
- Keep the org settings block **minimal**. peribolos treats a listed empty
  string as a desired value, and GitHub returns `null` for unset `company` /
  `email` / `location`, so listing them as `''` makes every run issue an
  `EditOrg` call. Omitted fields are left untouched.
- Usernames are GitHub login handles (lowercase, no `@`).
- The `peribolos` container image tag is pinned in the `Taskfile`
  (`PERIBOLOS_IMAGE`); bump it deliberately.
