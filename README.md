# Exaltation Station

A Path of Exile 2 crafting solver focused on finding the best route from a current item to a target item under explicit constraints such as budget, minimum stats, acceptable risk, and open affixes.

> Project status: concept / early prototype. The rules, data model, and solver strategy below are working hypotheses to validate with a narrow proof of concept before expanding scope.

## Goal

Exaltation Station should answer a question like:

> "Given this current item, how should I craft toward my target while minimizing cost and avoiding bad branches?"

The project is intended to be a **decision engine**, not just a crafting guide. It should model the current item, enumerate legal crafting actions, evaluate their outcomes, and recommend the next action according to a chosen objective.

The long-term target is an interactive workflow:

1. Copy an item from PoE2.
2. Paste it into Exaltation Station.
3. Define the target in terms of useful outcomes rather than one exact mod combination.
4. Ask the solver for the best route.
5. Perform the suggested craft in-game.
6. Paste the new item state.
7. Recompute the best next action.

The application should remain an **advisor**. It should not automate input or perform actions inside the game.

## Core hypotheses

### 1. Crafting is best represented as a state-transition problem

An item is represented as a state:

- base type
- item level
- rarity
- prefixes
- suffixes
- mod tiers
- mod tags / families where relevant
- relevant special state required by crafting mechanics

A crafting action transforms one state into one or more possible next states.

```text
Current item
  |
  +-- deterministic action --> next state
  |
  +-- probabilistic action --> outcome A (pA)
                           +--> outcome B (pB)
                           +--> outcome C (pC)
```

The solver therefore operates on a graph or decision tree rather than a static list of recipes.

### 2. The rules engine should be deterministic even when the game is not

For the same item state, crafting action, ruleset, and patch version, Exaltation Station should always produce the same set of possible outcomes and probabilities.

Randomness belongs to the modeled game mechanic, not to the solver implementation.

This distinction is important:

- **deterministic logic**: which actions are legal, which outcomes exist, how weights are calculated
- **probabilistic mechanic**: which of those outcomes the game actually gives the player

### 3. Targets should be expressed as constraints, not exact recipes

The solver should optimize for useful end states rather than one hard-coded affix combination.

Example target:

```text
physical DPS >= 550
attack speed >= 1.65
budget <= 8 Divine
at least 1 open suffix
```

Several different affix combinations may satisfy the same target.

This should let the solver discover routes that a hand-written recipe would miss.

### 4. The solver should optimize a policy, not just a sequence

A useful craft plan is conditional.

Example:

```text
Use action A

if result is strong:
  continue with route B

if result is acceptable:
  continue with route C

if result is bad:
  reset / sell / stop
```

The result should therefore be a **decision policy** with stop conditions, not merely a fixed ordered list of currencies.

### 5. Cost and risk are first-class optimization dimensions

Possible optimization modes may include:

- minimum expected cost
- maximum success probability under a fixed budget
- minimum number of crafting steps
- minimum probability of bricking the item
- best expected value
- weighted compromise between cost, success chance, and risk

The first proof of concept only needs one or two simple objectives.

### 6. Search-space control will matter more than raw compute

Naively enumerating every possible crafting branch will become intractable very quickly.

The solver will likely need some combination of:

- state hashing / memoization
- dominated-state pruning
- depth limits
- budget limits
- beam search
- best-first search / A*
- dynamic programming where applicable
- probability cutoffs for negligible branches

The goal is not to explore every theoretically reachable item. The goal is to preserve branches that can still lead to a useful target.

### 7. OR-Tools may be useful, but should not be a hard dependency initially

Constraint programming is a good fit for questions such as:

- does this state satisfy the target?
- can any reachable combination satisfy the requested constraints?
- which affix combinations are valid under a set of requirements?

However, the full crafting problem also contains sequential probabilistic decisions.

Working assumption:

```text
rules engine      -> models what crafting actions can do
constraint layer  -> evaluates whether states satisfy the target
search solver     -> chooses the best sequence / policy
```

The first implementation should use simple TypeScript predicates for constraints. OR-Tools or another solver should only be introduced if it provides a demonstrated advantage.

## Proposed architecture

The initial implementation is expected to be a static web application suitable for GitHub Pages.

```text
GitHub Pages
  |
  +-- UI
  +-- item parser
  +-- rules engine
  +-- target evaluator
  +-- solver
  +-- versioned PoE2 data
  +-- Web Worker
```

### Why a static application first?

A client-side implementation offers several advantages for the first versions:

- almost no infrastructure to maintain
- cheap or free hosting
- easy public deployment through GitHub Pages
- deterministic, reproducible local calculations
- no server required for the core solver
- straightforward open-source distribution

A backend should only be added when a concrete requirement justifies it, for example live market aggregation, heavy compute, shared caches, or APIs that cannot be queried safely from the browser.

## Data model hypothesis

Game data should be separate from solver logic and versioned by PoE2 patch.

```text
src/
  core/
  parser/
  rules/
  solver/
  constraints/
  ui/

data/
  poe2-<patch>/
    bases.json
    modifiers.json
    currencies.json
    crafting-rules.json
```

