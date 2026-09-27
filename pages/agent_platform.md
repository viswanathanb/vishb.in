---
title: Agent Platform
sidebar_position: 97
hide_title: true
categories: ai, agents, platform
---

# Building an Agent Platform on Kubernetes

Every team now wants "AI agents". Within a few weeks you have MCP servers running on laptops, prompts pasted into wikis, skills copied between repos, and API keys for three LLM providers sitting in `.env` files. Someone asks the obvious questions: *which agents exist, who owns them, which tools can they call, which models do they use, and who pays for it?*

That's a platform problem, not an AI problem. This is how I wired one together from open source components: **agentregistry**, **kagent** and **agentgateway**, on AKS with Cilium, using Claude on Microsoft Foundry as the model.

---

## The Pieces

Four jobs, each done by a separate component:

| Job | Component | What it owns |
|---|---|---|
| **Catalog** | agentregistry | What exists: agents, MCP servers, skills, prompts, models, versions |
| **Runtime** | kagent | Running agents and MCP servers as Kubernetes workloads |
| **Gateway** | agentgateway | Every LLM (and later MCP) call: credentials, allowed models, token usage |
| **Network** | Cilium | Who can talk to whom, the backstop when everything else fails |

And one thing that ties them together, which none of them provides: **identity**. More on that later.

```
  git / arctl
      │
      ▼
 ┌──────────────┐   HTTP (one way)   ┌──────────────┐
 │ agentregistry│ ─────────────────▶ │    kagent    │
 │   catalog +  │                    │  controller  │
 │  deployments │ ◀── discovery ──── │              │
 └──────────────┘    (every 60s)     └──────┬───────┘
                                            │ creates
                               ┌────────────┴────────────┐
                               ▼                         ▼
                        ┌─────────────┐           ┌─────────────┐
                        │  agent pod  │ ───MCP──▶ │ MCP server  │
                        │  (A2A, BYO) │           │    pod      │
                        └──────┬──────┘           └─────────────┘
                               │ LLM
                               ▼
                        ┌─────────────┐   key    ┌──────────────┐
                        │agentgateway │ ───────▶ │   Foundry    │
                        │             │          │   (Claude)   │
                        └─────────────┘          └──────────────┘
```

---

## 1. The Registry: A Catalog, Not a Runtime

agentregistry is a catalog with a deployment controller attached. It stores versioned, tagged records of:

- **MCP servers**: either a runnable package (npm, PyPI or OCI) or a remote URL
- **Skills**: pointers to a git repo and folder, pinned to a commit by a controller
- **Prompts**: the prompt text itself, versioned
- **Agents**: a container image plus references to the MCP servers, skills and prompts it uses
- **Runtimes**: *where* things can be deployed
- **Deployments**: *this* agent on *that* runtime with *these* overrides

Skills and prompts are shared, versioned content. Servers and agents are workload definitions, and the images and packages themselves live in the usual registries.

A nice touch: when you register an MCP server from npm or PyPI, the registry checks that the package actually declares that MCP name. You can't claim someone else's package.

**The catch:** the open source build has **no authentication and no authorization**. Anyone who can reach the API is an admin. There are clean interfaces to plug your own in, but out of the box it's safe only on localhost.

That settled the operating model for me: **git is the governance layer**. Manifests live in a repo, PRs are reviewed, CI runs a dry-run on the PR and applies on merge. The UI is for browsing, not for clicking "create". Registry writes and code review become the same thing.

---

## 2. The Runtime: kagent

The registry doesn't talk to Kubernetes at all. It talks to **kagent** over HTTP, and kagent creates the workloads. This split is easy to miss and matters a lot:

- The registry does **not** need to run in the same cluster. It only needs to reach kagent's API. One registry can drive several clusters, one Runtime record each.
- Traffic is **one way**. Agent pods never call back to the registry. MCP URLs, model settings and config are injected as environment variables at deploy time.
- There are **two levels of reconciliation**. The registry's controller reconciles Deployment records against kagent. kagent's controller reconciles its own resources into pods. Edit an Agent in the registry and its deployments roll forward.

For MCP servers, kagent does something clever: each MCP pod runs a small gateway as an adapter. It starts the stdio server (`npx …` or `uvx …`) and exposes it as HTTP MCP inside the cluster. Remote MCP servers get no pod at all, just a registration.

