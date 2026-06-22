# useEffect-cleaned-architecture

> A formal engineering skill specification defining a deterministic cognitive
> architecture for designing React systems that eliminate unnecessary
> `useEffect` usage and enforce correct state-flow design.

---

## 1. Skill Name

**useEffect-cleaned-architecture**

Identifier: `useEffect-cleaned-architecture`
Domain: frontend / reactive UI architecture
Type: cognitive architecture model (reasoning system, not a code generator)
Scope: applicable to any declarative reactive rendering system; grounded in React.

---

## 2. Skill Definition

### What the skill is

`useEffect-cleaned-architecture` is a decision and classification system that
governs how state, events, and side effects are modeled in React applications.
It defines a deterministic pipeline through which every piece of application
logic must pass before it may be expressed as an effect. The skill replaces
reflexive `useEffect` usage with a principled separation between **derived
state**, **source state**, **events**, and **external synchronization**.

### Problem solved

React applications degrade along four predictable axes when `useEffect` is
treated as a general-purpose lifecycle hook rather than a narrow
synchronization primitive:

1. **Overuse of `useEffect`.** Effects are used to compute values, respond to
   user actions, fetch data, and synchronize variables that are mathematically
   derivable from existing state. This introduces renders that exist only to
   feed other renders, producing cascade behavior the declarative model was
   designed to eliminate.

2. **State synchronization anti-patterns.** Two `useState` calls holding the
   same fact at different points in time, with an effect copying one into the
   other, create a window of inconsistency. The component renders with stale
   data between the commit and the effect flush. These windows are the root of
   the majority of "stale state" bugs.

3. **Performance degradation.** Every effect that writes state triggers an
   additional render pass. An effect chain of depth N produces up to N
   additional synchronous render passes per user interaction. Derived-state
   effects are the most common cause of "phantom" rerenders that profilers
   attribute to mysterious state updates.

4. **Derived-vs-stored confusion.** When a value is computable from props or
   other state, storing it in `useState` and keeping it current via an effect
   duplicates the source of truth. The duplication must be maintained, and any
   failure to maintain it (a missing dependency, a skipped update) is a bug.
   The declarative alternative — compute during render — has no failure mode
   because there is nothing to keep in sync.

### What the skill is not

- Not a lint rule. It encodes reasoning, not pattern matching.
- Not a ban on `useEffect`. It defines the narrow window in which effects are
  the correct tool and forbids everything outside it.
- Not a code generator. It produces decisions, not implementations.

---

## 3. Core Principle Model

The skill rests on five non-negotiable principles. Any design that violates
one of them is, by definition, incorrect under this architecture.

### 3.1 React is a declarative rendering engine

React's contract is: given state `S`, produce UI `f(S)`. The framework owns
the scheduling of `f`. The developer owns the transition of `S`. Anything
that attempts to manually sequence the rendering pipeline — "after this
renders, do X, which causes another render" — is a reimplementation of the
scheduler the framework already provides, and is strictly worse because it
cannot participate in React's batching, concurrent rendering, or priority
model.

### 3.2 State is a data-flow problem, not a lifecycle problem

The question "when does this update?" is answered by "when its inputs
change," not by "after the component mounts" or "after this other state
sets." State models facts; facts propagate through data flow. Lifecycle is an
implementation detail of how React commits to the DOM and must not appear in
the application's logic vocabulary.

### 3.3 Effects represent external synchronization only

An effect is a bridge between React's declarative world and an imperative
external system (a DOM API not yet abstracted, a subscription, a socket, a
timer). If the thing being synchronized is itself React state, the effect is
categorically wrong — the correct synchronization is derivation during
render.

### 3.4 Derived state must never be stored

A value is derived if it is a pure function of other state or props already
available to the component. Storing a derived value creates a second source
of truth that must be kept consistent with the first by imperative glue
(effects). Pure functions have no consistency cost; stored copies do.
**Derive during render, always.** Memoization is permitted as a performance
concern; storage as a correctness concern is not.

### 3.5 Events, not effects, drive user interactions

User intent arrives as an event (click, submit, input, keypress). The
response to that intent is business logic that belongs in an event handler,
executed once, at the moment of intent. Routing a user action through state
→ render → effect — so that an effect can "notice" the state change and
respond — delays the response by a render cycle, obscures the causal chain,
and couples the response to rendering rather than to intent.

---

