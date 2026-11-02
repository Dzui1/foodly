# Epicure-Core + Ollama Terminal App

## Final Project Plan

## 1. Goal

Build a local terminal application that turns abstract food requests into recipes using two separate systems with clearly defined responsibilities:

- **A small local LLM through Ollama** interprets natural language, moods, preferences, and constraints.
- **The Epicure-Core Word2Vec embedding space** performs the actual ingredient similarity search and vector arithmetic.
- **Python** connects the two, applies filters, validates results, and manages conversation state.

Example user request:

> "Give me rainy day vibes without seafood."

The LLM should not generate a recipe immediately. Instead, it translates the request into a small set of concrete seed ingredients and dietary constraints. Python then searches the Epicure-Core vector space, filters the results, and gives the resulting ingredient neighborhood back to the LLM. The LLM uses only those results, plus a small whitelist of pantry staples, to write the final recipe.

The system should prioritize reliability, local execution, explainability, and useful output over UI complexity.

---

## 2. Core Architecture

The application is split into four logical layers.

### 2.1 User Interface Layer

A terminal-based interface accepts natural-language requests and displays the final recipe.

Recommended stack:

- **Typer** for CLI commands and options.
- **Rich** for readable terminal output, progress indicators, formatted recipe sections, and optional debugging information.

A full-screen TUI is unnecessary for the first version. Rich provides enough structure without adding much complexity.

### 2.2 LLM Interpretation Layer

A small Ollama-hosted model acts as the language interpreter.

Its responsibilities are:

- Convert abstract ideas such as "rainy day," "summer in Sicily," or "comforting but light" into concrete ingredient seeds.
- Detect dietary restrictions and exclusions.
- Convert broad constraints such as "no seafood" into a structured category exclusion rather than attempting to list every seafood ingredient itself.
- Normalize ingredient names into likely Epicure vocabulary tokens.
- Preserve a short summary of the user's original vibe for use during recipe formatting.

Recommended primary model:

- **Qwen 2.5 3B Instruct**

Reason:

- Both project plans converge on Qwen as the more reliable option for structured output and tool use.
- For this architecture, reliable interpretation matters more than creative prose during the first pass.

Optional formatter model:

- A Gemma model may be used for the final recipe-writing pass if better prose is desired.
- This should remain optional. The first version should support using the same Qwen model for both interpretation and formatting to reduce memory usage and operational complexity.

### 2.3 Python Vector Engine

Python owns all deterministic logic.

Its responsibilities are:

- Load the Epicure-Core embedding matrix and vocabulary.
- Normalize vectors once at startup.
- Resolve ingredient names and synonyms.
- Build a query vector from seed ingredients.
- Apply positive and negative weighting when needed.
- Compute cosine similarity against all ingredients.
- Apply dietary and ingredient-level exclusions.
- Return ranked neighbors and metadata to the LLM.

The Epicure-Core model is small enough that approximate nearest-neighbor indexing is unnecessary for the first version. A full NumPy similarity calculation over 1,790 vectors is simpler and fast enough.

### 2.4 Recipe Formatting Layer

After the vector search returns results, the LLM performs a second pass.

Its responsibilities are:

- Turn the returned ingredient neighborhood into one coherent dish.
- Stay grounded in the returned ingredients.
- Use only a small whitelist of universal staples when necessary.
- Respect all exclusions already applied by Python.
- Tie the dish back to the user's original request.

This separation is important. The first LLM pass interprets intent. The vector engine decides the culinary neighborhood. The second LLM pass writes the recipe.

---

## 3. End-to-End Request Flow

The complete request flow should be:

1. The user enters a natural-language request.
2. The LLM interprets the request into structured fields.
3. Python validates and normalizes the fields.
4. Python resolves seed ingredients against the Epicure vocabulary.
5. Python builds the query vector.
6. Python expands and applies exclusions.
7. Python ranks the nearest ingredients.
8. Python returns structured search results.
9. The LLM receives the original request, interpreted intent, and returned neighbors.
10. The LLM writes the final recipe.
11. Conversation state is retained for follow-up requests such as "make it spicier" or "something lighter."

The vector engine should remain the source of truth for which ingredients are available to the recipe writer.

