# CLAUDE.md — Documentation Generation Workflow

## Project Overview

You are documenting two interconnected repositories for MishiPay, a retail technology platform:

1. **MishiPay** (`./MishiPay/`) — The main Django monolith (~2,955 Python files, 60+ Django apps). Handles retail operations, payments, items, customers, analytics, and retailer-specific integrations.
2. **service.promotions** (`./service.promotions/`) — A standalone Django promotions microservice (~106 Python files). Implements a sophisticated discount engine with 8 promotion family types.

The output is **unified Markdown documentation** in `./docs/` covering both repositories as one system.

---

## Directory Layout

```
/Users/theo/master-documentation/
├── CLAUDE.md              ← You are here. Read this first every session.
├── tasks.json             ← Task tracker. Read to find your next task.
├── docs/                  ← Documentation output (you write here)
│   ├── 00-index.md
│   ├── 01-architecture-overview.md
│   ├── 02-tech-stack.md
│   ├── 03-infrastructure.md
│   ├── promotions-service/
│   │   ├── overview.md
│   │   ├── data-models.md
│   │   ├── business-logic.md
│   │   ├── api-reference.md
│   │   └── devops.md
│   ├── platform/
│   │   ├── overview.md
│   │   ├── core.md
│   │   ├── authentication.md
│   │   ├── items.md
│   │   ├── orders.md
│   │   ├── payments.md
│   │   ├── customers.md
│   │   ├── analytics.md
│   │   ├── emails-alerts.md
│   │   ├── coupons-promotions.md
│   │   ├── subscriptions-referrals.md
│   │   ├── dashboard.md
│   │   ├── cashier-kiosk.md
│   │   ├── inventory.md
│   │   ├── integrations.md
│   │   ├── retailer-modules.md
│   │   ├── support-modules.md
│   │   └── infrastructure.md
│   └── cross-cutting/
│       ├── data-flow.md
│       ├── api-reference.md
│       └── glossary.md
├── MishiPay/              ← Cloned repo (read-only, do not modify)
└── service.promotions/    ← Cloned repo (read-only, do not modify)
```

---

## Session Workflow Algorithm

Every session follows this exact sequence. **Do not deviate.**

### Step 1: Read Instructions
```
Read: /Users/theo/master-documentation/CLAUDE.md
```

### Step 2: Read Task Tracker
```
Read: /Users/theo/master-documentation/tasks.json
```
Find the first task with `"status": "pending"`. This is your task for this session.
Change its status to `"in_progress"`.

### Step 3: Read Existing Documentation
Read the documentation files listed in the task's `"docs_to_update"` field. If the files don't exist yet, you'll create them. If they exist, you'll expand them.

**This step is critical for context.** The existing documentation is the accumulated knowledge from all previous sessions. Read it thoroughly before reading source code.

### Step 4: Read Source Code
Read the files listed in the task's `"source_paths"` field. Focus on:
- **Models** (models.py, models/) — data structures, relationships, fields
- **Views/APIs** (views.py, viewsets.py) — endpoints, request/response
- **Serializers** (serializers.py) — data validation, transformation
- **URLs** (urls.py) — routing, endpoint paths
- **Business logic** — core algorithms, services, helpers
- **Config** (settings.py, config.py) — configuration, constants
- **Admin** (admin.py) — admin interface customization
- **Tasks** (tasks.py) — background/async tasks

For each file, focus on understanding:
- What does this code do?
- How does it relate to the rest of the system?
- What are the key classes, functions, and their purposes?
- What external services/APIs does it interact with?

**Skim but don't deeply document:**
- Migration files (just note schema evolution patterns)
- Test files (note coverage scope and key test scenarios)
- Auto-generated code

### Step 5: Write/Expand Documentation
Update the documentation files specified in `"docs_to_update"`. Follow the documentation standards below.

**Key rule: Always read the existing doc file first, then expand it.** Never overwrite previous content unless correcting an error.

### Step 6: Update Task Status
In `tasks.json`, set the completed task's status to `"completed"`. Add a `"completed_notes"` field with a brief summary of what was documented.

### Step 7: Request Review
Tell the user:
- What task you completed
- What documentation files you created/updated
- A brief summary of key findings
- Any questions or concerns about the codebase
- Ask the user to review before proceeding

**Stop here. Do not start the next task.** Wait for user approval.

---

## Documentation Standards

### File Structure
Every documentation file should follow this template:

```markdown
# [Module/Component Name]

> Last updated by: Task [ID] — [Brief description]

## Overview
[2-3 sentence summary of what this module does and why it exists]

## Key Concepts
[Explain domain-specific terminology and concepts]

## Architecture
[How this module is structured, key design patterns used]

## Data Models
[Database tables/models, their fields, and relationships]

## API Endpoints
[If applicable — routes, methods, request/response formats]

## Business Logic
[Key algorithms, workflows, decision trees]

## Dependencies
[What this module depends on, and what depends on it]

## Configuration
[Environment variables, settings, feature flags]

## Notable Patterns
[Design patterns, conventions, or unusual approaches worth noting]

## Open Questions
[Anything unclear that needs further investigation]
```

### Writing Style
- Be factual and precise. Reference actual file paths and line numbers.
- Use code blocks for class names, function signatures, and configuration values.
- Include actual field names, types, and constraints from models.
- Document what the code **does**, not what it **should** do.
- Flag any inconsistencies, dead code, or potential issues you find.
- Use tables for structured data (model fields, API endpoints, config values).
- Keep paragraphs short (2-3 sentences max).

### Cross-References
When a module references another module, use this format:
```markdown
See [Module Name](../path/to/doc.md) for details on [specific aspect].
```

### Progressive Enhancement
Documentation builds up across tasks. Each task should:
1. Read what exists
2. Add new information
3. Correct any previous errors
4. Add cross-references to newly documented modules
5. Update the `> Last updated by` line

---

## Context Window Management

This workflow is specifically designed for Claude's context window limitations:

1. **Each task is self-contained.** You don't need history from previous conversations.
2. **Documentation IS the memory.** Previous sessions' work is captured in docs/.
3. **Read docs before code.** This gives you context without re-reading the entire codebase.
4. **Tasks are scoped to ~1 hour of work.** This limits how much code you need in context at once.
5. **Focus on key files, not every file.** A module with 649 files has maybe 30 important ones. The task's `source_paths` tells you where to focus.

If a task feels too large for your context window:
- Focus on the most important files first (models, views, core logic)
- Document what you've read
- Note in `completed_notes` that additional passes may be needed
- The user can split the task if necessary

---

## File Reading Priority

When exploring a Django app, read files in this order:
1. `models.py` or `models/` directory (understand the data)
2. `urls.py` (understand the API surface)
3. `views.py` or `viewsets.py` (understand the endpoints)
4. `serializers.py` (understand data shapes)
5. Core business logic files (services, helpers, utils)
6. `tasks.py` (background work)
7. `admin.py` (admin customization)
8. `signals.py` (event hooks)
9. `tests/` directory (understand test coverage scope)

---

## Rules

1. **Never modify source code.** Only write to `docs/` and `tasks.json`.
2. **Always read CLAUDE.md and tasks.json first.** Every session, without exception.
3. **One task per session.** Complete it, request review, stop.
4. **Read existing docs before writing.** Never overwrite accumulated knowledge.
5. **Be honest about gaps.** If you can't fully document something, say so in Open Questions.
6. **Reference actual code.** Use file paths like `mainServer/mishipay_core/models.py:45`.
7. **Update the index.** After creating/updating docs, update `docs/00-index.md` to reflect current state.
