# GitHub for Bitbucket Server Users (Focused Guide)

Scope: Core concept mapping, account & identity structure, authentication & authorization, and how effective repository permissions are resolved in GitHub.

Out of Scope (intentionally): Migration mechanics, GitHub Actions, CI/CD authoring, monorepo vs polyrepo strategy, workflow automation specifics.

---

## 1. Core Concept Mapping (Adjusted Scope)

| Theme | Bitbucket Server Concept | GitHub Equivalent | Notes (Source Control Governance Focus) |
|-------|--------------------------|-------------------|-----------------------------------------|
| Top-Level Grouping | Project | Organization | Orgs host repos, teams, policies, security features. |
| Cross‑Org Umbrella | (N/A) | Enterprise Account | Optional: central policy, billing, SSO, audit. |
| Repository | Repository | Repository | Git semantics unchanged. Topics improve discoverability. |
| Permission Grouping | Groups (project/repo) | Teams (nestable) | Teams map to roles: read, triage, write, maintain, admin. |
| Branch / Merge Control | Branch Permissions + Merge Checks | Rulesets (branch / push / tag) | Rulesets are the modern generalized policy layer. |
| Required Build Status | Merge Check (external CI) | Required status checks (in ruleset) | Jenkins posts statuses via Status or Checks API. |
| Default Reviewers | Default reviewers list | CODEOWNERS + Teams | Path-pattern driven ownership & auto-review requests. |
| Tag Governance | Manual / convention | Tag rulesets (e.g. `v*`) | Restrict creation, updates, deletions of release tags. |
| Forking Model | Personal / project forks | User/org forks | Internal teams usually branch instead of forking. |
| Wiki | Repo wiki | Repo wiki | Both are Git repos (`<repo>.wiki.git`). |
| Snippets | Limited | Gists | Prefer repos for shared, governed scripts. |
| Automation Identity | Service accounts + PATs | GitHub Apps / Fine‑grained PATs | Apps are scoped & token-rotated. |
| Compliance / Audit | Local logs | Org & Enterprise audit log | Centralized, API accessible. |
| Ownership Mapping | Per-repo defaults | CODEOWNERS file | Declarative, versioned; last matching rule wins. |

---

## 2. Account & Identity Structure

| Aspect | Bitbucket Server | GitHub |
|--------|------------------|--------|
| User Namespace | Local / LDAP bound | Global user accounts |
| Grouping Mechanism | Groups | Teams (hierarchical) |
| Higher Aggregation | Projects group repos | Organizations (optionally under Enterprise Account) |
| Cross‑Org Governance | (N/A) | Enterprise Account (policy, SSO, audit unification) |
| Per-Repo Roles | Read / Write / Admin via groups | read, triage, write, maintain, admin |
| Nested Teams | Not typical | Supported (inheritance) |
| Automation Identities | Service users + credentials | GitHub Apps or machine users (fallback) |
| SSH Keys | User + deploy keys | User keys, deploy keys, machine user keys, SSH certs (Enterprise) |
| Ownership of Paths | Manual assignment | CODEOWNERS (path → team/users) |
| Policy Expression | Branch permissions / merge checks | Rulesets targeting branch / push / tag refs |

### 2.1 Recommended Structuring Pattern

| Layer | Purpose | Example |
|-------|---------|---------|
| Enterprise (optional) | Central auth, policy, audit | `acme-enterprise` |
| Organization | Business / platform boundary | `acme-platform` |
| Teams | Domain / capability / function | `payments`, `security`, `qa`, `platform-sre` |
| Repos | Service / library / infra units | `svc-payments`, `lib-auth`, `infra-terraform` |

### 2.2 Team Role Guidelines

| Role Needed | Suggested Role | Notes |
|-------------|----------------|-------|
| Active development | write | Branch pushes; merges gated by rulesets. |
| Leads / Maintainers | maintain | Manage settings (no destructive admin ops). |
| Release Engineering | maintain (plus tag ruleset access) | Controlled tagging & release gates. |
| Security Reviewers | write or maintain | Included via CODEOWNERS for sensitive paths. |
| Automation (Jenkins) | GitHub App installation | Avoid broad PATs. |

### 2.3 CODEOWNERS Placement & Example

Valid paths: root `/CODEOWNERS`, `.github/`, or `docs/` (root preferred for visibility).

```
/services/payment/ @payments @qa-team
/auth/**           @security-team
*.sql              @data-team
```

_Last matching rule wins; keep ordering intentional._

---

## 3. Authentication & Authorization

Focus: Human + automation access, credential types, and enforcement levers (no GitHub Actions usage assumed).

### 3.1 Authentication Methods