---

## 4. Project Structure

A simple project structure is enough:

- `app` — CLI entry point, conversation loop, Ollama orchestration.
- `engine` — vector loading, normalization, similarity search, weighting, and filtering.
- `models` — validated request and response structures.
- `prompts` — interpreter and formatter prompt definitions.
- `state` — optional conversation state and previous search context.
- `data/`
  - vector matrix
  - vocabulary
  - ingredient categories
  - synonym mappings
- `tests/`
  - unit tests
  - golden prompt tests
  - end-to-end tests
- project configuration and README

The system should keep the LLM-facing logic separate from the vector engine so either side can be changed independently.

---

## 5. Environment and Dependency Management

The two plans recommend different environment tools: Poetry in one and `uv` in the other.

### Decision: use `uv` for the main development workflow

Reasons:

- Lightweight and fast for a local Python CLI project.
- Supports reproducible dependency locking.
- Less configuration overhead than Poetry for this size of application.

A normal virtual environment with pinned requirements remains an acceptable fallback for contributors who do not use `uv`.

Docker should not be required during initial development. It adds unnecessary setup for a project whose main goal is local interaction with an already installed Ollama instance.

Docker can be added later for distribution or reproducible demos.

---

## 6. Epicure-Core Model Loading

The plans describe several loading options, including the official loader, SafeTensors, NumPy files, JSON, HDF5, SQLite, and memory mapping.

### Decision: use the official/raw model format only for initial acquisition, then cache locally as NumPy plus metadata

Recommended local representation:

- vector matrix stored as a NumPy binary file
- vocabulary stored as JSON
- categories stored as JSON
- synonyms stored as JSON

Reasons:

- Very small runtime dependency surface.
- Easy to inspect and debug.
- Fast startup.
- Simple compatibility with NumPy.
- Avoids introducing HDF5, SQLite, or an ANN library for a dataset this small.

Memory mapping may be used if convenient, but it is not essential because the entire matrix is tiny.

The loader should verify that the number of vocabulary entries matches the number of vectors before allowing the application to start.

---

## 7. Vector Search Design

### 7.1 Vector Normalization

All ingredient vectors should be L2-normalized once after loading.

This makes cosine similarity equivalent to a dot product and keeps search logic simple.

### 7.2 Query Construction

The default query should use a normalized centroid of the seed ingredients.

Recommended process:

1. Resolve each seed ingredient to a vocabulary entry.
2. Retrieve its normalized vector.
3. Average the vectors.
4. Normalize the final query vector.
5. Rank all ingredients by cosine similarity.

This is the most useful default because it is simple, interpretable, and directly matches the intended use of the embedding space.

### 7.3 Weighted Seeds

The engine should support optional seed weights.

Example use cases:

- "mostly tomato and basil, with a little chili"
- "like beef stew, but lighter"
- follow-up requests where one part of the previous result should become more important

Weighted search should be supported in the engine even if the first CLI version does not expose manual weights to users.

### 7.4 Negative Direction

Negative vector weighting can be used as a secondary mechanism for stylistic adjustments such as:

- less rich
- less sweet
- less smoky

However, negative vector arithmetic should not be used as the primary safety or dietary filtering mechanism.

Hard constraints must be applied as actual filters.

### 7.5 Advanced Rotation / Creative Mode

The rotation approach described in one plan can be added later as an experimental creativity feature.

It should not be part of the first implementation because:

- it is harder to explain to users,
- it increases tuning complexity,
- and simple centroid search already satisfies the main product goal.

---

## 8. Constraint and Filtering Engine

Filtering is one of the most important parts of the system.

The LLM should not be responsible for manually naming every ingredient in a restricted class.

### 8.1 Primary Strategy: category-based hard filtering

Python should maintain or load mappings such as:

- Seafood
- Meat
- Pork
- Dairy
- Egg
- Nuts
- Gluten-containing grains
- Alcohol

When the user says something such as "no seafood," the interpreter should return a category-level exclusion. Python then removes every matching vocabulary item.

If Epicure-Core does not provide all required category metadata directly, the project should maintain a curated `categories.json` file.

### 8.2 Specific Ingredient Exclusions

