---
layout: post
title: "Building Production Agentic AI for Engineering Triage"
date: 2026-06-01
tags: [agentic-ai, langgraph, mcp, platform-engineering]
excerpt: >-
  A walk through the architecture of a production agentic-AI pipeline I built
  for performance-engineering triage — what it does, how it's wired, and the
  deliberate safety properties around an LLM with write-authority on
  enterprise systems.
---

*A walk through the architecture of a production agentic-AI pipeline I built
for performance-engineering triage — what it does, how it's wired, and the
deliberate safety properties around an LLM with write-authority on enterprise
systems. Codename details and proprietary specifics are removed; everything
below is the generic pattern.*

---

## The problem

Performance regressions in a large benchmark suite are a triage bottleneck.
Every nightly run produces hundreds of metrics, profiling artifacts, and
warning signals. A human engineer has to:

1. Notice the regression.
2. Pull profiling data, compare against baseline, identify the suspect change.
3. Open a ticket with the right component, severity, and a coherent
   description.
4. Attach the run artifacts so the on-call doesn't dig through CloudWatch.

This pattern repeats every day, on every benchmark execution. It is also the
exact shape of work that a well-bounded agent can take over — given the right
guardrails.

## What I built

A **production agentic AI pipeline** that watches benchmark runs and, when a
regression is detected:

- Classifies the regression by component and severity.
- Summarizes the relevant profiling runs into LLM-ready context.
- Files an audited ticket in the enterprise issue tracker with the artifact
  URLs attached.
- Hands off to a review agent and (in the multi-agent extension) to a coding
  agent that proposes a remediation pull request.

The whole pipeline runs unattended. The on-call engineer wakes up to a
triaged ticket with structured findings, not a wall of metrics.

## Architecture in one diagram

```
ECS benchmark run
     │
     ▼
EventBridge ─▶ SNS ─▶ Lambda ──┬──▶ LLM gateway
                                │
                                ├──▶ MCP server (issue tracker)
                                │
                                └──▶ Slack (structured blocks)
```

- **ECS** runs the benchmark and emits artifacts to S3.
- **EventBridge → SNS → Lambda** is the event fanout. The Lambda is the
  agent host.
- **LangGraph StateGraph** lives inside the Lambda. Nodes are the agents —
  classifier, executor, reviewer.
- **LLM gateway** is the governed entry point. Tool-calling and structured
  outputs go through it.
- **MCP server** exposes the issue tracker as typed tools — `create_problem`,
  `add_comment`, `assign_to_sprint`. The agent's authority is bounded by the
  tools the server chooses to expose.
- **Slack** receives a structured-block summary so humans see what the agent
  did and can intervene.

The whole thing is codified in Terraform: Lambda + IAM role + scoped Secrets
Manager allow-list, ECS task definition, SQS/SNS VPC endpoints, and a
reproducible LangGraph Lambda layer build pipeline. Reproducible means a
fresh AWS account can deploy the whole stack from `terraform apply`.

## The multi-agent extension

The simple version is a single classifier-then-tool agent. The richer version
implements the classic **planner / reviewer / executor** pattern over a
shared LangGraph state:

```
                       (loop back if review fails)
                ┌───────────────────────────────────┐
                ▼                                   │
    ┌────────────┐    ┌──────────┐    ┌──────────┐  │
    │ regression │ ─▶ │ ticket   │ ─▶ │ review   │ ─┤
    │ detector   │    │ filer    │    │ agent    │  │
    └────────────┘    └──────────┘    └──────────┘  │
                                            │       │
                                            ▼       │
                                      ┌──────────┐  │
                                      │ coding   │ ─┘
                                      │ agent    │
                                      └──────────┘
                                            │
                                            ▼
                                       PR + ticket
```

- **Regression detector** compares run metrics against a rolling baseline
  and emits a structured "what regressed and by how much" finding.
- **Ticket filer** uses the MCP tool to create an audited ticket.
- **Review agent** verifies the ticket has the right component, severity,
  and reproduction steps. If it doesn't, the loop goes back.
- **Coding agent** picks up confirmed tickets, proposes a remediation, and
  opens a pull request.

Each agent has narrow authority. The review agent can't file tickets. The
coding agent can't edit metrics. Authority and capability are separated by
construction.

## Safety properties

When an LLM has authority to write to enterprise systems, you build
defense-in-depth.

| Layer | What it stops |
|-------|---------------|
| **Classification** | Off-topic or obvious-attack inputs never reach tool calls |
| **Closed-component allowlist** | The agent can only file tickets in pre-approved components |
| **Sandbox-isolated runtime** | The agent runs in Lambda, not on a long-lived server with broad IAM |
| **Scoped IAM + Secrets allow-list** | The Lambda role has only the secrets and APIs the agent needs |
| **Deterministic provenance footer** | Every ticket the agent files carries a footer naming the agent run, version, and source artifact URLs |
| **Audit trail** | The full graph state is logged per invocation; you can replay any decision |

These layers are independent. Any one of them can fail and the others still
contain the blast radius.

## The evaluation flywheel

A pipeline that takes write-actions on production needs an honest evaluation
discipline, not just unit tests. The flywheel:

1. Every run emits structured telemetry — what the agent classified, what
   tools it called, how long each call took, which decisions changed the
   ticket vs. left it unchanged.
2. A **baseline + rolling-history tracker** stores reference behavior for
   well-known scenarios.
3. Regressions in agent behavior — wrong component picked, missing artifact
   URL, latency spike — surface in the same Slack channel as benchmark
   regressions, treated as first-class quality signals.
4. The eval harness has unit-test coverage across tool-call parsing, error
   paths, profiling analysis, and recommendation math. The recommendation
   tests caught a 3× CPU under-reporting bug that had been live for weeks
   before the agent was built.

The flywheel is what turns "we built an agent" into "we operate an agent."

## Lessons learned

A few things that surprised me building this end to end:

- **The hard part is bounding authority, not bolting on capability.** Adding
  another tool to the agent is one MCP endpoint and a docstring. Deciding
  what the agent is *not allowed to do* — and making that property robust to
  prompt drift — is most of the engineering.
- **MCP makes tool surfaces self-documenting.** The schema-as-interface
  pattern means the agent rarely calls a tool with wrong types. The few
  failures we had were semantic ("right tool, wrong target") not syntactic.
- **Closed-component allowlists are underrated.** A simple list of approved
  ticket components, checked at the tool boundary, prevented a category of
  failure modes that no amount of prompt engineering would have caught.
- **The data flywheel pays off slowly, then suddenly.** For the first month
  the eval harness produced flat lines. Then a model upgrade shifted
  classification distribution overnight, and the rolling-history tracker
  caught it before any human did.

## What's next

- Replace the rule-based regression detector with a learned scorer over the
  run telemetry.
- Move the LLM gateway from per-invocation calls to **prompt caching** so the
  fixed system context (component catalog, classification rubric, tool
  schemas) doesn't count toward every token bill.
- Extend the coding agent to also draft a unit test that would have caught
  the regression — closing the loop from "filed a ticket" to "would not
  ship the same bug twice."

---

*If any of this is useful, the multi-agent pattern lives in
[github.com/kanaparthikiran/multi-agent-langgraph-demo](https://github.com/kanaparthikiran/multi-agent-langgraph-demo)
as a reference implementation you can run on NVIDIA NIM in a few minutes.*