| Method | Description | Primary Use | Notes |
|--------|-------------|-------------|-------|
| Web Login (Password + SSO/MFA) | Browser auth | UI access | Enforce SSO + MFA. |
| SSH Keys | Public key auth | Git clone/push | Prefer `ed25519`; rotate stale keys. |
| HTTPS + PAT | Token via HTTP Basic | CLI / scripts | Prefer fine‑grained PATs, short-lived. |
| Fine-Grained PATs | Repo-scoped, permission-scoped | Temporary automation / personal scripting | Avoid broad classic PATs. |
| GitHub Apps | Installation tokens (short TTL) | Jenkins, bots, policy tools | Least privilege; tokens rotate. |
| Deploy Keys | Single-repo SSH key | Read-only or narrow automation | Use when App is overkill for a single repo. |
| GPG / SSH Signing Keys | Commit signature | Integrity/provenance | Combine with signed commit requirement. |

### 3.2 Authorization Constructs

| Construct | Scope | Purpose |
|-----------|-------|---------|
| Organization Roles | Org-wide | Governance & policy management. |
| Team Membership | Org / repo list | Aggregate permission assignment. |
| Repository Roles | Repo | read / triage / write / maintain / admin. |
| Rulesets | Ref patterns | Enforce reviews, checks, signatures, push limits. |
| CODEOWNERS | Path patterns | Auto reviewer requests (ownership mapping). |
| Status Checks | Commit contexts | Gate merges (e.g., Jenkins build). |
| Bypass Actors | Specific users/teams | Controlled override, audited. |

### 3.3 Rulesets vs Traditional Branch Protection

Rulesets:
- Multiple prioritized sets (branch, tag, push).
- Pattern-based inclusion/exclusion (`main`, `release/*`, `v*`).
- Central (org-level) or repo-level definition.
- Enforce: required reviews, required status checks, signature requirements, no force push, deletion blocks, PR-only changes, tag restrictions.

### 3.4 Status Checks (External CI: Jenkins)

| Aspect | Recommendation |
|--------|---------------|
| Context Names | Stable, lowercase (`jenkins/ci`, `jenkins/security-scan`). |
| Aggregation | One gating status for multiple internal steps to reduce ruleset churn. |
| Reliability | Ensure final success/failure posted to prevent “pending” merges. |
| Security | Use a GitHub App with minimal scopes (checks/statuses). |

### 3.5 Credential Hygiene

| Risk | Mitigation |
|------|------------|
| Broad, long-lived PATs | Replace with fine-grained PATs or Apps; enforce expirations. |
| Shared user tokens | Disallow; use App or machine user per integration. |
| Unused deploy keys | Periodic API audit → prune. |
| Overused bypass actors | Restrict & audit usage in logs. |
| Unsigned commits on critical branches | Enforce `required_signatures` in rulesets. |

### 3.6 Recommended Enforcement Baseline

| Control | Baseline Setting |
|---------|------------------|
| MFA & SSO | Mandatory |
| Ruleset on `main` | 2 approvals, Jenkins status required, block force push & deletions, signed commits |
| Ruleset on `release/*` | Inherit `main` + security review path (via CODEOWNERS) |
| Tag Ruleset `v*` | Restrict tag creation/deletion to Release team |
| CODEOWNERS | Critical services & sensitive paths covered |
| Automation Identity | Jenkins via GitHub App (no admin PAT) |

### 3.7 Example Ruleset Elements (Conceptual)

| Rule Type | Key Parameters | Purpose |
|-----------|----------------|---------|
| required_pull_request_reviews | approvals=2, dismiss_stale=true | Quality & freshness |
| required_status_checks | contexts=`["jenkins/ci"]`, strict=true | Build gate |
| restrict_force_pushes | block=true | Preserve history |
| restrict_deletions | block=true | Prevent accidental loss |
| required_signatures | — | Commit provenance |
| non_fast_forward | — | Linear history (optional) |

### 3.8 Commit Signing Options

| Method | Pros | Considerations |
|--------|------|----------------|
| GPG | Standard practice | Key distribution & trust setup required |
| SSH Signing | Simpler if SSH used | Requires recent Git |
| Enforcement | Ensures integrity | Provide setup docs & fallback guidance |

### 3.9 Least Privilege Automation Decision Tree

1. Multi-repo or dynamic access? → GitHub App.  
2. Single-repo read or simple clone? → Deploy key (read) or App.  
3. Temporary personal script? → Fine-grained PAT (short expiry, limited repos).  
4. Legacy broad PAT? → Replace with App; revoke old PAT.

### 3.10 Auditing & Monitoring

| Artifact | API / Source | Purpose |
|----------|--------------|---------|
| Rulesets | Org & repo rulesets endpoints | Detect drift & misconfigurations |
| Collaborators & Teams | Repos collaborators API + Teams API | Permission matrix review |
| Deploy Keys | Repo deploy keys API | Remove unused credentials |
| Audit Log | Org/Enterprise audit log | Track bypass, token events, ruleset changes |
| Status Check History | Commit status / checks APIs | Identify flaky gates |

---

## 4. Repository Access & Permission Resolution