## 4. State Classification System (CORE ENGINE)

This section is the engine of the skill. Every piece of state in an
application must be assigned exactly one category. The category determines
the handling strategy, the storage location, and whether any effect is
permitted to touch it.

### 4.1 Server State

**Definition.** State that originates on a server and is owned by a remote
system. The client holds a cache, not the truth. It is inherently
asynchronous, may be mutated by other actors, and may become stale without
local action.

**Examples.** A user profile fetched from an API; a paginated list of
records; a search result set; an order status.

**Correct handling.** Managed by a dedicated server-state layer (React
Query, SWR, TanStack Query, Relay, or an equivalent). The layer owns cache
invalidation, refetching, optimistic updates, race condition handling, and
loading/error states. Components consume the cache via hooks; they do not
own the data's lifecycle.

**Common misuse.** Storing fetched data in `useState` inside a component,
populated by a `useEffect` that runs on mount. This loses invalidation,
duplicates the cache across components, races on rapid prop changes, and
cannot participate in background refresh. It is the single most common
effect abuse.

---

### 4.2 UI State

**Definition.** State local to a component's presentation that has no
meaning outside the render it controls. It is ephemeral, synchronous, and
owned entirely by the component.

**Examples.** Which accordion panel is open; whether a dropdown is visible;
the current sort indicator; hover/focus transient flags; a controlled
input's value when it is not yet submitted.

**Correct handling.** `useState` (or `useReducer` when transitions are
complex). Updated exclusively by event handlers. Never the input to an
effect that writes other state.

**Common misuse.** Mirroring a prop into UI state via an effect "to keep it
in sync" — e.g., an effect that copies `props.value` into a local
`useState` whenever the prop changes. This is the stored-derived-state
anti-pattern; the prop already is the value. Reset keyed components by
remounting (`key={...}`) instead of syncing.

---

### 4.3 Form State

**Definition.** State representing the in-progress values of a form,
including validation status, dirtiness, submission status, and field-level
errors. It is a structured aggregate with its own semantics distinct from
generic UI state.

**Examples.** A registration form's field values and per-field validation
errors; a multi-step wizard's step data; a dirty/submitting flag.

**Correct handling.** A form-state library (React Hook Form, TanStack Form,
Conform, or equivalent) that owns field registration, validation timing,
and submission. For trivial cases, local `useState` with event-driven
validation in handlers. Validation is computed (derived) or run on submit;
it is not an effect that watches fields and writes errors.

**Common misuse.** An effect that watches form values and writes computed
errors into a separate `useState` — a stored-derived-state pattern that
adds a render cycle and a consistency window per field.

---

### 4.4 Global State

**Definition.** State shared across many components that are not in a direct
parent-child relationship, where the state represents a client-side fact
(not a server fact). It is application-wide, synchronous on the client, and
single-owner.

**Examples.** The current theme; the authenticated session token (client
side); an open/closed sidebar; feature flags resolved client-side; the
current locale.

**Correct handling.** A store (Zustand, Jotai, Redux, or Context with a
reducer) exposed via selector hooks. Components subscribe to slices, not the
whole. Mutations occur through actions/handlers, never through effects that
watch other state and propagate to the store.

**Common misuse.** An effect that watches a prop or local state and calls a
global setter to "push it up." This inverts data flow: the store should be
the source, and the component should read from it, not write to it as a
side effect of rendering.

---

### 4.5 Derived State

**Definition.** State that is a pure function of other state, props, or
server state already available to the component. It carries no information
not present in its inputs.

**Examples.** A filtered/sorted list derived from a source array and a
filter; a total price derived from line items; a "is valid" flag derived
from field values; a formatted display string derived from a raw value.

**Correct handling.** Computed during render. `const filtered =
items.filter(...)`. Memoize with `useMemo` only when the computation is
measurably expensive and the inputs are referentially stable. Never stored
in `useState`. Never updated via an effect.

**Common misuse.** Storing `filteredItems` in `useState` and running an
effect whenever `items` or `filter` changes to recompute. This adds a
render, introduces a frame where `filteredItems` is stale, and is strictly
more code than computing it inline.

---

### 4.6 External System State

**Definition.** State owned by an imperative external system that React does
not control: a WebSocket, a subscription, a browser API (IntersectionObserver,
MediaQuery, geolocation), a third-party widget, an interval, or the document
itself.

