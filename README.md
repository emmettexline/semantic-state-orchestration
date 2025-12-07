# semantic-state-orchestration

Semantic State Orchestration

By Emmett D. Exline

Domain and Ground Rule
=

Domain: multi‑module language model orchestration with a single interface process and multiple tool modules.
​

Ground rule: all user/task information lives in an explicit semantic state S; all interaction with modules is via typed events; any information loss is performed only by explicit summarization operations.

Core Sets and Primitive Types
=
    
Let
    
    Tok – finite set of tokens
    Text – set of finite token sequences over Tok (Text = Tok*)
    U – set of user identifiers
    Conv – set of conversation identifiers
    Mod – set of module identifiers
    ItemId – set of item identifiers
    CallId – set of tool‑call identifiers

These sets are assumed countable and effectively encodable as strings or integers.

Semantic State
=

2.1 Item universe

Define:

    Fact := ItemId × Text
    Constr := ItemId × Text
    Subtask := ItemId × Text × {Unassigned, InProgress, Done, Failed}
    
Let
    
    Item :=
    {Fact} × Fact
    ∪ {Constr} × Constr
    ∪ {Subtask} × Subtask
    
    For x ∈ Item, write id(x) ∈ ItemId for its identifier (the first component of its payload).
    
    2.2 Dependencies and state
    
    A dependency graph is a function
    
    deps : ItemId → P(ItemId)
    
    A semantic state is a pair
    
    S = (I, deps)

with:
    
    I ⊆ Item is finite
    ∀ x ∈ I, deps(id(x)) ⊆ { id(y) | y ∈ I }

Informal invariant:

    All user‑visible facts, constraints, and subtasks are represented as elements of I.

    Forgetting or compaction of information happens only by removing items from I and optionally adding new items (e.g. summaries).
    
Events, Alphabet, and Traces
=

3.1 Event component sets

Define:
    
    UserMsg := U × Text
    ToolCall := Mod × CallId × Text
    ToolResult := Mod × CallId × Text
    AddItem := Item
    UpdateItem := Item
    ForgetItems := P(ItemId)
    FinalAnswer := Text

3.2 Base alphabet

The base event alphabet is the disjoint union

    Σ_B :=
    {UserMsg} × UserMsg
    ∪ {ToolCall} × ToolCall
    ∪ {ToolResult} × ToolResult
    ∪ {AddItem} × AddItem
    ∪ {UpdateItem} × UpdateItem
    ∪ {ForgetItems} × ForgetItems
    ∪ {FinalAnswer} × FinalAnswer

A finite base trace is an element of Σ_B*.

Orchestrator State and Base Dynamics
=

4.1 Control state

Let C be a finite set of control states.

Example shape (not fixed by the spec, but typical):

    C ⊆ Phase × Aux

where Phase is a finite set such as {Idle, Planning, CallingTool, Answering} and Aux is a finite product of flags, counters, and optional identifiers (e.g. current CallId).
​

4.2 Orchestrator state

An orchestrator state is

    O := S × C × Σ_B*

Write o = (S, c, τ), where S is the semantic state, c is control, τ is the trace so far.

4.3 External input

Define the set of external inputs

    In :=
    {InUserMsg} × UserMsg
    ∪ {InToolResult} × ToolResult
    ∪ {InNone}

where InNone denotes an internal step (no new external event).

4.4 Step function

A single base step is a deterministic function

    step_base : O × In → O × Σ_B*

