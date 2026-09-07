# Exaltation Station

A Path of Exile 2 crafting engine and route solver focused on finding good crafting decisions from a current item toward a target under explicit constraints such as budget, minimum physical DPS, attack speed, acceptable risk, and open affixes.

> Project status: concept / early prototype. The current priority is correctness of the item model, crafting rules, and probabilities before solver sophistication.

## Project language

English is the default language for the project, including documentation, issues, code comments, test names, and user-facing technical terminology.

## Goal

Exaltation Station should eventually answer a question like:

> "Given this current item, what should I do next to maximize my chance of reaching the target within my budget?"

The project is intended to be a **decision engine**, not just a crafting guide.

The long-term interaction is expected to look like this:

1. Copy an item from PoE2 using the detailed / advanced item text format.
2. Paste it into Exaltation Station.
3. Define a target in terms of useful outcomes rather than one exact affix combination.
4. Ask the engine for the best next action.
5. Perform the action in-game.
6. Paste the resulting item state.
7. Recompute the best next action.

Exaltation Station should remain an **advisor**. It should not automate game input or perform crafting actions inside PoE2.

---

## Engineering priority

The project should be built in this order:

```text
correct mechanics
  -> verified eligible mod pools
  -> verified probability model
  -> correct derived stats such as pDPS
  -> engine proof
  -> solver proof
  -> UI and performance work
```

A sophisticated search algorithm built on an incorrect item or probability model would be worse than a small correct engine.

---

## Core hypotheses

### 1. Crafting is a state-transition problem

An item is represented as a canonical state. A crafting action transforms that state into one or more possible next states.

```text
Current item
  |
  +-- deterministic action --> next state
  |
  +-- probabilistic action --> outcome A (pA)
                           +--> outcome B (pB)
                           +--> outcome C (pC)
```

The full problem can therefore become a graph or decision tree.

### 2. The rules engine must be deterministic even when crafting is probabilistic

For the same:

- item state;
- crafting action;
- game-data snapshot;
- ruleset;
- patch assumptions;

Exaltation Station should always generate the same legal outcome set and the same calculated probabilities.

Randomness belongs to the modeled game mechanic, not to the implementation.

### 3. Targets are constraints, not recipes

The solver should optimize for useful end states rather than require one exact affix combination.

Example:

```text
physical DPS >= 550
attack speed >= 1.65
```

Several different modifier combinations may satisfy the same target.

### 4. A useful route is eventually a policy, not just a fixed sequence

A real craft plan is conditional:

```text
perform action A

if result is strong:
  continue

if result is usable but suboptimal:
  take an alternate route

if result is bad:
  stop / reset / sell
```

This is the long-term solver target.

### 5. Cost and risk are first-class inputs

Possible future optimization objectives include:

- maximum success probability under a fixed budget;
- minimum expected cost;
- minimum brick probability;
- minimum crafting steps;
- expected value;
- weighted compromises between cost, probability, and risk.

The initial implementation should support only the minimum objective necessary to validate the engine.

### 6. Search-space control will matter later

Once genuinely competing crafting actions are supported, naïve exhaustive enumeration may become too expensive.

Possible techniques include:

- state hashing and memoization;
- dominated-state pruning;
- budget and depth limits;
- best-first search;
- beam search;
- dynamic programming where applicable;
- probability cutoffs for negligible branches.

None of these should be implemented before profiling demonstrates a need.

### 7. OR-Tools is optional and deferred

Constraint programming may eventually help with complex target combinations, but it is not required for the first implementation.

Initial target evaluation should use normal TypeScript predicates.

```text
rules engine      -> what can happen
constraint layer  -> whether an item satisfies the target
search solver     -> which action or policy is best
```

OR-Tools or another constraint solver should only be introduced after a concrete use case demonstrates that it is better than simpler code.

---

## Proposed architecture

The initial application should be compatible with static hosting such as GitHub Pages.

```text
Static web application
  |
  +-- item parser
  +-- canonical item model
  +-- normalized PoE2 data
  +-- crafting rules engine
  +-- derived-stat calculator
  +-- target evaluator
  +-- V0a decision logic
  +-- later: route solver
  +-- UI
```

### No backend by default

A client-side implementation gives us:

- very little infrastructure to maintain;
- deterministic local calculations;
- cheap or free hosting;
- simple public distribution;
- easy reproducibility of a calculation against a known data snapshot.

A backend should only be added for a demonstrated requirement such as live market aggregation, heavy shared computation, or an API that cannot safely be called from the browser.

### No Web Worker until profiling requires it

The first narrow implementation should run in-process.

A Web Worker should only be introduced if real measurements show that calculations block the UI. V0a is intentionally too small to justify concurrency infrastructure in advance.

---

## Canonical data model

Game-data definitions and rolled item state must be separate concepts.

### Base item definition

A base definition contains immutable game data required to evaluate a bow.

Conceptually:

```ts
interface BaseItemDefinition {
  id: string;
  itemClass: string;
  basePhysicalMin: number;
  basePhysicalMax: number;
  baseAttacksPerSecond: number;
  tags: string[];
}
```

