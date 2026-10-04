---
title: Vibe Coding a CRUD App
sidebar_position: 96
hide_title: true
categories: ai, agents, golang, react, render
---

# Vibe Coding a CRUD App: From One Prompt to a Live URL

Every platform team eventually needs the same boring app: a list of *things*, a detail page per thing, a form to edit it, and some notion of who is allowed to see what. Clusters, machines, tenants, environments, feature flags. It's never the interesting part of the job, and it's never small enough to skip.

This time I wanted to see how far I could get by typing a few sentences and letting Claude Code do the rest. The *thing* was Talos Linux clusters and the machines inside them, which is the inventory I was keeping in a spreadsheet after [building the on-prem cluster](/on_prem_k8s). Here's the full transcript of what I typed, what came back, where it stalled, and what I had to do myself.

---

## The Setup: Skills, Not Prompts

The trick that made this work wasn't a clever prompt. It was a **skill**: a folder of markdown instructions, templates and shell scripts that tells the agent *how this kind of app is built here*. I used a set of `crud-*` skills installed as a Claude Code plugin:

| Skill | Owns |
|---|---|
| `crud-app-scaffold` | the spec, project layout, root files, orchestration of the others |
| `crud-backend-go` | Go + Gin + GORM conventions, server wiring, errors, pagination |
| `crud-auth` | email/password login, JWT in an HttpOnly cookie, CSRF guard, admin users |
| `crud-authz` | RBAC roles plus ReBAC relation tuples: owners, editors, viewers, teams, parent inheritance |
| `crud-resource` | adding one resource end to end, cloned from a golden `project` example |
| `crud-frontend-react` | Vite + React + shadcn/ui + TanStack Query shell and conventions |
| `crud-deploy-render` | Dockerfile, `render.yaml`, deploying with Task |
| `crud-quality-gates` | gofmt, golangci-lint, eslint, tsc, vitest, osv-scanner |

The skill pins the stack, the folder layout, the error envelope, the pagination contract, even the comment markers where new code gets inserted. The agent copies and adapts. It does not get to invent an architecture at 2 a.m.