Given (o, inp), with o = (S, c, τ) and step_base(o, inp) = (o', σ), write o' = (S', c', τ').

Operational constraints:

Trace extension:

    τ' = τ · σ (concatenation)

Semantic updates:

    S' is obtained from S by interpreting in order the subsequence of σ
    consisting of AddItem, UpdateItem, ForgetItems events; no other
    changes to S are allowed.

Determinism:

    For fixed (o, inp), the sequence σ (up to Text contents) and the
    next control state c' are uniquely determined; stochasticity in
    implementations appears only inside Text fields.    ​

This defines the base process as a labeled transition system over O with labeled outputs σ.

Modules and Context Construction
=

5.1 Module specification

Each module M ∈ Mod is specified by:

    A weight function

    w_M : S × Σ_B* × ItemId → ℝ_≥0

assigning a non‑negative utility score to including a given item in this module’s context, given current state and trace.

    A cost function

    c_M : S × Σ_B* × ItemId → ℕ

giving the estimated token cost of including that item in this module’s prompt, including any formatting overhead.

    A black‑box semantics

    σ_M : Text → Text

modeling the module’s behavior on raw textual requests and responses (probabilistic in practice, abstracted here as a map).

5.2 Token‑constrained context selection

Given:

    state S = (I, deps)

    trace τ ∈ Σ_B*

    module M ∈ Mod

    budget T ∈ ℕ

Define:

    U := { id(x) | x ∈ I } ⊆ ItemId

For each i ∈ U:

    w_i := w_M(S, τ, i)
    c_i := c_M(S, τ, i)

Feasible subsets:

    F_M(S, τ, T) :=
    { K ⊆ U |
    Σ_{i∈K} c_i ≤ T
    and ∀ i ∈ K, deps(i) ⊆ K }

Objective:

For K ⊆ U, define utility

    U_M(K) := Σ_{i∈K} w_i

Define the optimal set as any maximizer

    K_M*(S, τ, T) ∈ argmax_{K ∈ F_M(S, τ, T)} U_M(K)

with a fixed deterministic tie‑breaking rule.

5.3 Serialization

Each module M has a serialization function

    ser_M : S × Σ_B* × P(ItemId) → Text

The context builder for M is then

    Ctx_M(S, τ, T) :=
    (K_M*(S, τ, T),
    ser_M(S, τ, K_M*(S, τ, T)))

Interpretation:

    K_M* is the selected subset of semantic items to expose to M

    ser_M packs those items and any necessary call metadata into a Text
    payload not exceeding the token budget T by construction of c_i.

Note: Ctx_M does not change S; it only reads S and τ. All semantic information loss, if any, must be realized through explicit summarization steps (next section).

Summarization and Forgetting
=

A summarizer is a distinguished module Sum with the usual w_Sum, c_Sum, σ_Sum, plus a semantic compaction operator

    f_sum : S × P(ItemId) → (Item, P(ItemId))

Given S = (I, deps) and a cluster C ⊆ U where U = { id(x) | x ∈ I }:

    f_sum(S, C) = (x_new, F)

where:

    x_new ∈ Item is a new “summary” item

    F ⊆ C is the set of item ids to be forgotten

The corresponding state update is:
  
    I' := (I \ { x ∈ I | id(x) ∈ F }) ∪ { x_new }
    deps' : obtained from deps by
    – restricting to ids of items in I'
    – optionally adding deps(id(x_new)) for x_new
  
    S' := (I', deps')

The base step function uses this by:

    Selecting a cluster C in some control state c

    Constructing a context for Sum focused on items with ids in C

    Emitting a ToolCall to Sum, receiving a ToolResult

    Emitting AddItem(x_new) and ForgetItems(F), which transform S to S'

This is the only mechanism that changes the semantic information content of S; all other operations (Ctx_M, tool calls, etc.) are invariant with respect to S.

Overall Orchestration Dynamics
=

The runtime consists of:

    A base process given by step_base over states O = S × C × Σ_B*

    A family of modules { M ∈ Mod } each with (w_M, c_M, σ_M, ser_M)

    A token‑constrained context builder Ctx_M for each M

    An explicit summarization mechanism via f_sum
  
A global execution is an interleaving of:

    Applications of step_base with inputs InUserMsg or InNone, which may
    emit ToolCall events constructed using Ctx_M.

    Calls to σ_M for each ToolCall to M, producing ToolResult events,
    which are then fed back as InToolResult to step_base.

    Occasional summarization cycles using Sum and f_sum to compact S.

All user‑visible behavior is given by projections of the global trace τ ∈ Σ_B*. Constraints on where and how S changes, and how contexts are constructed under budgets, are sufficient to guide a direct functional implementation of this orchestration system.
