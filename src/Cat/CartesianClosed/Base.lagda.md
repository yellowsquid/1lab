```agda
open import Cat.Diagram.Terminal
open import Cat.Functor.Adjoint
open import Cat.Diagram.Product
open import Cat.Prelude

import Cat.Functor.Bifunctor as Bifunctor
import Cat.Reasoning

module Cat.CartesianClosed.Base
  where
```

# Cartesian closed categories

Recall that we defined a **cartesian category** to be one which admits
all binary products, hence products of any finite positive cardinality.
Such a category is called **cartesian closed** (abbreviation: ccc) if it
has a terminal object (hence products of any natural number of objects),
and, for any object $A$, the functor $- \times A$ has a [right adjoint],
to be denoted $[A,-]$.

[right adjoint]: Cat.Functor.Adjoint.html

The object $[A,B]$ provided by this functor is called the **exponential
of $B$ by $A$**, and thus it is also written $B^A$. The adjunction is
best understood in terms of isomorphisms between [Hom-functors]: In a
ccc, the following Hom-sets are naturally isomorphic.

[Hom-functors]: Cat.Functor.Hom.html

$$
\hom(A \times B, C) \cong \hom(A, [B,C])
$$

The right-to-left direction of this isomorphism is called **currying**;
The left-to-right direction can thus be called **uncurrying**.
Generally, if you have an object in one side, its image under the
isomorphism is called its **exponential transpose**. The interpretation
of $[A,B]$ is that it is the _space of maps_ between $A$ and $B$.
Indeed, every _actual_ map $f : A \to B$ in the category corresponds to
a unique map $\ulcorner f \urcorner : 1 \to [A,B]$ (called the **name**
of $f$), by the following sequence of isomorphisms:

$$
\hom(A,B) \cong \hom(1 \times A, B) \cong \hom(1, [A,B])
$$

```agda
record is-cc {o ℓ} (C : Precategory o ℓ) : Type (o ⊔ ℓ) where
  field
    cartesian : ∀ A B → Product C A B

  open Cat.Reasoning C public
  open Cartesian C cartesian public

  private
    module ×-Bifunctor = Bifunctor {C = C} {C} {C} ×-functor

  field
    [_,-]      : Ob  → Functor C C
    tensor⊣hom : ∀ A → ×-Bifunctor.Left A ⊣ ([ A ,-])
    terminal   : Terminal C

  module [-,-] (a : Ob) = Functor [ a ,-]
  module T⊣H {a : Ob} = _⊣_ (tensor⊣hom a)

  [_,_] : Ob → Ob → Ob
  [ A , B ] = [-,-].₀ A B
```

We now make the structure of a ccc more explicit.

```agda
module CartesianClosed {o ℓ} {C : Precategory o ℓ} (cc : is-cc C) where
  open Functor
  open is-cc cc public
  private variable X Y Z : Ob
```

Each pair of objects $X$, $Y$ gives rise to an **evaluation** map
$\mathrm{ev} : [X, Y] \times X \to Y$. This is the counit of the
tensor-hom adjunction. The [adjuncts] (the exponential transposes
mentioned before) generated by a map $f$ give the currying and
uncurrying transformations:

[adjuncts]: Cat.Functor.Adjoint.Base.html

```agda
  ev : Hom ([ X , Y ] ⊗ X) Y
  ev = T⊣H.counit.ε _

  curry : Hom (X ⊗ Y) Z → Hom X [ Y , Z ]
  curry = L-adjunct (tensor⊣hom _)

  uncurry : Hom X [ Y , Z ] → Hom (X ⊗ Y) Z
  uncurry = R-adjunct (tensor⊣hom _)
```

By the triangle identities, `curry`{.Agda} and `uncurry`{.Agda} are
inverse equivalences.

```agda
  curry∘uncurry : ∀ {X Y Z} → is-left-inverse (curry {X} {Y} {Z}) uncurry
  curry∘uncurry f = L-R-adjunct (tensor⊣hom _) f

  uncurry∘curry : ∀ {X Y Z} → is-right-inverse (curry {X} {Y} {Z}) uncurry
  uncurry∘curry f = R-L-adjunct (tensor⊣hom _) f
```