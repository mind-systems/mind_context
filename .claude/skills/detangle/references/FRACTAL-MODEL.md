# Fractal Architecture Model

## The Pattern

Architecture repeats the same three-role structure at every zoom level:

| Role | What it IS | What it DOES | Knows about |
|---|---|---|---|
| **Trunk (domain)** | State, invariants, business rules | Owns data, enforces constraints | Nothing above itself |
| **Branch (subdomain)** | Transport, reactive layer, adapters | Moves data between trunk and leaves | Trunk + leaf interfaces |
| **Leaf (usage)** | Surface, interface, consumer | Uses without owning | Only its direct branch |

**The rule:** A leaf must not know the trunk directly. A trunk must not know its leaves.
Any violation is a coupling problem — and a clue that something is in the wrong place.

---

## Zoom Levels in This Codebase

### Level 0 — Forest (cross-project)

```
mind (monorepo root)
├── mind_api        ← trunk: API domain (NestJS, PostgreSQL)
├── mind_mobile     ← leaf: mobile consumer
├── mind_landing    ← leaf: web consumer
└── mind_mcp        ← leaf: MCP tool consumer
```

**Branch at this level:** HTTP/REST, gRPC, proto contracts.
**Forest-level triggers:** proto files, shared DTOs, JWT shape, auth flow.

### Level 1 — Inside mind_api (trunk)

```
mind_api
├── Domain modules (users, breathing, sessions, ...)  ← trunk
├── Controllers / HTTP layer                           ← branch (transport)
└── External consumers (mobile, mcp, landing)         ← leaves
```

### Level 1 — Inside mind_mobile (a leaf that is also a tree)

```
mind_mobile
├── Core/Api (Dio, interceptors, DTOs)    ← branch (transport)
├── Feature modules (breathing, auth...)  ← trunk (local domain)
└── UI screens / widgets                  ← leaves
```

### Level 2 — Inside a feature module (e.g. Breathing)

```
BreathingModule
├── BreathingService / data source         ← trunk (state, persistence)
├── ViewModel / state machine / reactivity ← branch (reactive transport)
└── BreathingScreen / UI components        ← leaves
```

### Level 3 — Inside a ViewModel

```
BreathingViewModel
├── State (immutable data class)     ← trunk
├── Reactive streams (StateNotifier) ← branch
└── Exposed methods / getters        ← leaves (used by screen)
```

---

## Cross-Project Contract Map

These elements live at the **forest level** — changing one requires raising context
across all projects that consume it.

| Contract | Defined in | Consumers |
|---|---|---|
| Proto files | `mind_api/proto/` | mind_mcp, mind_mobile |
| REST API shapes (DTOs) | `mind_api/src/*/dto/` | mind_mobile (Dart models), mind_mcp |
| JWT payload shape | `mind_api/src/auth/` | mind_mobile (AuthInterceptor), mind_mcp |
| Auth token storage | `mind_mobile/lib/User/` | mind_api (validation) |
| Drift schema (cache) | `mind_mobile/lib/*/data/` | mirrors mind_api response shapes |

**Rule:** When you touch any row in this table, read ALL columns before acting.

---

## Climbing Algorithm

```
1. Start at the element in question.
2. Ask: "What owns this? What is it a part of?"
   → Move up one level (leaf → branch).
3. Ask: "What domain invariant does this serve?"
   → Move up one level (branch → trunk).
4. Ask: "Does this cross a project boundary?"
   → If YES: identify all consumers, read a slice of each.
5. Produce context map. Then act.
```

Stop when you have answered: **what is the trunk-level invariant this leaf exists to serve?**

If you cannot answer that question, you have not climbed high enough.