**Examples.** The current connection status of a socket; whether an element
is intersecting the viewport; the window's match-media result; a countdown
derived from a timer.

**Correct handling.** A `useEffect` that subscribes on mount and
unsubscribes on cleanup, writing into local state when the external system
emits. This is the canonical — and narrow — legitimate use of `useEffect`.
Prefer a custom hook (`useIntersectionObserver`, `useMediaQuery`) that
encapsulates the effect so it appears once, not scattered.

**Common misuse.** Treating any asynchronous thing as external state. A
fetch is not external state — the server owns the data, so it is server
state (4.1). A setTimeout used to debounce a value is not external state;
it is derivable via a debounce utility called from the handler or a
purpose-built hook.

---

## 5. Decision Engine (HOW TO THINK)

Every feature, every state addition, every handler, and every proposed
effect must pass through the following pipeline in order. The pipeline is
deterministic: a feature that fails a stage is not permitted to proceed to
the next. The output is an implementation decision, not a code skeleton.

### Stage 1 — Identify the state type

Classify every piece of state the feature touches into exactly one of the
six categories in §4. If a value does not fit a category, the model of the
feature is incomplete; do not proceed. If a value fits two categories, it
has been mis-modeled — refine until exactly one applies.

### Stage 2 — Determine derived vs source-of-truth

For each state value, ask: *can this be computed from other state already
present?* If yes, it is derived (§4.5) and must not be stored. If no, it is
a source of truth and its category (§4.1–4.4, 4.6) determines its owner.
This stage eliminates the largest class of effect bugs by construction.

### Stage 3 — Determine whether external synchronization is required

Ask: *does this logic need to reach outside React's declarative tree to an
imperative system that React cannot express declaratively?* If yes, an
effect is a candidate (proceed to §6 for governance). If no — the logic is
purely internal to React state — effects are forbidden; proceed to Stage 4.

### Stage 4 — Decide event-driven vs reactive model

- If the logic is a response to a discrete user or system **intent** (click,
  submit, message, timer fire), it is **event-driven**: handle it in the
  event handler, once, at the moment of intent.
- If the logic is a continuous **transformation** of inputs that must hold
  at all times, it is **reactive**: derive it during render.

These are the only two permitted control-flow models inside React. Effects
are neither; effects are the escape hatch reserved for Stage 3.

### Stage 5 — Choose implementation strategy

Select the minimal mechanism that satisfies the decision, in this priority:

1. Derive during render (no state, no effect).
2. Event handler (state mutation as a direct response to intent).
3. Server-state tool (for any remote data).
4. Store action (for global client facts).
5. Encapsulated custom-hook effect (for external synchronization only).

`useEffect` appears only at rung 5, and only when Stage 3 certified the
synchronization as external. Any design that reaches rung 5 without Stage 3
certification is incorrect.

---

## 6. useEffect Governance Rules

### 6.1 Allowed — only when ALL hold

A `useEffect` is permitted only when **all** of the following are true:

1. **External system.** The effect synchronizes with an imperative external
   system (DOM API not abstracted by React, browser API, subscription, socket,
   timer, third-party imperative widget, or the document/window).
2. **No declarative alternative.** React provides no built-in, hook, or
   declarative expression for the same behavior. (If `useSyncExternalStore`
   fits, it is preferred over a manual effect.)
3. **Subscription lifecycle.** The work is genuinely subscribe-on-mount /
   unsubscribe-on-unmount (or subscribe-on-dep-change / unsubscribe-on-prev).
   One-shot setup with no teardown is suspicious; one-shot work in an effect
   usually belongs in an event handler or a module-level initializer.

### 6.2 Required justification for every allowed effect

Every permitted `useEffect` must carry, in the team's reasoning record (a
comment, a PR description, or an ADR), three explicit statements:

- **Justification:** what external system is being synchronized.
- **External dependency:** the specific imperative API or subscription
  being bridged.
