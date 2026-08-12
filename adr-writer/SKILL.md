---
name: adr-writer
description: Write an ADR file. Use this whenever the user asks to create an ADR, document a decision, explain an approach, or justify why X was chosen over Y.
---

# When to Use

**Always:**
- When the user asks to create an ADR, document a decision, explain an approach, or justify why X was chosen over Y.

**Exceptions (ask your human partner):**
- It's unclear whether the current task requires a complex decision.
- The current task is about debugging, fixing an issue, or answering a simple question.

## Steps

### 1. Create an ADR

Create an Architecture Decision Record and save it to the `docs/` directory. The ADR must contain at least:

- **Title**: Sequential number + active-voice decision statement (e.g., `ADR-042: Grab your gun and bring in the cat`)
- **Category**: See [Category Mapping](#category-mapping) below.
- **Status**: Lifecycle stage (Proposed, Accepted, Rejected, or Superseded)
- **Implemented**: Link to PR, if it exists
- **Created**: Datetime using the format `YYYY-MM-DD HH:MM`
- **Context**: Forces, requirements, and background circumstances
- **Options Considered**: Serious alternatives with pros and cons
- **Decision**: Chosen solution and brief justification
- **Consequences**: Positive and negative implications, including trade-offs
- **References**: Link to other ADRs, other PRs, or external web references

#### Category Mapping

| Category | Topics |
|----------|--------|
| Core Architecture | Foundational technology choices, framework decisions, language/runtime choices |
| Integration | Exchange-specific modules, traits, instrument resolution, URL config |
| Persistence & Storage | Database choice, schema, migrations, retention policies |
| Metrics & Visualization | Prometheus, Grafana, dashboard layout, endpoint structure |
| Operations | Deployment, networking, signaling, reliability, workflow changes |

### 2. Save ADR

#### ADR File Name

The file name must follow this format: `ADR-<sequential number %03d>-<YYYYMMDD>-<short-title>.md`

Example: `ADR-042-20260801-grab-your-gun-and-bring-in-the-cat.md`

The sequential number is one more than the highest existing ADR in `docs/adr/**`.

```bash
$ tree docs/
docs
└─ adr
   ├── Core Architecture
   │  ├── ADR-001-....
   │  └── ADR-002-....
   └── Integration
      ├── ADR-003-....
      └── ADR-004-....
```

In this example, the next sequential number is `005`.

#### Where to Save

Save the ADR file in `docs/adr/<category>/`. Create the directory if it doesn't exist.

## Gotchas

If only creating a plan (not executing it), return only the full path of the ADR file that will be created. Never create an ADR during the `creating plan` stage.
