# A Stateful MIR for Rust

I here propose an enrichment of Rust's core language (the MIR) to support more advanced features.
The list of features I use or enable in the surface language is broad, and should include:

 - "First-class "initializedness" i.e. making the type system aware of initialized locations (or even values!)
   - Typestate
 - [`&move` and friends](https://github.com/rust-lang/rfcs/issues/998)
   - Moving out of `&mut` temporarily [isn't there an issue for this?]
 - Enum variant subtyping
 - Safe `DerefMove`
 - [Placer API](https://github.com/rust-lang/rfcs/pull/809)
 - Partial Moves
   - [Non lexical lifetime](http://smallcultfollowing.com/babysteps/blog/2016/05/04/non-lexical-lifetimes-based-on-liveness/)'s "fragments"
 - Drop
   - [By-value / by `&move` Drop](https://github.com/rust-lang/rfcs/pull/1180)
   - [Cyclic drop](https://github.com/rust-lang/rfcs/pull/1327)
   - Cyclic init
   - [Linear types](https://github.com/rust-lang/rfcs/pull/776)
 - Dynamism
  - Enums with tags in a separate location
  - Desugaring [non-zeroing dynamic drop](https://github.com/rust-lang/rfcs/pull/320)
  - [Non-lexical lifetimes](http://smallcultfollowing.com/babysteps/blog/2016/05/09/non-lexical-lifetimes-adding-the-outlives-relation/)'s "outlives at"

Most previous discussion (what I've linked) focuses on the surface language, backwards compatibility, and the like.
Where the discussion gets stuck is it's not clear what hidden complexity (especially soundness) may be overlooked.
I've instead opted for the route of defining a core language rigorously enough so that readers might convince themselves (or prove!) that the core language is sound and both today's Rust and the proposed extensions can be desugared to it.

The core idea here is to bring back typestate---in this presentation the ability for locations to change type.
With that, many nice abilities arise naturally.


## First-class "Initializedness"

Rust already has limited support for creating locations and initializing them later---see `let x;`.
The problem with its current support is that it's anti-modular: whatever sort of flow analysis pass implements this just looks at the body of a single function and does not allow any operation on the uninitialized local variable.
The solution is two-fold: first, being able to infer "initializedness"---what I am calling the status of whether something is initialized or not---from types alone, and second, allowing locations to change type.

The need for more types is simple enough to understand.
From a tool writer's perspective, types are *the* way to divide-and-conquer or suspend-and-resume some sort of global static analysis:  a little local analysis is done, code/data is tagged with what was proved, and that information can can now be used remotely.

One question is whether lvalues/locations and rvalues/values both need to know about initializedness: on one hand one only writes to or takes references to lvalues, and currently reading uninitialized locations is prohibited.
On the other hand, the type system doesn't currently discriminate between rvalues and lvalues, and in the interest of simplicity it would be nice to keep it that way.
Here's my tiebreaker: functional struct update (`{ foo: bar, ..baz }`) both could benefit from initializedness (field `foo` need not be initialized in `baz`) and only concerns rvalues.
(While a valid desugaring might introduce some temporaries and assign them, this is not the only approach.)
Therefore I opt to include initializedness to types for rvalues and lvalues alike.
Without any way to inspect an undefined value, this should be perfectly safe.

With that alone, however, an uninitialized location can never be used for anything, and an initialized location must stay initialized---we are worse off than before!
But, if we allow a location to change its type (as long as size and alignment is preserved), we get back where we started, and more!
So moving out would change the type of a location from `initialized T` to `uninitialized T`, and the opposite for writes.
("Dropping then writing" is best though of as two separate steps, but can still look like one step only in the surface language.
More on this and dropping in general later.)
In fact, we could even allow changing from `T` to `U` if they have the same size!
Likewise, we only need one uninitialized type per size.
That said, it's possible to stage all this in the core language or surface language, just starting with stateful initializedness for locations.

Let's now start to formalize our new MIR, limiting ourselves to what features have been introduced so far.

For convenience,
```
anything* ::= | anything ',' anything*
```
aka comma-separated repetition, 0 or more times.
As these grammars are really "language agnostic ADTs" and not here for parsing, I may play fast and loose with trailing commas and the like.

lvalues are global or whole-function scoped:
```
Static, s  ::= 'static0' | 'static1' | ...
Local,  l  ::= 'local0'  | 'local1'  | ...
Param,  p  ::= 'param0'  | 'param1'  | ...
LValue, lv ::= 'ret_slot' | Static | Local | Param
```

Types, constants, and rvalues are largely "what one would expect".
Additionally, there is a (size-indexed) uninitialized type and an inscrutable constant inhabiting it, and an absurd type, with no inhabitants:
```
Size,        N  ::= '0b' | '1b' | '2b' | ...
TypeBuiltin     ::= 'Uninit'<Size> | ! | ...
TypeParam,   TP ::= 'TParam0' | 'TParam1' | ...
TypeUserDef     ::= 'User0'   | 'User1'   | ...
Type,        T  ::= TypeBuiltin | TypeParam | TypeUserDef
```
The absurd type is so absurd it can be all sizes for all lvalues (thanks @RalfJung!); there is no need to give it a size index.

```
Constant, c  ::= 'uninit'::<Size> | ...
Operand,  o  ::= 'const'(Constant)
              |  'consume'(LValue)
Unop,     u  ::= ...
Binop,    b  ::= ...
RValue,   rv ::= 'use'(Operand)
              |  'unop'(Unop, Operand, Operand)
              |  'binop'(Binop, Operand, Operand)
```

An *lvalue context* represents the contents of each lvalue *at a node in the CFG*.
A well-formed lvalue context must assign a type to each lvalue exactly once---it is conceptually a function or total map from lvalues to types.
A *static context* is just an lvalue context restricted to statics.
```
LValueContext, LV ::= (LValue: Type)*
StaticContext, S  ::= (Static: Type)*
```

For now, we only will worry about `Copy` implementations, but more generally an *type context* keeps track of user defined types, type parameters, traits bounds.
A trait bound can only involve user-defined types, in which case it represents an impl, or it can involve type parameters, in which case it represents a postulate from a where clause.
[N.B. this roughly corresponds to rustc's `ParameterEnvironment`.]
```
Trait,       tr  ::= # some globally-unique identifier
TraitBound,  trb ::= Type ':' Trait
TypePremise      ::= TraitBound | TypeParam | TypeUserDef
TypeContext, TC  ::= TypePremise*
```
As a basic scoping rule, a parameter should come before any bound that uses it in a type context.

Nodes in the CFG we will think of as continuations: they do not return, and while they don't take any arguments, the types of locations can be enforced as a prerequisite to them being called.
```
Label,      k ::= 'enter' | 'exit'
               |  'k0' | 'k1' | ...
Node,       e ::= 'Assign'(LValue, Operand, Label)
               |  'DropCopy'(LValue, Label)
               |  'If'(Operand, Label, Label)
               |  'Call'(Operand, Label)
               |  'Unreachable'
               |  ... # et cetera
NodeType,  ¬T ::= ¬(LValueContext)
CfgContext, K ::= (Label : NodeType)*
```

And now the judgments.
Operands and rvalues have not one but two lvalue contexts, an "in context" and "out context".
This pattern allows us to thread some state.

Constants can become rvalues whenever, and the in-context and out-context are only constrained to be equal.
```
Const:
  TC  ⊢  c: T
  ────────────────────────────
  TC;  LV;  LV  ⊢  const(c): T
```

Consumption is more complex.
We need to uninitialize the lvalue iff the type is !Copy.
```
MoveConsume:
  ───────────────────────────────────────────────────────────────
  TC, T: !Copy;  LV, lv: T;  LV, lv: Uninit<_>  ⊢  consume(lv): T
```
```
CopyConsume:
  ──────────────────────────────────────────────────────
  TC, T: Copy;  LV, lv: T;  LV, lv: T  ⊢  consume(lv): T
```

The actual threading of the state is in the rvalue intruducers.
Note that the order of the threading does not matter---our state transformations are communicative.
```
Use:
  TC; LV₀; LV₁  ⊢  o: T
  ──────────────────────────
  TC; LV₀; LV₁  ⊢  use(o): T
```
```
UnOp:
  TC; LV₀; LV₁  ⊢  o: T
  u: fn(T) -> Tᵣ        # primops need no context
  ──────────────────────────────
  TC; LV₀; LV₁  ⊢  use(u, o): Tᵣ
```
```
BinOp:
  TC; LV₀; LV₁  ⊢  o₀: T₀
  TC; LV₁; LV₂  ⊢  o₁: T₁
  b: fn(T₀, T₁) ->        # primops need no context
  ───────────────────────────────────
  TC; LV₀; LV₂  ⊢  use(b, o₀, o₁): Tᵣ
```

Nodes do not have two lvalue contexts, because viewed as continuations they don't return.
The out contexts of their operand(s) instead constrain their successor(s).
Assignment is perhaps the most important operation:
```
Assign:
  TC; S, LV, lv: Uninit<_>;  S, LV, lv: Uninit<_>  ⊢  rv: T
  TC; S;  K  ⊢  k: ¬(LV, lv: T)
  ───────────────────────────────────────────────────────────
  TC; S;  K  ⊢  assign(lv, rv, k): ¬(LV, lv: Uninit<_>)
```
Note that the lvalue to be assigned must be uninitialized prior to assignment, and the rvalue must not affect it, so moving from an lvalue to itself is not prohibited.
[Also note that making `K, k: _ ⊢ ...` the conclusion instead of making `... ⊢ k: _` a premise would work equally well, but this is easier to read.]

Call resembles `Unop` and `Binop`, since its the moral equivalent for calling user-defined instead of primitive functions.
Functions do have type parameters, so we must substitute type args for type parameters.
While it's not too interesting now, it will become more interesting later.
```
Call:
  T₀ = for<TP*> fn(T₁..Tₙ) -> Tᵣ where trb*
  ∀i.
    TC; S, LVᵢ;  S, LVᵢ₊₁  ⊢ oᵢ : Tᵢ [Tₜₐ/TP]*
  TC, trb*; S;  K  ⊢  k: ¬(LVₙ₊₁, lv: Tᵣ)
  ────────────────────────────────────────────────────────────────
  TC, trb*; S;  K  ⊢  call<Tₜₐ*>(lv, o*, k): ¬(LV₀, lv: Uninit<_>)
```

We can define diverging functions simply by never calling 'exit' and creating a cylic in the CFG instead.
But when calling a non-terminating function, we still need to provide a successor.
This unreachable node can be used whenever there exists an lvalue with type `!`
Since that is the return type of a diverging function, after we return we will have an lvalue with that type (the return slot), and thus we can use this as a successor.
This is also useful for unreachable branch in an enum match (corresponding to an absurd variant).
```
Unreachable:
  ──────────────────────────────────────
  TC; S;  K  ⊢  unreachable: ¬(LV₀, lv: !)
```

In this formulation, everything is explicit, so we also need to drop copy types (even if such a node is compiled to nothing) to mark them as uninitialized.
```
CopyDrop:
  TC, T: Copy; S;  K  ⊢  k: ¬(LV, lv: Uninit<_>)
  ────────────────────────────────────────────────
  TC, T: Copy; S;  K  ⊢  drop(lv, k): ¬(LV, lv: T)
```

And here is `if`.
I could go on, but hopefully the basic pattern is clear.
```
If:
  TC; S, LV₀;  S, LV₁  ⊢  o: T
  TC; S; K  ⊢  k₀: ¬(LV₁)
  TC; S; K  ⊢  k₁: ¬(LV₁)
  ──────────────────────────────────
  TC; S; K  ⊢  if(o, k₀, k₁): ¬(LV₀)
```

And finally, the big "let-rec" that ties the "knot" of continuations together into the CFG --- and a function.
Every node in the CFG is postulated (node `eᵢ`, with type `¬Tᵢ`), and bound to a label (`k₀`).
```
Fn:
  k₀ = entry
  T₀ = ¬((s: Tₛ)*, (a: Tₚ)*, (l: Uninit<_>)*, ret_slot: Uninit<_>)
  ∀i.
    TC; # trait impls
    S;  # statics
    l*; # locals (just the location names, no types)
    K,  # user labels, K = { kₙ: ¬Tₙ | n }
      exit:  ¬((s: Tₛ)*, (a: Uninit<_>)*, (l: Uninit<_>)*, ret_slot: Tᵣ);
    ⊢ eᵢ: ¬Tᵢ
  ─────────────────────────────────────────────────────────────────────────────
  TC; S  ⊢  Mir { params, locals, labels: { (k: ¬T = e)* }, .. }: fn(Tₚ*) -> Tᵣ
```
Note the two special labels, 'enter' and 'exit'.
'enter' is defined like any other node, but must exist and match the function's signature.
Specifically, it requires that all locals are uninitialized, and all parameters are initialized to match the type of the function.
'exit' isn't defined at all, but bound in the CfgContext so nodes can choose to exit as a successor.
It requires that all locals and args are uninitialized, but the "return slot" is initialized.

For completeness, we can parameterize the MIR with type parameters and trait bounds like this:
```
FnGeneric:
  TC, TP*, trb*; S  ⊢  f: fn(Tₚ*) -> Tᵣ
  ───────────────────────────────────────────────────────────
  TC; S ⊢ (Λ<TP*, trb*> f): for<TP*> fn(Tₚ*) -> Tᵣ where trb*
```

Ok, make sense? I've of course left many parts of the existing MIR unaccounted for: compound lvalues, lifetimes, references, panicking, mutability, aliasing, and more.
Also, I only gave the introducers (I trust the MIRi devs to figure out the eliminators!).
Yet this IR is pretty advanced in some other ways.
Besides changing the types of locations, we have full support for linear types---note that everything must be manually deinitialized.
Going forward, I'll extend this IR little by little until everything should be covered.

[A final note, `CopyDrop` and `CopyConsume` could easily refer to different traits instead of both to `Copy`, to disassociate the ability to be memcopied from the ability to be forgotten.
I have proposed splitting `Copy` like this in the past, but will not mention it again, as nothing else depeons on this.


## Enum Switch

One thing that I haven't explicitly mentioned yet is the subtyping relation over continuation types.
There is no width subtyping because the lvalue must assign a type to all lvalues.
Otherwise, values could be forgotten without being dropped.
There is (contravariant) depth-subtyping, however:
```
SubContLValue:
  b <: a
  ────────────────────────────
  ¬(LV, lv: a) <: ¬(LV, lv: b)
```
the intuition being that a continuation can care only somewhat about an lvalue.
Overall, nothing non-standard here.

Why do I bring this up? Well, while the MIR is mostly safe, enum tagging currently needs a downcast in each branch.
Now the downcast is adjacent to the switch, so it's not that hard to verify, but still it would be nice to have a single node with a safe interface.
Enum variant subtypes have been proposed for a variety of reasons, but fit perfectly here.
Enum switch becomes:
```
Switch:
  (∪ₙ Tₙ) :> T
  ∀i
    TC; S; K  ⊢  kᵢ: ¬(LV, lv: Tᵢ)
  ────────────────────────────────────────────
  TC; S; K  ⊢  switch(lv, t, k*): ¬(LV, lv: T)
```
[On the first line, that's a union not the letter 'U'.] The union isn't me introducing union types (whew!), but just saying that these Tᵢ "cover" t.
The nodes branched from the switch each expect a different variant, and the switch dispatches to the one expecting the variant that's actually there.


## Lifetimes

Niko's recent blog posts cover this very well, so I will build on them.

The "non-lexical" in "non-lexical lifetime" is valid, but leaves out the important detail that such lifetimes *are* lexical with respect to the MIR.
Indeed that this is one of the main motivations for the MIR.
Not only does this make lifetime inference as we currently do simpler, but it also means we can explicitly represent lifetimes.
As I mentioned above, in formalisms like this, I believe everything should be explicit, and so lifetimes will be too.

Lifetimes will be abstract function-global labels, just as lvalues have been defined as abstract function-global labels.
Furthermore, just as lvalues can correspond to local variables or parameters, so lifetimes can exist internal to the function body or be parameters.
Finally, there is one static lifetime, but many static lvalues (less symmetry, oh well).
```
LifetimeLocal, 'l ::= '\'local0' | '\'local1' | ...
LifetimeParam, 'p ::= '\'param0' | '\'param1' | ...
Lifetime,      'a ::= '\'static' | LifetimeLocal | LifetimeParam
```

As a first approximation, continuation types will be extended to include the set of liftimes the node inhabits, hereafter called the "active lifetimes".
```
LifetimeContext, LC ::= Lifetime*
NodeType,        ¬T ::= ¬(LValueContext; LifetimeContext)
```
As lvalues contexts must be proper maps, so lifetime contexts must be proper sets when invoking any judgment.
All node introducers so far will be modified to simply propagate the lifetime context: whatever lifetimes include a node's successors will also include the node itself.
Similarly, there is no continuation subtyping related to "active liftimes", they must match exactly.

Now, everything so far, alone, does render lifetimes worthless, because all nodes would inhabit the same lifetimes!
To remedy this, we'll have dedicated nodes to begin and end lifetimes: their single successor will have one more or less active lifetime than they do.
For now, lets define them as:
```
LifetimeBegin:
  TC; S; K  ⊢  k: ¬(LV; LC, 'l)
  ────────────────────────────────────
  TC; S; K  ⊢  begin('l, k): ¬(LV; LC)
```
```
LifetimeEnd:
  'l ∉ LV
  TC; S; K  ⊢  k: ¬(LV; LC)
  ──────────────────────────────────────
  TC; S; K  ⊢  end('l, k): ¬(LV; LC, 'l)
```
The additional premise, that `'l` is not in any (current) type of any location, should ensure that values do not outlive their lifetime.
It would be nicer to use a "strict-outlives" relation, but we don't have that.
This is a bit unduly restrictive if we choose to support contravariant lifetimes, but either way it is sound.
Finally, recall that `'l` is a meta-variable for local lifetimes. Parameter lifetimes and `'static` cannot be begun or ended.

I should remark on the design principles that led me to this formalism.
Niko's [third blog post](http://smallcultfollowing.com/babysteps/blog/2016/05/09/non-lexical-lifetimes-adding-the-outlives-relation/) goes over the limitations of single-entry-multiple-exit lifetimes defined with dominators, and the series instead opts to define lifetimes as subsets of CFG nodes which satisfy some conditions.
He also pointed out that lifetimes could be thought of as "sets of paths" through a graph.
I like the imagining lifetimes as "sets of paths", and that leads me to believe we should focus on the path's endpoints more than their interiors.
A set of nodes focuses on active lifetimes and leaves the lifetime boundary implicit, but having explicit start/end nodes does the opposite.
Also, while one could make variants of nodes that introduce lifetimes so we need not introduce extra "no-op" begin/end nodes, that would lead to either explosion of rules, or more complicated rules.

Niko's [second blog post](http://smallcultfollowing.com/babysteps/blog/2016/05/04/non-lexical-lifetimes-based-on-liveness/) on non-lexical lifetimes proposes that instead of requiring the type of a binding to outlive the *scope* of the binding, we should merely require that it outlives the *live-range* of the binding.
In my formulation, that is a particularly natural strategy.
Consider that only during the lifetime in which a variable is live will its lvalue have that (initialized) type.
Its last use is in fact its destruction, after which the lvalue's type is `Uninit<_>`.
To me, this signifies that the "moral equivalent" of a scope for this IR is in fact exactly this "live-range" lifetime.
While it is possible to derive lifetimes based on scopes in the surface language, they are fairly meaningless.

Niko's third blog post also goes over the need for an "outlives-at" relation.
The basic idea is that we often (always?) only care which lifetime ends later, and don't care which began earlier.
More on that for a bit, but for now I'll explain how this can justify introducing references within an already active lifetime.
Obviously the newly-introduced reference didn't exists from the beginning of the lifetime, but as long as it is destroyed before the lifetime ends, the lifetime *outlives* the reference *at* the moment of introduction.
Because lifetimes are based on scopes today, we are already allow the bounding lifetime of a reference to extend past the reference's last use, and thus can be confident that a dedicated sort of node just to end lifetimes is sufficiently expressive.
If allowing the creation of references in already active lifetimes is indeed sound, then a dedicated sort of node to begin lifetimes works too.

### Outliving

Now, let us talk about the outlives relationship in detail.
I tried to think to think of a situation where the new "outlives-at" relation wouldn't suffice, and the old "outlives" relation was needed, but I failed.
The rules I came up with, keep the "at" implicit however, so our `x: y` syntax will remain the same.

As before, a lifetime can bound another lifetime or a type.
Lifetime bounds are bundled in a *bound context* [running out of names, I am].
```
LifetimeBound, lb ::= Lifetime ':' Lifetime
                   |  Type     ':' Lifetime
BoundContext,  BC ::= LifetimeBound*
```

Recall the outlives relation defined in [RFC 1214](https://github.com/rust-lang/rfcs/blob/master/text/1214-projections-lifetimes-and-wf.md).
It doesn't concern itself with surface language terms or the CFG, but simply the derivation of outlive bounds from other outlive bounds.
We will use it here (with the slight exception of a different rule for unique references, as those will be defined differently when they are introduced in a later section).
Where the judgements refer "the environment", we will make that precise by using the bound context.
Thus, prepend all judgements with `BC; ` in the original rules, like
```
BC; R ⊢ 'foo: 'bar
```
The reasoning behind keeping that "at" part implicit is that when working with outlives-at judgements that all share the same "at", I believe one can ignore the position altogether.
If that is not the case, RF 1214's outlives will indeed need to be further modified.

Unfortunately, we must modify continuation types again, giving them---you guessed it---a bound context.
```
NodeType, ¬T ::= ¬(LValueContext; LifetimeContext; BoundContext)
```
Again, all existing rules will blindly propagate the bound context to their successors.
But, this time there is a new subtyping rule:
```
SubContOblig:
  ∀lb ∈ OB₁.  OB₀; <>  ⊢ lb
  ────────────────────────────────
  ¬(LV; LC; OB₁) <: ¬(LV; LC; OB₀)
```
Note the the 0 and 1 subscripts are reversed from what one might expect.
Instead of imaging the bounds as *proofs our continuations require*, imagine them as
*obligations our continuations discharge*.
So, given what (all) successors are obligated to carry out, one can deduce bounds using RFC 1412's outlives, and give those new bounds as obligations the current node can carry out.

Why all this?
Consider that there is no evidence in the "past" or "present" with which to prove the outlives-at relation.
The best one can do is charge all successors to witness `'b` dying no later than `'a` for `'a: 'b`.
Consider also that if we had stayed with the original outlives relation, each node would both obligate its successors and require a proof from its predecessors.

Hopefully it is clear now that we will need to modify `LifetimeEnd` to discharge these obligations:
```
LifetimeEnd:
  'l ∉ LV
  TC; S; K  ⊢  k: ¬(LV; LC; BC)
  ───────────────────────────────────────────────────────────────
  TC; S; K  ⊢  end('l, k): ¬(LV; LC, 'l; BC, {'l: 'a | 'a ∈ LC })
```
The set-builder notation says we discharge obligations for each lifetime in `LC`.
(Of course, it should be fine if the node's predecessors doesn't care about every lifetime in `LC` being outlived. The subtyping rule takes care of that.)

We can now redefine `Call`, `Fn`, and `FnGeneric`.
`Call` has three additional jobs.
First, it needs to provide lifetime arguments drawn from the set of active lifetimes.
Second, it needs to substitute those as well for lifetime parameters in the argument types.
Third, it needs to propagate obligations to satisfy the lifetime bounds from the where clause.
```
Call:
  TC ⊢ trb*
  OB ⊢ lb*
  T₀ = for<TP*, 'p*> fn(T₁..Tₙ) -> Tᵣ where trb*, lb*
  ∀i.
    TC; S, LVᵢ;  S, LVᵢ₊₁  ⊢ oᵢ : Tᵢ [Tₜₐ/TP]* ['a/'p]*
  TC; S;  K  ⊢  k: ¬(LVₙ₊₁, lv: Tᵣ; LC; OB)
  ───────────────────────────────────────────────────────────────────────
  TC; S;  K  ⊢  call<Tₜₐ*, 'a*>(lv, o*, k): ¬(LV₀, lv: Uninit<_>; LC; OB)
```
[I switched from `TC, trb* ⊢ ...` to making `TC ⊢ trb*` a separate premise just for legibility.]

`Fn`, not `FnGeneric` is responsible for lifetime parameters and lifetime bounds.
lifetime parameters and `'static` become active lifetimes for `enter` and `exit`.
`exit` also satisfies obligations for all lifetime bounds, allowing the bounds to be propagated backwards into the rest of the CFG.
```
Fn:
  k₀ = entry
  T₀ = ¬((s: Tₛ)*, (a: Tₚ)*, (l: Uninit<_>)*, ret_slot: Uninit<_>; 'static, 'p*)
  ∀i.
    TC; # trait impls
    S;  # statics
    l*; # locals (just the location names, no types)
    K,  # user labels, K = { kₙ: ¬Tₙ | n }
      exit:  ¬((s: Tₛ)*, (a: Uninit<_>)*, (l: Uninit<_>)*, ret_slot: Tᵣ; 'static, 'p*; BC);
    ⊢ eᵢ: ¬Tᵢ
  ─────────────────────────────────────────────────────────────────────────────────────────
  TC; S  ⊢  Mir { params, locals, labels: { (k: ¬T = e)* }, .. }:
            for<'p*> fn(Tₚ*) -> Tᵣ where BC
```

`FnGeneric` is basically the same but prepends type parameters to lifetime parameters and prepends trait bounds to lifetime bounds:
```
FnGeneric:
  TC, TP*, trb*; S  ⊢  f: for<'p*> fn(Tₚ*) -> Tᵣ where BC
  ────────────────────────────────────────────────────────────────────
  TC; S ⊢ (Λ<TP*, trb*> f): for<TP*, 'p*> fn(Tₚ*) -> Tᵣ where trb*, BC
```

I should note that the outlives-at relation defined this way has some perhaps surprising consequences.
Consider a CFG of a single loop with two lifetimes twice overlapping.
Cut and unrolled, the loop looks like:
```
   'a ┌─────────────────────┐
┅━━━━━┷━━━━┯═══════════╤════╧━━━━━┅
┄──────────┘        'b └──────────┄

  → CFG edge direction (time)
  ━ 'a: 'b
  ═ 'b: 'a
```
the imaginary cut is at the dotted lines, and the brackets labeled `'a` and `'b` denote each lifetime.
Moving from left to right, after `'b` ends and until `'a` ends, `'a` can derive that `'b: 'a`, because `'b` is alive at `'a`'s ending.
But during the other half of the CFG loop, the opposite can be derived!
This shows that no precaution is taken against lifetimes resurrecting after they are presumed dead.
This might seem highly dangerous, but note that when a lifetime dies, nothing associated with it can still exist because no lvalue can include it in its type.
Thus, no pointer invalidation can occur.

To end the section, I want to close with Niko's most unruly example from the third blog.
In Rust, it looks like:

> ```rust
  let mut map1 = HashMap::new();
  let mut map2 = HashMap::new();
  let key = ...;
  let map_ref1 = &mut map1;
  let v1 = map_ref1.get_mut(&key);
  let v0;
  if some_condition {
      v0 = v1.unwrap();
  } else {
      let map_ref2 = &mut map2;
      let v2 = map_ref2.get_mut(&key);
      v0 = v2.unwrap();
      map1.insert(...);
  }
  use(v0);
> ```

As our CFG it looks like:

```
            A [ map1 = HashMap::new()       ]
            1 [ map2 = HashMap::new()       ]
            2 [ key: K = ...                ]
            3 [ begin('x)                   ]
            3 [ map_ref1 = &mut map1        ]
            4 [ v1 = map_ref1.get_mut(&key) ]
            5 [ if some_condition           ]
                      |               |
                     true           false
                      |               |
                      v               v
  B [ v0 = v1.unwrap() ]   C [ destroy(v0)                 ]
  1 [ goto             ]   1 [ end('x)                     ]
                      |    2 [ begin('x)                   ]
                      |    3 [ map_ref2 = &mut map2        ]
                      |    4 [ v2 = map_ref2.get_mut(&key) ]
                      |    5 [ begin('x)                   ]
                      |    6 [ v0 = v2.unwrap()            ]
                      |    7 [ map1.insert(...)            ]
                      |    8 [ goto                        ]
                      |               |
                      v               v
                    D [ use(v0)       ]
                    1 [ end('x)       ]
```

As is shown, we can build this with a single lifetime!
Start before `A3`, to include `v1`.
Give `V0`'s ref the same lifetime on both branches.
On the right branch, end and begin again before `C3`; as the next section will demonstrate, this "clears" the borrow on `map1`.
Finally, begin again before assigning `v0` so that it can be given the lifetime as prescribed above.
Note that `B1` and `C8` have different sets of lvalues borrowed before the merge at `D0`; that's fine.
One can imagine borrowing and then immediately throwing away the reference, but the location must stay borrowed until the lifetime the referenced was associated with ends.
Thus it fine to skip the intermediate step, and allow one to coerce lvalues to their borrowed equivalents.


## Unique References (Generalizing `&mut`)

Lvalue typestate might be the great enabler of what everything else proposed in this document, but is somewhat useless on its own.
With references as they exist today, all uninitialized lvalues correspond to locals, and the one thing one can do with them---write to them---is already supported.
As to changing the type of locals, one can simply use more locals and hope an optimization pass will figure out how to reuse stack slots.

But consider how we might generalize unique references.
In brief, references provide access to locations, so one can convert a reference to an lvalue (deref), or an lvalue to a reference (ref).
That much will stay the same between the existing MIR and this.
With unique references in particular, for every reference created, one preexisting lvalue is made unusable, so there is never more than one way to access "real" location.
Thus, in some sense, a unique reference is as good as "direct" access to the original lvalue.
It would thus be nice if one could do everything via a unique reference that can be done with a "top-level" lvalue (i.e. one *not* from a deref).
With first class initializedness, that "everything" boils down to changing the type of references' contents.

A first approximation of this might keep the existing `&mut T` type, and simply change a reference's type parameter along with its backing lvalue on writes and move-outs.
There a two problems with this, however.
First, when a function takes some unique reference arguments, the backing lvalues aren't in scope, so there's no way to change them.
The caller would have no idea what the callee is doing with the references.
Second, because of control flow merges, it may not be known which lvalue a reference points to, and thus which lvalues' types to update.
For example, see `D0` in the CFG at the end of the lifetimes section.
`v0` may point in either `map1` or `map2`, there's no way of knowing!

The solution is to give unique references *two* type parameters.
I will use the syntax `&mut<'a, Tᵢ, tₒ>`.
The first parameter is the current type or input type, and represents what's currently in the borrowed location.
The second is the residule type or output type, and represents what must be there when the reference is dropped.
This solves the first problem because functions' types now state the condition they will leave any borrowed locations in.
This also solves the second problem because when we mark a location borrowed, we can simultaneously set it to the newly-created reference's residule type.
Doing so ensures that on the branch that the location *wasn't* mutated via the reference, it already has the type it would have had it been.

The subtyping rule is as follows:
```
SubUniqRef:
  'a₀ <: 'a₁
  Tᵢ₀ <: Tᵢ₁
  ──────────────────────────────────────────
  &mut<'a₀, Tᵢ₀, tₒ> <: &mut<'a₁, Tᵢ₁, tₒ>
```
Unique references are covariant with respect to the first parameter because it affects the first *read*.
They would be contravariant with respect to the the second parameter because it affects the last *write*, but since we don't support contravariance they are invariant instead.

The well-formedness (from RFC 1214) rule is as follows:
```
WfUniqRef:
  R ⊢ Tᵢ WF
  R ⊢ Tₒ WF
  ───────────────────────
  R ⊢ &mut<'a, Tᵢ, Tₒ> WF
```
Note that neither type argument need outlive the lifetime of the reference!
This is a fairly subtle point that deserves some remark.
The lifetime of the unique reference is the lifetime that the *location* is borrowed; the *contents* of that location is fully owned by the reference.
It could well be that the contents cannot outlive a different lifetime that ends earlier,
and thus must be destroyed first.
That's no problem, because as was already stated, the unique reference allows one change the type of the contents just as one can do with a top-level lvalue.
The residule type must outlive the references lifetime *at* the moment when the reference is dropped, but symmetrically that type may not be inhabited when the reference was created.

The outlives (again from RFC 1214) is:
```
OutlivesUniqRef:
  R ⊢ 'a₀: 'a₁
  R ⊢ Tᵢ:  'a₁
  R ⊢ Tₒ:  'a₁
  ──────────────────────────
  R ⊢ &mut<'a₀, Tᵢ, Tₒ>: 'a₁
```
Requiring `Tᵢ: 'a₁` is easy to decide on as it imposes no onerous restriction.
If the reference is left containing `Tᵢ` after `Tᵢ` is no longer alive, and the lifetime(s) in which `Tᵢ` is alive does not begin again until after the reference must be dropped (if it begins again at all), the reference is incapable of being dropped.
`Tₒ: 'a₁` while somewhat conservative, follows from making the output type parameter invariant instead of contravariant.
It also matches the outlive rules for traits objects and function types.

To ensure that the residule type does indeed reflect the location pointed to by the reference before it is dropped, we only allow dropping unique references when the current type matches the residule type.
```
DropUniqRef:
  TC, T; S;  K  ⊢  k: ¬(LV, lv: Uninit<_>; LC; BC)
  ───────────────────────────────────────────────────────────────
  TC, T; S;  K  ⊢  drop(lv, k): ¬(LV, lv: &mut<'a, T, T>; LC; BC)
```
Notice that both type arguments are `T`.

Besides unique reference types themselves, there are two more constructs that must be introduced: a borrowed type, and projections.
Uniquely borrowing a location in Rust prevents all interaction with that location except through the reference.
Currently we prevent access to the borrowed lvalue through more static analysis, just as we prevent access to uninitialized lvalues.
And just as I instead opted for an uninitialized type family, so I will opt for a borrowed type family: `BorrowedMut<'a, T>`.
`BorrowedMut<'a, T>` is a simple newtype around `T`, "protecting" `T` during `'a`.
`'a` would be contravariant and so is invariant.
```
SubBorrowedMut:
  T₀ <: T₁
  ────────────────────────────────────────
  Borrowed<'a, T₀> <: Borrowed<'a, T₁>
```
The reason for the would-be contravariance is simple, while a `&mut<'a, _, T>` must be destroyed before `'a` ends, it's associated `BorrowMut<'a, T>` must turn into back `T` after `'a` ends, to guarantee that aliasing is prevented.

Because it turns back to `T` after, `BorrowMut<'a, T>`'a the well-formedness rule has the addition restriction that `T: 'a`.
```
WfBorrowedMut:
  R ⊢ T WF
  R ⊢ T: 'a
  ─────────────────────────
  R ⊢ BorrowedMut<'a, T> WF
```
[Just like with `LifetimeEnd`, it might be nice to use a strict outlives relation here, but we don't have it at our disposal.
Thankfully not using strict outlives won't create any soundness problems but rather simply delay the catching of errors.]

Outlives for `BorrowedMut` retains a shred of contravariance in that it outlives its parameter (and must do so far reasons explained above).
Hopefully this doesn't cause any problems.
```
OutlivesBorrowedMut:
  R ⊢ T: 'a₁
  ─────────────────────────
  R ⊢ BorrowedMut<'a₀, T>: 'a₁
```

Additionally, any unborrowed lvalue is a subtype of its borrowed equivalent.
This is needed for control flow merges as I talked about above; the shared successor needs to conservatively prohibit access to potentially-borrowed locations.
(In the example but `map1` and `map2` would need to be borrowed at `D0`.)
```
SubBorrowUniquely:
  ──────────────────────────────────────
  T <: BorrowedMut<'a, T₁>
```
Note, I think It should be possible to achieve this with an "lvalue cast" instead of subtyping if that is desired.

The crucial attribute of `BorrowedMut<T>` is any lvalue of it cannot be inspected in any way.
There has been some talk of non-movable types, so I will presume the existence of a `Move` trait whose sole purpose is to gate the use the consume operand.
While we're at it, one might as also make `Uninit<_>: !Move` so the `uninit` constant can be dispensed with.
If the new trait sounds like mission creep, it is perfectly possible to also just dispense with the `Move` trait and special-case prohibit `BorrowedMut` (and `Uninit` too) from being consumed.

Now we have enough in place for unique reference introduction.
References are introduced as an rvalue, with the same grammar as today.
```
BorrowKind ::= 'unique' | 'aliased'
RValue, rv ::= 'use'(Operand)
            |  'unop'(Unop, Operand, Operand)
            |  'binop'(Binop, Operand, Operand)
            |  'ref'(Lifetime, BorrowKind, LValue)
```
Because we can only introduce references for lifetimes that are active, we will need to additionally route the lifetime context to all rvalue introducers.
(This is done for the others in the appendix.)
```
RefUnique:
  ───────────────────────────────────
  TC;
    LV, lv: Tᵢ;
    LV, lv: BorrowedMut<'a, Tₒ>;
    LC, 'a
    ⊢ ref('a, unique, lv): &mut<'a, Tᵢ, Borrowed<'a, Tₒ>>
```
As I said earlier, we change the type of the lvalue to the residule type, ahead of a value of that type actually being written to the reference.
That is manifested as `lv: BorrowedMut<'a, T₀>` in the output lvalue context.
One thing I didn't mention, and I didn't initially expect, is that the residule type would *also* be borrowed.
The reason for this is that when reborrowing an indeterminate number of time (e.g. when descending through a tree), it is impossible to keep all intermediate references around, for that would take an indeterminate number of lvalues.
As the deref rule will make far clearer, this allows intermediate references to be dropped.
Note finally that because one can borrow an lvalue without taking a reference, this doesn't constrain consumers who don't reborrow the reference.

And the last prerequisite, projections. Projections are rustc's umbrella concept for getting lvalues from lvalues.
Examples include (primitive) array indexing, fields access, and deref.
```
Projection, prj ::= 'deref'
LValue,     lv  ::= 'ret_slot' | Static | Local | Param
                 |  Projection(LValue)
```
Deref is all we need concern ourselves with for now.

Ok, finally everything is in place to introduce the deref rule.
This rule is at the heart of what references do, and perhaps take the least expected form.
Considering that projections extract lvalues from lvalues, one might think the deref rule would look like:
```
LV ⊢ lv: &<'a, Tᵢ, Tₒ>
──────────────────────
LV ⊢ deref(lv) : Tᵢ
```
or similarly:
```
──────────────────────────────────────
LV, lv: &<'a, Tᵢ, Tₒ> ⊢ deref(lv) : Tᵢ
```
This, however, breaks because node introducers themselves (e.g. assign) "change" the lvalue context (i.e. nodes' successors often take a different lvalue context than the nodes themselves).
Those changes need to be propagated back to types of the references being dereferenced.
Instead we will use this rule:
```
WithDeref:
  TC, T; S;
    {  k:  ¬(LVᵢ, deref(lv): Tᵢ; LCᵢ; BCᵢ) | i ≥ 1 }
    ⊢  k': ¬(LV₀, deref(lv): T₀; LC₀; BC₀)
  ─────────────────────────────────────────────────────────────────────
  TC, T; S;
    K, {          k:   ¬(LVᵢ, lv: &mut<'a, Tᵢ, Tᵣ>; LCᵢ; BCᵢ) | i ≥ 1 }
    ⊢  with_deref(k)': ¬(LV₀, lv: &mut<'a, T₀, Tᵣ>; LC₀; BC₀)
```
This convoluted rule allows one to build a node where it and its successors just see the dereferenced lvalue instead of the reference itself.
Then, provided we can build real successors which see a reference instead, this node too can be built with the reference instead.
The type of the deref lvalue becomes the current type, and the type of the residule type can be anything but must stay the same.
Let's put this into context with assign.
Say we want to move out of a unique reference.
The premised assign node expects `deref(lv): T`, and its single successor expects `deref(lv): Uninit<_>`.
Then the concluded `with_deref(assign(..))` node expects `lv: &mut<'a, T, Whatever>`, and its successor expects `lv: &mut<'a, Uninit<_>, Whatever>`.
Thus, we've moved out of a reference!

Another crucial example is reborrowing.
First, its instructive to see how the current and residule types of the references are "threaded together".
The new references's (initial) current type is the old references (previous) current type, and the new reference's residule type becomes the old reference's (new) current type.
Pictorially this looks like:
```
('a: 'b)
    ┌───────────────────────────────────────────────────────────┐
    │&mut<'b, T₀,                               BorrowedMut<T₁>>│
    └──────────┬────────────────────────────────────┬───────────┘
               ↑                                    ↓
┌──────────────┴──────────────────┐   ┌─────────────┴─────────────────────────────┐
│&mut<'a,     T₀, BorrowedMut<T₂>>├ → ┤&mut<'a, BorrowedMut<T₁>, BorrowedMut<<T₂>>│
└─────────────────────────────────┘   └───────────────────────────────────────────┘
```
On top is the new reference created from the reborrow.
On bottom is the old reference, before and after the reborrow.
See now also what I crudely described before when introducing the unique reference dropping rule.
If `T₁ = T₂`, then the old reference can be dropped before the new reference, allowing an indefinite chain of reborrowing with no more than 2 references alive at a time.

Second, it is good to be aware of the increase in capabilities on reborrowed references between the status quo and this proposal.
If the dereferenced lvalue is borrowed and assigned, the successor's reborrowed reference  (not the newly created reference!) will look like:
```
&mut<'old, BorrowedMut<'new, T>, T>
```
whereas today, one would end up with the moral equivalent of
```
BorrowedMut<&mut<'a, T>, T>>
```
this is the difference between (on top, my plan) a reference where the *referenced location* is borrowed, and (on bottom, status quo) a borrowed location holding a reference.
The latter can only be dropped, but the former can be moved around too.
I don't have a complete plan, but this opens the door to storing a reborrowed reference and the reference that reborrows it together in the same struct.
This is related to the [first wishlist item](https://github.com/tomaka/vulkano/blob/master/TROUBLES.md) of Tomaka's Vulkano library.
[I do have a plan for making Box and other owning pointers unique reference newtypes.
I will present it a few sections later.]

### Other proposals and the surface language

Finally, before I end this section, a few words on how this relates to other proposals.
`&mut`, `&move`, and `&out` are but special case of `&mut<_, _, _>`:
```
&'a mut  T = &mut<'a, T, T>
&'a move T = &mut<'a, T, Uninit<_>>
&'a out  T = &mut<'a, Uninit<_>, T>
```
The subtyping rule for `&mut<_, _, _>` likewise imply the subtyping rules for the three.
Because unique references' current types parameter is covariant, `&move`'s parameter is also covariant.
Because unique references' residule type parameter is invariant (or contravariant), `&out`'s parameter is invariant (or contravariant).
Invariance overrides the others (or covariance and contravariance cancel out), so `&mut`'s parameter is invariant.
Additionally, there has been some interest in relaxing the well-formedness restriction on `&mut`.
While I do not know if this is backwards compatible, this proposal would indicate that it can be done soundly.
I should finally note that the chief criticism of these proposals is the worry that by introducing more primitive reference types, we'll open the floodgates and end up with an overflowing menagerie of confusing and non-orthogonal primitive pointer types.
Well, I am very please to report that with this proposal there are and will be no new pointer types, as `&mut T` becomes but a synonym for its generalization.

Thanks to default type parameters, we can backwards-compatibly extend `DerefMut` to support any unique reference:
```rust
pub trait DerefMut<ResiduleSelf = Self>: Deref {
    type ResiduleTarget = Self::Target;
    fn deref_mut(self: &mut<Self, ResiduleSelf>) -> &mut<Self::Target, ResiduleTarget>;
}
```
This removes the need for any `DerefMove` trait or similar.


## Appendix: Grammar and Rules in Full

All the extension and modifications of each section, squashed together.

### Grammar

```
anything* ::= | anything ',' anything*
```

```
Static,     s   ::= 'static0' | 'static1' | ...
Local,      l   ::= 'local0'  | 'local1'  | ...
Param,      p   ::= 'param0'  | 'param1'  | ...
Projection, prj ::= 'deref'
LValue,     lv  ::= 'ret_slot' | Static | Local | Param
                 |  Projection(LValue)
```
```
LifetimeLocal, 'l ::= '\'local0' | '\'local1' | ...
LifetimeParam, 'p ::= '\'param0' | '\'param1' | ...
Lifetime,      'a ::= '\'static' | LifetimeLocal | LifetimeParam
```
```
Size,        n  ::= '0b' | '1b' | '2b' | ...
TypeBuiltin     ::= 'Uninit'<Size> | !
                 |  &'mut'<Lifetime, Type, Type> | 'BorrowedMut'<Type>
                 |  ...
TypeParam,   TP ::= 'TParam0' | 'TParam1' | ...
TypeUserDef     ::= 'User0'   | 'User1'   | ...
Type,        T  ::= TypeBuiltin | TypeParam | TypeUserDef
```
```
Constant, c  ::= 'BorrowedMut'::<Type>(Constant) | ...
Operand,  o  ::= 'const'(Constant)
              |  'consume'(LValue)
Unop,     u  ::= ...
Binop,    b  ::= ...
BorrowKind   ::= 'unique' | 'aliased'
RValue,   rv ::= 'use'(Operand)
              |  'unop'(Unop, Operand, Operand)
              |  'binop'(Binop, Operand, Operand)
              |  'ref'(Lifetime, BorrowKind, LValue)
```
```
LValueContext, LV ::= (LValue: Type)*
StaticContext, S  ::= (Static: Type)*
```
```
Trait,       tr  ::= # some globally-unique identifier
TraitBound,  trb ::= Type ':' Trait
TypePremise      ::= TraitBound | TypeParam | TypeUserDef
TypeContext, TC  ::= TypePremise*
```
```
LifetimeBound       ::= Lifetime ':' Lifetime
                     |  Type     ':' Lifetime
BoundContext,  BC ::= LifetimeBound*
```
```
Label,           k  ::= 'enter' | 'exit'
                     |  'k0' | 'k1' | ...
Node,            e  ::= 'Assign'(LValue, Operand, Label)
                     |  'DropCopy'(LValue, Label)
                     |  'If'(Operand, Label, Label)
                     |  ... # et cetera
LifetimeContext, LC ::= Lifetime*
NodeType,        ¬T ::= ¬(LValueContext; LifetimeContext; BoundContext)
CfgContext,      K  ::= (Label : NodeType)*
```

### Rules

#### Operand Introduction Rules
```
Const:
  TC  ⊢  c: T
  ────────────────────────────
  TC;  LV;  LV  ⊢  const(c): T
```
```
MoveConsume:
  ──────────────────────────────────────────────────────────────────────
  TC, T: Move + !Copy;  LV, lv: T;  LV, lv: Uninit<_>  ⊢  consume(lv): T
```
```
CopyConsume:
  ──────────────────────────────────────────────────────
  TC, T: Copy;  LV, lv: T;  LV, lv: T  ⊢  consume(lv): T
```

#### RValue Introduction Rules
```
Use:
  TC; LV₀; LV₁  ⊢  o: T
  ──────────────────────────
  TC; LV₀; LV₁; LC  ⊢  use(o): T
```
```
UnOp:
  TC; LV₀; LV₁  ⊢  o: T
  u: fn(T) -> Tᵣ        # primops need no context
  ──────────────────────────────
  TC; LV₀; LV₁; LC  ⊢  use(u, o): Tᵣ
```
```
BinOp:
  TC; LV₀; LV₁  ⊢  o₀: T₀
  TC; LV₁; LV₂  ⊢  o₁: T₁
  b: fn(T₀, T₁) ->        # primops need no context
  ───────────────────────────────────
  TC; LV₀; LV₂; LC  ⊢  use(b, o₀, o₁): Tᵣ
```
```
RefUnique:
  ───────────────────────────────────
  TC;
    LV, lv: Tᵢ;
    LV, lv: BorrowedMut<'a, Tₒ>;
    LC, 'a
    ⊢ ref('a, unique, lv): &mut<'a, Tᵢ, Borrowed<'a, Tₒ>>
```

#### Node/Continuation Introduction Rules
```
Assign:
  TC;  S, LV, lv: Uninit<_>;  S, LV, lv: Uninit<_>;  LC  ⊢  rv: T
  TC; S; K  ⊢  k: ¬(LV, lv: T; LC; BC)
  ─────────────────────────────────────────────────────────────────
  TC; S; K  ⊢  assign(lv, rv, k): ¬(LV, lv: Uninit<_>; LC; BC)
```
```
Call:
  TC ⊢ trb*
  OB ⊢ lb*
  T₀ = for<TP*, 'p*> fn(T₁..Tₙ) -> Tᵣ where trb*, lb*
  ∀i.
    TC; S, LVᵢ;  S, LVᵢ₊₁  ⊢ oᵢ : Tᵢ [Tₜₐ/TP]* ['a/'p]*
  TC; S;  K  ⊢  k: ¬(LVₙ₊₁, lv: Tᵣ; LC; OB)
  ───────────────────────────────────────────────────────────────────────
  TC; S;  K  ⊢  call<Tₜₐ*, 'a*>(lv, o*, k): ¬(LV₀, lv: Uninit<_>; LC; OB)
```
```
Unreachable:
  ──────────────────────────────────────────────
  TC; S;  K  ⊢  unreachable: ¬(LV₀, lv: !; LC; BC)
```
```
CopyDrop:
  TC, T: Copy; S;  K  ⊢  k: ¬(LV, lv: Uninit<_>; LC; BC)
  ────────────────────────────────────────────────────────
  TC, T: Copy; S;  K  ⊢  drop(lv, k): ¬(LV, lv: T; LC; BC)
```
```
If:
  TC; S, LV₀;  S, LV₁  ⊢  o: T
  TC; S; K  ⊢  k₀: ¬(LV₁; LC; BC)
  TC; S; K  ⊢  k₁: ¬(LV₁; LC; BC)
  ──────────────────────────────────────────
  TC; S; K  ⊢  if(o, k₀, k₁): ¬(LV₀; LC; BC)
```
```
Switch:
  (∪ₙ Tₙ) :> T
  ∀i
    TC; S; K  ⊢  kᵢ: ¬(LV, lv: Tᵢ; LC; BC)
  ────────────────────────────────────────────────────
  TC; S; K  ⊢  switch(lv, t, k*): ¬(LV, lv: T; LC; BC)
```
```
LifetimeBegin:
  TC; S; K  ⊢  k: ¬(LV; LC, 'l; BC)
  ────────────────────────────────────────
  TC; S; K  ⊢  begin('l, k): ¬(LV; LC; BC)
```
```
LifetimeEnd:
  LV₁ = LV₀[T / BorrowedMut<'l, T>] # Unborrow for this lifetime
  'l ∉ LV₁
  TC; S; K  ⊢  k: ¬(LV₁; LC; BC)
  ────────────────────────────────────────────────────────────────
  TC; S; K  ⊢  end('l, k): ¬(LV₀; LC, 'l; BC, {'l: 'a | 'a ∈ LC })
```
```
DropUniqRef:
  TC, T; S;  K  ⊢  k: ¬(LV, lv: Uninit<_>; LC; BC)
  ───────────────────────────────────────────────────────────────
  TC, T; S;  K  ⊢  drop(lv, k): ¬(LV, lv: &mut<'a, T, T>; LC; BC)
```
```
WithDeref:
  TC, T; S;
    {  k:  ¬(LVᵢ, deref(lv): Tᵢ; LCᵢ; BCᵢ) | i ≥ 1 }
    ⊢  k': ¬(LV₀, deref(lv): T₀; LC₀; BC₀)
  ─────────────────────────────────────────────────────────────────────
  TC, T; S;
    K, {          k:   ¬(LVᵢ, lv: &mut<'a, Tᵢ, Tᵣ>; LCᵢ; BCᵢ) | i ≥ 1 }
    ⊢  with_deref(k)': ¬(LV₀, lv: &mut<'a, T₀, Tᵣ>; LC₀; BC₀)
```

#### `Fn` Introduction Rules
```
Fn:
  k₀ = entry
  T₀ = ¬((s: Tₛ)*, (a: Tₚ)*, (l: Uninit<_>)*, ret_slot: Uninit<_>; 'p*, BC)
  ∀i.
    TC; # trait impls
    S;  # statics
    l*; # locals (just the location names, no types)
    K,  # user labels, K = { kₙ: ¬Tₙ | n }
      exit:  ¬((s: Tₛ)*, (a: Uninit<_>)*, (l: Uninit<_>)*, ret_slot: Tᵣ; 'p*, BC);
    ⊢ eᵢ: ¬Tᵢ
  ────────────────────────────────────────────────────────────────────────────────
  TC; S  ⊢  Mir { params, locals, labels: { (k: ¬T = e)* }, .. }:
            for<'p*> fn(Tₚ*) -> Tᵣ where BC
```
```
FnGeneric:
  TC, TP*, trb*; S  ⊢  f: for<'p*> fn(Tₚ*) -> Tᵣ where BC
  ────────────────────────────────────────────────────────────────────
  TC; S ⊢ (Λ<TP*, trb*> f): for<TP*, 'p*> fn(Tₚ*) -> Tᵣ where trb*, BC
```

#### Subtyping
```
SubRefl:
  ──────
  T <: T
```
```
SubTrans:
  T₀ <: T₁
  T₁ <: T₂
  ──────
  T₀ <: T₂
```
```
SubContLValue:
  b <: a
  ────────────────────────────────────────────
  ¬(LV, lv: a; LC; OB) <: ¬(LV, lv: b; LC; OB)
```
```
SubContOblig:
  ∀lb ∈ OB₁.  OB₀; <>  ⊢ lb
  ────────────────────────────────
  ¬(LV; LC; OB₁) <: ¬(LV; LC; OB₀)
```
```
SubUniqRef:
  'a₀ <: 'a₁
  Tᵢ₀ <: Tᵢ₁
  Tₒ₀ :> Tₒ₁ # optional contravariance
  ──────────────────────────────────────────
  &mut<'a₀, Tᵢ₀, tₒ₀> <: &mut<'a₁, Tᵢ₁, tₒ₁>
```
```
SubBorrowedMut:
  'a₀ :> 'a₁ # optional contravariance
  T₀  <: T₁
  ──────────────────────────────────────
  Borrowed<'a₀, T₀> <: Borrowed<'a₁, T₁>
```
```
SubBorrowUniquely:
  ──────────────────────────────────────
  T <: BorrowedMut<'a, T>
```

#### Outlives
Changes from [RFC 1214](https://github.com/rust-lang/rfcs/blob/master/text/1214-projections-lifetimes-and-wf.md).
```
OutlivesUniqRef:
  R ⊢ 'a₀: 'a₁
  R ⊢ Tᵢ:  'a₁
  R ⊢ Tₒ:  'a₁ # only if invariant
  ──────────────────────────
  R ⊢ &mut<'a₀, Tᵢ, Tₒ>: 'a₁
```
```
OutlivesBorrowedMut:
  R ⊢ T: 'a₁
  ─────────────────────────
  R ⊢ BorrowedMut<'a₀, T>: 'a₁
```

#### Well-Formedness
Changes from [RFC 1214](https://github.com/rust-lang/rfcs/blob/master/text/1214-projections-lifetimes-and-wf.md).
```
WfUniqRef:
  R ⊢ Tᵢ WF
  R ⊢ Tₒ WF
  ───────────────────────
  R ⊢ &mut<'a, Tᵢ, Tₒ> WF
```
```
WfBorrowedMut:
  R ⊢ T WF
  R ⊢ T: 'a
  ─────────────────────────
  R ⊢ BorrowedMut<'a, T> WF
```
