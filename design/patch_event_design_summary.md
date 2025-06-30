# 📝 Design Proposal  
## PatchEvent Mechanism for Session Pruning  
### Based on Issues #1013 & #1588  

**Author**: [Prathamesh Joshi](https://www.linkedin.com/in/jpratham)  
**Status**: _Draft for Review_  
**Created**: 2025-06-30  

> **TL;DR**  
> Introduce an **append-only `PatchEvent`** that lets developers _logically_ splice, truncate, summarise session data without ever mutating the underlying immutable event log. The design is additive (no breaking changes), keeps audit trails intact, and scales with snapshots or incremental folding.

---

## 1  Problem Statement

`SessionService.append_event()` is currently the **only** mutator.  
Large tool results and RAG passages quickly bloat `session.events`, forcing crude *oldest-first* truncation and hitting model context/rate limits. Developers need precise, audit-safe ways to:

* Delete obsolete tool dumps.  
* Replace verbose spans with summaries.

Hard deletion conflicts with ADK’s _append-only_ contract, hence **issue #1013**.

---

## 2  Goals & Non-Goals

|  Goals |  Out of scope |
|---|---|
| Precise, explicit log pruning. | Mutating or deleting rows in place. |
| Preserve auditability / replay. | Automatic retention policies. |
| Zero breaking API changes. | Redesign of Runner truncation logic. |

---

## 3  Proposed Solution — `PatchEvent`

### 3.1 Schema

```python
class PatchEvent(Event):
    patch_type: Literal[
        "splice",            # delete/replace slice
        "truncate_before",   # drop all events before `event_id`
        "summarise",         # replace slice with a summary Event
    ]
    # splice
    start: int | None = None
    count: int | None = None
    replacement: list[Event] | None = None
    # truncate_before
    event_id: str | None = None
    # summarise
    summary_event: Event | None = None
```

### 3.2 Service API (additive)

```python
class BaseSessionService:
    ...
    @abstractmethod
    async def apply_patch(
        self, session: Session, patch: PatchEvent
    ) -> PatchEvent: ...
```

### 3.3 Materialisation

A helper `fold(events_with_patches)` walks the raw stream once, applies each patch, and **replays surviving `state_delta`s** to produce:

```python
visible_events: list[Event]
state: dict
```

This keeps history verifiable while ensuring the prompt/context reflects developer-requested edits.

---

## 4  Complexity & Optimisation

| Operation | Baseline cost |
|---|---|
| `append_event()` / `apply_patch()` | **O(1)** time / space |
| `fold()` (naïve) | **O(N + P)** time, O(N′) memory <br>_N = events, P = patches, N′ ≤ N_ |

### 4.1 Why O(N) is fine

* Typical RAG / tool-heavy chats hit the model window at **200–400 events** → < 1 ms fold.  
* Even 10 k events replay in ~8 ms on CPython 3.11.

### 4.2 Scaling path

| Optimisation | Net read cost | When to enable |
|---|---|---|
| **Incremental fold cache** (store `last_idx`, `cached_state`) | O(Δ) | _Phase 1_, trivial (<20 loc). |
| **Periodic snapshots / projected state column** | O(S + Δ) (usually O(1)) | When sessions exceed ~5 k events. |
| **Range tree for patches** | O(log P + Δ) | Only if P ≫ N (rare). |

_All optimisations honour append-only storage._

---

## 5  Developer UX

```python
patch = PatchEvent(patch_type="splice", start=5, count=2)
await session_svc.apply_patch(sess, patch)   # O(1)

# Next read
sess = await session_svc.get_session(...)

# 'visible' log and session.state now reflect the splice,
# audit log still contains the original events + PatchEvent.
```

### What changed vs. today?

**Before** — `session.events` ⟶ *audit log* (raw, unfiltered).  
**After**  — `session.events` ⟶ *visible log* (folded view after applying all `PatchEvent`s).  
The unfiltered list is retained internally (or in the database) for audit/debugging.

*No existing code breaks*: any code that previously just **read** `session.events` now gets the pruned view—which is typically what it wanted.

Power-users who need the raw list can call a future helper such as:

```python
raw = await session_svc.get_raw_events(session_id)
# or
sess.audit_events
```

but normal agent execution continues to operate on the visible log only.

---

## 6  Testing Plan

* **Unit** – fold correctness for each `patch_type`; overlapping patches; state integrity.  
* **Integration** – In-memory & DB services; concurrent patch writers; rollback paths.  
* **Regression** – Legacy sessions (no patches) must pass unmodified test suite.  
* **Fuzz** – Random event/patch streams replayed against imperative reference.

---

## 7  Roll-out

| Phase | Deliverable | Flag |
|---|---|---|
| 0 | Core `PatchEvent`, `splice`; in-memory impl; docs draft. | off |
| 1 | `truncate_before`; incremental fold cache; DB backend. | `ADK_PATCH_EXPERIMENTAL=1` |
| 2 | `summarise`; snapshot helper. | on (default) |

No migrations—`PatchEvent` stores as opaque JSON in existing `events` table.

---

## 8  Implications

* **Performance** – sub-ms fold for common cases; snapshots keep worst-case bounded.  
* **Backward Compatibility** – additive API; older SDKs ignore unknown `patch_type`.  
* **Security** – explicit API discourages accidental data loss.

---