For agents, the path that works through the registry is **bring your own image**: any container that speaks A2A. kagent also has its own "declarative" agents (a prompt plus tools, run by kagent's engine) and its own UI to create them. That overlaps with the registry, and I keep the kagent UI for operations only: chatting with an agent, checking which tools were discovered.

Two things surprised me when I read the RBAC:

- kagent's API runs with **authentication off by default**. Anything that can reach it can create agents. A Cilium policy restricts it to the registry and the agents' own namespace.
- The MCP controller has **cluster-wide** rights to create Deployments and service accounts, regardless of which namespaces you told kagent to watch. Worth scoping down before tenants share the cluster.

---

## 3. The Gateway: One Door for Every LLM Call

Agents never talk to the LLM provider directly. They talk to **agentgateway**, and the gateway holds the real credentials.

```
  agent ──(placeholder key)──▶ agentgateway ──(real key)──▶ Foundry
                                    │
                                    ├─ allowed models
                                    ├─ token usage per call
                                    └─ one place to rotate keys
```

This gives you, almost for free:

- **No provider keys in agent pods.** The agent gets a gateway URL and a dummy key. The gateway swaps it for the real one.
- **A model allow-list.** Only models configured in the gateway exist, as far as agents are concerned.
- **Cost visibility.** Every call is logged with model and token counts. FinOps starts here.
- **Swappable backends.** Switching an agent between the gateway and the provider directly, or between two gateways (agentgateway vs Envoy AI Gateway, say), is an environment variable. They coexist fine for evaluation.

agentgateway has native support for Claude on Foundry: it builds the right Anthropic path and headers. Two gotchas cost me time. First, setting the chart's config **replaces all of its defaults**, including the listener, so you have to declare the port yourself. Second, our Foundry endpoint accepted the key as a Bearer token and rejected it in an `api-key` header.

The same gateway should front **MCP traffic** too: tool-level allow-lists per caller, one audit trail. That's the next step. Today agents still reach MCP servers directly.

---

## 4. Identity: The Common Factor Nobody Issues

Every policy question comes down to *who is calling*:

- Which agent may use which MCP server?
- Which tools may it call?
- Which models, and how many tokens?

The registry names things. kagent runs them. The gateway checks tokens. **None of them issues identity.** That has to come from the platform.

The plan that fits AKS:

```
 registry Deployment "opsassistant"
        │  (predictable name)
        ▼
 kagent Agent "opsassistant-<hash>"
        │  Kyverno injects the platform-owned service account
        ▼
 pod runs as SA ──▶ Workload Identity / projected token
        │
        ├──▶ Cilium:       pod labels → allowed destinations
        └──▶ agentgateway: token claims → allowed models, servers, tools
```

- **kagent supports setting an agent's service account**, and falls back to creating one per agent. A **Kyverno** mutation can set a platform-created one, so no registry changes are needed and the registry can't choose an identity.
- The attack surface moves to **names and namespaces**. Whoever can overwrite a Deployment record or create a Runtime pointing at another namespace inherits that identity. Governed writes (GitOps and review) still matter.
- An admission policy is the in-cluster backstop: agent pods may only use service accounts bound to them.

---

## 5. Building an Agent

The flow from nothing to a running agent:

1. **Scaffold** with `arctl init agent`. It generates an ADK (Python) project, the agent manifest, and wiring for the MCP servers you name.
2. **Point the model at Foundry.** I swapped the template's model adapter for the Anthropic SDK's Foundry client, which authenticates with a key or Entra ID and can target either Foundry directly or the gateway.
3. **Build and push** the image to ACR.
4. **Register** the Agent in the registry.
5. **Deploy**: a Deployment record binds the agent to a runtime, with env overrides that point the model at the gateway.

One wrinkle: the registry's own **Model** kind can't describe a Foundry model yet, so the model settings travel as Deployment env instead. And the agent's persona comes from the code built into the image, not from the registry's description field. An Agent can reference registry prompts and skills, and the registry validates those references and redeploys when they change, but their content is never delivered to a BYO agent on kagent.

The first real test worked end to end: an A2A request to the agent, two LLM calls through the gateway (one to decide on a tool, one to answer), a Microsoft Learn tool call, and a sourced answer back.

The agent is **reactive**: idle until someone sends it a message, autonomous only in choosing its tools. Making it proactive means adding a trigger (a CronJob, an Alertmanager webhook, another agent), and once no human is in the loop, the gateway policies stop being optional.

---

## Where It Stands

| Area | Today | Target |
|---|---|---|
| Namespaces | one namespace for everything | a namespace and Runtime per tenant |
| AuthN / AuthZ | none on registry, kagent API or gateway | Entra ID, registry authz, gateway JWT |
| Agent identity | per-agent service account, unused | platform-owned service account, Workload Identity |
| LLM credentials | API key in the gateway | gateway Workload Identity |
| MCP traffic | direct, bypasses the gateway | through the gateway, Cilium egress lock |
| Registry | on a laptop, port-forwarded | in-cluster |
| Delivery | manual apply | GitOps CI |

---

## Lessons Learned

- **Read the code, not the README.** The registry README says it configures gateway routing automatically. The current code doesn't. Plan to wire the gateway yourself.
- **The UI is always the least capable path.** The registry UI only offers "Deploy" for OCI packages, while the backend happily deploys npm, PyPI and remote servers. Another reason for GitOps.
- **Defaults are insecure everywhere.** No auth on the registry, no auth on kagent's API, no inbound auth on the gateway. Network policy is doing all the work until identity lands.
- **Identity is the design.** Catalog, runtime and gateway are the easy parts. Deciding who issues the agent's identity, and how it flows into every policy, is where the real work is.

---

## The Real Gaps

The pipeline works. What's missing is everything that makes it safe to share. In rough order of importance:

**1. Identity.** Nothing in the stack issues an identity to an agent. kagent creates a service account per agent, but nothing uses it: no Workload Identity, no token the gateway checks, no link to an owner. Until agents have an identity, every other policy is keyed on IP addresses and namespaces.

**2. Authentication and RBAC.** None of the three control planes checks who is calling. The registry treats every request as an admin, kagent's API trusts whatever user ID you send, and the gateway accepts anything that reaches it. There's no concept of "team A owns these agents" or "only platform admins may create runtimes". The hooks exist, but you have to write the providers.

**3. Tenancy.** Everything runs in one namespace. The registry has namespaces in its data model but hides them, and kagent's MCP controller has cluster-wide rights regardless. Real multi-tenancy needs a namespace and Runtime per tenant, scoped kagent RBAC, and authorization that actually enforces the boundary.

**4. Tool-level control.** Only LLM traffic goes through the gateway. MCP calls go straight from the agent to the server, so there's no answer to "which agent called which tool, with what arguments". Some tool descriptions even tell the model it now has internet access. Tool allow-lists and egress control belong in the gateway and in Cilium, not in the agent's good behaviour.

**5. Human in the loop.** Agents choose their own tools, and nothing approves a risky call before it happens. That's fine for a read-only assistant answering questions. It isn't fine once an agent can change infrastructure, or runs on a trigger with nobody watching.

**6. Configuration spread across three places.** The model is configured in the gateway, in the registry's Model kind (which can't express Foundry yet), and in kagent's ModelConfig, and none of them know about each other. The agent's prompt and persona are baked into the image, so the registry's prompts don't reach it. Changing how an agent behaves means a rebuild, not a reviewed config change.

**7. Credentials and supply chain.** The gateway holds a provider API key from a manually created secret. Agent images are pushed with mutable tags, unsigned, and pulled without provenance checks. Workload Identity for the gateway, secrets from Key Vault, immutable signed images, and an admission policy that enforces them are all still to do.

**8. Delivery and drift.** The registry is push-based: nothing reconciles git against it, removed manifests aren't pruned, and anything that can reach the API can change state outside git. GitOps only governs what goes through the pipeline. Everything else needs to be closed off.

**9. Audit end to end.** Each component logs its own piece: the registry its records, kagent its sessions, the gateway its token counts. Nothing ties a single user request to the agent, the tools it called, and the tokens it spent. For incident review and FinOps, that correlation needs building.

None of these are exotic. They're the same gaps every platform has on day one. With agents they matter sooner, because the workload makes its own decisions about what to call next.

---

_Next: routing MCP through the gateway, and giving agents a real identity with Workload Identity and Kyverno._