The user may also exclude specific ingredients such as:

- cilantro
- mushroom
- peanut
- anchovy

These should be resolved through the synonym layer and removed directly.

### 8.3 Synonym Expansion

A curated synonym file should normalize common variants before search and filtering.

Examples:

- scallion -> green onion
- chickpea -> garbanzo bean, depending on vocabulary
- aubergine -> eggplant

The synonym map should also support category terms when useful.

### 8.4 Embedding-Based Exclusion

Similarity-based exclusion can exist as a fallback for unusual terms that are not represented in the curated metadata.

It should not be trusted for strict allergies or dietary restrictions because semantic similarity does not guarantee category membership.

### 8.5 Allergy Handling

The application can respect user-stated exclusions at the ingredient-search level, but it should not present itself as medically reliable for severe allergy decisions.

The interface should avoid claims such as "allergy-safe." Its guarantee is only that known matching vocabulary items were filtered according to the configured metadata.

---

## 9. LLM-to-Python Interface

### Decision: Ollama native tool calling as the primary interface

The interpreter should return one structured tool call rather than prose.

The tool payload should contain the following conceptual fields:

- seed ingredients
- optional seed weights
- excluded categories
- excluded ingredients
- short vibe or intent summary
- optional intensity value
- requested number of neighbors

Native tool calling is preferable because it minimizes fragile parsing logic.

### Fallback: JSON mode

If a selected model does not reliably support tool calls, the same schema should be requested as JSON and validated by Python.

Validation should reject malformed structures and unknown field types rather than silently guessing.

A ReAct-style text protocol should not be used. It adds complexity without providing a benefit for this project.

---

## 10. Interpreter Prompt Behavior

The interpreter prompt should be short and operational.

It should teach the model the following rules:

### Abstract language becomes concrete food concepts

The model may use its general knowledge to map moods and settings into common ingredients.

For example:

- rainy, cozy, winter -> warm savory ingredients
- bright, fresh, summer -> citrus, herbs, fresh produce
- rich, romantic -> deeper or more indulgent ingredients

These mappings should be examples, not a rigid lookup table.

### Use common canonical ingredient names

The model should prefer common vocabulary likely to exist in the embedding space.

It should avoid:

- complete dish names
- brands
- preparation descriptions
- non-food mood words as vector seeds

### Constraints are handled first

Before choosing seeds, the model should identify requirements such as:

- vegan
- vegetarian
- no seafood
- dairy-free
- no mushrooms

The model should express broad restrictions as categories whenever possible.

### One interpretation call per user turn

The interpreter should produce a single structured request per turn.

This keeps latency predictable and avoids autonomous tool loops.

### Preserve the user's vibe separately

A short natural-language intent summary should be passed through the system so the final formatter can retain the emotional or stylistic character of the request without contaminating the vector query with non-food words.

---

## 11. Out-of-Vocabulary Handling

Out-of-vocabulary ingredients are expected and should be handled explicitly.

Recommended resolution order:

1. Exact vocabulary match.
2. Canonicalized token match.
3. Synonym map.
4. Interpreter-provided replacement if clearly appropriate.
5. Drop the seed and record it as unresolved.

The engine should return both:

- recognized seeds
- unresolved seeds

If all seeds fail, the application should not invent a query vector. It should either retry interpretation with safer/common ingredients or ask the user for clarification.

The final response may briefly mention that a term was normalized or substituted when this materially changes the interpretation.

---

## 12. Search Result Contract

The vector engine should return structured data rather than formatted text.

Useful fields include:

- normalized seed ingredients
- excluded categories
- excluded ingredients
- unresolved input terms
- ranked neighbors
- similarity scores
- ingredients removed by filtering
- search settings used

The formatter should receive only the information it needs.

Similarity scores may be available in debugging mode, but the normal user experience does not need to expose raw vector values unless they improve the explanation.

---

## 13. Recipe Generation Rules

The second LLM pass should produce a practical, concise recipe.

Recommended output:

1. Dish name
2. One short explanation of why the ingredients fit the request
3. Ingredient list
4. Approximately 4-8 preparation steps
5. A short closing line connecting the dish back to the requested vibe

### Grounding rule

