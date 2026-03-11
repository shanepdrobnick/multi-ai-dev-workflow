# The Multi-Repo Bridge Pattern
### An extension of the Multi-AI Bridge Pattern for multi-repository infrastructure and platform work

**Invented by Shane Drobnick — March 2026**
Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — free to use, attribution required.

> This document extends the Multi-AI Bridge Pattern (Shane Drobnick, March 2026).
> If you use either pattern in your work, a credit to Shane Drobnick with a link to
> [https://github.com/shanepdrobnick/multi-ai-bridge-pattern](https://github.com/shanepdrobnick/multi-ai-bridge-pattern)
> is required under the licence terms.

---

## The Problem With Multi-Repo Work

The original Multi-AI Bridge Pattern solves coordination within a single project.
But real platform and infrastructure work doesn't live in one repo. It lives across
many — infrastructure, Helm charts, deployment manifests, workflow orchestration,
application code — each with its own conventions, patterns, and tribal knowledge.

The result is a compounding coordination problem:

- Each repo requires its own discovery phase before anything can be written
- Decisions made in one repo have downstream consequences in others
- Tribal knowledge about naming conventions, module versions, and architectural patterns
  is scattered across four or five codebases
- Context is lost not just between sessions but between repos within the same session
- The IDE AI has no way to know that the IAM role name it writes in Terraform must
  exactly match the service account name in the Helm chart, which must match what
  ArgoCD deploys, which must match what the orchestration layer passes to the pod

Without a coordination layer, you make that mistake. Then you spend two hours debugging
a pod that won't start because a service account name has a hyphen where there should
be an underscore.

---

## The Solution

A **context.json bridge file** that spans all repos, plus a **recon-first discipline**
that ensures the creative AI understands each repo's patterns before the IDE AI writes
a single line.

```
┌─────────────────┐                                    ┌─────────────────┐
│                 │  ──── context.json (all repos) ──► │                 │
│   Creative AI   │                                    │     IDE AI      │
│  (Architect /   │  ◄──── scan reports + output ───── │ (Implementer /  │
│ Decision Layer) │                                    │  Executor)      │
└─────────────────┘                                    └─────────────────┘
         ▲                                                      │
         │                 Developer                            │
         └──────────── (Technical Director) ───────────────────┘
                    routes prompts, relays results,
                    makes final calls
```

The context.json is the single source of truth that travels across every repo.
It accumulates knowledge as each repo is explored, and it carries that knowledge
forward — not just through the current session, but into future sessions and
future integrations built on the same pattern.

---

## The Recon-First Discipline

The most important rule of the Multi-Repo Bridge Pattern:

**Never author before you recon.**

Before any file is written in any repo, the creative AI generates a targeted scan
prompt for the IDE AI to execute. The scan returns the current state of the repo —
existing patterns, module versions, naming conventions, what already exists and
what doesn't. The creative AI analyses the results, resolves open questions, and
only then generates authoring prompts.

This prevents the most expensive class of mistake in multi-repo work: writing
something that is internally correct but externally inconsistent — a Helm chart
that references a service account the Terraform IAM module named differently,
or an ArgoCD manifest that points to a chart path that doesn't exist yet.

```
For each repo:
  1. Recon scan    → IDE AI reads, reports, does not write
  2. Analysis      → Creative AI resolves open questions, updates context.json
  3. Authoring     → IDE AI writes exactly what the creative AI specified
  4. Review        → IDE AI checks its own output against a checklist
  5. Assessment    → Creative AI reviews, approves or issues fix prompt
  6. Commit        → Developer commits with message from creative AI
```

---

## context.json Structure for Multi-Repo Work

Unlike the single-project `ai_instructions.json`, the multi-repo context.json
is designed to travel. It captures the full pipeline, not just one task.

```json
{
  "project": "Human-readable project name",
  "owner": "Shane Drobnick",
  "repos": {
    "repo_a": "purpose and location",
    "repo_b": "purpose and location"
  },
  "approach": "AI-Bridge Pattern description",
  "ide_ai_conversation_guidance": {
    "new_conversation_when": ["switching repos", "new distinct task", "context drift"],
    "continue_same_conversation_when": ["iterating same file", "running review after author"],
    "standard_workflow": "author → review in same conversation → relay to creative AI"
  },
  "architecture": {
    "description": "what the system does",
    "components": {}
  },
  "environments": {
    "pattern": "how environments are separated",
    "current_target": "which environment is being built now"
  },
  "resolved": {
    "question_id": "answer and evidence"
  },
  "open_questions": [
    {
      "id": "OQ-1",
      "area": "which repo or concern",
      "question": "what needs answering",
      "impact": "HIGH | MEDIUM | LOW — what it blocks"
    }
  ],
  "known_patterns": {
    "repo_name": {
      "modules": {},
      "naming_conventions": {},
      "known_arns_and_ids": {}
    }
  },
  "files_to_modify": {
    "repo/path/file.ext": "what to add and why"
  },
  "progress": {
    "repo_a": "COMPLETE | IN PROGRESS | NOT STARTED | BLOCKED",
    "repo_b": "NOT STARTED"
  },
  "for_next_instance": {
    "what_changes": ["list of things that vary per integration/service/tenant"],
    "what_stays_the_same": ["list of things that are invariant"]
  },
  "session_log": [
    {
      "date": "ISO date",
      "summary": "what was decided and done"
    }
  ]
}
```

---

## The Conversation Management Rule

Multi-repo work with an IDE AI has a specific failure mode: **context drift**.
The IDE AI accumulates assumptions from earlier in a conversation and applies
them incorrectly to later tasks. A scan conversation that went deep on Terraform
module patterns will colour how the IDE AI interprets a Helm authoring task
if you stay in the same conversation.

The rule is simple:

| Situation | Conversation |
|-----------|-------------|
| Switching repos | **New conversation** |
| Starting a new distinct task | **New conversation** |
| Context has drifted or assumptions are wrong | **New conversation** |
| Iterating on the same file | **Continue** |
| Running review after authoring | **Continue** |
| Following up on the previous output | **Continue** |

The creative AI flags this explicitly at the top of every prompt it generates:
`# NEW CONVERSATION` or `# CONTINUE SAME CONVERSATION`.

The developer never has to guess.

---

## The Review Prompt Pattern

Every authoring prompt is followed by a review prompt in the same conversation.
The IDE AI checks its own output against a structured checklist and writes the
results to a temporary markdown file (`_review_output.md`) that the developer
can copy and paste to the creative AI.

The review file is never committed to git. It is a working artefact of the session.

```
# _review_output.md — never git add this file

Checklist item 1: PASS — reason
Checklist item 2: FAIL — what is wrong and what the correct version should be
Checklist item 3: PASS — reason
```

The creative AI assesses the review. If all items pass, work proceeds.
If any item fails, the creative AI generates a targeted fix prompt.
The developer never decides what constitutes a fix — that is the creative AI's job.

---

## The Template Principle

The most powerful application of the Multi-Repo Bridge Pattern is when the same
pipeline must be repeated for multiple similar services — microservices, integrations,
tenants, environments.

The context.json `for_next_instance` section captures exactly what changes between
instances and what stays the same. When the next instance begins, a new session is
loaded with the existing context.json plus a note that a new instance is starting.
The creative AI immediately knows:

- Which patterns are established and must not be reinvented
- Which values need to be substituted
- Which repos have already been explored and what their conventions are
- What the full pipeline looks like and what order to follow

The second integration takes a fraction of the time of the first.
The third takes a fraction of the second.
The tribal knowledge compounds.

---

## Real-World Example — Third-Party Data Integration Pipeline

The Multi-Repo Bridge Pattern was applied in March 2026 to set up the AWS
infrastructure and deployment pipeline for a third-party data integration,
spanning four repositories:

| Repo | Purpose | Work Done |
|------|---------|-----------| 
| `terraform-aws-resources` | AWS infrastructure | S3 buckets, SQS queue + DLQ, Pod Identity IAM roles |
| `platform-data-helm` | Kubernetes Helm charts | Processor Deployment chart, Poller ServiceAccount chart |
| `ingestion-argo` | ArgoCD manifests | AppProject, stg and prd Application manifests |
| `airflow-dags` | Workflow orchestration | Poller DAG with KubernetesPodOperator |

**Before the first file was written:** nine open questions were identified and
systematically resolved through targeted recon scans of each repo. Each scan
was a read-only prompt that returned the current state — existing module versions,
naming patterns, IAM conventions, Helm chart structures — without making any changes.

**No file in any repo was authored until all open questions relevant to that file
were closed.** This prevented naming inconsistencies, module version mismatches,
and cross-repo reference errors entirely.

**The context.json built during the first integration session** now serves as the
complete specification for repeating the same pipeline for further integrations.
Each subsequent integration inherits all discovered patterns, all resolved questions,
and the full tribal knowledge accumulated during the first run — with only the
integration-specific values substituted.

---

## When To Use This Pattern

Use the Multi-Repo Bridge Pattern when:

- A single feature or service requires changes across three or more repositories
- Naming consistency across repos is critical (IAM roles, Kubernetes service accounts,
  S3 bucket ARNs, Helm chart paths)
- The same pipeline will be repeated for multiple similar services
- Tribal knowledge about repo conventions is not documented anywhere
- You cannot afford to make a cross-repo inconsistency mistake in production

Do not use it for single-repo work — the original Multi-AI Bridge Pattern
with `ai_instructions.json` and `project_sync.py` is the right tool for that.

---

## What This Pattern Is Not

This is not a tool. There is no CLI, no SDK, no SaaS wrapper.
It is a **discipline** — a set of rules for how a developer coordinates
two AI tools across multiple codebases with a JSON file as the shared brain.

The specific AI tools are irrelevant. Claude and GPT-4.1 were used in the
original application. Any creative AI with strong architectural judgment and
any IDE AI with file-editing capability will work. The pattern is the thing,
not the tools.

---

## Licence

**Creative Commons Attribution 4.0 International (CC BY 4.0)**

Copyright 2026 Shane Drobnick

You are free to:
- **Use** — personally, commercially, in products and services
- **Adapt** — build on it, modify it, extend it
- **Share** — distribute it, teach it, publish it

Under the following terms:
- **Attribution required** — You must credit Shane Drobnick as the originator
  of the Multi-AI Bridge Pattern and the Multi-Repo Bridge Pattern, and include
  a link to [https://github.com/shanepdrobnick/multi-ai-bridge-pattern](https://github.com/shanepdrobnick/multi-ai-bridge-pattern)
- **No additional restrictions** — You may not apply terms that prevent others
  from doing what this licence allows

Full licence: https://creativecommons.org/licenses/by/4.0/

---

## Author

**Shane Drobnick**
Software Engineer — Australia, March 2026

*"The engineers who figure out how to orchestrate multiple AIs efficiently
are going to be very valuable very soon."*