### Modifier definition

A modifier definition represents what may spawn, not what actually rolled on an item.

Conceptually:

```ts
interface ModifierDefinition {
  id: string;
  generationType: "prefix" | "suffix";
  group: string;
  requiredItemLevel: number;
  tags: string[];
  generationWeights: GenerationWeightRule[];
  stats: StatDefinition[];
}
```

The canonical model must preserve conditional / ordered generation-weight rules when the upstream data uses them.

**Do not flatten conditional generation weights into one global `spawnWeight` number.**

Eligibility and weight must be resolved against the actual item/base context.

### Rolled modifier

An affix on a real item is a specific roll of a modifier definition.

```ts
interface ModifierRoll {
  definitionId: string;
  tier?: number;
  values: Record<string, number>;
}
```

This distinction matters because a tier can expose a value range while the real item contains one specific value inside that range.

### Item state

Conceptually:

```ts
interface ItemState {
  baseTypeId: string;
  itemLevel: number;
  rarity: Rarity;
  quality: number;
  prefixes: ModifierRoll[];
  suffixes: ModifierRoll[];
}
```

Base physical damage and base attack speed should come from the referenced `BaseItemDefinition` rather than be duplicated across every state.

Additional fields should only be added when a supported PoE2 mechanic proves they are required.

---

## Item input hypothesis

### V0a input

V0a should target **English advanced / detailed copied item text**, preferably the format that exposes enough modifier metadata to identify prefixes, suffixes, tiers, and values reliably.

The parser should reject unsupported or ambiguous input rather than guess.

The first parser does not need to support:

- every game language;
- screenshots or OCR;
- fuzzy modifier identification;
- every item class;
- every historical item-text format.

Tests should use committed text fixtures representing real copied bows.

---

## Derived-stat model

The pDPS target makes the stat model part of the core engine, not a UI concern.

At minimum the calculation must correctly account for the supported interactions involving:

- bow base physical damage;
- local physical-damage modifiers;
- flat added physical damage where applicable;
- local attack-speed modifiers;
- quality;
- actual rolled modifier values rather than only tiers.

The exact calculation order should be covered by manually verified fixtures before being treated as authoritative.

---

## Craft-action model

Each supported crafting action should expose eligibility and possible outcomes.

Conceptually:

```ts
interface CraftAction {
  isAvailable(item: ItemState, context: CraftContext): boolean;
  outcomes(item: ItemState, context: CraftContext): OutcomeSet;
}
```

An outcome should contain enough information for deterministic evaluation:

```ts
interface Outcome {
  probability: number | "unknown";
  item: ItemState;
  cost: Cost;
}
```

This creates a clean boundary between PoE2 rules and future search algorithms.

---

## Probability model

Probability accuracy is a critical project risk.

There are at least two distinct probability layers:

```text
1. probability that a modifier / tier is selected
2. probability of the numeric roll inside that modifier's allowed range
```

V0a must not silently treat these as the same problem.

Modifier-selection probabilities should be derived from validated eligible pools and generation weights.

Numeric roll distributions must be explicitly verified before Exaltation Station claims an exact probability for thresholds such as `pDPS >= X` when success depends on the value rolled inside an affix range.

If the numeric distribution is not yet validated, the engine should:

- expose the assumption explicitly; or
- return a probability range / unknown result; or
- restrict the test case to outcomes where the uncertain roll distribution does not affect the assertion.

**Never invent precision.**

---

## Target model

A target should be evaluable against any supported item state.

V0a targets:

```ts
pdps(item) >= targetPdps
```

with optional:

```ts
attackSpeed(item) >= targetAttackSpeed
```

More expressive target languages should be deferred.

---

## Cost model

V0a should not depend on live market prices.

To support a fixed budget, use an explicit configurable cost model for the supported crafting components.

Conceptually:

```ts
interface CostModel {
  transmutation: number;
  augmentation: number;
  regal: number;
  exalted: number;
}
```

The values may initially be manually supplied normalized cost units.

This means V0a can test budget-aware decision logic without pretending to know current Divine-equivalent prices.

Live pricing and trade-vs-craft optimization remain separate future concerns.

---

## Data ingestion hypothesis

External sources should never leak their schema directly into the crafting engine.

```text
external PoE2 data
  -> snapshot
  -> importer
  -> normalization
  -> validation
  -> canonical Exaltation Station data
  -> crafting engine
```

Suggested layout:

```text
src/
  parser/
  model/
  rules/
  stats/
  constraints/
  solver/
  ui/

ingestion/
  importers/
  validators/

data/
  fixtures/
  normalized/
```

The final repository structure should remain as small as the implementation allows; this is a conceptual separation, not a requirement to create every directory immediately.

---

## Initial data-source hypothesis

For the first proof of concept, **RePoE PoE2 data is considered sufficient to start testing the engine**.

Initial upstream source:

- `repoe-fork/poe2`

Working assumption:

```text
RePoE PoE2 snapshot
  -> import / normalize
  -> canonical Exaltation Station subset
  -> manually validate the supported bow data
  -> run engine tests
```

RePoE is an **initial engineering source**, not unquestioned authoritative truth.

