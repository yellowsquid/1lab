<!--
```agda
open import 1Lab.Path.IdentitySystem
open import 1Lab.Reflection.HLevel
open import 1Lab.HLevel.Retracts
open import 1Lab.HLevel.Universe
open import 1Lab.HIT.Truncation
open import 1Lab.Reflection using (arg ; typeError)
open import 1Lab.Univalence
open import 1Lab.HLevel
open import 1Lab.Equiv
open import 1Lab.Path
open import 1Lab.Type

open import Data.List.Base

open import Meta.Idiom
open import Meta.Bind
```
-->

```agda
module 1Lab.Resizing where
```

# Propositional Resizing {defines="propositional-resizing"}

Ordinarily, the collection of all $\kappa$-small predicates on
$\kappa$-small types lives in the next universe up, $\kappa^+$. This is
because _predicates_ are not special in type theory: they are ordinary
type families, that just so happen to be valued in [[propositions]]. For
most purposes we can work with this limitation: this is called
**predicative mathematics**. But, for a few classes of theorems,
predicativity is too restrictive: Even if we don't have a single
universe of propositions that can accommodate all predicates, we would
still like for universes to be closed under power-sets.

Using some of Agda's more suspicious features, we can achieve this in a
way which does not break computation too much. Specifically, we'll use a
combination of `NO_UNIVERSE_CHECK`, postulates, and rewrite rules.

We start with the `NO_UNIVERSE_CHECK` part: We define the **small type
of propositions**, `Ω`, to be a record containing a `Type` (together
with an irrelevant proof that this type is a proposition), but contrary
to the universe checker, which would like us to put `Ω : Type₁`, we
instead force `Ω : Type`.

```agda
{-# NO_UNIVERSE_CHECK #-}
record Ω : Type where
  constructor el
  field
    ∣_∣   : Type
    is-tr : is-prop ∣_∣
open Ω public
```

This type, a priori, only contains the propositions whose underlying
type lives in the first universe. However, we can populate it using a
`NO_UNIVERSE_CHECK`-powered higher inductive type, the "small
[[propositional truncation]]":

```agda
{-# NO_UNIVERSE_CHECK #-}
data □ {ℓ} (A : Type ℓ) : Type where
  inc    : A → □ A
  squash : (x y : □ A) → x ≡ y
```

Just like the ordinary propositional truncation, every type can be
squashed, but unlike the ordinary propositional truncation, the
`□`{.Agda}-squashing of a type always lives in the lowest universe.  If
$T$ is a proposition in any universe, $\Box T$ is its name in the zeroth
universe.

<!--
```agda
instance
  H-Level-□ : ∀ {ℓ} {T : Type ℓ} {n} → H-Level (□ T) (suc n)
  H-Level-□ = prop-instance squash

  open hlevel-projection
  Ω-hlevel-proj : hlevel-projection (quote Ω.∣_∣)
  Ω-hlevel-proj .has-level = quote Ω.is-tr
  Ω-hlevel-proj .get-level x = pure (quoteTerm (suc zero))
  Ω-hlevel-proj .get-argument (arg _ t ∷ _) = pure t
  Ω-hlevel-proj .get-argument _ = typeError []
```
-->

We can also prove a univalence principle for `Ω`{.Agda}:

```agda
Ω-ua : {A B : Ω} → (∣ A ∣ → ∣ B ∣) → (∣ B ∣ → ∣ A ∣) → A ≡ B
Ω-ua {A} {B} f g i .∣_∣ = ua (prop-ext! f g) i
Ω-ua {A} {B} f g i .is-tr =
  is-prop→pathp (λ i → is-prop-is-prop {A = ua (prop-ext! f g) i})
    (A .is-tr) (B .is-tr) i

instance abstract
  H-Level-Ω : ∀ {n} → H-Level Ω (2 + n)
  H-Level-Ω = basic-instance 2 $ retract→is-hlevel 2
    (λ r → el ∣ r ∣ (r .is-tr))
    (λ r → el ∣ r ∣ (r .is-tr))
    (λ x → Ω-ua (λ x → x) λ x → x)
    (n-Type-is-hlevel {lzero} 1)
```

