# Design Validation Lenses

Use these checks during source validation. They operationalize *A Philosophy of Software Design* without turning metrics or stylistic preferences into verdicts.

## 1. Complexity burden

Ask how much developers must know and how often they pay that cost.

- **Change amplification:** Would one conceptual change require edits or coordinated knowledge in several places?
- **Cognitive load:** How many facts, modules, modes, or setup steps must a caller or maintainer hold at once?
- **Unknown unknowns:** Are dependencies, side effects, invariants, or constraints difficult to discover locally?
- **Exposure:** Is the complexity in a frequently read, changed, or widely used path, or isolated behind a stable boundary?

Local complexity matters less when a deep module contains it. High churn, fan-in, or developer traffic can make otherwise moderate complexity expensive.

## 2. Module depth

Ask whether a simple interface hides substantial useful functionality and knowledge.

Inspect both parts of the interface:

- **Formal:** signatures, types, public methods, errors, and configuration
- **Informal:** invariants, ordering constraints, side effects, conventions, and knowledge callers must possess

Then assess:

- **Information hiding:** Is each important design decision localized or leaked across modules?
- **Knowledge-based decomposition:** Are boundaries organized around stable concepts and hidden knowledge, or merely execution order?
- **Abstraction integrity:** Do adjacent layers offer distinct abstractions, or mostly pass calls and variables through?
- **Together or apart:** Would combining related state and behavior simplify the interface, or would separation create a genuinely distinct abstraction?

Long cohesive implementations behind simple interfaces may be strong deep modules. Chains of tiny classes or functions may multiply interfaces and cognitive load.

## 3. Complexity direction

Ask whether complexity is handled once below the interface or repeatedly pushed onto callers.

Look for:

- flags, callbacks, setup sequences, and caller coordination
- repeated defensive checks or scattered special cases
- invalid or partial states that the interface could define away
- configuration that could be inferred or given a safe default
- storage, protocol, framework, or representation details exposed to ordinary callers
- inconsistent names, argument order, contracts, or error behavior

Assess generality by asking whether the interface is the simplest strong abstraction that covers current needs and plausible reuse. Do not reward speculative mechanisms or an interface that makes the current use awkward.

A wrapper is shallow when it adds a layer without hiding information, changing abstraction, or simplifying callers.

## 4. Strategic direction

Ask whether recent work improves the design or tactically adds complexity.

Look for:

- another exception, mode, flag, dependency, or leaked implementation detail
- recurring friction patched locally instead of addressed at the boundary
- duplicated policy likely to diverge
- comments or names that cannot state the abstraction clearly
- changes that preserve behavior but leave future modifications less local

When an interface is unclear, sketch its public contract and explanatory comment. Difficulty stating the contract simply is design evidence. For a non-trivial redesign, compare at least two plausible interface shapes by module depth, information hiding, special-case reduction, consistency, and total cognitive load.

## Interpretation cautions

- High fan-in may identify a valuable stable module; inspect what callers must know.
- High fan-out matters when callers coordinate leaked details, not when orchestration is legitimate.
- File and symbol complexity are partly size signals.
- Duplication matters most when it spreads policy or a design decision and risks divergence.
- Cycles and coupling are contextual; expected orchestration and test relationships are not findings by themselves.
- `SIMILAR_TO` is token similarity, not proof code should be unified.
- Low graph degree or missing edges may reflect dynamic behavior or coverage gaps.
- Comment density is not a quality metric. Judge whether comments add contracts, precision, intuition, or rationale that code cannot express.
- Do not recommend abstraction merely to reduce line counts or metric scores.
- Prefer interface simplicity over implementation simplicity when it lowers total system complexity.

## Redesign discipline

Do not prescribe a redesign without examining callers and constraints. Prefer the smallest direction that:

- localizes knowledge
- deepens a useful module
- simplifies an interface
- removes caller coordination
- defines a special case out of existence

State uncertainty when evidence cannot distinguish a successful deep module from a risky concentration of responsibility.