Main recipe ingredients must come from the returned vector neighborhood.

A short whitelist of universal staples may be used without appearing in the vector results, for example:

- salt
- water
- black pepper
- neutral oil or olive oil
- garlic
- onion

The whitelist should stay small. Otherwise the LLM can bypass the embedding system and generate an ordinary recipe from memory.

### Similarity explanations

Raw cosine scores should not dominate the recipe output.

They may optionally be used for one or two explanatory statements, especially in a debug or "why this works" mode.

The default recipe should remain readable and food-focused.

### Chemistry and cultural claims

The formatter should avoid making unsupported claims about flavor chemistry or cultural authenticity simply because two ingredients are close in an embedding space.

The embedding captures relationships learned from its training data; similarity is not automatically proof of a particular chemical mechanism or traditional pairing.

---

## 14. Conversation State and Follow-Ups

The application should keep lightweight state for the current session.

Useful stored values:

- original user request
- interpreted vibe
- most recent seed ingredients
- most recent query vector or centroid
- current exclusions
- most recent neighbor list

This enables follow-up requests such as:

- "make it spicier"
- "less creamy"
- "now vegetarian"
- "something similar but for breakfast"
- "more like the last one"

A follow-up should still generate one fresh structured interpretation request, but the LLM may receive the previous state as context.

For "more like this," the system may reuse the previous query centroid as an additional search signal.

---

## 15. Terminal Experience

The first version should stay intentionally simple.

### Default mode

Interactive REPL:

- user types a request
- application shows a short processing indicator
- recipe is printed cleanly
- user may continue with a follow-up

### Optional CLI flags

Useful options may include:

- model selection
- number of neighbors
- debug mode
- show similarity scores
- creative mode
- one-shot versus interactive mode

A full graphical or web interface should be deferred until the search and interpretation behavior is stable.

---

## 16. Testing Strategy

Testing should cover the deterministic engine and the probabilistic LLM layer separately.

### 16.1 Unit Tests

Test:

- tensor loading
- vocabulary alignment
- normalization
- cosine ranking
- weighted queries
- category exclusions
- specific ingredient exclusions
- synonym resolution
- empty inputs
- out-of-vocabulary terms

### 16.2 Property Tests

Important invariants:

- excluded ingredients never appear in final neighbors
- returned neighbor count never exceeds the requested limit
- all returned ingredients exist in the vocabulary
- category exclusions are consistently applied
- similarity scores are sorted correctly

### 16.3 Golden Prompt Tests

Maintain a small collection of representative requests such as:

- rainy day comfort without seafood
- vegetarian and fresh
- rich date-night food
- spicy but no mushroom
- bright breakfast

The test does not need one exact recipe. It should verify that:

- the interpretation is reasonable
- required exclusions are recognized
- expected broad ingredient families appear in the search results

### 16.4 Model Comparison

Run the same prompt set against supported local models and compare:

- valid tool-call rate
- vocabulary hit rate
- constraint accuracy
- number of retries
- latency
- recipe grounding

Qwen should remain the default unless testing shows another model performs better on these concrete metrics.

### 16.5 Latency Tests

Track separately:

- interpreter latency
- vector-search latency
- formatter latency

The vector-search portion should be effectively negligible relative to local LLM inference.

---

## 17. Failure Modes

### No valid seed ingredients

Ask for clarification or retry interpretation using a restricted common-ingredient vocabulary.

### Too few neighbors after filtering

Return fewer ingredients and create a simpler dish rather than relaxing hard constraints automatically.

### Contradictory request

Examples:

- vegan with eggs
- dairy-free but specifically requesting cream

The system should ask one clarification question before running the search.

### Model returns malformed structured output

Validate the result and retry once using a stricter schema-oriented prompt or JSON fallback.

### Ollama unavailable

Return a clear local error and explain that the language interpretation layer is unavailable. The vector engine itself should remain testable independently.

### Missing or corrupt model data

Fail during startup with a validation error rather than producing partial search results.

---

## 18. Recommended Technology Decisions