- **Why not derived/event:** why neither derivation during render nor an
  event handler can express this. (Usually: "the system pushes updates to
  us asynchronously; we cannot pull it during render.")

An effect that cannot state all three is, by this standard, forbidden.

### 6.3 Forbidden uses

The following are categorically forbidden, regardless of whether they
"work":

- **Derived state computation.** Computing a value from other state/props
  inside an effect and storing it. (Stage 2 failure.)
- **Data fetching.** Triggering network requests in an effect to populate
  component state. Server state (§4.1) belongs to a server-state layer.
- **User action response.** Responding to a user action by mutating state
  in an effect that "notices" the action via a state change. The handler
  exists; use it. (Stage 4 failure.)
- **Internal synchronization.** Copying one `useState` into another to
  "keep them in sync." If they must agree, one is derived and must not be
  stored; if they are independent, do not couple them.
- **Prop-to-state mirroring.** Copying a prop into state on prop change.
  Use the prop directly, or remount via `key` to reset.
- **Chaining.** An effect that writes state consumed as a dependency by
  another effect. Each link is a forbidden use; the chain compounds them.
- **Logging/analytics as a side door.** Event-based analytics belong in
  handlers; route them explicitly. (Exception: analytics that must observe
  committed render state may use an effect, but this is rare and must be
  justified per §6.2.)

---

## 7. Correct Architecture Patterns

### 7.1 Event-driven handlers (primary control flow)

All user intent flows through handlers. A handler is executed exactly once
per intent, has direct access to the triggering event, and may call
multiple state setters, server mutations, or store actions in a single
synchronous batch. **Why it is stable:** the causal chain from intent to
result is linear and inspectable; there is no render-mediated indirection;
React batches the updates atomically.

### 7.2 Derived computation in render

Values that are functions of available state are computed inline during
render, optionally memoized. **Why it is stable:** there is no second source
of truth and no consistency window; the value is correct by construction at
every commit; it participates automatically in React's concurrent rendering
and memoization.

### 7.3 Server-state separation

All remote data flows through a dedicated cache layer with first-class
invalidation, refetch, optimistic-update, and race handling. **Why it is
stable:** the cache is the single owner; components are pure consumers;
race conditions on rapid prop changes are handled by the layer, not
re-implemented per component; background refresh and invalidation are
automatic.

### 7.4 Feature-based modular architecture

State, handlers, and derived selectors for a feature are colocated and
exposed through a narrow surface. **Why it is stable:** the decision engine
of §5 is applied once per feature at the module boundary, not re-litigated
in every component; the state classification of §4 is visible in one place.

### 7.5 Service-layer business logic isolation

Business rules (validation, transformation, domain invariants) live in
plain functions in a service/module layer, not in components or effects.
Components call services from handlers; services are pure and testable.
**Why it is stable:** logic is testable without a rendering environment;
it is reusable across components and frameworks; it cannot accidentally
become coupled to React's commit cycle.

### 7.6 Store-based global state management

Cross-tree client facts live in a store with selector-based subscription.
**Why it is stable:** single owner; components subscribe to slices, so
unrelated changes do not rerender them; mutations flow through explicit
actions, not through render-mediated effects.

---

## 8. Anti-Patterns (FAILURE MODES)

### 8.1 Effect chains

An effect writes state that is a dependency of another effect, which writes
state consumed by a third. **Consequences:** O(N) extra render passes per
interaction; intermediate states are observable (flicker, inconsistent UI);
dependencies are fragile and a single missing dep silently breaks the chain;
the system is un-debuggable because cause and effect are separated by render
boundaries. **Root cause:** each link is a forbidden use (§6.3); the chain
is forbidden uses composed.

### 8.2 Derived state stored in useState

A value computable from existing state is stored and kept current via an
effect. **Consequences:** a frame of staleness between the source change and
the effect flush; duplicated truth that must be maintained; any missed
dependency is a silent desync bug; extra render per source change. **Root
cause:** Stage 2 failure — the value was never source-of-truth.

### 8.3 Fetching inside useEffect

Network reads triggered by an effect, results written to `useState`.
**Consequences:** race conditions when props change before the response
arrives (no cancellation by default); no invalidation; no sharing across
components; loading/error states hand-rolled per component; refetch-on-focus
and background refresh absent. **Root cause:** Stage 3 failure — a fetch is
not external synchronization; it is server-state acquisition.

### 8.4 Duplicated state sources

Two states hold the same fact, kept in sync by effects. **Consequences:**
the two drift on any sync miss; every mutation must touch both; the sync
effects themselves rerender the component. **Root cause:** one of the two
is derived (Stage 2) and must be removed; the other is the source.

### 8.5 Business logic inside components

Domain rules inlined in component bodies or effects. **Consequences:** logic
is untestable, duplicated across components that need the same rule, and
coupled to the render cycle. **Root cause:** service-layer isolation (§7.5)
was skipped.

### 8.6 Lifecycle-driven thinking

Designing with vocabulary like "after mount, do X," "on prop change, do Y,"
"before render, prepare Z." **Consequences:** the design fights the
declarative model; every lifecycle coupling is a potential stale-state bug
under concurrent rendering; the component cannot be safely reused in
different rendering schedules. **Root cause:** §3.2 violation — state was
modeled as lifecycle rather than data flow.

---

## 9. Refactoring Heuristics (FIXING BAD CODE)

Deterministic transformation rules. Apply in order; stop when none apply.

1. **If state is derived → remove the state variable.** Delete the `useState`
   and the effect that maintains it; compute the value during render from
   its sources. The value is now correct by construction.

2. **If an effect sets state → redesign the flow.** An effect that writes
   state is almost always either (a) computing a derived value (apply rule
   1) or (b) responding to an event that should have been handled in a
   handler (apply rule 3). Identify which and eliminate the effect.

3. **If an effect replaces an event handler → move the logic into the
   handler.** If the effect exists to respond to a user action observed via
   a state change, the handler that produced the state change is the correct
   home. Delete the effect; call the logic from the handler.

4. **If duplication exists → consolidate the source of truth.** If two
   states hold the same fact, identify the source and delete the copy. If
   the copy was derived, apply rule 1. If it was a mirrored prop, use the
   prop directly or remount via `key`.

5. **If an effect fetches data → replace with a server-state tool.** Move
   the fetch into a query hook with cache, invalidation, and race handling.
   Delete the effect and its local state. The component becomes a consumer.

6. **If an effect synchronizes with a real external system → keep it, but
   encapsulate.** Extract it into a named custom hook (`useX`) so the effect
   appears once, with its §6.2 justification, behind a clean read API. Use
   `useSyncExternalStore` where the system can be modeled as a subscribable
   store.

7. **If logic is inlined in a component → extract to a service.** Move
   business rules to plain functions; call them from handlers. The component
   retains only rendering and state wiring.

---

## 10. Allowed vs Forbidden Matrix

| Concept | Allowed | Forbidden |
|---|---|---|
| `useState` / `useReducer` | UI state, form state (trivial), source-of-truth client state | storing derived values; mirroring props |
| `useEffect` | external synchronization only (§6.1), with justification (§6.2) | derivation, fetching, action response, internal sync, chaining |
| Derived values | computed during render, optionally `useMemo` | stored in `useState`; recomputed in effects |
| Event handlers | primary control flow for all user intent | replaced by effects observing state |
| Server-state tools (React Query, SWR, Relay) | required for all remote data | optional / hand-rolled |
| Manual fetch in effect | never | always |
| Global store (Zustand, Jotai, Redux, Context+reducer) | cross-tree client facts | duplicating server state; written-to by effects |
| `useSyncExternalStore` | preferred for subscribable external stores | reimplemented with manual effects |
| Prop-to-state sync | never; use prop directly or remount via `key` | effect copying prop into state |
| Effect writing state consumed by another effect | never | always |
| Business logic | in service-layer functions | in components or effects |
| Lifecycle vocabulary in design | never | "after mount/on prop change do X" |

---

## 11. Real-World Application Mapping

### 11.1 Dashboards

A dashboard aggregates server state (metrics, records), UI state (filters,
selected range, active tab), and derived state (filtered/rolled-up numbers).
The dominant failure mode is filtering server data via an effect: fetch →
`useState` → effect that re-filters on filter change. Correct: server-state
layer keys queries by filter (the filter is a query parameter, not local
state to sync); derived rollups computed in render or in a memoized
selector. Effects appear only for genuine subscriptions (live-updating
widgets via WebSocket) and live in dedicated hooks.

### 11.2 Authentication flows

Auth combines server state (the session itself, owned by the server and
mirrored via a server-state or session hook), global client state (cached
user profile, theme), and event-driven mutations (login, logout). The
dominant failure mode is an effect that "checks auth on mount" and
redirects. Correct: the session is read from a session provider; redirects
happen in the login/logout handlers or in route guards that read the
session synchronously — not in an effect that watches session state and
imperatively navigates. Token refresh is an external-system concern and may
live in an encapsulated hook with full §6.2 justification.

### 11.3 CRUD systems

CRUD is server state plus event-driven mutations. The dominant failure mode
is fetching the entity in an effect keyed by id, with no race handling on
rapid id changes. Correct: a server-state query keyed by id (the layer
cancels superseded requests); mutations via the layer's mutation API with
optimistic update and automatic invalidation. List filtering and sorting
are derived (computed in render) unless they change the query itself (then
they are query keys). No effects for data flow.

### 11.4 Real-time apps

Real-time combines server state (the canonical data), external system state
(the socket connection and its emitted events), and derived state
(e.g., presence rolls up from connection events). The dominant failure mode
is scattering socket logic across components via effects. Correct: one
encapsulated connection hook owns the socket lifecycle; it feeds a store or
a server-state cache that components subscribe to. Components never touch
the socket directly. The socket effect is the canonical §6.1 case; it
appears exactly once.

### 11.5 Complex forms

Forms combine form state (field values, dirtiness), derived state
(validation errors, cross-field invariants, submit-enabled flag), and an
event-driven submission. The dominant failure mode is an effect that
watches fields and writes errors into state. Correct: form-state library
owns values and submission; validation is a derived function of the values
(computed on submit, or on blur via a handler, or live via a memoized
selector) — never an effect writing into an errors `useState`. Cross-field
rules are service-layer functions called from the validation entry point.

---

## 12. Cross-Framework Transferability

The skill is named for `useEffect` because that is where the disease is most
visible, but the underlying model is a **data-flow architecture skill**. The
principles transfer wherever a declarative reactive renderer meets imperative
systems.

### 12.1 React

Canonical ground. `useEffect` is the escape hatch; the decision engine of §5
governs its use; `useSyncExternalStore` is the preferred bridge for
subscribable external stores.

### 12.2 React Native

Identical model. The "external systems" broaden (native modules, the bridge,
`BackHandler`, `AppState`, `Keyboard`), so the legitimate-effect window is
slightly wider — but the governance of §6 still applies: subscription
lifecycle, encapsulated in a hook, justified per §6.2. Data flow, derived
state, and event handling are unchanged.

### 12.3 Next.js / TanStack Start (server components & routing)

The model sharpens. Server components make "fetch in effect" doubly wrong —
data belongs on the server, requested during render of the server tree, or
in a route loader. Client-side effects shrink further: route transitions
are loader-driven, not effect-driven. The classification of §4 holds;
server state is acquired upstream of the client component, not inside it.

### 12.4 Vue (conceptually)

Vue's `watch`/`watchEffect` are the analog of `useEffect` and carry the same
abuse risk. The model maps directly: `computed` is the blessed home for
derived state (Vue's reactivity makes derivation idiomatic), `methods`/event
handlers drive intent, and `watch` is reserved for external sync — the
governance of §6 transfers verbatim. Vue's fine-grained reactivity makes
derivation cheaper, which reinforces rule 3.4.

### 12.5 Flutter (conceptually)

Flutter is imperative at the widget level but uses declarative state
frameworks (Provider, Riverpod, BLoC). The skill maps to: derived state via
`Provider.select` / computed providers (never stored in a `StatefulWidget`
field kept in sync by `initState`/`didUpdateWidget`); events via BLoC or
handler callbacks; external sync via a dedicated stream subscription with
lifecycle management (the analog of an encapsulated effect). The
classification of §4 and the decision engine of §5 apply unchanged.

### 12.6 Backend event-driven systems

The classification generalizes: "server state" becomes "state owned by
another service"; "external system state" becomes "state owned by an
external integration or stream"; "derived state" is still pure
transformation and still must never be stored as a duplicated source. The
forbidden pattern of "effect watches X and writes Y" maps to "consumer
re-emits a derived event that downstream consumers also re-emit" — the same
chain anti-pattern (§8.1) with the same cure (derive at the consumer, do
not re-emit). The decision engine of §5 is a state-modeling discipline
that applies to any event-driven topology.

### 12.7 Principle of transfer

Across all targets the invariant is: **state is data flow; effects
(whatever the local name) are external synchronization; derivation is
computed, never stored.** A system that internalizes §3 and §5 designs
correctly in any of the above. A system that reaches for the local
equivalent of `useEffect` to express anything other than external
synchronization is incorrect, regardless of framework.

---

## End of Specification

This document is a reasoning standard, not a code template. Apply §5 to
every decision; apply §6 to every effect; apply §4 to classify before
implementing. When in doubt, the answer is the higher rung of §5: derive
during render, handle in the handler, and reserve the effect for the
external world only.
