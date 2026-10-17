# Inductive Types and Structural Recursion

**Status**: Draft. Companion to the [enums](enums.md) and
[pattern matching](pattern-matching.md) proposals.

Date: Oct 17, 2026

## Introduction

Recursive data structures—such as abstract syntax trees (ASTs), intermediate
representations (IRs), JSON documents, and computational graphs—are central to
compiler engineering, symbolic mathematics, AI runtimes, and systems
programming.

In modern functional languages like **OCaml**, **Standard ML**, and **Lean 4**,
algebraic inductive types and pattern-directed structural recursion are
foundational. They allow developers to express complex data transformations
concisely, with compiler checks for exhaustiveness. Lean also checks termination
for total recursive definitions; OCaml and Standard ML allow unrestricted
recursion. Their memory models differ too: OCaml uses a tracing garbage
collector (GC), while [Lean uses reference counting][lean-runtime]. Allocation,
indirection, and reclamation costs still matter for low-level systems
programming, custom memory layouts, and accelerator-driven environments.

Mainstream systems languages like C++ and Rust provide building blocks for
recursive data. [Rust already has recursive algebraic enums][rust-recursive],
but neither language connects recursive data to compiler-checked structural
recursion. Developers choose explicit indirection (such as `std::unique_ptr`
or `Box<T>`) and use hand-rolled worklists when native recursion would overflow
on deep structures. Meanwhile, Python provides clean, high-level syntax, but
lacks both native algebraic data types and checked structural recursion. Its
call-stack limit is configurable through `sys.setrecursionlimit`, but raising
that limit does not make deep recursion stack-safe.

Mojo has a unique opportunity to unite the best of both paradigms:

1. **The expressive power and mathematical rigor of modern functional
   languages:** First-class inductive enums, pattern-directed recursion,
   structural descent checking, and guaranteed exhaustiveness.
2. **The performance, ownership model, and deterministic control of a systems
   language:** Predictable memory layouts, explicit runtime costs, atomic
   reference counting without a tracing GC, safe structural sharing, and
   guaranteed stack-safe structural execution.

This proposal introduces a cohesive language design for recursive data in Mojo
through two complementary features:

1. **Inductive Enums (`inductive enum`):** Nominal algebraic sum types whose
   payloads may contain recursive references to `Self` (directly, inside
   payload structs, or within container types like `List[Self]` and
   `Dict[K, Self]`). At runtime, inductive enums represent immutable, directed
   acyclic graphs (DAGs) backed by atomic reference counting. They provide
   cheap $O(1)$ copies through reference sharing, safe subtree reuse, and
   guaranteed stack-safe iterative cleanup.
2. **Structural Functions (`match recursive` and `recur`):** First-class
   structural recursion designed specifically for inductive values. A function
   ending in `match recursive` enables pattern arms to request child results
   declaratively using the `recur` keyword. The compiler verifies that every
   constructor is exhaustively covered, checks that recursive calls strictly
   descend along structural sub-elements, and lowers execution onto an explicit
   heap frame stack so that native stack usage for structural calls remains
   strictly $O(1)$ regardless of data depth. Termination also requires each
   activation to perform finite local work and issue finitely many child
   requests.

This proposal builds naturally upon Mojo's existing design—including the
`EnumLike` trait, pattern matching, arm ordering, and guards. Derived operations
(such as equality checkers, hashing, and printing) leverage the exact same
stack-safe traversal mechanics as structural functions. A synthesized
`Inductive` trait, proposed as a follow-up, would extend them to generic library
code such as serializers and pretty-printers.