| Area | Final Choice | Why |
|---|---|---|
| Python environment | `uv` | Fast, reproducible, lightweight |
| Deployment | Native local first | Simplest Ollama integration |
| Optional packaging | Docker later | Useful for demos and distribution |
| Vector storage | NumPy matrix + JSON metadata | Fast, simple, inspectable |
| Similarity engine | NumPy cosine search | Dataset is too small to require ANN |
| Query vector | Normalized centroid | Simple and interpretable |
| Advanced query control | Optional weights + negative direction | Useful without overcomplicating default behavior |
| Constraint system | Hard category map + exact ingredient filters + synonyms | More reliable than semantic filtering |
| Semantic exclusion | Fallback only | Not appropriate for strict restrictions |
| Interpreter model | Qwen 2.5 3B Instruct | Better structured/tool behavior in both plans |
| Formatter model | Same Qwen by default, Gemma optional | Simpler baseline with optional prose upgrade |
| LLM bridge | Native Ollama tools | Structured and less brittle |
| Fallback bridge | Validated JSON mode | Model-compatible fallback |
| Terminal framework | Typer + Rich | Useful UI without TUI complexity |
| Full-screen TUI | Deferred | Not required for MVP |
| ANN index | Not needed | Only 1,790 vectors |
| Creative rotation | Future option | Interesting but not essential |

---

## 19. Build Order

### Phase 1: Data and Vector Engine

Build and verify:

- model download/import
- cached vector representation
- vocabulary lookup
- normalized similarity search
- exact exclusions
- category filtering
- synonym resolution

At the end of this phase, the search engine should work without any LLM.

### Phase 2: Structured LLM Interpreter

Add:

- Ollama integration
- interpreter prompt
- native tool calling
- schema validation
- JSON fallback
- out-of-vocabulary handling

At the end of this phase, natural-language requests should reliably become valid search queries.

### Phase 3: Recipe Formatter

Add the second LLM pass and enforce grounding to the returned neighbors.

### Phase 4: Terminal UX and State

Add:

- Rich formatting
- interactive history
- follow-up context
- debug output
- model selection

### Phase 5: Validation and Packaging

Add:

- golden prompt suite
- property tests
- model comparison
- latency measurements
- README
- optional Docker setup

---

## 20. MVP Scope

The first usable version should support:

- local Ollama model
- one terminal command or interactive REPL
- abstract vibe interpretation
- 2-5 seed ingredients
- category and specific ingredient exclusions
- Epicure-Core cosine search
- ranked ingredient neighborhood
- grounded recipe generation
- simple follow-up requests
- clear handling of unknown ingredients and empty results

The MVP should intentionally exclude:

- web UI
- image generation
- approximate nearest-neighbor search
- complex agent loops
- multiple simultaneous vector models
- rotation-based creativity controls
- large recipe databases
- external APIs

---

## 21. Future Enhancements

Once the baseline system is reliable, useful additions include:

### Multiple Epicure spaces

Allow users to switch or blend between:

- core
- chemistry-oriented embeddings
- co-occurrence-oriented embeddings

This could become a simple "tastes similar" versus "commonly cooked together" control.

### Analogy search

Support questions such as:

- vegetarian alternative to a meat ingredient
- ingredient playing a similar role in another cuisine

### Mood interpolation

Allow multiple vibes or seed groups to be blended with explicit weights.

### Ingredient explorer

Add an optional TUI or local web page for exploring nearest neighbors directly.

### Custom Ollama model configuration

Once prompts stabilize, package the interpreter behavior into a dedicated Ollama configuration for easier setup.

### User preference profiles

Store persistent preferences such as vegetarian, no cilantro, low spice, or preferred cuisines locally and automatically merge them into each query.

---

## 22. Final Architecture Summary

The most important architectural decision is to keep language interpretation and ingredient search separate.

The LLM should answer:

> "What concrete food concepts does this user mean?"

The Epicure vector engine should answer:

> "What ingredients are close to those concepts in this embedding space?"

Python should answer:

> "Which of those results are valid under the user's constraints?"

The final LLM pass should answer:

> "How can I turn those validated ingredients into a useful recipe?"

This design keeps the system small, explainable, and practical. It also makes failures easier to debug because interpretation errors, vector-search behavior, filtering errors, and recipe-writing errors can be tested independently.