Explains how GitHub derives a user’s effective permissions on a repository and how to inspect them (including base role, team roles, direct collaborator assignment, organization ownership, and visibility). Also clarifies that rulesets gate operations but do not alter the underlying permission level.

### 4.1 Permission Levels (Highest → Lowest)

`admin` > `maintain` > `write` > `triage` > `read` > `none`

| Level | Can (Highlights) | Cannot |
|-------|------------------|--------|
| admin | All, including delete repo, manage rulesets, collaborators | — |
| maintain | Manage settings, branches, labels | Delete repo, manage sensitive org settings |
| write | Push branches, merge PRs (if rules satisfied) | Change protected settings |
| triage | Manage issues/PR metadata (label, assign) | Push code |
| read | View, clone, comment | Push or triage |
| none | No access (except public visibility) | Everything else |

### 4.2 Sources of Permission

| Source | Description | Notes |
|--------|-------------|-------|
| Organization Base Role | Default minimum for all members (None/Read/Write) | Often set to None or Read. |
| Team Role Assignment | Team granted role on repo | Nested teams inherit. |
| Direct Collaborator Role | User explicitly assigned | Used for outside collaborators too. |
| Org Ownership | Owners implicitly have admin | Universal across org repos. |
| Repo Visibility | Public/Internal may grant read | Public: everyone; Internal: all enterprise members (read). |
| Fork Ownership | Fork grants control of fork, not base repo | Base repo permissions still govern merge. |

Effective permission = highest role from any source.

### 4.3 Resolution Algorithm (Conceptual)

```
if user is org owner: return admin

candidates = []

# Visibility-derived read
if repo public: candidates += [read]
else if repo internal and user in enterprise: candidates += [read]

# Base role
if base_role != none and user is org member: candidates += [base_role]

# Direct collaborator
if user is direct collaborator: candidates += [direct_role]

# Teams
for each team granting repo access where user is member (direct or inherited):
    candidates += [team_role]

effective = highest(candidates)
return effective
```

Then apply rulesets to allow/deny specific operations (push, merge, tag creation).

### 4.4 Interaction with Rulesets (Operation Gating)

| Operation | Underlying Needed Role | Gated Further By |
|-----------|------------------------|------------------|
| Clone private repo | read | SSO, IP allow list (enterprise), suspension |
| Push branch | write | Ruleset (may require PR-only, restrict actors) |
| Merge PR | write | Required reviews, status checks, signatures, linear history |
| Create / Update tag | write | Tag ruleset (restrict creation/deletion) |
| Change repo settings | maintain | Some actions require admin |
| Manage rulesets | admin | Org-level rulesets: org admin required |

Rulesets do not downgrade permission, they impose conditions.

### 4.5 Edge Cases

| Case | Effect |
|------|--------|
| Outside collaborator | Only explicit repo role (no base role). |
| Pending invite | No access until accepted. |
| SSO not authorized | Access blocked until user re-authenticates SSO. |
| Suspended user | No functional access despite stored assignments. |
| Fine-grained PAT with fewer scopes | Token limited even if user’s role higher. |
| GitHub App action | App’s installation permissions independent of user roles. |

### 4.6 Checking Effective Permission (UI)

1. Confirm org membership (Org → People).
2. Check team memberships (Org → Teams → search user).
3. In repository (Settings → Collaborators & teams) review direct collaborators & teams (requires maintain/admin).
4. If not listed:
   - Access may derive from base role (if ≥ read).
   - Or repo is public/internal.

### 4.7 REST API Methods

| Goal | Endpoint | Notes |
|------|----------|-------|
| Effective permission | `GET /repos/{owner}/{repo}/collaborators/{username}/permission` | Returns `permission` + `role_name`. |
| List collaborators | `GET /repos/{owner}/{repo}/collaborators` | Use `affiliation` filter if needed. |
| List repo teams | `GET /repos/{owner}/{repo}/teams` | Shows each team’s permission. |
| Check org membership | `GET /orgs/{org}/members/{username}` | 204 = member, 404 = not member. |
| User’s teams (self) | `GET /user/teams` | Requires `read:org` scope. |

Example (CLI):
```bash
gh api repos/YourOrg/your-repo/collaborators/some-user/permission
```

Sample response (abridged):
```json
{
  "permission": "write",
  "role_name": "write",
  "user": { "login": "some-user" }
}
```

Use `role_name` to distinguish `triage` vs `write` vs `maintain` (legacy `permission` field may simplify categories).

### 4.8 GraphQL to Inspect Permission Sources

````graphql
query ($org: String!, $repo: String!, $login: String!) {
  organization(login: $org) {
    repository(name: $repo) {
      name
      permissionSources(login: $login) {
        permission
        source {
          __typename
          ... on Repository { name }
          ... on Team { name slug }
          ... on Organization { name }
        }
      }
    }
  }
  user(login: $login) {
    login
  }
}