The objective is to make patch updates primarily a data/rules maintenance task rather than a solver rewrite.

Every probability shown to users should be traceable to the exact rules/data version used by the calculation.

## Item input hypothesis

The preferred input is the textual item representation copied from the game.

The parser should convert it into a canonical internal `ItemState`.

Conceptually:

```ts
interface ItemState {
  baseType: string;
  itemLevel: number;
  rarity: Rarity;
  prefixes: Modifier[];
  suffixes: Modifier[];
}
```

The exact schema should remain minimal until real PoE2 mechanics prove additional fields are required.

## Craft-action hypothesis

Each supported crafting mechanic should expose two basic operations:

```ts
interface CraftAction {
  isAvailable(item: ItemState): boolean;
  outcomes(item: ItemState): Outcome[];
}

interface Outcome {
  probability: number;
  item: ItemState;
  cost: Cost;
}
```

This creates a clean boundary between PoE2 rules and search algorithms.

## Target model hypothesis

A target should be evaluable against any item state.

Initially this can simply be a collection of predicates, for example:

```ts
pdps(item) >= 550
attackSpeed(item) >= 1.65
openSuffixes(item) >= 1
```

Later, the target language may support richer constraints, scoring, alternatives, and preferences.

## Solver output hypothesis

The solver should not pretend that probabilistic crafts are guaranteed.

A useful result should include at least:

- recommended next action
- estimated cost
- success probability where calculable
- relevant failure branches
- stop / reset conditions
- assumptions used by the calculation

Longer term, it may expose a complete decision tree or compact policy.

Example:

```text
Recommended action: <craft action>

If outcome A:
  continue with action B

If outcome B:
  item remains usable; switch to route C

If outcome C:
  stop and reset

Expected cost: X
Success probability within budget: Y%
```

## Market-price hypothesis

Live pricing is useful but should not be required for the first solver.

The core engine should work with an abstract cost model. Prices can initially be:

- manually entered
- bundled snapshots
- later sourced from an appropriate API or market data provider

This keeps the crafting engine independent from market availability and API reliability.

A future version could compare:

```text
expected craft cost
vs.
market cost of an equivalent item
```

and recommend crafting, buying, or stopping.

## Data quality is a critical dependency

The quality of the solver can never exceed the quality of its PoE2 rules and modifier data.

Every supported mechanic should therefore have tests validating known examples and edge cases.

When mechanics or probabilities are uncertain, the application should expose that uncertainty rather than presenting an invented exact number.

## Initial data-source hypothesis

For the first proof of concept, **RePoE PoE2 data is considered sufficient to start testing the solver**.

The purpose of the first milestone is not to guarantee perfect live-patch fidelity across all of Path of Exile 2. It is to validate the core calculation loop with enough structured data to model a narrow crafting domain correctly.

Working assumption:

```text
RePoE PoE2 snapshot
  -> import / normalize
  -> canonical Exaltation Station dataset
  -> manually validate a narrow subset
  -> run solver experiments
```

RePoE should therefore be treated as an **initial engineering data source**, not as unquestioned authoritative truth.

Before expanding beyond the proof of concept, the project should add stronger freshness and provenance controls, including where practical:

- upstream snapshot / commit identification
- associated PoE2 patch version
- import timestamp
- consistency checks
- manual verification of representative mod pools and weights
- explicit marking of uncertain or community-inferred values

For the PoC, a limited RePoE-backed dataset is acceptable if the exact item class, modifier pool, weights, and supported crafting actions used by the test cases are independently checked against another trusted reference or known in-game behavior.

The solver itself must remain independent of RePoE's schema so that upstream data sources can be replaced or supplemented later without rewriting the crafting engine.

## Initial proof of concept

The first milestone should deliberately be small.

Suggested scope:

- one item category
- a limited set of relevant modifiers
- a small number of crafting actions
- deterministic item parsing for fixtures
- simple target predicates
- one search strategy
- expected-cost or success-probability optimization
- automated tests against manually verified crafting cases

The proof of concept succeeds when the engine can take an initial item, a target, and a limited ruleset and independently return a sensible craft route.

It does **not** require complete PoE2 data, live prices, a polished UI, AI integration, or OR-Tools.

## What Exaltation Station is not

At least initially, this project is not intended to be:

- an in-game automation bot
- a macro system
- a generic LLM crafting chatbot
- a replacement for deterministic game data
- a full market-trading platform
- a complete simulator of every PoE2 mechanic from day one

An LLM may eventually be useful as an optional interface for translating natural-language goals into constraints and explaining solver results. It should not be the authority for crafting rules or probabilities.

## Design principle

**Model the game accurately, keep the engine deterministic, and add complexity only when a real crafting case requires it.**

The project should favor a small, testable core over an ambitious architecture built before the underlying crafting model has been validated.

## Disclaimer

Exaltation Station is an independent community project and is not affiliated with or endorsed by Grinding Gear Games.

Path of Exile and Path of Exile 2 are trademarks of Grinding Gear Games.