The `□`{.Agda} type former is a functor (in the handwavy sense that it
supports a "map" operation), and can be projected from into propositions
of any universe. These functions compute on `inc`{.Agda}s, as usual.

```agda
□-map : ∀ {ℓ ℓ'} {A : Type ℓ} {B : Type ℓ'}
      → (A → B) → □ A → □ B
□-map f (inc x) = inc (f x)
□-map f (squash x y i) = squash (□-map f x) (□-map f y) i

□-rec!
  : ∀ {ℓ ℓ'} {A : Type ℓ} {B : Type ℓ'}
  → {@(tactic hlevel-tactic-worker) pa : is-prop B}
  → (A → B) → □ A → B
□-rec! {pa = pa} f (inc x) = f x
□-rec! {pa = pa} f (squash x y i) =
  pa (□-rec! {pa = pa} f x) (□-rec! {pa = pa} f y) i

out! : ∀ {ℓ} {A : Type ℓ}
     → {@(tactic hlevel-tactic-worker) pa : is-prop A}
     → □ A → A
out! {pa = pa} = □-rec! {pa = pa} (λ x → x)

elΩ : ∀ {ℓ} (T : Type ℓ) → Ω
∣ elΩ T ∣ = □ T
elΩ T .is-tr = squash
```

<!--
```agda
□-elim
  : ∀ {ℓ ℓ'} {A : Type ℓ} {P : □ A → Type ℓ'}
  → (∀ x → is-prop (P x))
  → (∀ x → P (inc x))
  → ∀ x → P x
□-elim pprop go (inc x) = go x
□-elim pprop go (squash x y i) =
  is-prop→pathp (λ i → pprop (squash x y i)) (□-elim pprop go x) (□-elim pprop go y) i

□-rec-set
  : ∀ {ℓ ℓ'} {A : Type ℓ} {B : Type ℓ'}
  → (f : A → B)
  → (∀ x y → f x ≡ f y)
  → is-set B
  → □ A → B
□-rec-set f f-const B-set a =
  fst $ □-elim
    (λ _ → is-constant→image-is-prop B-set f f-const)
    (λ a → f a , inc (a , refl))
    a

□-idempotent : ∀ {ℓ} {A : Type ℓ} → is-prop A → □ A ≃ A
□-idempotent aprop = prop-ext squash aprop (out! {pa = aprop}) inc

□-ap
  : ∀ {ℓ ℓ'} {A : Type ℓ} {B : Type ℓ'}
  → □ (A → B) → □ A → □ B
□-ap (inc f) (inc g) = inc (f g)
□-ap (inc f) (squash g g' i) = squash (□-ap (inc f) g) (□-ap (inc f) g') i
□-ap (squash f f' i) g = squash (□-ap f g) (□-ap f' g) i

□-bind
  : ∀ {ℓ ℓ'} {A : Type ℓ} {B : Type ℓ'}
  → □ A → (A → □ B) → □ B
□-bind (inc x) f = f x
□-bind (squash x x' i) f = squash (□-bind x f) (□-bind x' f) i

instance
  Map-□ : Map (eff □)
  Map-□ .Map._<$>_ = □-map

  Idiom-□ : Idiom (eff □)
  Idiom-□ .Idiom.pure = inc
  Idiom-□ .Idiom._<*>_ = □-ap

  Bind-□ : Bind (eff □)
  Bind-□ .Bind._>>=_ = □-bind

is-set→locally-small
  : ∀ {ℓ} {A : Type ℓ}
  → is-set A
  → is-identity-system {A = A} (λ x y → □ (x ≡ y)) (λ x → inc refl)
is-set→locally-small a-set .to-path = out! {pa = a-set _ _}
is-set→locally-small a-set .to-path-over p = is-prop→pathp (λ _ → squash) _ _

to-is-true
  : ∀ {P Q : Ω} ⦃ _ : H-Level ∣ Q ∣ 0 ⦄
  → ∣ P ∣
  → P ≡ Q
to-is-true prf = Ω-ua (λ _ → hlevel 0 .centre) (λ _ → prf)
```
-->
