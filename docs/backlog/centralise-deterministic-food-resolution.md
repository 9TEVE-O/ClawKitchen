# Centralise deterministic food/entity resolution before LLM fallback

## Status

Backlog / inspection-first implementation brief.

This file preserves the planning work because GitHub Issues are currently disabled for this repository.

Do not treat this as approval to implement immediately. The first agent pass is inspection and PR planning only.

## Decision

Do not ask the coding agent to optimise LLM usage generally.

Ask it to centralise deterministic resolution logic currently isolated in the recipe composition path, then route all relevant app entrypoints through that resolver before falling back to the language model.

This keeps the target narrow, bounded, and reviewable.

## Problem

The app already contains deterministic resolution logic, but it appears to be used only in the recipe composition function.

That means some flows can resolve structured food/entity inputs without a model, while other flows still send similar data directly to the language model.

Example user input:

> I need a chicken rice bowl. I had 100 grams of chicken, 50 grams of rice, and my homemade buffalo sauce.

The app should not require an LLM for every part of this. Known foods, saved recipes, aliases, quantities, units, and user-defined ingredients should resolve deterministically where possible.

## Goal

Create a shared resolver layer that all relevant flows use before invoking the LLM.

The resolver should determine whether a request can be handled by deterministic app logic, whether it needs partial model assistance, or whether it genuinely requires full LLM interpretation.

## Non-goals

- Do not rebuild the whole food parsing system.
- Do not change nutrition calculations unless required for resolver integration.
- Do not replace all LLM usage.
- Do not merge automatically.
- Do not make broad UI changes.
- Do not implement speculative optimisation outside the resolver path.

## Implementation shape

### New or extracted resolver boundary

Create or extract a shared resolver module with a contract similar to:

```ts
export type ResolutionStatus =
  | "resolved"
  | "partially_resolved"
  | "ambiguous"
  | "unresolved";

export type ResolutionSource =
  | "known_food"
  | "saved_recipe"
  | "user_alias"
  | "deterministic_parser"
  | "llm_fallback";

export interface ResolutionInput {
  rawText: string;
  userId?: string;
  context?: {
    mealType?: string;
    locale?: string;
    timezone?: string;
  };
}

export interface ResolvedEntity {
  id: string;
  label: string;
  source: ResolutionSource;
  quantity?: number;
  unit?: string;
  confidence: number;
  metadata?: Record<string, unknown>;
}

export interface ResolutionResult {
  status: ResolutionStatus;
  entities: ResolvedEntity[];
  unresolvedText?: string[];
  requiresLlm: boolean;
  reason?: string;
}
```

### Resolver function

```ts
export async function resolveFoodEntities(
  input: ResolutionInput
): Promise<ResolutionResult> {
  // 1. Parse obvious quantities and units.
  // 2. Match known foods.
  // 3. Match user-saved recipes.
  // 4. Match aliases / previous user entries.
  // 5. Return unresolved spans for LLM fallback only when needed.
}
```

### LLM gate

Every relevant flow should pass through a pre-LLM gate:

```ts
export async function resolveBeforeModel(input: ResolutionInput) {
  const resolution = await resolveFoodEntities(input);

  if (resolution.status === "resolved" && !resolution.requiresLlm) {
    return {
      mode: "deterministic",
      result: resolution,
    };
  }

  return {
    mode: "llm_fallback",
    result: resolution,
  };
}
```

The LLM should only receive:

```ts
{
  originalInput,
  deterministicResolution,
  unresolvedParts,
  task: "Resolve only the unresolved or ambiguous parts. Do not reinterpret resolved entities unless explicitly required."
}
```

## Search targets for the coding agent

Ask the agent to inspect for these patterns first:

```txt
recipe composition
resolver
resolve
parse food
ingredient lookup
nutrition lookup
LLM call
model call
completion
chat completion
natural language
food entry
meal entry
saved recipe
user recipe
alias
```

Priority locations:

```txt
Recipe composition function
Natural-language meal entry flow
Ingredient search / lookup flow
Saved recipe lookup
Voice input handling
LLM wrapper / model client
Nutrition calculation pipeline
Any direct model-call sites
```

## Required refactor steps

### 1. Inventory direct LLM call sites

Produce a list of files/functions that send user food/meal input to a model.

For each call site, classify:

- A. Should definitely use deterministic resolver first
- B. Might use resolver first
- C. Should remain LLM-first
- D. Out of scope

### 2. Extract current deterministic logic

Find the deterministic resolution logic currently used in the recipe composition path.

Move it into a shared resolver module without changing behaviour.

The first PR should preserve existing behaviour.

### 3. Add tests before wiring broadly

Add tests for:

- Known food exact match
- Known food alias match
- Saved recipe match
- Quantity + unit parsing
- Multiple ingredient input
- Partially unresolved input
- Ambiguous match
- No-match fallback

Example cases:

```txt
"100 grams chicken"
"50g rice"
"homemade buffalo sauce"
"chicken rice bowl"
"100 grams chicken, 50 grams rice, homemade buffalo sauce"
```

### 4. Route additional flows through the resolver

After tests pass, update the relevant entrypoints so they call the shared resolver before model fallback.

Do this incrementally.

Recommended PR split:

- PR 1: Extract resolver and preserve recipe composition behaviour
- PR 2: Add tests and resolver telemetry
- PR 3: Route natural-language meal entry through resolver
- PR 4: Route voice/phone entry through resolver
- PR 5: Remove duplicate or bypassed resolution logic

### 5. Add observability

Track:

```txt
deterministic_resolved_count
partial_resolution_count
llm_fallback_count
ambiguous_resolution_count
resolver_error_count
```

Each model call should record whether deterministic resolution was attempted first.

## Acceptance criteria

The PR is acceptable only if:

- Existing recipe composition behaviour remains unchanged or improves.
- All relevant resolver tests pass.
- At least one non-recipe-composition flow now uses the shared resolver.
- Resolved entities are not resent to the model for reinterpretation unless ambiguous.
- LLM fallback still works for unresolved inputs.
- The change is reviewable in small PRs.
- No protected/default branch is modified directly.
- No automatic merge occurs.

## Agent prompt

Copy this into the coding agent:

```txt
You are working on a food/recipe app where deterministic resolution logic currently appears to exist in the recipe composition path but is not consistently used across the app.

Objective:
Centralise the deterministic food/entity resolver and route relevant LLM entrypoints through it before model fallback.

Do not implement immediately.

First inspect the codebase and produce:
1. A file/function map of all relevant resolver, recipe composition, food parsing, ingredient lookup, saved recipe, voice input, natural-language input, and LLM call sites.
2. A classification of each model call site:
   - definitely resolver-first
   - maybe resolver-first
   - should remain LLM-first
   - out of scope
3. The smallest safe PR plan.
4. The tests that should be added before broad wiring.
5. Any risks or unclear assumptions.

Important constraints:
- Do not rewrite the app.
- Do not remove LLM fallback.
- Do not change nutrition calculations unless required for resolver integration.
- Do not merge automatically.
- Do not mark this complete until tests cover deterministic, partial, ambiguous, and unresolved cases.
- Prefer extracting existing logic over inventing a new resolver from scratch.
- Resolved entities should not be resent to the LLM for reinterpretation unless they are ambiguous.
```

## Closure rule

Do not treat this backlog item as complete until an inspection pass, test plan, and small PR plan have been reviewed.
