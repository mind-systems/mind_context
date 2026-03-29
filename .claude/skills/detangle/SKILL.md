---
name: detangle
description: >-
  Raise full fractal context before answering any question or starting any task.
  Instead of analyzing a leaf in isolation, climbs the tree: leaf → branch (subdomain) →
  trunk (domain) → forest (cross-project contracts). Use when a task touches a UI element,
  a feature module, a data model, a transport contract, or any code whose meaning depends
  on its position in the architecture. Especially critical for shared contracts (proto, API
  shapes, DTOs) — raises context across all consuming trees simultaneously.
argument-hint: "[topic or file or question]"
---

# Detangle

You are working in a fractal architecture. Every element — a file, a component, a model,
a proto field — exists at a specific level of the tree. You cannot reason about it correctly
without knowing its position in the full tree.

**Architecture is a fractal:**
- Leaf = usage surface (UI component, controller endpoint, CLI interface)
- Branch = transport/reactive layer (HTTP, gRPC, WebSocket, state machine, ViewModel)
- Trunk = domain (state, invariants, business rules, persistence)
- Forest = cross-project contracts (proto, shared DTOs, API shapes)

The same three roles repeat recursively at every zoom level.

---

## Workflow

### Step 1 — Identify the entry point

From the argument `$1` (or conversation context if no argument), determine:
- What is the **leaf** being discussed? (file, component, field, endpoint, UI element)
- What project does it live in?

### Step 2 — Climb the tree (leaf → trunk)

Do NOT stay at the leaf level. Raise context level by level:

**Leaf level:**
Read the file/component itself. Understand what it does on the surface.

**Branch level (subdomain):**
Ask: why does this leaf exist? What transport or reactive mechanism connects it to the domain?
- For UI: which ViewModel/service/coordinator owns it? What state does it consume?
- For an endpoint: which service/use-case does it delegate to?
- For a model field: which layer uses it — persistence, transport, or presentation?

**Trunk level (domain):**
Ask: what is the core invariant this leaf ultimately serves?
- What state does the domain own?
- What are the business rules enforced here?
- Who else touches this domain?

### Step 3 — Check for cross-project reach (forest)

Ask: does this element touch a shared boundary?

**Triggers for forest-level context:**
- The element is a proto field, message, or service → read the `.proto` file AND check all consumers (broker, core, mobile, mcp)
- The element is an API response shape (DTO) → check both the API definition and every client that deserves it
- The element is an auth token, session, or credential shape → check all services sharing the same secret/contract
- The element is a database model that feeds a mobile cache (Drift) → check both ORM model and Dart model

When cross-project reach is detected: **read a slice of each affected tree**, not just the originating one.

### Step 4 — Synthesize

Before answering or acting, produce a brief context map:

```
Leaf:    <what it is>
Branch:  <what connects it to the domain>
Trunk:   <what domain invariant it serves>
Forest:  <cross-project contracts affected, if any>

Impact:  <what changes here, what else must move with it>
```

This map is your reasoning foundation. Now answer the question or begin the task.

---

## Depth rules

- Never stop at 2-3 levels if the tree goes deeper.
- Go as deep as the **flow requires** — not as deep as is easy.
- If you hit a proto file: you are at the forest level. Read it and read its consumers.
- If you hit a shared JWT secret or auth shape: you are at the forest level. Check all services.
- If you hit a Drift table that mirrors an API response: you are at the forest level. Check both.

## What NOT to do

- Do not analyze a leaf and answer without climbing.
- Do not assume you understand a component from its name alone.
- Do not read only the originating project when a contract is shared.
- Do not treat "it's just a UI change" as an excuse to skip the branch and trunk.

---

## Reference

See [references/FRACTAL-MODEL.md](references/FRACTAL-MODEL.md) for the full fractal model
and project-specific tree maps.
