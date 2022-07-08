# a MIR Formality, goals and the reality

From [the repository](https://github.com/nikomatsakis/a-mir-formality) (emphasis mine):

> This repository is an early-stage experimental project that aims to be a complete, authoritative formal model of the [Rust MIR](https://rustc-dev-guide.rust-lang.org/mir/index.html). **Presuming these experiments bear fruit, the intention is to bring this model into Rust as an RFC and develop it as an official part of the language definition.**

The goal of this document is to explain the high-level structure as well as giving a sense for the overall development roadmap I have in mind.

[ToC]

## How I see formality being used

Formality will become the "primary source" for "what Rust means":

* When a new language feature is designed, one stage in the process will be formalizing that feature into formality. 
    * Currently, since formality focuses on MIR, this will only make sense for "core language features" like implied bounds or specialization. Eventually I'd like to extend formality to cover other areas like name resolution and Rust surface syntax.
    * This will help to uncover interactions between features.
* Formality can be fuzzed and checked against rustc, much as the [AWS S3 team does to check their code](https://www.amazon.science/publications/using-lightweight-formal-methods-to-validate-a-key-value-storage-node-in-amazon-s3).
    * We can extend rustc to generate formality declarations.
* For the trait solver, formality will map quite closely to its overall structure, which allows us to experiment with new algorithms or structure and see their effects quickly.
    * As an initial example, I plan to prototype a new approach for associated type normalization.

## Why a new repo? Why PLT redex?

I had originally hoped that chalk could serve as a kind of "executable semantics" for Rust. The idea was that the separation between "generating clauses" and "the solver" would allow us to specify how Rust works on a high-enough level that the code is nicely generic. Over time, though, I've become convinced that this approach won't scale. There is a lot of "incidental complexity" that is created by integrating chalk into rustc, as well as engineering for efficiency. Ultimately, I don't see chalk being sufficiently malleable for our purposes.

I chose to use PLT Redex because it's a really great tool for exploration. It allows you to write very high-level structures, execute them, and see what happens. I think it's sufficiently approachable that we can onboard contributors and grow the types team. I'm not sure the same is true of other alternatives. Plus I like it.

## Formality and the types team

I imagine formality being "owned and maintained" by the types team, like other rust-lang projects. 

## Layers of formality

Formality is structured into several layers. These layers are meant to also map fairly closely onto chalk and the eventual Rust trait solver implementation. Ideally, one should be able to map back and forth between formality and the code with ease.

* **formality-ty:** the "types" layer
    * Defines Rust types and functions for equating/relating them.
        * The representation is meant to cover all Rust types, but is optimized for extracting their "essential properties".
    * Defines core logical predicates (`Implemented(T: Trait)`, etc) and solvers.
        * But doesn't define what they *mean* -- i.e., the conditions in which they are true.
* **formality-decl:** the "Rust declarations" layer
    * Defines Rust "top-level items" and their semantics
        * This includes crates, structs, traits, impls, but excludes function bodies.
    * Semantics are defined by converting Rust items into predicates
        * e.g., `impl<T: Eq> Eq for Vec<T>` becomes a "program clause" (axiom) like `forall<T> { Implemented(T: Eq) => HasImpl(Vec<T: Eq>) }` (the distinction between `HasImpl` and `Implemented` is covered below).
    * Defines the well-formedness checks for those items
        * e.g., `struct Foo { f1: T1 }` is well-formed if 
* **formality-mir:** the "type system for MIR" layer
    * Defines MIR and rules for its type checker (corresponds roughly to the MIR borrow checker + polonius).
    * Caveat: I've sketched this out, but nothing more.
* **formality-mir-op:** the "operational semantics for MIR" layer
    * Extends the above level with an operational semantics. Basically equivalent to miri.
    * Caveat: I haven't even sketched this out yet.

## A closer look at formality-ty

Let's take a closer look at the formality-ty layer. 

### Defining Rust types

The current definition of types looks like this ([source](https://github.com/nikomatsakis/a-mir-formality/blob/47eceea34b5f56a55d781acc73dca86c996b15c5/src/ty/grammar.rkt#L25-L37)):

```scheme
(define-language formality-ty
  ...
  (Ty :=
      (TyApply TyName Parameters) ; Application type
      VarId                       ; Bound or existential (inference) variable
      (! VarId)                   ; Universal (placeholder) variable
      )
  (TyName :=
          AdtId           ; enum/struct/union
          TraitId         ; trait
          AssociatedTy    ; Associated type
          ScalarId        ; Something like i32, u32, etc
          (Ref MaybeMut)  ; `&mut` or `&`, expects a lifetime + type parameter
          (Tuple number)  ; tuple of given arity
          )
   ...
   (ScalarId := i8 u8 i16 u16 i32 u32 i64 u64 i128 u128 bool)
   ...
   (AssociatedTy := (TraitId AssociatedTyId))
   ...
   (Parameters := (Parameter ...))
   (Parameter := Ty Lt)
   ...
   ((AdtId VarId TraitId AssociatedTyId AnyId) := variable-not-otherwise-mentioned)
)
```

As you can see, it's woefully incomplete, but it should give you some idea for the level of abstraction we are shooting for and also how PLT Redex works. The idea here is that a type is either a variable, a placeholder that represents a generic type, or an "application" of a type name to some parameters. Let's see some examples:

* A generic type like `T` could either be `T` or `(! T)`:
    * `T` is used when the generic has yet to be substituted, e.g., as part of a declaration.
    * `(! T)` is used as a placeholder to "any type `T`".
* A type like `Vec<T>` in Rust would be represented therefore as:
    * `(TyApply Vec (T))`, in a declaration; or,
    * `(TyApply Vec ((! T)))` when checking it.
* A scalar type like `i32` is represented as `(TyApply i32 ())`.

As I said, this defintion of types is woefully incomplete. I expect it to eventually include:

* "alias types" like associated types and type aliases
* "existential" types like `dyn`
* "forall" quantifies to cover `for<'a> ...`
* "function" types `fn(A1...An) -> R`
* "implication" types `where(...) T`-- these don't exist in Rust yet =)

You can also see that the definition of types is aligned to highlight their "essential" characteristics and not necessarily for convenience elsewhere. Almost every Rust type, for example, boils down to *some* kind of "application" (it's likely that we can even represent `fn` types this way).

### Type unification

A key part of the type layer is that it includes *type unification*. That is, it defines the rules for making types equal. This will eventually have to be extended to cover subtyping (more on that a bit later) so that we can properly handle variance.

Unification is done via a "metafunction", which just means a function that operates on terms (versus a function in the Rust program being analyzed):

```scheme
(define-metafunction formality-ty
  most-general-unifier : Env TermPairs -> EnvSubstitution or Error
```

This function takes an environment `Env` and a list of pairs of terms that should be equated and gives back either:

* a new environment and substitution from inference variables to values that will make the two terms syntactically equal; or,
* the term `Error`, if they cannot be unified.

The unifier is a bit smarter than the traditional unification in that it knows about *universes* and so can handle "forall" proofs and things (that's what is found in the environment). This is the same as chalk and rustc. 

I won't cover the details but I'll just give an example. This is actually modified from a unit test from the code ([source](https://github.com/nikomatsakis/a-mir-formality/blob/47eceea34b5f56a55d781acc73dca86c996b15c5/src/ty/unify.rkt#L254-L269)). Invoking `most-general-unifier` like so:

```scheme
(most-general-unifier Env_2 ((A X)
                             (X (TyApply Vec (Y)))
                             (Y (TyApply i32 ()))))
```

corresponds to saying that `[A = X, X = Vec<Y>, Y = i32]` must all be true, where `A`, `X` and `Y` are inference variables. The resulting output is a substitution that maps `A`, `X` and `Y` to the following values:

* `A -> Vec<i32>`
* `X -> Vec<i32>`
* `Y -> i32`

### Predicates

Formality-ty also defines the core predicates used to define Rust semantics. The current definition is as follows ([source](https://github.com/nikomatsakis/a-mir-formality/blob/47eceea34b5f56a55d781acc73dca86c996b15c5/src/ty/grammar.rkt#L121-L130)):

```scheme
  (Predicate :=
             ; `TraitRef` is (fully) implemented.
             (Implemented TraitRef)
             ; an impl exists for `TraitRef`; this *by itself* doesn't mean
             ; that `TraitRef` is implemented, as the supertraits may not
             ; have impls.
             (HasImpl TraitRef)
             ; the given type or lifetime is well-formed.
             (WellFormed (ParameterKind Parameter))
             )
```

These core predicates are then used to define a richer vocabulary of goals (things that can be proven) and various kinds of "clauses" (things that are assumed to be true, axioms) ([source](https://github.com/nikomatsakis/a-mir-formality/blob/47eceea34b5f56a55d781acc73dca86c996b15c5/src/ty/grammar.rkt#L136-L143)):

```scheme
  (Goals = (Goal ...))
  (Goal :=
        Predicate
        (Equate Term Term)
        (All Goals)
        (Any Goals)
        (Implies Hypotheses Goal)
        (Quantifier KindedVarIds Goal)
        )

  ((Hypotheses Clauses Invariants) := (Clause ...))
  ((Hypothesis Clause Invariant) :=
                                 Predicate
                                 (Implies Goals Predicate)
                                 (ForAll KindedVarIds Clause)
                                 )
```

Importantly, the *types layer* defines a solver that gives semantics to all the "meta" parts of goals and clauses -- e.g., it defines what it means to prove `(All (G1 G2))` (prove both `G1` and `G2`, duh). But it doesn't have any rules for what it means to prove the *core* predicates true -- so it could never prove `(Implemented (Debug ((! T))))`. Those rules all come from the declaration layer and are given to the types layer as part of the "environment".

You might be curious about the distinction between goal and clause and why there are so many names for clauses (hypothesis, clause, invariant, etc). Let's talk briefly about that.

* **Goals vs clauses:** 
    * The role of `ForAll` in goals and clauses is different.
        * Proving $\forall X. G$ requiers proving that `G` is true for any value of `X` (i.e., for a placeholder `(! X)`, in our setup).
        * In contrast, if you know $\forall X. G$ as an axiom, it means that you can give `X` any value `X1` you want.
    * Clauses have a limited structure between that keeps the solver tractable. The idea is that they are always "ways to prove a single predicate" true; we don't allow a clause like `(Any (A B))` as a clause, since that would mean "A or B is true but you don't know which". That would then be a second way to prove an `Any` goal like `(Any ...)` and introduce lots of complications (we got enough already, thanks).
* **Hypotheses vs clauses vs invariants:**
    * These distinctions are used to express and capture implied bounds. We'll defer a detailed analysis until the section below, but briefly:
        * "Hypotheses" are where-clauses that are assumed to be true in this section of the code.
        * "Clauses" are global rules that are always true (derived, e.g., from an impl).
        * "Invariants" express implied bounds (e.g., supertrait relationships like "if `T: Eq`, then `T: PartialEq`").

### Solver

Putting all this together, the types layer currently includes a relatively simple solver called `cosld-solve`. This is referencing the classic [SLD Resolution Algorithm](https://en.wikipedia.org/wiki/SLD_resolution) that powers prolog, although the version of it that we've implemented is extended in two ways:

* It covers Hereditary Harrop predicates using the [techniques described by Gopalan Nadathur](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.107.2510&rep=rep1&type=pdf).
* It is coinductive as [described by Luke Simon et al.](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.102.9618&rep=rep1&type=pdf) -- this means it permits cycles, roughly speaking.

In terms of chalk solvers, it is "similar" to slg, but much simpler in its structure (it doesn't do any caching). 

All those fancy words aside, it's really quite simple. It's defined via induction rules, which PLT Redex lets us write in a natural style. The definition begins like so:

```scheme
(define-judgment-form formality-ty
  #:mode (prove I I I O)
  #:contract (prove Env Predicates_stack Goal EnvSubstitution)
```

This says that we are trying to prove something written as `(prove Env Predciates Goal EnvSubstitution)`, where the first three are 'inputs' and the final name is an 'output' (the input vs output distinction is often left implicit in Prolog and other languages). The idea is that we will prove that `Goal` is true in some environment `Env`; the environment contains our hypotheses and clauses, as well as information about the variables in scope. The `Predicates` list is the stack of things we are solving, it's used to detect cycles. The `EnvSubstitution` is the *output*, it is a modified environment paired with a substitution that potentially gives new values to inference variables found in `Goal`.

Here is a simple rule. It defines the way we prove `Any` ([source](https://github.com/nikomatsakis/a-mir-formality/blob/main/src/ty/cosld-solve/prove.rkt#L62-L65)). The notation is as follows. The stuff "above the line" are the conditions that have to be proven; the thing "under the line" is the conclusion that we can draw.

```scheme
  [(prove Env Predicates_stack Goal_1 EnvSubstitution_out)
   ------------------------------------------------------- "prove-any"
   (prove Env Predicates_stack (Any (Goal_0 ... Goal_1 Goal_2 ...)) EnvSubstitution_out)
   ]
```

This rule says:

* Given some goal `(Any (Goal_0 ... Goal_1 Goal_2 ...))` where `Goal_1` is found somewhere in that list...
    * if we can prove `Goal_1` to be true, then the `Any` goal is true.

Or read another way:

* If we can prove `Goal_1` to be true, then we can prove `(Any Goals)` to be true so long as `Goal_1` is somewhere in `Goals`.

It shows you a bit of the power of PLT Redex (and Racket's pattern matching), as well. We are able to write the rule in a "non-deterministic" way -- saying, "pick any goal from the list" and prove it. Redex will search all the possibilities.

## A closer look at formality-decl

Now that we've surveyed the type layer, let's look at the declaration layer. It begins by declaring an "extended" language `formality-decl` that adds new stuff to `formality-ty` ([source](https://github.com/nikomatsakis/a-mir-formality/blob/47eceea34b5f56a55d781acc73dca86c996b15c5/src/decl/grammar.rkt#L5)):

```scheme
(define-extended-language formality-decl formality-ty ...)
```

For example, a set of crates looks like this ([source](https://github.com/nikomatsakis/a-mir-formality/blob/47eceea34b5f56a55d781acc73dca86c996b15c5/src/decl/grammar.rkt#L7-L10)):

```scheme
 (CrateDecls := (CrateDecl ...))
  (CrateDecl := (CrateId CrateContents))
  (CrateContents := (crate (CrateItemDecl ...)))
  (CrateItemDecl := AdtDecl TraitDecl TraitImplDecl)
```

Basically a list of items like `(a (crate (item1 item2 item3)))` where `item{1,2,3}` are either structs/enums (`AdtDecl`), traits (`TraitDecl`), or impls (`TraitImplDecl`).

### Declaring traits in `FormalityDecl`

Let's look more closely at one of those kinds of items. A trait declaration looks like this ([source](https://github.com/nikomatsakis/a-mir-formality/blob/47eceea34b5f56a55d781acc73dca86c996b15c5/src/decl/grammar.rkt#L26-L27)):

```scheme
  (TraitDecl := (TraitId TraitContents))
  (TraitContents := (trait KindedVarIds WhereClauses TraitItems))
  ...
  (WhereClauses := (WhereClause ...))
  (WhereClause :=
               (ForAll KindedVarIds WhereClause)
               (Implemented TraitRef)
               )
```

Here:

* `TraitId` is the name of the trait
* `KindedVarIds` is a list of generic parameters like `((TyVar Self) (TyVar T))`. Note that the `Self` parameter is made explicit. 
* `WhereClauses` is a list of where-clauses, which are currently just `T: Foo` trait references (though potentially higher-ranked).
* `TraitItems` are the contents of the trait, currently not used for anything.

So the following Rust trait

```rust
trait Foo<T: Ord>: Bar { }
```

would be represented as

```scheme
(Foo (trait ; KindedVarIds -- the generics:
            ((TyKind Self) (TyKind T))
            ; where clauses, including supertraits:
            ((Implemented (Ord (T))) (Implemented (Bar (Self))))
            ; trait items, empty list:
            ()
            ))
```

### Lowering crate items to clauses

The next part of `formality-decl` is the metafunction `env-with-crate-decls` ([source](https://github.com/nikomatsakis/a-mir-formality/blob/47eceea34b5f56a55d781acc73dca86c996b15c5/src/decl/decl-to-clause.rkt#L20-L23)):

```scheme
(define-metafunction formality-decl
  ;; Add the clauses/hypothesis from multiple crates
  ;; into the environment, where CrateId names the current crate.
  env-with-crate-decls : Env CrateDecls CrateId -> Env
  ; ...
  )
```

What this does is to convert *Rust declarations* into the *environment* from the type layer. Note that it takes:

* a base environment `Env` (typically just the constant `EmptyEnv`)
* a set of crates `CrateDecls` (this is meant to include the imports)
* the id of the current crate `CrateId`. This is because the set of rules we generate for a particular item can be different depending on whether we are compiling the crate where it was declared or some other crate (consider e.g. `#[non_exhaustive]`).

### Generating the clauses for a single crate item

The `env-with-crate-decls` function just iterates over all the items in all the crates and ultimately invokes this helper function, `crate-item-decl-rules` ([source](https://github.com/nikomatsakis/a-mir-formality/blob/47eceea34b5f56a55d781acc73dca86c996b15c5/src/decl/decl-to-clause.rkt#L57-L63)):

```scheme
(define-metafunction formality-decl
  ;; Given a crate item, return a tuple of:
  ;;
  ;; * The clauses that hold in all crates due to this item
  ;; * The invariants that hold in all crates due to this item
  ;; * The invariants that hold only in the crate that declared this item
  crate-item-decl-rules : CrateDecls CrateItemDecl -> (Clauses Invariants Invariants)
```

`crate-item-decl-rules` takes 

* the full set of crates `CrateDecls` and
* the declaration of a single item `CrateItemDecl`

and it returns a 3-tuple. As the comment says, this contains both clauses (rules that can be used to derive true facts) along with two sets of invariants. We'll cover the invariants later.

Metafunctions are basically a gigantic match statement. They consist of a series of clauses, each of which begins with a "pattern match" against the arguments. 

To see how it works, let's look at the case for an `impl` from that section. To begin with, here is the comment explaining what we aim to do:

```scheme
 [;; For an trait impl declared in the crate C, like the following:
   ;;
   ;;     impl<'a, T> Foo<'a, T> for i32 where T: Ord { }
   ;;
   ;; We consider `HasImpl` to hold if (a) all inputs are well formed and (b) where
   ;; clauses are satisfied:
   ;;
   ;;     (ForAll ((LtKind 'a) (TyKind T))
   ;;         (HasImpl (Foo (i32 'a u32))) :-
   ;;             (WellFormed (TyKind i32))
   ;;             (WellFormed (TyKind i32))
   ;;             (Implemented (Ord T)))
```

The actual code for this ([source](https://github.com/nikomatsakis/a-mir-formality/blob/47eceea34b5f56a55d781acc73dca86c996b15c5/src/decl/decl-to-clause.rkt#L141-L166)) begins by matching the item that we are generating rules for:

```scheme
   (crate-item-decl-rules CrateDecls (impl KindedVarIds_impl TraitRef WhereClauses_impl ImplItems))
```

The next line is the final result. The function this is part of  -- this says we will produce one global clause

```scheme
   ((Clause) () ())
```

This final result is allowed to refer to variables defined on the following lines, shown below. A `(where/error <pattern> <value>)` clause is effectively just `let <pattern> = <value>`, in Rust terms; the `/error` part means "if this pattern doesn't match, generate an error". You can also write `(where <pattern> <value>)`, but that means that if the pattern doesn't match, the metafunction should to see if the next rule works.

The impl code begins by pattern matching against the `TraitRef` that the impl is implementing (in the example from the comment, that would be `(Foo (i32 'a T))`). We extract out the `TraitId` and the self type `Parameter_trait`:

```scheme
   (where/error (TraitId (Parameter_trait ...)) TraitRef)
```

Next we look up the definition of the trait itself to find out its generics, using a helper function `item-with-id`. This matches and extracts out the parameter kinds (lifetimes vs types, in this code):

```scheme
   (where/error (trait KindedVarIds_trait _ _) (item-with-id CrateDecls TraitId))
   (where/error ((ParameterKind_trait _) ...) KindedVarIds_trait)
```

Next we convert the where clauses (e,g, `T: Ord`) into goals, using a helper function `where-clauses-to-goals` (described below):
```scheme
   (where/error (Goal_wc ...) (where-clauses-to-goals WhereClauses_impl))
```

Finally we can generate that variable `Clause` that was referenced in the final result. Note that we use the `...` notation to "flatten" together the list of goals from the where-clauses (`Goal_wc ...`) and `WellFormed`-ness goals that we must prove (i.e., to use the impl, we must show that its input types are well-formed):

```scheme
   (where/error Clause (ForAll KindedVarIds_impl
                               (Implies
                                ((WellFormed (ParameterKind_trait Parameter_trait)) ...
                                 Goal_wc ...
                                 )
                                (HasImpl TraitRef))))
   ]
```

The `where-clause-to-goal` helper is fairly simple. Here is the source, I'll leave puzzling it out as an exercise to the reader ([source](https://github.com/nikomatsakis/a-mir-formality/blob/47eceea34b5f56a55d781acc73dca86c996b15c5/src/decl/decl-to-clause.rkt#L197-L211)):

```scheme
(define-metafunction formality-decl
  ;; Convert a where clause `W` into a goal that proves `W` is true.
  where-clause-to-goal : WhereClause -> Goal

  ((where-clause-to-goal (Implemented TraitRef))
   (Implemented TraitRef)
   )

  ((where-clause-to-goal (ForAll KindedVarIds WhereClause))
   (ForAll KindedVarIds Goal)
   (where/error Goal (where-clause-to-goal WhereClause))
   )

  ; FIXME: Support lifetimes, projections
  )
```

### Generating the "ok goals" for a crate

In addition to "clauses", there is also a function `crate-ok-goal` that generate goals for each crate item in a given crate ([source](https://github.com/nikomatsakis/a-mir-formality/blob/47eceea34b5f56a55d781acc73dca86c996b15c5/src/decl/decl-ok.rkt#L7-L11)):

```scheme
(define-metafunction formality-decl
  ;; Given a set of crates and the decl for the current crate,
  ;; generate the goal that proves all declarations in the current crate are
  ;; "ok". Other crates are assumed to be "ok".
  crate-ok-goal : CrateDecls CrateDecl -> Goal
```

The idea is that the crate is "ok" (i.e., passes the type check) if these goals are satisfied. For the declarations layer, these goals correspond roughly to rustc's `wfcheck`. Here is the rule for impls ([source](https://github.com/nikomatsakis/a-mir-formality/blob/47eceea34b5f56a55d781acc73dca86c996b15c5/src/decl/decl-ok.rkt#L59-L71)):

```scheme
  [;; For a trait impl declared in the crate C, like the following:
   ;;
   ;;     impl<'a, T> Foo<'a, T> for u32 { }
   ;;
   ;; we require that the trait is implemented.
   (crate-item-ok-goal _ (impl KindedVarIds_impl TraitRef WhereClauses_impl ImplItems))
   (ForAll KindedVarIds_impl
           (Implies ((WellFormed KindedVarId_impl) ... WhereClause_impl ...)
                    (All ((Implemented TraitRef)))))

   (where/error (KindedVarId_impl ...) KindedVarIds_impl)
   (where/error (WhereClause_impl ...) WhereClauses_impl)
   ]
  )
```

In short, an impl is well-formed if the trait is implemented. We'll look at the definition of `Implemented` in more detail in [the next section](#Case-study-Implied-bounds-and-perfect-derive), but for now it suffices to say that a trait is implemented if (a) it has an impl and (b) all of its where-clauses are satisfied. Since we know there is an impl (we're checking it right now!) this is equivalent to saying "all of the trait's where clauses are satisfied".

## Case study: Implied bounds and perfect derive

The current code doesn't really model Rust as it is today. It actually models Rust extended with support for two new features, "implied bounds" and "perfect derive":

* **Implied bounds:** Given `struct Foo<T: Ord>`, can `impl<T> Foo<T> { ... }` just know that `T: Ord`?
    * (We actually have implied bounds today, but they are limited to supertraits (e.g., `T: Eq => T: PartialEq`), so maybe a better way to describe implied bounds would be *expanded implied bounds*.)
* **Perfect derive:** Given `#[derive(Clone)] struct Foo<T> { x: Rc<T> }`, we "just know" that `impl<T> Clone for Foo<T>` works, and that `T: Clone` is not necessary?
    * The idea is that `derive` would generate `impl<T> Clone for Foo<T> where Rc<T>: Clone`. Seems simple, right?
    * The trick is that we have to extend all trait matching to work like auto-traits does, and accept cycles. Consider deriving clone on
        * `struct List<T> { value: Rc<T>, next: Option<Rc<List<T>>> }`
        * here you would get `impl<T> Clone for List<T> where Rc<T>: Clone, Option<Rc<List<T>>>: Clone` -- if you try that today, you'll find it is a cycle error.
    * We are going to refer to this "accept cycles" as coinductive; it's basically the [co-LP formulation by Luke Simon et al.](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.102.9618&rep=rep1&type=pdf) that I referred to earlier.

These two features are a bit tricky to integrate because accepting cycles, if you're not careful, can easily lead you into assuming implied bounds that are not true. The classic example is this:

```rust
trait Copy { }
trait Magic: Copy { }
```

Clearly, given these traits, we know that `T: Magic => T: Copy`, right? But what about if someone writes this rather tautological impl:

```rust
impl<T: Magic> Magic for T { }
```

If we're not careful, we can use this impl to show that every type implements `Magic` -- and yet there is no `impl Copy` anywhere. Something is off!

The solution to that is based on the scheme that [scalexm](https://github.com/scalexm) invented for Chalk; I've tweaked it somewhat by integrating it a bit more deeply into the "core logic", which simplifies the predicates that we need. I find I like this formulation better, and it allows us to simplify a few other things too.

### Distinguish `HasImpl` and `Implemented`

The first key part of the system is to distinguish *having an impl* (`HasImpl`) from *being implemented* (`Implemented`). The former says that the user wrote an impl. The latter says that all the requirements are met to implement the trait, including in particular that all of its where clauses (which includes the supertraits) are satisfied.

Using the code for impls we saw earlier, the `Magic` impl would generate the following clause:

```scheme
; forall<T> { Implemented(T: Magic) => HasImpl(T: Magic) }
(ForAll ((TyKind T)) 
        (Implies ((Implemented (Magic (T))))
                 (HasImpl (Magic (T)))))
```

To actually prove `Implemented(T: Magic)`, This clause has to be combined with the clauses generated by the trait declarations ([source](https://github.com/nikomatsakis/a-mir-formality/blob/47eceea34b5f56a55d781acc73dca86c996b15c5/src/decl/decl-to-clause.rkt#L95-L118)). For the `Copy` trait, which has no where clauses, this clause is very simple. To be implemented, the impl must exist, and the type must be well-formed:

```scheme
; forall<T> { (
;               HasImpl(T: Copy), 
;               WellFormed(T),
;             ) => Implemented(T: Copy) }
(ForAll ((TyKind T))
        (Implies ((HasImpl (Copy (T)))
                  (WellFormed (TyKind (T))))
                 (Implemented (Copy (T)))))
```

For `Magic`, the rule includes the where clause that `T: Copy`:

```schheme
; forall<T> { (
;               HasImpl(T: Magic), 
;               WellFormed(T),
;               Implemented(T: Copy),
;             ) => Implemented(T: Magic) }
(ForAll ((TyKind T))
        (Implies ((HasImpl (Magic (T)))
                  (WellFormed (TyKind (T)))
                  (Implemented (Copy (T))))
                 (Implemented (Magic (T)))))
```

Now we start to see how this works -- if I want to call a function with a `T: Magic` where clause, like this...

```rust
fn make_the_magic_happen<T: Magic>(t: T) {
    let u = t;
    drop((t, u)); // both t, u are valid
}
```

...it is not enough to show that `HasImpl` is satisfied, I also have to prove that `T: Copy`. To do that, I have to show that `HasImpl(T: Copy)`, and I can't do that.

### But wait, implied bounds?

Actually though, the above is not sufficient to solve the problem. That's because we haven't added in implied bounds yet! The *naive* version of implied bounds is that we want to add in a rule like so:

```
Implemented(T: Magic) => Implemented(T: Copy)
```

i.e., if I know that `T: Magic`, I also know that `T: Copy`. But if we literally added that clause, it would be unsound, at least in a coinductive setting. Why is that? Say I want to prove that `String: Copy`...

* `Implemented(String: Copy)`? Well, that's true if...
    * `Implemented(String: Magic)`? Well, that's true if...
        * `HasImpl(String: Magic)`? Well, that's true if...
            * `Implemented(String: Magic)` -- and that's on the stack, so that's ok!
        * `WellFormed(String)` -- yep
        * `Implemented(String: Copy)` -- well, that's on the stack, so that's ok!

The traditional solution so this sort of problem is to impose some kind of limits on the impls people can write so they must be "productive". It's a bit tricky to define what productivity means, but intuitively it means "not tautological". The challenge is that the various schemes I've seen for showing productivity don't accept impls like the ones that perfect derive would create, so they wouldn't really work for us. The solution in the impl works a different way.

The co-LP formulation acccepts any cycle as valid, so it's very easy to create these kind of "tautological rules". Now, if the user actually *wrote* those impls, I don't see that as a problem. It's ok to have mutually dependent impls, all we want to know basically is "when I call a method, there will be some impl to go to" (see example below). But it's not good if it's unsound. =)

```rust
trait Foo {
    fn foo(&self);
}

trait Bar {
    fn bar(&self);
}

impl<T: Bar> Foo for T {
    fn foo(&self) {
        if something() { self.bar() }
    }
}

impl<T: Foo> Bar for T {
    fn bar(&self) {
        if something() { self.foo() }
    }
}
```

### Enter: invariants

The insight is that it's not ok to use implied bounds out of thin air. You only want to use them for where-clauses that you have in scope. In this way, they are categorically different from program clauses, which always hold. I've decided to refer to implied bounds as *invariants* -- the idea is that they are things which "must be true if the program is valid". So for our program we would have one **invariant**:

```
forall<T> { Implemented(T: Magic) => Implemented(T: Copy) }
```

To express this a bit more formally, let `F` be the set of all "facts" we can generate from the clauses alone (a "fact" here is just a predicate that refers to some concrete types and thing)s. Because there are an infinite set of types, the set of facts is also infinite, but that's ok. In our example, given the rules we've seen so far (but ignoring the implied bound), we can show that `HasImpl(i32: Magic)` and `HasImpl(u32: Magic)` easily enough. We don't have a `HasImpl(i32: Copy)` fact, though, and because of that we also can't have a `Implemented(i32: Copy)` fact. Given this set of facts `F`, then we ought to be able to prove each invariant `I`, or something is broken in our type rules. In our example, the invariant `forall<T> { Implemented(T: Magic) => Implemented(T: Copy) }` does in fact hold, because there are no `Implemented(T: Magic)` facts.

### Integrating invariants into the solver

The solver is able to make use of invariants to generate proofs, but only in a limited way. Whereas we can always use a program clause, we can only apply invariants to the *hypotheses* that are in scope -- a *hypotheses* is some where-clause that we are assuming to be true. The idea here is that the caller must have proven that hypothesis to be a *fact* -- if they did so, then unless our type rules are broken, the invariant holds, which means that any facts we can derive with the invariants are also true.

This in turn implies that the seemingly tautological impl of `Magic` is actually **legal**! Recall the "ok goals" we saw before, that are used to decide which declarations are legal. The "ok" goal for the magic impl looks like this:

```scheme
(ForAll ((TyKind T))
        (Implies ((Implemented (Magic (T))))
                 (Implemented (Magic (T)))))
```

Basically, "if we assume that `T: Magic` is implemented, then we can show that `T: Magic` is implemented". Well, that's obviously true.

OK, so the impl is legal, but what about this function `make_the_magic_happen`?

```rust
fn make_the_magic_happen<T: Magic>(t: T) {
    let u = t;
    drop((t, u)); // both t, u are valid
}
```

We don't currently have type-checking logic in formality but, if we did, type-checking this function would require copying `t` and hence proving that:

```
forall<T> {
    Implemented(T: Magic) => Implemented(T: Copy)
}
```

Here the `Implemented(T: Magic) => ...` comes from the where-clauses on the function. To solve this, the solver puts `Implemented(T: Magic)` into the environment as a hypothesis using the "prove-implies" rule ([source](https://github.com/nikomatsakis/a-mir-formality/blob/47eceea34b5f56a55d781acc73dca86c996b15c5/src/ty/cosld-solve/prove.rkt#L67-L71)):

```scheme
  [(where Env_1 (env-with-hypotheses Env Hypotheses))
   (prove Env_1 Predicates_stack Goal EnvSubstitution_out)
   -------------------------------------------------------- "prove-implies"
   (prove Env Predicates_stack (Implies Hypotheses Goal) (reset Env () EnvSubstitution_out))
   ]
```

Next it can apply the "prove-hypothesis-imply" rule ([source](https://github.com/nikomatsakis/a-mir-formality/blob/47eceea34b5f56a55d781acc73dca86c996b15c5/src/ty/cosld-solve/prove.rkt#L41-L45)):

```scheme
  [(where #f (in? Predicate Predicates_stack))
   (Hypotheses-imply Env () Predicate EnvSubstitution_out)
   --------------- "prove-hypotheses-imply"
   (prove Env Predicates_stack Predicate EnvSubstitution_out)
   ]
```

This rule usess [`Hypotheses-imply`](https://github.com/nikomatsakis/a-mir-formality/blob/47eceea34b5f56a55d781acc73dca86c996b15c5/src/ty/cosld-solve/hypothesize.rkt#L9-L11), another typing judgment which determines whether `Predicate` is either *directly* in the environment as a hypothesis **or can be derived via an invariant**. This last part is what we need here! The only hypothesis in the environment is `Implemented(T: Magic)`, but we can use the invariant

```
Implemented(T: Magic) => Implemented(T: Copy)
```

to expand that to `Implemented(T: Copy)`, so we are happy.

### Wait, so is this sound?

But accepting this impl and this function this doesn't mean we have an unsound program -- the question is, who is going to *call* that function, and with what type? And this is where the errors come in. Consider this `main` function:

```rust
fn main() {
    make_the_magic_happen("Die, cruel world, die!".to_string());
}
```

For this program to type-check, we must prove the where-clauses on `make_the_magic_happen`, which means proving

```
Implemented(String: Magic)
```

But in this case, there are no hypotheses in the environment, so we can't make use of the invariants. We have to use the program clause, it requires also showing that `Implemented(String: Copy)` which in turn means showing `HasImpl(String: Copy)`, and we cannot do that.

Thinking a bit more abstractly, no matter what where clauses we have on various functions, we will "bottom out" in a `main` function somewhere, and `main` has no where clauses. Therefore, if our program relies on `Implemented(i32: Magic)`, that must be provable in an environment with no hypotheses. Put another way, `Implemented(i32: Magic)` must be a member of that infinite set of facts that we described earlier, the ones which categorize "everything that is true in this program". But we already argued that this set does not include `Implemented(i32: Magic)`, because the only way to get such a fact is to use the program clause, and the program clause requires that `HasImpl(i32: Copy)`, which is not true.

### Productivity again?

xxx -- didn't get time to to finish this, but I think that you can frame the previous two sections in terms of the typical "productivity" rules. There is a nice thesis I've been slowly working through on this. The TL;DR is something like this: "we accept all cycles but require that for any proof of `Implemented(T: Foo)`, `HasImpl(T: Foo)` must appear somewhere in the cycle", but that's not quite stating it right.

## Future directions

Here are some of the next steps I've been thinking about and we could discuss:

* There are some fixes to how `Hypothesize-implies` works that I want to make (I think it should be "bottom-up", not "top-down" as it is today, if that makes sense -- basically adopt the elaboration strategy from)
* Sketch out the MIR type-checker
* Integrate higher-ranked types and regions, polonius, etc -- I've been reading up on subtyping systems and I think I have some ideas for a fresh approach here
* Implement the "recursive solver" as a metafunction
    * it is useful to be able to compare the "cosld-solver", which I believe is sound+complete, against ours, which (I hope) is merely sound
* Extend rustc with `-Zformality` to allow it to generate formality programs
* Use PLT redex to fuzz the existing solver and test the "invariant logic"

# Niko's FAQ

## If this is a complete model, isn't that just an implementation? 

Not really. It leaves out a ton of stuff:

* diagnostics
* low-level ABI details
* parser and concrete rust syntax
* running C code and FFI (though you can imagine translating C code to equivalent unsafe Rust, which would be supported to some extent)
* platform specific things, interactions with the operating system

Currently, it's focusing on MIR and back, which means we can exclude a lot of things

* name resolution
* layout (we can take struct layouts as inputs)
* 

# Questions

## Are there any fundamental differences to Chalk, so far?

nikomatsakis: Yes. Chalk does not have the concept of invariants and handles implied bounds in a different way (`FromEnv` goals). Chalk also has one structural setup that has to be ported to formality -- it generates program clauses "on demand". This is required because actually the set of program clauses for a Rust program is potentially infinite (think of e.g. auto traits). I've been debating the best way to model this with PLT Redex and in particular whether it's important to preserve the various layers I've shown here, it might be easier if we merged everything into one.

## Is this an "ideal" Rust model, or a "practical" one?

- Current Rust semantics, or the one we want

nikomatsakis: You're asking the right question, and it's something I wanted to discuss. Right now it's the "ideal" model in the sense that I am using it to show how some extensions to Rust would work, but I think it's important that it be the "practical" one too. It's not hard to disable the implied bounds (actually, limit them to supertraits) and remove support for cycles. What's a bit more interesting, and I'm not sure about it, is the best way to have "feature gates" so that we can show the "diff required" to support a new feature. 

## Is it practical to try to develop Chalk alongside? I.e. update Chalk's rules as a-mir-formality is updated?

nikomatsakis: I think so, and I think we should. This is partly why Formality's structure intentionally mirrors Chalk (and it should eventually include the solver, so that we can show "these are things the compiler should be able to prove" versus "these are things that are provable"). The idea is that we can prototype and explore more quickly in this environment, but chalk would scale up to real programs (the PLT Redex solver is way too slow for that). Also, Chalk's structure would be more optimized for performance, for giving good error messages, for dealing with low-level ABI details, and all the other fun stuff that the model does NOT include.

## What about Coq?

> it seems that coq is the most popular at POPL etc. How well do you think the framework you're using will integrate into other PL tools/efforts?

nikomatsakis: As you can see, the notation is quite high-level, so if nothing else I think it would be easy to manually transcribe things. But I would assume people have made efforts to "export" rules and things from PLT Redex to other formats.

## How would types team signoff integrate into feature process?

* add a checkbox "types team signoff" for new language features
* open an issue on formality repository


## How/where does coercion fit in?