Each imported snapshot should eventually record:

- upstream repository;
- upstream commit / snapshot identifier;
- associated game patch when known;
- import timestamp;
- validation status;
- provenance for uncertain or community-inferred values where relevant.

For V0a, the exact bow bases, modifier pools, generation weights, and supported crafting rules used in acceptance tests should be independently cross-checked against another trusted reference or known in-game behavior.

The engine must remain independent of the RePoE schema so another source can replace or supplement it later.

### Data redistribution caution

The repository is public. Do not automatically vendor or redistribute the complete upstream game dataset until the relevant licensing / redistribution terms have been reviewed.

The safest initial path is:

```text
ingestion code
+
small derived / manually validated V0a fixtures
```

rather than copying a complete external database into the repository by default.

---

# Validated development baseline

## V0a — Crafting engine proof

The previously discussed V0 is now classified more precisely as **V0a: engine proof**.

Its purpose is to prove that Exaltation Station models a small part of PoE2 crafting correctly.

### Scope

```text
Item class: Bows only
Input: English advanced / detailed copied item text
Crafts:
  - Orb of Transmutation
  - Orb of Augmentation
  - Regal Orb
  - Exalted Orb
Target:
  - physical DPS threshold
  - optional attack-speed threshold
Budget:
  - fixed budget using a configurable normalized CostModel
Data:
  - RePoE snapshot
  - manually cross-checked supported subset
Output:
  - legal outcomes
  - probabilities where validated
  - derived stats
  - continue / stop recommendation
```

### Why this is engine proof rather than solver proof

With this restricted currency set, much of the crafting sequence is mechanically constrained.

The main decisions are often whether a current result is good enough to continue spending currency or whether the process should stop.

That is enough to validate:

- parsing;
- item-state reconstruction;
- modifier eligibility;
- generation-weight handling;
- outcome generation;
- probability calculations;
- pDPS calculations;
- cost-aware continue / stop logic.

It is **not enough to prove that a general route solver can choose intelligently between several genuinely competing crafting strategies**.

### V0a success criteria

V0a is complete when manually verified fixtures demonstrate that the engine can:

1. parse supported copied bow text deterministically;
2. identify the correct base and rolled affixes;
3. reconstruct quality and relevant item state;
4. derive the correct eligible modifier pool for each supported action;
5. correctly exclude blocked modifier groups and invalid tiers;
6. resolve the correct conditional generation weights;
7. produce outcome probabilities that sum correctly where the model is fully known;
8. calculate resulting attack speed and pDPS correctly;
9. enforce a fixed normalized budget;
10. return the expected continue / stop recommendation for known cases.

A polished UI is not required to close V0a.

### Minimum acceptance fixture set

Before expanding scope, create a small suite of manually verified bow cases covering at least:

- a Normal bow eligible for Transmutation;
- a Magic bow with one affix eligible for Augmentation;
- a full Magic bow eligible for Regal;
- a Rare bow eligible for Exalted;
- an item where modifier groups remove otherwise plausible outcomes;
- a case where actual roll values change pDPS enough to affect the target decision.

The exact number of fixtures is less important than proving these mechanics independently.

---

## V0b — Solver proof

V0b starts only after V0a passes its acceptance tests.

V0b should add **exactly one meaningful source of competing crafting decisions** so the project can demonstrate route optimization rather than only outcome simulation and continue / stop logic.

The additional mechanic should be selected based on:

- data availability;
- confidence in its rules;
- usefulness for bow crafting;
- ability to create genuinely competing routes;
- minimal implementation complexity.

V0b succeeds when the same starting item can plausibly follow more than one legal strategy and the solver can justify why one route is preferable under the chosen objective and budget.

Do not add several advanced crafting systems at once.

---

## Explicitly deferred

The following are not required for V0a:

- other item classes;
- Essences, Omens, or broad advanced crafting support;
- a general-purpose crafting DSL;
- live market pricing;
- trade-vs-craft optimization;
- OR-Tools;
- Web Workers;
- a backend service;
- LLM decision-making;
- screenshot / OCR item parsing;
- multilingual item parsing;
- exhaustive simulation of every PoE2 mechanic.

An LLM may eventually be useful as an optional interface for translating natural-language goals into formal constraints and explaining solver results. It must not be the authority for crafting rules or probabilities.

---

## Licensing and public-project hygiene

Before accepting significant external contributions or redistributing external datasets:

- choose an explicit license for Exaltation Station's own source code;
- verify the redistribution terms of imported game/community data;
- preserve source attribution and provenance where required;
- keep the public non-affiliation disclaimer visible.

---

## Design principle

**Model the game accurately, keep the core deterministic, verify probabilities, and add complexity only when a real crafting case requires it.**

The preferred progression is:

```text
small + correct
  -> tested
  -> useful
  -> optimized
```

not:

```text
large architecture
  -> hope the model is correct later
```

---

## Disclaimer

Exaltation Station is an independent community project and is not affiliated with or endorsed by Grinding Gear Games.

Path of Exile and Path of Exile 2 are trademarks of Grinding Gear Games.