The skills are open source: [github.com/viswanathanb/vibe-app-skill](https://github.com/viswanathanb/vibe-app-skill). In Claude Code they install as a plugin:

```bash
# add the marketplace once, then install the plugin
claude plugin marketplace add viswanathanb/vibe-app-skill
claude plugin install vibe-app-skill@vibe-app-skill

# later: pick up new skill versions
claude plugin update vibe-app-skill@vibe-app-skill

# in a new, empty project directory
claude
> /vibe-app-skill:crud-app-scaffold
```

The [README](https://github.com/viswanathanb/vibe-app-skill#readme) also covers installing the skills as plain folders for other coding agents, what each `crud-*` skill owns, and how to maintain them. The app that came out of this post is at [github.com/viswanathanb/talos-manager](https://github.com/viswanathanb/talos-manager).

```bash
   me ──prompt──▶ Claude Code
                     │ reads
                     ▼
             ┌────────────────┐
             │  crud-* skills │  conventions + scripts + golden example
             └───────┬────────┘
                     │ runs
                     ▼
   new-app.sh ──▶ backend/ + frontend/ + Taskfile + Dockerfile + render.yaml
                     │
   add-resource.sh ──▶ one package + one feature folder per resource
                     │
   task check ──▶ lint, test, build, osv
```

---

## 1. The Prompt

```bash
/vibe-app-skill:crud-app-scaffold

Create "Talos Manager", Go module github.com/viswanathanb/talos-manager.
It manages Talos Linux Kubernetes clusters.
Resources:
- Cluster (owned + shareable): name, endpoint URL, Kubernetes version,
  Talos version, status (provisioning|ready|degraded|deleting), description.
- Machine (child of Cluster): hostname, IP address, role (controlplane|worker),
  status (pending|ready|error).
Members can create clusters and share them with teams; viewers only see
what's shared with them.
Start with the spec and confirm it with me before generating code.
```

That last line matters. The skill's first step is to write an `APP_SPEC.md` and stop. The agent filled in everything I hadn't said and listed its assumptions back to me:

- cluster name unique per owner, endpoint must be an `https://` URL, status defaults to `provisioning`
- hostname and IP unique within a cluster, IP validated as IPv4 or IPv6, role defaults to `worker`
- machines live as a table inside the cluster detail page, no standalone page
- anyone with `editor` on a cluster can add or edit its machines; viewers see them read-only
- deleting a cluster cascades to its machines

Each of those is a decision I would otherwise have discovered as a bug three weeks later. Reading a one-page spec and saying "yes" took two minutes. **Spec first is the cheapest review you will ever do.**

---

## 2. The Skeleton

One script call. It copied the root files, generated the Go module, ran `bun create vite`, installed shadcn, copied the skeleton UI, and verified gofmt, vet, build, tests, prettier, eslint, tsc and the production bundle. Output:

```bash
talos-manager/
  AGENTS.md  APP_SPEC.md  Taskfile.yml  docker-compose.yml
  Dockerfile  render.yaml  .env.example  .gitignore
  backend/   Go API: auth, users, RBAC + ReBAC, teams
  frontend/  React SPA: login/signup, teams + members, admin users
```

Before I had written a single line about clusters, I had login, sessions, an admin page, teams, and a generic sharing API. That's the part of every internal tool that eats the first two weeks.

---

## 3. Two Resources, Two Access Patterns

This is where the skills earned their keep. The authz skill defines three patterns and makes you pick one per resource:

| Pattern | Who can see it | Example |
|---|---|---|
| **owned** | creator is owner, shares with users or teams as editor/viewer | Cluster |
| **child of parent** | inherits from the parent via a `parent` tuple | Machine |
| **admin-managed catalog** | admins edit, everyone signed in reads | reference data |

Under the hood it's a small Zanzibar: relation tuples in Postgres (`cluster:1#owner@user:7`, `cluster:1#viewer@team:3#member`, `machine:9#parent@cluster:1`) and a schema that says which relations grant which permissions.

```go
"cluster": {
    Relations: {"owner": {"user"}, "editor": {"user", "team#member"}, "viewer": {"user", "team#member"}},
    Permissions: {"view": {"viewer", "edit"}, "edit": {"editor", "manage"}, "delete": {"owner"}, "manage": {"owner"}},
},
"machine": {
    Relations:   {"parent": {"cluster"}},
    Permissions: {"view": {"parent->view"}, "edit": {"parent->edit"}, "delete": {"parent->edit"}, "manage": {"parent->manage"}},
},
```

Machines have **no relations of their own**. Every permission check walks up to the cluster. Share a cluster with a team, and the team sees every machine in it. Revoke it, and they're gone. No per-machine ACLs to keep in sync.

**Cluster** came out of the `add-resource.sh` script in one shot, then the agent swapped the placeholder fields for the real ones: a composite unique index on `(created_by_id, name)`, a service-level check that the endpoint parses as `https`, an allow-list of sortable columns, and a React form whose zod schema mirrors the Go binding tags.

**Machine** was hand-written from the same template because the child pattern wants different plumbing: routes nested under the parent (`POST /api/clusters/:id/machines`), `Authorize("edit", cluster)` instead of an RBAC create action, a `parent` tuple granted in the same transaction as the row, and a `ClusterID` that is simply absent from the update input so it can't be changed.

One wrinkle the agent solved on its own: the cluster's delete needs to cascade to machines, but `machine` imports `cluster` for the parent type. It added a tiny interface on the cluster side, implemented it on the machine service, and wired them together in `server.go`. Textbook dependency inversion, unprompted, and it even left a compile-time assertion.

---

## 4. "A Few Eternities Later": Testing

`task check` passed. That proves the code compiles and the linters are happy. It proves nothing about whether Bob can read Alice's cluster.

The agent wrote a curl script with four cookie jars (admin, two members, one viewer) and ran it against a real Postgres. Fifty assertions:

```bash
# isolation
  ok   bob list has only his own (200)      bob total=1
  ok   bob detail -> 403
  ok   bob machines -> 403
  ok   bob patch machine -> 403
# share with bob as viewer
  ok   bob now sees it in list              bob total=2
  ok   bob detail ok                        bob perms=['view']
  ok   bob still cannot edit cluster (403)
# share with a team (carol is a member) as editor
  ok   carol sees cluster                   perms=['edit', 'view']
  ok   carol can add machine via team editor (201)
  ok   carol cannot delete cluster (403)
  ok   alice removes team from cluster (204)
  ok   carol lost access (403)
# cascade delete
  ok   alice deletes cluster (204)
  ok   machine gone (admin -> 404)
```

followed by a `select object_type, count(*) from relation_tuples` to confirm nothing dangled.

Things that went wrong, in order:

1. **Port 5432 was taken** by a Postgres already on my laptop. The agent moved the compose port to 5433 in `.env` and moved on.
2. **The first test run reported 29 failures.** All of them were a quoting bug in the agent's own test helper. It noticed, fixed the helper, reset the database, and re-ran. 44 passed.
3. **Three failures survived.** Sharing with a team returned 400. Running the same request by hand returned 204. The culprit was macOS shipping **bash 3.2**, which treats a `#` inside `"$(...)"` as a comment and silently truncated `team:1#member"}` out of the JSON body. The API was fine; the test harness was from 2007.
4. **I don't have the Chrome extension**, so there was no browser for the agent to drive. It found Google Chrome on disk, launched it headless with a remote debugging port, and wrote an 80-line Bun script speaking the DevTools protocol: log in, click through, fill forms with React-compatible setters, take screenshots. It then *read the screenshots* to confirm the machine table, the 409 toast and the share dialog rendered.
5. **One real UI bug** fell out of that: the IP field said "Invalid IPv4 address" for a bad IPv6 because zod's union only reports its first branch. Fixed, rebuilt, re-tested.

> please deploy to local and test it

turned into: build the production Docker image, run it in production mode against the local database with a bootstrap admin, re-run the fifty API checks against the container, re-run the browser walkthrough. All green.

---

## 5. Deploy to Render: Where the Vibe Stops

> lets deploy to render

This is the part that is **not automatic**, and I think it's right that it isn't. The agent did everything it could without my credentials:

- reviewed `render.yaml` (one Docker web service, one Postgres, same region, `JWT_SECRET` generated by Render, bootstrap admin as `sync: false`, signup off)
- checked the Dockerfile's Go image matched `go.mod`
- confirmed `.env` was ignored and no secret literals were in the tree
- pushed `main` to GitHub

Then it stopped and handed me a short list:

```bash
1. Log the Render CLI in:   ! render login
2. Create the Blueprint in the browser:
   https://render.com/deploy?repo=https://github.com/viswanathanb/talos-manager
   Render will prompt for BOOTSTRAP_ADMIN_EMAIL and BOOTSTRAP_ADMIN_PASSWORD.
3. Tell me when it's applied, ideally with the srv-... ID from the dashboard URL.
```

Three clicks and one password later I pasted the service ID back. It stored the ID in `.env` for `task deploy`, then probed the live URL:

```bash
healthz                         {"status":"ok"} [200]
login page                      200 text/html
deep link /clusters/1           200 text/html      (SPA fallback works)
unauthenticated /api/clusters   401
POST without X-Requested-With   403 missing X-Requested-With header
signup                          403 signup is disabled
security headers                HSTS, CSP, X-Frame-Options DENY, nosniff
```

Auto-deploy is on, so every push to `main` rebuilds the image and rolls it out.

---

## What I Actually Did

| Me | The agent |
|---|---|
| Typed four prompts | Wrote ~120 files across Go and TypeScript |
| Read a one-page spec and said yes | Listed its assumptions before writing code |
| Watched | Ran lint, tests, build, osv-scanner after every change |
| Watched | Wrote and ran 50 API assertions with four accounts |
| Watched | Found Chrome, drove it headless, read its own screenshots |
| Ran `render login`, clicked Apply, typed a password | Reviewed the Blueprint, pushed, verified the live URL |
| Committed to GitHub | Pointed out my PAT was embedded in the git remote URL |

Honest caveats:

- **The skill is the product.** Without it the same prompt would have produced a plausible-looking app with a different auth story each time. The constraints are what made the output boring, and boring is the goal for a CRUD app.
- **Dependencies aren't free.** osv-scanner flagged two advisories with no fixed version yet, one in `golang.org/x/crypto` and one transitive npm package. The agent reported them and did not add ignores. That is the correct behaviour and also a reminder that a generated app is still an app you own.
- **Free tier is a demo tier.** The Render service sleeps when idle and the free Postgres expires. The agent said so, twice.
- **I still own the decisions.** Who can edit machines, whether a viewer-role user granted editor on a cluster may add machines (yes), whether cluster delete cascades (yes). The agent proposed; I approved. The spec file records all of it.

---

## Summary: The Flow

1. Install a skill that encodes your conventions, golden example and scripts ([vibe-app-skill](https://github.com/viswanathanb/vibe-app-skill), see its README)
2. One prompt: name, module, resources, access patterns
3. Agent writes the spec, you say yes
4. Script generates the skeleton: auth, users, teams, sharing, quality gates
5. One resource at a time, parents first, `task check` after each
6. Two-account tests against a real database, then a real browser
7. Production image locally, same tests again
8. Push; the human does the three steps that need credentials
9. Agent verifies the live URL

From an empty directory to `https://talos-manager-p9jd.onrender.com` in one sitting, with the only hand-written artefacts being four prompts and a password. The spreadsheet is retired.