We have implemented and validated the core storage and traversal mechanisms in
an open-source prototype, [FP Mojo][fp], which successfully verifies iterative
evaluation and stack-safe cleanup on graphs a million levels deep.
[The Prototype](#the-prototype) section details its test suite and highlights
where this native proposal builds upon that foundation.

> **Scope Note:** This proposal focuses specifically on finite inductive data
> and well-founded structural computations over it. Coinductive infinite
> streams, dependent pattern matching, and general-purpose termination provers
> for arbitrary functions are outside the scope of this document.

## A First Example

To see how inductive enums and structural recursion feel in practice, consider a
classic arithmetic expression tree containing numbers, additions, and
conditional expressions:

```mojo
inductive enum Expr:
    case num(Int)
    case add(Self, Self)
    case if_else(Self, Self, Self)


def evaluate(e: Expr) -> Int:
    match recursive e:
        case .num(value):
            return value
        case .add(recur left, recur right):
            return left + right
        case .if_else(recur condition, then_branch, else_branch):
            if condition != 0:
                return recur then_branch
            return recur else_branch


def simplify(e: Expr) -> Expr:
    match recursive e:
        case .add(recur .num(0), recur right):
            return right
        case recur rebuilt:
            return rebuilt


def main():
    var zero = Expr.num(0)
    var x = Expr.if_else(Expr.num(1), Expr.add(zero, Expr.num(5)), zero)
    print(evaluate(x))            # 5
    print(evaluate(simplify(x)))  # 5
```

*(Note: The native syntax shown above represents this proposal's proposed
language design. The [FP Mojo prototype](#the-prototype) demonstrates equivalent
behavior working on today's pinned Mojo toolchain.)*

### How It Works

- **Recursive Payloads with `Self`:** In the declaration of `Expr`, using `Self`
  within a case payload designates a recursive position. In `main()`, assigning
  `zero` twice within `x` does not duplicate memory; both branches share a
  single reference-counted node.
- **Eager vs. On-Demand Evaluation in `evaluate`:**
  - In the `.add(recur left, recur right)` arm, `recur` appears directly inside
    the pattern. This instructs the runtime to evaluate both child expressions
    before executing the arm's body, binding `left` and `right` as computed
    integers (`Int`).
  - In the `.if_else(recur condition, then_branch, else_branch)` arm, only the
    `condition` child is evaluated up front. The branches remain un-evaluated
    `Expr` handles. In the arm body, `return recur then_branch` lazily descends
    into only the branch selected by the runtime condition, completely skipping
    the unused branch.
- **Non-Destructive Rewriting in `simplify`:**
  - The first arm of `simplify` evaluates both child expressions and matches the
    left result against `.num(0)`. If matched, it simplifies `0 + right` by
    returning `right`.
  - If the first pattern does not match, execution falls through to the
    catch-all arm: `case recur rebuilt`. This arm automatically collects the
    recursively simplified child results and rebuilds the node. Importantly, if
    neither child changed during simplification, `rebuilt` returns the
    *original* immutable node handle without allocating any new memory.
  - Because `case recur rebuilt` covers all remaining constructors, the compiler
    verifies that `simplify` is fully exhaustive.

## Inductive Enums

### Declaration and Syntax

In Mojo, standard enums (as proposed in [`enums.md`](enums.md)) define closed
sum types where each case may carry an independent payload. However, standard
enums require a statically bounded size at compile time, meaning a case cannot
directly contain the enum itself.

An `inductive enum` lifts this restriction. It declares a nominal algebraic type
whose cases may reference the enclosing enum at designated **recursive
positions**:

```mojo
inductive enum Expr:
    case num(Int)
    case add(Self, Self)
    case if_else(Self, Self, Self)
```

Within an inductive enum declaration, recursive positions may appear:

1. **Directly in case payloads or tuples:** e.g., `case add(Self, Self)`.
2. **As fields within a payload struct:** e.g., `case add(Add[Self])`.
3. **Inside standard child containers:** e.g., `List[Self]`, `Optional[Self]`,
   or values of `Dict[K, Self]` (see
   [Describing Children](#describing-children)).

To preserve structural integrity and enable the compiler to verify termination,
the enum's own type (written either as `Self` or by its nominal name) may *only*
appear in these validated positions. The same rule applies to other members of
an [inductive group](#mutual-recursion-inductive-groups).

> **Safety Invariant: Strictly Verified Containment**  
> The compiler inspects all payload fields after substituting generic
> parameters. It explicitly rejects recursive occurrences hidden behind raw
> pointers, inside function types, or within uninspected collections. Allowing
> opaque references would break the compiler's ability to prove acyclicity,
> track structural descent, or orchestrate stack-safe deallocation.
>
> Furthermore, `Self` represents the enum instantiated with its identical
> generic parameters. Non-regular recursion (where a generic type recursively
> contains a different instantiation of itself, such as `struct Tree[T]`
> containing `Tree[List[T]]`) is prohibited in v1.

By default, an inductive enum is closed: its set of cases is fixed. A library
can instead declare an inductive enum *open*, so that adding a case later is not
a breaking change. Like other enums, an inductive enum may declare comptime
values in its body; declaring the `_enum_is_exhaustive` member that the
[pattern matching proposal](pattern-matching.md) defines for `EnumLike` types
overrides the synthesized default:

```mojo
inductive enum OpenExpr:
    comptime _enum_is_exhaustive = False

    case num(Int)
    case add(Self, Self)
```

Structural functions outside the declaring module must then include a catch-all
arm (see [Exhaustiveness Checking](#exhaustiveness-checking)).

#### Composing with Named Payload Structs

For complex ASTs, tuple payloads can quickly become unwieldy. Developers can use
standard generic structs to provide named fields, methods, and trait
conformances, as [suggested in the forum discussion][composition-discussion]:

```mojo
@fieldwise_init
struct Add[R: Movable & Deinitable](Movable):
    var left: Self.R
    var right: Self.R


@fieldwise_init
struct If[R: Movable & Deinitable](Movable):
    var condition: Self.R
    var then_branch: Self.R
    var else_branch: Self.R


inductive enum NamedExpr:
    case num(Int)
    case add(Add[Self])
    case if_else(If[Self])
```

Here, `Add` and `If` are regular Mojo structs. After generic parameter
substitution, the compiler identifies the recursive positions within `left`,
`right`, `condition`, `then_branch`, and `else_branch`. Pattern matching then
destructures them naturally:

```mojo
case .add(Add(left=recur left, right=recur right)):
    return left + right
```

[Named case fields][named-fields-discussion] from the forum discussion would be
an alternative spelling for the same concept, and remain fully compatible with
this proposal.

#### Destructor Restrictions on Payloads

To guarantee stack safety during cleanup, inductive types impose two
straightforward rules on payload types:

1. **No custom destructors on recursive structs:** A struct containing recursive
   positions cannot define its own `__deinit__`. When an inductive node is
   destroyed, the runtime must extract its child handles into an iterative
   worklist *before* releasing the rest of the payload. Payload types without
   recursive positions may define their own destructors normally.
2. **Payloads must allow implicit destruction:** A linear type marked
   `@explicit_destroy` cannot be stored in an inductive enum. Because inductive
   nodes are shared via reference counting, payloads are destroyed implicitly
   whenever the final handle goes out of scope.

These rules bound stack use for structural release. Non-recursive payload
destructors retain their ordinary behavior and may use the native stack.

### Values, Representation, and Construction

At the machine level, an inductive enum value is a lightweight, pointer-sized
**handle** to an immutable heap-allocated node. The node header stores the
active case tag, an atomic reference count, and the case payload.

Constructing an inductive value uses the synthesized static factory methods of
standard enums:

```mojo
var leaf = Expr.num(2)
var sum = Expr.add(leaf, leaf)  # Both child slots point to the exact same node
```

#### Value Semantics and Copying

In functional languages like OCaml and Lean, data structures are persistent and
immutable by default. Mojo marries this mental model with explicit ownership
semantics:

- **Cheap $O(1)$ Copies:** Every inductive enum is `ImplicitlyCopyable`.
  Copying an inductive handle simply copies a pointer and performs an atomic
  reference count increment.
- **Payloads Are Not Duplicated:** Copying a handle never clones the underlying
  payload. Consequently, types that are only `Movable` (or expensive to
  duplicate) can be stored in inductive nodes without being cloned. Sharing
  across threads still requires thread-safe payloads, and atomic count updates
  have a runtime cost.
- **Copying Payloads Out:** Extracting or cloning a payload *out* of a node
  requires that payload type to conform to `Copyable`.
- **Rebuilding Requirements:** If a child changes during
  [rebuilding](#rewriting-and-tree-transformations), the newly constructed
  parent requires copies of the other payload fields (as the old parent may
  still be shared elsewhere). Those fields must therefore be `Copyable`, even
  when the rewriting function owns its handle.

#### Immutability and Persistent Updates

Inductive nodes are strictly immutable once initialized. A variable holding an
inductive value can be reassigned, but the underlying node data never changes:

```mojo
var e = Expr.num(0)
e = Expr.add(e, Expr.num(1))  # Creates a new parent node; the original leaf remains shared
```

Transforming or "updating" a tree follows the standard functional paradigm: new
nodes are constructed along the path of modification (the spine), while all
unmodified subtrees are shared by reference.

As with other immutable references in Mojo, shallow immutability does not freeze
external mutable state referenced through a non-recursive payload (such as an
interior pointer or reference cell). The case, stored fields, and child links do
stay fixed. [Child containers](#describing-children) must keep those links
unchanged for the node's entire lifetime. This distinction also affects
[hashing](#derived-operations): an external payload's hash can mutate even
though the enclosing node is immutable.

#### Identity vs. Equality

- **`e1 is e2` (Identity):** Evaluates whether two handles point to the exact
  same heap node in $O(1)$ time. A handle and its direct copies always share
  identity.
- **`e1 == e2` (Structural Equality):** Performs a deep structural comparison
  across the entire subtree.

Whether two separately constructed identical values share the same node is
unspecified. This leaves implementations free to deduplicate unit cases (cases
without payloads). Inline leaves would need to preserve the same observable
identity and payload-access rules, as discussed under optimization
opportunities.

### Sharing, Acyclicity, and Stack-Safe Deallocation

In compiler IRs and symbolic computation, structural sharing is critical. For
example, starting with a base variable `x` and constructing `Expr.add(x, x)`
repeatedly 60 times creates a directed acyclic graph (DAG) of exactly **61
nodes**. In a naive tree representation that forbids sharing (such as unique
`Box<T>` in Rust or `std::unique_ptr` in C++), expanding this value would
duplicate nodes exponentially, requiring $2^{61}-1$ nodes (tens of exabytes of
memory).

```text
        add (Node 60)
       /   \
      +-----+
         |
        add (Node 59)
       /   \
      +-----+
         :
        add (Node 1)
       /   \
      +-----+
         |
      num(1) (Node 0)
```

Inductive enums preserve shared DAG structure while maintaining safety:

#### 1. Guaranteed Acyclicity

Because an inductive node can only be constructed from child handles that
*already exist*, and because nodes are strictly immutable once created,
**inductive links can never form cycles**. The graph is guaranteed to be a
finite, directed acyclic graph (DAG).

*(Note: While inductive links are mathematically acyclic, opaque external
payloads—such as a mutable cell or a `PythonObject` updated after construction
to refer back to the node—could theoretically introduce cycles. Such cycles
would leak memory identically to circular `ArcPointer` references today, but
cannot corrupt structural traversals.)*

#### 2. Stack-Safe Iterative Deallocation

In standard systems programming, dropping a deeply nested recursive structure
can trigger a cascading chain of recursive `__deinit__` calls. An AST or JSON
structure 50,000 levels deep can overflow the call stack upon release, depending
on frame size and the available stack.

To eliminate this class of crashes, inductive enums use **iterative worklist
deallocation**:

1. When a handle goes out of scope and decrements a node's reference count to
   zero, the runtime does *not* recursively invoke destructors down the call
   stack.
2. Instead, all of the node's immediate child handles are moved onto a
   thread-local pending deallocation list before the remaining payload is
   destroyed and the node's memory is reclaimed.
3. The runtime processes this worklist iteratively in a flat loop:
   - If a child's reference count decrements to zero, its children are queued
     onto the worklist before its remaining payload is destroyed and its memory
     is reclaimed.
   - If a child is still shared elsewhere, its count is decremented and it is
     removed from the worklist.
4. For mutually recursive inductive groups, a single worklist handles nodes
   across all member types seamlessly.

The prototype's [`Node.__deinit__`][data] demonstrates iterative release, and
its [cleanup tests][cleanup] successfully drop trees a million levels deep.

Iterative release may alter the destruction order across siblings and subtrees:
a node's payload is destroyed before its children are released, but the relative
order among separate subtrees is unspecified.

### Cost Model and Optimization Opportunities

Inductive enums establish a transparent, predictable cost model:

| Operation                 | Complexity                             | Implementation Details                                           |
|:--------------------------|:---------------------------------------|:-----------------------------------------------------------------|
| **Construct Node**        | $O(1)$, excluding payload construction | One heap allocation + header initialization (tag & refcount)     |
| **Copy Handle**           | $O(1)$                                 | Single pointer copy + atomic reference count increment           |
| **Move or Borrow Handle** | $O(1)$                                 | No refcount modification                                         |
| **Inspect Borrowed Node** | $O(1)$                                 | Direct pointer dereference (no refcount modification)            |
| **Release Handle**        | $O(1)$ plus reclamation                | Atomic decrement; if zero, node enters iterative worklist        |
| **Structural Hash**       | $O(V + E + H)$                         | Visits unique nodes and child slots; $H$ is payload hashing cost |

Here, $V$ counts distinct nodes and $E$ counts child slots, including repeated
references to the same node. Reclamation visits each released node and its child
slots once, plus the cost of destroying non-recursive payloads.

Releasing a parent decrements every child handle it stores. If two fields point
to the same shared child, both references are decremented, though the child's
memory is freed at most once.

#### Compiler & Runtime Optimizations

The language specifies the formal semantics rather than a rigid allocator.
Implementations are free to adopt proven functional-systems optimizations:

- **Trivial Leaf Inlining (Future Work):** Small payloads could be encoded
  inside the handle word when both the payload and tag fit; full-width integers
  may still require boxing. The initial representation keeps leaves in heap
  nodes. Inlining needs rules that preserve node identity and borrowed payload
  access before it can replace that representation.
- **Node Pooling:** Allocating nodes from per-type slab allocators with
  thread-local free lists, drastically reducing allocator latency.
- **Uniqueness Detection and In-Place Reuse (Perceus / Lean 4):** When a
  structural transformation owns an input handle (`var e: Expr`) with a
  reference count of 1, there are no other owning handles. Storage reuse also
  requires the compiler to establish that no live borrow or later use needs the
  original contents, under the
  [owned-subject rules](#owned-subjects-var-e-expr). The runtime can then reuse
  the existing memory buffer in place. This combines functional purity with the
  performance of imperative in-place updates (as demonstrated by Koka's
  [*Perceus*][perceus] and Lean 4's [*Counting Immutable Beans*][beans]).
- **Thread-Local Elision:** When the implementation can establish that a node is
  reachable only from the current thread, atomic operations on that node can be
  relaxed to non-atomic increments and decrements. Escape analysis of a single
  value is not enough: a value built on one thread may point to child nodes that
  other threads also hold, so the property must hold for every node whose count
  is updated. Lean 4 tracks this at runtime by marking every object reachable
  from a value when the value is shared with another thread, and uses atomic
  counts only for marked objects.

Atomic reference counts ensure handles can be safely copied and dropped across
multiple threads. Concurrent reads of shared payloads depend on the
thread-safety of the payload types, as with any shared Mojo value. Inductive
values are not supported inside GPU kernels in v1.

### Integration with `EnumLike` and Pattern Matching

The compiler synthesizes an `EnumLike` conformance for every inductive enum,
matching standard enums. The active case's payload resides within its heap node,
so `_unsafe_get_enum_payload` returns a reference into the node with the origin
of the handle.

Because inductive nodes are shared and immutable, reading an active payload must
always yield an *immutable* reference:

- In the `EnumLike` trait, we introduce a compile-time trait member:

  ```mojo
  comptime _enum_payloads_are_mutable: Bool = True
  ```

- The default preserves ordinary enums' existing mutability rules. Inductive
  enums explicitly set this to `False`. Payload projection must then return an
  immutable reference even for a mutable handle, including in generic
  `EnumLike` code and `ref` patterns, preventing mutations that would violate
  sharing invariants.

This requires changing the signature of `_unsafe_get_enum_payload` in the
[enums proposal](enums.md): its result is currently `ref[self]`, which is
mutable whenever the handle is. Its result origin must instead be the handle's
origin, made immutable when `_enum_payloads_are_mutable` is `False`, so that
generic `EnumLike` code receives an immutable reference from the signature
alone. The FIXME on that method already calls for a different origin; the two
changes could be made together.

Standard pattern matching (`match`, `if case`, `while case`) works out of the
box for shallow inspection:

```mojo
match e:
    case .add(.num(0), right):
        use(right)
    case _:
        pass
```

In standard `match` statements, pattern matching inspects stored handles without
triggering recursive evaluation. Recursive computation is requested explicitly
via `match recursive` and `recur`.

### Compile-Time Evaluation (`comptime`)

An expression graph constructed at compile time can be used to specialize a
kernel, perform constant folding, or drive code generation. Inductive values are
well suited for this, but runtime handles (which have node identity and
reference counts) differ from static compile-time parameters:

For compile-time evaluation, we propose:

- Inductive values can be constructed, matched, and passed to structural
  functions in compile-time code, wherever the compile-time interpreter
  supports heap allocation.
- In v1, they cannot be used as static parameters of types or functions, and
  values built at compile time cannot be materialized directly into runtime
  memory without explicit serializing/deserializing logic.
- Hashing works at compile time as it does at runtime: no digests are stored,
  and the result depends only on the value and the hasher the caller supplies.
  These values do not cross into runtime.

## Choosing a Storage Strategy

Any recursive type requires a layer of indirection: an `Expr` cannot embed two
`Expr` values inline without having infinite size. We chose shared,
reference-counted nodes so that programs can cheaply maintain multiple versions
of a data structure, share identical subtrees, and reclaim nodes the instant
they become unreachable.

Other memory representations—such as unique owned boxes or region arenas—excel
in specific domains. They could also support checked descent and iterative
traversal, but carry different ownership trade-offs:

| Property             | Owned Box (`OwnedPointer[T]`)   | Shared Atomic Refcounting       | Region Arena                 |
|:---------------------|:--------------------------------|:--------------------------------|:-----------------------------|
| **Construct Node**   | Allocate heap buffer            | Allocate buffer + init refcount | Bump-allocate in region      |
| **Copy Value**       | Deep clone (if payloads permit) | Pointer copy + atomic increment | Copy pointer                 |
| **Borrow Node**      | Zero overhead                   | Zero overhead                   | Zero overhead                |
| **Release Storage**  | Free each node individually     | Decrement; free when zero       | Reclaim entire arena at once |
| **Subtree Sharing**  | Forbidden (strictly unique)     | Yes (arbitrary DAG sharing)     | Yes (within arena lifetime)  |
| **Lifetime in Type** | None                            | None                            | Region origin parameter      |

### Owned Box

In an owned box model (comparable to Rust's `Box<T>` or C++'s
`std::unique_ptr`), every child node is strictly unique. This eliminates
reference counting overheads and allows in-place mutation without uniqueness
checks. It works well for purely linear trees that are consumed or transformed
destructively without preserving earlier versions.

However, strict ownership forbids structural sharing. As illustrated earlier,
constructing `Expr.add(x, x)` 60 times requires 61 nodes in a shared DAG, but
would demand $2^{61}-1$ nodes in an owned box tree. Deeply cloning an owned tree
requires copying every payload along the tree. Shared nodes allow programs to
retain intermediate revisions and share common sub-expressions with zero copy
overhead.

### Region Arena

An arena allocates nodes consecutively within a pre-allocated memory chunk using
fast pointer bumps, and reclaims all allocated memory simultaneously when the
arena itself is destroyed. Handles carry the arena's origin parameter, allowing
the compiler to verify that handles do not outlive the arena. Sharing requires
no reference counts.

This model is ideal for phase-oriented workloads—such as parsing an entire AST,
performing a compiler pass, and then discarding the entire AST at once.

However, if a long-lived application maintains evolving versions of a tree in an
arena, dead nodes cannot be freed individually; they remain allocated until the
entire arena is released, so occupied memory grows without bound. Freeing a
single node requires knowing that it is dead, and with sharing that depends on
runtime data: it requires either tracing from roots, which is garbage
collection, or counting references, which is the shared strategy below with a
different allocator. Region inference does not remove the problem: the ML Kit
compiler inferred regions for every allocation and still had to add a garbage
collector ([Hallenberg, Elsman, and Tofte][mlkit]).

Mojo does not use garbage collection, and a production system cannot let
occupied memory grow without bound. We therefore rejected arenas, along with the
region origins they would add to every type that holds an inductive value.

### Shared Nodes with Atomic Reference Counts

Shared, reference-counted nodes provide the ideal default for general-purpose
inductive data:

- Copying a handle is $O(1)$.
- Borrowing incurs zero reference count updates.
- Keeping a subtree alive requires an increment only at its root.
- Values can freely outlive the lexical scope in which they were constructed.

While atomic operations incur modest overhead, compiler optimizations (inline
leaves, node pools, thread-local elision, and Perceus-style in-place reuse)
substantially mitigate these costs.

The [FP Mojo prototype][fp] validates this model, maintaining 60 levels of DAG
sharing in 61 nodes and safely reclaiming structures a million levels deep.

## Why Not a Library Box?

The companion enums proposal and the [forum discussion][storage-discussion]
suggest that recursive cases could be deferred and represented via a library box
such as `HeapBox[Self]`. While a library box
provides the necessary memory indirection, it cannot solve the core systems and
language problems on its own:

1. **Stack Overflow on Deallocation:** A standard `HeapBox` destructor
   recursively frees its boxed child. On deep trees, dropping an instance blows
   the native stack. Preventing this requires custom iterative deallocation
   integrated into the type's drop logic.
2. **Loss of Structural Sharing:** A uniquely owned `HeapBox` cannot share
   subtrees between two owning parents.
3. **Lack of Termination and Descent Checking:** While a box can expose its
   contents to shallow pattern matching, the compiler cannot verify that a
   recursive function strictly descends along structural child paths. Without
   language-level knowledge of inductive positions, the compiler cannot prove
   termination or lower traversals onto an explicit heap stack.

FP Mojo demonstrates that while shared nodes, coverage analysis, and explicit
stack execution *can* be implemented in a library, doing so requires verbose
metaprogramming: clause limits, copying payload fields into lambda arguments,
and slow compilation times. Native language support allows structural
computations to use Mojo's pattern matching, ownership rules, and borrow checker
directly, backed by robust compiler diagnostics.

## Describing Children

The `EnumLike` trait describes an enum's cases and payload types. Structural
recursion also requires knowing *where* recursive children reside within those
payloads. For each recursive position, the compiler computes and records a path
through the payload fields leading to a child or child container.

### Child Containers

A **child container** holds an ordered collection of recursive child slots. Once
embedded in an inductive node, neither the slots nor their handles may be
mutated, even through an alias. Two slots may point to the same child node, but
each slot is visited exactly once during traversal.

To participate in structural recursion, a container must expose four primitive
capabilities to the compiler:

1. **Borrow Iteration:** Visit child handles by immutable reference in order.
2. **Result Mapping:** Construct a container of the identical shape holding
   computed child results (e.g., executing `recur items` on a `List[Self]`
   field yields a `List[R]`).
3. **Rebuilding:** Reconstruct the container from transformed child handles
   (used by `case recur rebuilt`).
4. **Destructive Worklist Extraction:** Relinquish child handles via move in
   order, enabling the iterative deallocation worklist to drain children
   without invoking recursive destructors.

Initially, `List`, `Optional`, and `Dict` provide these operations through a
built-in interface known to the compiler (not a public trait):

```mojo
inductive enum Json:
    case null
    case boolean(Bool)
    case number(Float64)
    case string(String)
    case array(List[Self])
    case object(Dict[String, Self])
```

Mapping results across a `Dict[K, Self]` preserves its keys, so `K` must conform
to `Copyable` (in addition to standard dictionary key constraints). Rebuilding
any container similarly requires that non-recursive elements be `Copyable`.

User-defined child containers are deferred until future work can establish
formal verification or safe trait contracts ensuring that custom containers
cannot mutate aliased children or enter infinite enumeration loops.

### The `Inductive` Trait and Generic Code

In v1, the subject of `match recursive` must have a concrete inductive enum
type, possibly generic (such as `Tree[T]`). Generic library code over *any*
inductive enum, such as a serializer or a pretty-printer, can reflect over
`EnumLike`, but it must recurse natively and does not receive the stack-safety
guarantee; only the derived operations do. The matching proposal cannot match
over a generic `EnumLike` type yet either, because case analysis is performed by
the parser.

As a follow-up, the compiler could synthesize an `Inductive` conformance for
every inductive enum. Extending `EnumLike`, `Inductive` would expose the
structural reflection needed to inspect child paths, iterate over child
containers, and rebuild nodes. Structural functions could then operate over
generic type parameters constrained by `T: Inductive`:

- Generic library algorithms (such as serializers, tree visualizers, or generic
  rewriters) would gain access to the same checked, stack-safe traversal
  mechanics as derived operations.
- When generic code accesses a child through an `Inductive` path, the compiler
  would preserve the proof of structural descent, allowing generic arms to
  invoke `recur` safely.
- Generic code would handle unknown constructors via a catch-all arm (`case _`
  or `case recur node`). To use `case recur node`, the generic function would
  require that retained payload fields are `Copyable`.

Specifying this trait requires settling its members, how generic code types
`recur` on children of other members of an inductive group, and how the
conformance is reserved to the compiler: handwritten implementations cannot
assert the finite structure and strict child relationships it guarantees.

## Structural Functions

### Form and Syntax

A **structural function** is a function whose final statement is
`match recursive` over one of its parameters, termed the **subject**:

```text
def name(subject: Type, other: Arg, ...) -> Result:
    # optional statements, run once per node
    match recursive subject:
        case pattern:
            ...
            return result_for_this_node
```

The subject must have a concrete inductive enum type, possibly generic (see
[Generic Code](#the-inductive-trait-and-generic-code)). It is borrowed by
default, unless declared `var` (see
[Owned Subjects](#owned-subjects-var-e-expr)). Additional parameters are treated
as **per-node arguments**.

Inside a structural function:

- Each `case` arm computes a result for the active node.
- At the outermost call, `return` yields the function's final result.
- Within a recursive invocation, `return` supplies the value for that child
  request.
- Structural functions return values, not references.
- `recur` is a reserved keyword indicating a compiler-checked structural call.

Methods on inductive enums can be structural functions by matching over `self`.
Inside the method, `recur child` invokes the identical method on the child:

```mojo
inductive enum Expr:
    case num(Int)
    case add(Self, Self)
    case if_else(Self, Self, Self)

    def size(self) -> Int:
        match recursive self:
            case .num(_):
                return 1
            case .add(recur left, recur right):
                return 1 + left + right
            case .if_else(recur c, recur t, recur f):
                return 1 + c + t + f
```

### Requesting Results in Patterns

Placing `recur` before a child position in a pattern requests that the child be
recursively evaluated with the current per-node arguments. The subsequent
pattern matches against the computed result:

| Pattern Syntax                | Matches or Binds                                     |
|:------------------------------|:-----------------------------------------------------|
| `child`                       | The un-evaluated child handle (borrowed)             |
| `recur child`                 | The computed child result (borrowed)                 |
| `recur var child`             | The computed child result (owned by the arm body)    |
| `recur .num(0)`               | Matches the computed result against a nested pattern |
| `recur items` on `List[Self]` | A `List[R]` of computed results, one per element     |

#### Arm Matching Pipeline

To maximize performance, pattern matching evaluates child requests lazily and
in a strictly defined sequence:

1. **Constructor & Stored Value Tests:** The case tag and any patterns
   matching stored values are tested *before* requesting any child evaluations
   for this arm.
2. **Child Evaluation:** Requested children are evaluated in payload field order
   (and element order within child containers).
3. **Result Pattern Matching:** Patterns on computed results (such as
   `recur .num(0)`) are tested.
4. **Guard Execution:** If present, the arm's `if guard` expression executes.

If step 1, 3, or 4 fails, execution falls through to the next arm. The runtime
caches any child results already computed for this node; subsequent arms reuse
these cached results without re-evaluating the children.

Nested patterns can request deeper descendants directly:
`case .add(.add(recur a, recur b), recur c)`.

Guards borrow computed results and cannot contain `recur`. An arm body can take
full ownership of a computed result using `recur var`.

### Recursion in Arm Bodies

`recur` may also be invoked as an expression directly within an arm body or
loop. This enables on-demand, short-circuiting traversals:

```mojo
inductive enum Tree:
    case leaf(Int)
    case node(List[Self])


def first_negative(t: Tree) -> Optional[Int]:
    match recursive t:
        case .leaf(value):
            if value < 0:
                return value
            return None
        case .node(children):
            for child in children:
                var found = recur child
                if found:
                    return found
            return None
```

Here, traversal terminates at the first negative leaf encountered. The compiler
lowers the arm by splitting execution at each `recur` expression—analogous to
how coroutines are split at suspension points—preserving live local variables in
the node's explicit stack frame. `recur` cannot be used inside nested functions
or closures.

### Per-Node Arguments

Calling `recur child` passes the function's per-node arguments down unmodified.
To pass updated arguments (e.g., maintaining an environment in an interpreter),
invoke the structural function explicitly:

```mojo
inductive enum Env:
    case empty
    case bind(String, Int, Self)


inductive enum Term:
    case num(Int)
    case variable(String)
    case add(Self, Self)
    case let_in(String, Self, Self)


def lookup(env: Env, name: String) raises -> Int:
    match recursive env:
        case .empty:
            raise Error("unbound variable: " + name)
        case .bind(bound_name, value, rest):
            if bound_name == name:
                return value
            return recur rest


def evaluate(t: Term, env: Env) raises -> Int:
    match recursive t:
        case .num(value):
            return value
        case .variable(name):
            return lookup(env, name)
        case .add(recur left, recur right):
            return left + right
        case .let_in(name, recur bound, body):
            return recur evaluate(body, Env.bind(name.copy(), bound, env))
```

In `.let_in`, `recur bound` evaluates the binding expression in the current
environment. The explicit call `recur evaluate(...)` descends into the body with
the extended environment. The first argument to the explicit call must be a
proven structural child of the subject. The call may also name a different
structural function, such as `recur depth(child)` inside `evaluate`; all
structural functions that reach one another through `recur` share a single
explicit stack.

### Continuations and Tail Calls

When an arm executes `return recur x`, the result of `x` directly becomes the
result of the current node. The compiler detects this tail call and replaces the
current frame on the explicit stack, provided no active cleanups, borrows, or
exception handlers depend on it.

This ensures that linear chains of continuations operate in $O(1)$ frame space
(as demonstrated in `lookup` above). If an enclosing `try` block or live borrow
requires the frame to remain active, the frame is retained on the explicit
stack—which still avoids consuming native call stack frames.

### Mutual Recursion (Inductive Groups)

Compilers and abstract syntax trees frequently involve mutually recursive types:
for example, statements contain expressions, while expressions contain
statements (such as blocks or lambdas). Inductive enums that reference each
other form an **inductive group**. All members of a group must be declared
within the same module.

```mojo
inductive enum Expression:
    case num(Int)
    case add(Self, Self)
    case block(List[Statement], Self)


inductive enum Statement:
    case show(Expression)
    case seq(Self, Self)


def count_numbers(e: Expression) -> Int:
    match recursive e:
        case .num(_):
            return 1
        case .add(recur left, recur right):
            return left + right
        case .block(recur counts, recur result):
            var total = result
            for count in counts:
                total += count
            return total


def count_numbers(s: Statement) -> Int:
    match recursive s:
        case .show(recur count):
            return count
        case .seq(recur first, recur second):
            return first + second
```

A structural function over an inductive group is defined using function
overloads for each member type. The overloads share the same name and per-node
arguments:

- Invoking `recur x` automatically dispatches to the overload corresponding to
  `x`'s type.
- In `.block(recur counts, recur result)`, `counts` is a `List[Int]` produced by
  the `Statement` overload, while `result` is produced by the `Expression`
  overload.
- Every member type that a bare `recur` reaches must have an overload; a
  missing overload is a compile-time error.
- Overloads in a group share a single explicit stack and descent checker.
- [`memo`](#memoization-match-recursive-memo) is declared per overload. Each
  memoized overload keeps its own table, and a group may mix memoized and
  unmemoized overloads.

### Memoization (`match recursive memo`)

By default, reaching a shared DAG node via multiple paths evaluates it multiple
times. On a densely shared graph (such as the 61-node tree with $2^{61}-1$
paths), naive traversal results in exponential execution time.

Declaring a function with `match recursive memo` ensures that each distinct node
is evaluated at most once per outermost call, with subsequent requests reusing
the cached result:

```mojo
def count_leaves(e: Expr) -> Int:
    match recursive memo e:
        case .num(_):
            return 1
        case .add(recur left, recur right):
            return left + right
        case .if_else(recur c, recur t, recur f):
            return c + t + f


def main():
    var x = Expr.num(1)
    for _ in range(60):
        x = Expr.add(x, x)
    print(count_leaves(x))  # 1152921504606846976, computed in 61 arm runs!
```

#### Memoization Rules

1. **Identity-Keyed:** Results are keyed strictly by node pointer identity
   (`is`). No payload hashing is performed.
2. **`Copyable` Results:** Because a cached result may be returned to multiple
   parents, the function's return type must conform to `Copyable`.
3. **Fixed Arguments:** All `recur` calls to the memoized function must pass its
   per-node arguments unmodified, as arguments are not part of the memo key.
4. **Borrowed Subject:** Memoized functions require a borrowed subject.
5. **Errors Are Not Cached:** If an evaluation raises an error, the error is not
   cached; subsequent requests will re-attempt evaluation.

### Rewriting and Tree Transformations

Bottom-up tree rewriting is expressed via the catch-all pattern
`case recur rebuilt`:

```mojo
def simplify(e: Expr) -> Expr:
    match recursive e:
        case .add(recur .num(0), recur right):
            return right
        case recur rebuilt:
            return rebuilt
```

`case recur rebuilt` requests the recursive results of all children and binds
`rebuilt` to the reconstructed parent node. This form requires the function to
return its subject's inductive type, so child results fit the original recursive
slots. For a mutually recursive group, each participating overload must return
its own subject type.

#### Identity Preservation

If every child result evaluates to the exact same node handle as the original
child (`result is original`), the runtime avoids allocation and binds `rebuilt`
directly to the *original* parent node. This preserves existing structural
sharing across unchanged subtrees.

#### Rewriting Shared Data

Without `memo`, a rewrite evaluates a shared node once per path that reaches it,
like any other structural function. Each occurrence of a changed node is rebuilt
separately, so the result loses the sharing of its input, and the cost grows
with the number of paths rather than the number of nodes. Applied to the
61-node DAG above, `simplify` would evaluate $2^{61}-1$ node occurrences, even
though nothing in it changes. Rewrites over data that may be shared should use
[`match recursive memo`](#memoization-match-recursive-memo), which rewrites
each distinct node once and gives every request of that node the same result
node, preserving the sharing of the input.

#### Copyability of Retained Fields

When a child changes, constructing a new parent requires copying any
non-recursive payload fields (such as variable names, source locations, or
dictionary keys). The compiler verifies that all non-recursive fields across all
rebuilt constructors conform to `Copyable`. If a constructor contains a
non-copyable resource (such as a unique file handle), automatic rebuilding is
disallowed for that case. `case recur rebuilt` then does not cover that
constructor: an earlier arm must handle it, or the match is reported as
non-exhaustive.

#### Fixed-Point Rewriting

A single call to `simplify` performs one bottom-up pass. To apply rewrites until
a fixed point is reached, use an ordinary `while` loop with an identity check:

```mojo
var current = e
while True:
    var next = simplify(current)
    if next is current:
        break
    current = next
```

Because `is` is a single pointer comparison, detecting that a pass produced no
changes is extremely fast. The loop terminates only if a pass eventually returns
the same node. A rewrite that keeps allocating equal but distinct nodes will
not satisfy this check; termination of each pass does not guarantee convergence.

### Owned Subjects (`var e: Expr`)

By default, structural functions borrow their subject, preserving the caller's
original tree. When a function takes an owned subject (`var e: Expr`), it
receives an owning handle:

```mojo
def simplify_owned(var e: Expr) -> Expr:
    match recursive e:
        case .add(recur .num(0), recur right):
            return right
        case recur rebuilt:
            return rebuilt


var y = simplify_owned(x^)
```

Taking an owned subject enables potential **in-place storage reuse**:

- Before reusing a node's buffer, the runtime must verify that the node's
  reference count is exactly 1. The compiler must also establish that no live
  borrow, subsequent pattern, guard, arm, or cleanup needs the original
  contents.
- If an arm could fall through to a subsequent arm, the original children must
  remain intact. In such cases, child calls receive copied handles to prevent
  premature consumption of the node.
- Once the original value is definitively no longer needed, its storage can be
  recycled directly for the new parent, provided reuse respects live origins
  and preserves observable copying and destruction effects.

### Parallel Evaluation Considerations

Child computations within an arm are often independent and could theoretically
execute in parallel. However, parallel execution introduces complexities
regarding side effects, thread synchronization, error handling, and memo table
contention. For v1, child evaluation follows a strictly deterministic sequential
order.

### Ownership and Error Handling

Structural functions integrate seamlessly with Mojo's typed error model:

1. **Pattern Errors:** If an error is raised during the evaluation of a child
   requested in a pattern, the error propagates into the parent frame as if the
   pattern itself raised. No further arms are attempted for that node.
2. **Body Errors and `try`/`except`:** An error raised by `recur` within an arm
   body can be caught using a standard `try`/`except` block. For example, in an
   evaluator for a language with an `or_else` fallback construct:

   ```mojo
   case .or_else(body, fallback):
       try:
           return recur body
       except:
           return recur fallback
   ```

3. **Unwinding:** If an unhandled error escapes, the runtime unwinds the
   explicit frame stack, properly releasing all pending frames and temporary
   results without leaking memory.

## Compiler Checking and Verification

### Exhaustiveness Checking

Every constructor of an inductive enum must be covered by an unguarded arm or a
catch-all (`case _`, or `case recur rebuilt` for the constructors it can
[rebuild](#copyability-of-retained-fields)). Missing a constructor produces a
compile-time error:

```text
error: non-exhaustive structural match: missing Expr.if_else
```

#### Maranget's Pattern-Matrix Algorithm

Unlike standard enums with finite combinations, inductive data structures
represent an infinite universe of possible trees. Standard bitset enumeration
cannot check nested patterns over recursive types.

To verify exhaustiveness and detect unreachable arms, the compiler implements
**[Maranget's pattern-matrix algorithm][maranget]**, the gold standard used in
OCaml and Rust. The algorithm expands only the finite prefix of the tree
inspected by the patterns:

- For child containers, sequence patterns like `.array([])` and
  `.array([first, *rest])` are treated as constructor families, where `*rest`
  covers all lengths greater than or equal to the prefix.
- Guarded arms and refutable result patterns (e.g. `recur .num(0)`) do not count
  toward coverage because they may fail at runtime.
- For open enums (declared with `_enum_is_exhaustive = False`, see
  [Declaration and Syntax](#declaration-and-syntax)), external modules must
  provide a catch-all arm. When the function returns its subject's type,
  `case recur rebuilt` is a good choice: the rebuild is generated with the
  declaration, so it also handles cases added later.

### Structural Descent Verification

To establish structural descent, every structural call must pass a
**proven child expression**—a value the compiler can prove is a strict
structural sub-element of the current subject.

The compiler establishes proof of descent from:

1. Pattern variables bound to recursive positions of the subject.
2. Bindings obtained by matching over an already-proven child.
3. Elements obtained by iterating over, or indexing, an immutable view of a
   child container.

#### Preservation of Proofs Across Control Flow

The compiler tracks child provenance across variable bindings and assignments:

- If a child variable is reassigned to a newly constructed node or an unrelated
  value, the proof of descent is invalidated, and subsequent `recur` calls on
  that variable are rejected.
- At control-flow join points (such as `if`/`else` branches), all incoming paths
  must supply a proven strict descendant of the same subject.
- Arbitrary helper functions returning an inductive value do not carry proof of
  descent; only structural pattern bindings and verified container projections
  qualify.

### Termination Guarantees

Because every structural call strictly descends to a sub-element of a finite,
acyclic DAG, every descending chain of structural calls is finite. The whole
computation terminates if each activation performs finite local work and issues
finitely many child requests. This follows directly by the principle of
well-founded structural induction.

Arm bodies remain regular Mojo code. While the compiler guarantees that `recur`
descends, it does not prove that arbitrary loops or external functions within an
arm will terminate, or that an arm will stop requesting the same child. The
compiler emits helpful diagnostics in two common scenarios:

- **Warning on Unmarked Child Recursion:** Emitted when a structural function
  invokes another structural function on a child using an ordinary function call
  instead of `recur`.
- **Warning on Native Recursion:** Emitted when an ordinary function attempts to
  recurse directly on an inductive child, warning the developer that native
  stack frames will grow with tree depth.

### Explicit Stack Lowering and Bounded Stack Use

In standard programming languages, deep recursion consumes native stack frames,
leading to crashes on large trees.

Mojo guarantees that native stack usage for structural traversal and release
remains strictly bounded ($O(1)$) through **explicit stack lowering**:

- Structural functions are lowered onto an explicit heap-allocated frame stack.
- Calling `recur` pushes a logical frame onto the heap stack rather than
  allocating an OS call stack frame.
- Releasing an inductive value operates iteratively via a worklist.
- Derived operations (`Equatable`, `Hashable`, `Writable`) execute iteratively.

This bound applies to traversal through recursive positions. Ordinary calls in
arm bodies and non-recursive payload operations, including destructors, retain
their own stack behavior.

#### Debugger and Tooling Integration

To maintain an excellent developer experience, compiler debug metadata maps
heap-allocated frames to logical call frames. Debuggers (such as LLDB) and
profilers can reconstruct logical backtraces, allowing developers to step
through recursive algorithms exactly as they were written.

#### Hybrid Execution Optimization

To maximize performance on shallow trees while retaining safety on deep ones,
the runtime can employ a hybrid strategy: executing on the native stack up to a
threshold chosen for the target's stack budget and generated frame sizes, and
switching to the explicit heap stack when that limit is reached.

In the FP Mojo prototype's [recorded x86-64 benchmarks][architecture], with
minimal work per clause, a hand-written explicit-stack loop took about 2.4x
the time of native recursion on a balanced tree of 9,556 values; `fp.match`
took about 3x. On a chain 10,000 levels deep, the ratios were about 0.9x and
1.2x respectively. Separate tests validate evaluation and cleanup at a million
levels; they do not establish a universal failure depth for native recursion.

## Derived Operations

For inductive enums whose payloads conform to standard traits, the compiler
automatically derives implementations using `EnumLike` and its description of
children:

- **`Hashable`:** Traverses the DAG in a single iterative pass, caching digests
  for shared nodes during the call. Each unique node is hashed exactly once,
  combining the case tag, non-recursive payloads, and child digests. This avoids
  exponential hashing on shared DAGs.
- **`Equatable`:** Iteratively compares node tags and payloads. When comparing
  two DAGs, the runtime records pairs of nodes that have already compared equal,
  preventing duplicate traversals of shared subgraphs.
- **`Writable`:** Iteratively formats the tree into a text stream. Shared nodes
  are expanded structurally (meaning output text size can grow exponentially if
  deeply shared DAGs are formatted as expanded trees).

Payload equality and hash results must remain stable during each call; these
operations do not take a snapshot of external mutable state. Hash combination
must also respect container equality: equal dictionaries must hash equally
regardless of insertion order.

### Equality Identity Short-Circuiting

Handles pointing to the exact same node (`e1 is e2`) do not automatically skip
payload comparison unless the compiler can prove that the payload's `==`
operator is reflexive and that skipping the comparison has no observable
effects. Floating-point NaN is a counterexample to reflexivity: `NaN != NaN`,
even when both handles refer to the same node containing it.

### Digest Caching Between Calls

By default, nodes do not cache hashes between calls, because non-recursive
payloads could depend on external mutable state. Storing permanent digests
inside nodes would require the compiler or an explicit checked capability to
establish stable hash and equality behavior for the node's lifetime, and that
skipping later hash calls preserves observable effects. Neither `Hashable` nor
shallow immutability alone establishes these properties.

The cost is that every hash call walks the value: hashing a handle costs
$O(V + E + H)$ over the subgraph it reaches, not $O(1)$. Hash-consing and
common-subexpression tables keyed by subtrees, a staple of compiler IRs, hash
each new node as it is built, so building a deep value through such a table can
take time quadratic in its depth. An opt-in way to store digests is an
[open question](#open-questions).

## Alternatives Considered

- **Other Storage Strategies:** Owned boxes and region arenas are compared in
  detail in [Choosing a Storage Strategy](#choosing-a-storage-strategy).
- **Multiple Storage Declarations:** Allowing declarations to choose their
  storage strategy (e.g. `@arena` vs `@shared`) was considered and rejected.
  Two kinds of inductive values with different copying, lifetime, and
  reclamation rules would confuse users and split libraries. Shared atomic
  reference counting is the single storage strategy.
- **Unmarked Recursion (Lean/Coq/Agda style):** Languages like Lean and Coq
  allow standard function calls and verify descent automatically. We require the
  explicit `recur` marker because:
  1. It clearly distinguishes stack-lowered structural calls from ordinary
     function calls.
  2. It is syntactically necessary for requesting child results within pattern
     destructuring.
- **Clause-Only Match Expressions:** An expression-based `match` construct was
  considered. We chose a statement-based `match recursive` form because it
  integrates naturally with Mojo's multi-statement arm bodies, imperative
  loops, local variables, and typed error handling (`raises`/`try`).
- **Automatic Memoization:** The FP Mojo prototype memoizes automatically
  whenever the result type is copyable. However, because arm bodies may execute
  arbitrary side effects, native Mojo requires memoization to be explicitly
  requested via `match recursive memo`.
- **Separate Rewrite Constructs:** Providing a dedicated `rewrite` keyword was
  evaluated. We chose `case recur rebuilt` instead, providing identical
  functionality inside `match recursive` without introducing additional language
  keywords.

## Implementation Staging

Inductive types and their structural computation model should be accepted
together, so recursive storage does not ship with checked recursion left
undecided. We propose implementing that agreed design in three structured
phases:

### Phase 1: Declarations, Storage, and Cleanup

- Parsing of `inductive enum` syntax and verification of recursive positions.
- Heap node representation with atomic reference counting.
- Synthesis of the `EnumLike` conformance with immutable payload access.
- Iterative worklist deallocation in `__deinit__`.
- Shallow pattern matching support.

### Phase 2: Structural Functions and Stack Lowering

- Parsing of `match recursive` and `recur` in patterns and arm bodies.
- Implementation of Maranget's pattern-matrix exhaustiveness checker for nested
  constructors and child containers.
- Structural descent tracking in the compiler semantic analyzer.
- Coroutine-style frame splitting and explicit heap stack lowering.
- Continuation tail-call optimization.

### Phase 3: Transformations and Derived Operations

- Automatic node reconstruction (`case recur rebuilt`) with copyability checks.
- Identity-keyed memoization (`match recursive memo`).
- Derived `Equatable`, `Hashable`, and `Writable` implementations.
- In-place buffer reuse optimizations for unique owned subjects (`var`).

## The Prototype

We have implemented and verified the core concepts of this proposal in
[FP Mojo][fp], an open-source library built for Mojo 1.1.

The prototype's test suite validates:

- Shared immutable nodes backed by atomic reference counts.
- Iterative cleanup on trees **1,000,000 levels deep** without stack overflows.
- Exhaustiveness checking and rejection of missing constructors.
- Lazy evaluation of pattern-requested children.
- Non-destructive bottom-up rewriting with identity reuse.

The prototype uses typed clauses to request child results and `Next` to select
a continuation. `Next` checks for decreasing height at runtime; the native
proposal instead checks structural descent in the compiler. The prototype also
copies clause payloads where the native design can borrow, and memoizes shared
nodes automatically when results are `Copyable`.

Native syntax, arbitrary `recur` requests in arm bodies, changing per-node
arguments, mutual structural functions, and owned storage reuse go beyond this
prototype. So do `Dict` children, nested and result patterns, derived
operations, the `EnumLike` interface, open enums, and compile-time use. These
require compiler implementation and validation of their own.

### Validation Matrix

| Feature                                     | Verified by Prototype Component                                |
|:--------------------------------------------|:---------------------------------------------------------------|
| **Shared Immutable Nodes**                  | [`Node` and `_Cell`][data]                                     |
| **Iterative 1M-Deep Cleanup**               | [`Node.__deinit__`][data], [cleanup tests][cleanup]            |
| **Exhaustiveness Verification**             | [`_admit` and constructor plans][engine]                       |
| **Explicit Stack Execution**                | [`_run` and `_continue_with`][engine]                          |
| **Lazy Child Evaluation**                   | [runtime tests][single]                                        |
| **Container Children (`List`, `Optional`)** | [runtime tests][single]                                        |
| **Rewriting with Identity Preservation**    | [rewrite tests][rewrite]                                       |
| **Rejection of Non-Exhaustive Matches**     | [missing][missing], [guard][guard], [unreachable][unreachable] |

### Running the Prototype

To clone and run the prototype test suite locally:

```bash
git clone https://github.com/Spellbound-Mojo/fp_mojo.git
cd fp_mojo
git checkout 2c2b163ecec70c88440b336d4068cb6b3ac9f91b
pixi run mojo run -I src docs/examples/expression.mojo
pixi run mojo run -I src tests/runtime/test_match_single.mojo
pixi run mojo run -I src tests/ownership/test_match_cleanup.mojo
pixi run mojo run -I src tests/runtime/test_rewrite.mojo
pixi run compile-fail --source --filter match_ --werror --jobs 2
```

## Open Questions

1. **Syntax and Keywords:** Are `inductive enum`, `match recursive`, `recur`,
   and `memo` the most natural keywords, or would alternative spellings (such as
   `rec match` or `recurse`) better align with Mojo conventions?
2. **Consuming Non-Copyable Payloads:** Should there be an explicit consuming
   match construct that allows extracting non-copyable payloads from uniquely
   owned nodes, and how should it signal failure if the node is shared?
3. **Measure-Checked Recursion:** How should advanced structural continuations
   checked by a user-defined measure function (e.g., Ackermann-like functions or
   lexicographic descent) be expressed if needed?
4. **Multi-Subject Recursion:** What syntax should be used for structural
   recursion over multiple subjects simultaneously (e.g. zipping two trees or
   comparing two inductive values)?
5. **Verified User Containers:** What type-system capabilities or unsafe
   contracts are needed to allow third-party custom child containers to
   participate in structural recursion?
6. **Parallel Traversal:** What formal execution guarantees should a future
   parallel `match recursive` mode provide regarding side-effect ordering and
   memoization?
7. **Compile-Time Reification:** How should inductive values constructed at
   compile time be serialized or materialized as static runtime globals?
8. **Cached Digests:** What checked, opt-in capability should let a declaration
   store a digest in each node, so that hashing costs $O(1)$ per call, and what
   must it establish about the stability of payload hashing and equality?
9. **Open Declarations:** Should open inductive enums have a dedicated spelling
   (such as a decorator) instead of overriding `_enum_is_exhaustive`?
10. **Generic Structural Functions:** What members should a synthesized
    `Inductive` trait expose, and how should generic code type `recur` on
    children of other members of an inductive group?

[fp]: https://github.com/Spellbound-Mojo/fp_mojo/tree/2c2b163ecec70c88440b336d4068cb6b3ac9f91b
[architecture]: https://github.com/Spellbound-Mojo/fp_mojo/blob/2c2b163ecec70c88440b336d4068cb6b3ac9f91b/docs/content/architecture/matching.md
[data]: https://github.com/Spellbound-Mojo/fp_mojo/blob/2c2b163ecec70c88440b336d4068cb6b3ac9f91b/src/fp/adt/data.mojo
[engine]: https://github.com/Spellbound-Mojo/fp_mojo/blob/2c2b163ecec70c88440b336d4068cb6b3ac9f91b/src/fp/matching/_engine.mojo
[single]: https://github.com/Spellbound-Mojo/fp_mojo/blob/2c2b163ecec70c88440b336d4068cb6b3ac9f91b/tests/runtime/test_match_single.mojo
[cleanup]: https://github.com/Spellbound-Mojo/fp_mojo/blob/2c2b163ecec70c88440b336d4068cb6b3ac9f91b/tests/ownership/test_match_cleanup.mojo
[rewrite]: https://github.com/Spellbound-Mojo/fp_mojo/blob/2c2b163ecec70c88440b336d4068cb6b3ac9f91b/tests/runtime/test_rewrite.mojo
[missing]: https://github.com/Spellbound-Mojo/fp_mojo/blob/2c2b163ecec70c88440b336d4068cb6b3ac9f91b/tests/compile_fail/match_missing_constructor.mojo
[guard]: https://github.com/Spellbound-Mojo/fp_mojo/blob/2c2b163ecec70c88440b336d4068cb6b3ac9f91b/tests/compile_fail/match_guard_only.mojo
[unreachable]: https://github.com/Spellbound-Mojo/fp_mojo/blob/2c2b163ecec70c88440b336d4068cb6b3ac9f91b/tests/compile_fail/match_unreachable_clause.mojo
[storage-discussion]: https://forum.modular.com/t/pattern-matching-enum-proposals/3473/13
[composition-discussion]: https://forum.modular.com/t/pattern-matching-enum-proposals/3473/50
[named-fields-discussion]: https://forum.modular.com/t/pattern-matching-enum-proposals/3473/56
[beans]: https://arxiv.org/abs/1908.05647
[perceus]: https://doi.org/10.1145/3453483.3454032
[mlkit]: https://doi.org/10.1145/512529.512547
[maranget]: https://doi.org/10.1017/S0956796807006223
[lean-runtime]: https://lean-lang.org/functional_programming_in_lean/Programming___-Proving___-and-Performance/Summary/
[rust-recursive]: https://doc.rust-lang.org/stable/reference/types.html#recursive-types
