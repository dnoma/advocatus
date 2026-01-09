# Advocatus: Argumentative LLMs via Quantitative Bipolar Argumentation

Implementation of Argumentative LLMs based on the ArgLLM paper (arXiv:2405.02079). This project implements claim verification using **Quantitative Bipolar Argumentation Frameworks (QBAFs)**.

## Core Idea

Instead of asking an LLM directly "True or False?", generate structured arguments for and against a claim, then compute a **gradual strength score** through structured argumentation semantics.

```
                          ┌─────────────────────┐
                          │      📝 CLAIM       │
                          └──────────┬──────────┘
                                     │
           ┌─────────────────────────┼─────────────────────────┐
           │                         │                         │
           ▼                         ▼                         ▼
┌─────────────────────┐   ┌─────────────────────┐   ┌─────────────────────┐
│      SUPPORTS       │   │       OUTPUT        │   │      ATTACKS        │
├─────────────────────┤   ├─────────────────────┤   ├─────────────────────┤
│ Arg A  [τ=0.8] ─────┼───┤  TRUE / FALSE       │   ├─────▶ Arg C [τ=0.7] │
│ Arg B  [τ=0.6] ─────┼───┤  Strength: 0.73     │   ├─────▶ Arg D [τ=0.5] │
│                     │   │                     │   │                     │
│  F() = 1-∏(1-vi)    │   │  📋 Auditable       │   │  F() = 1-∏(1-vi)    │
│  σ = C(τ,va,vs)     │   │  Debate + Citations │   │  σ = C(τ,va,vs)     │
└─────────────────────┘   └─────────────────────┘   └─────────────────────┘
```

## Installation

```bash
pip install -e ".[dev]"
```

## Project Structure

```
advocatus/
├── src/
│   ├── frameworks/
│   │   ├── baf.py              # BAF: ⟨A, R−, R+⟩ (arguments + attack/support relations)
│   │   └── qbaf.py             # QBAF: ⟨A, R−, R+, τ⟩ (BAF + base scores)
│   ├── semantics/
│   │   ├── base.py             # Abstract gradual semantics interface
│   │   ├── df_quad.py          # DF-QuAD algorithm (deterministic)
│   │   └── qem.py              # Quadratic Energy Model (alternative)
│   ├── components/
│   │   ├── argument_generation.py    # Γ(x, G, θ) → BAF
│   │   ├── strength_attribution.py   # E(B, E) → QBAF
│   │   └── strength_calculation.py   # Σ(x, Q, σ) → σQ(x)
│   ├── prompts/
│   │   ├── argument_gen.py     # Prompt for generating supporting/attacking arguments
│   │   ├── strength_arg.py     # Prompt for estimating argument strength
│   │   └── strength_claim.py   # Prompt for estimating claim strength
│   └── pipeline.py             # Full pipeline: Claim → QBAF → True/False
├── experiments/
│   ├── configs/                # Experiment configurations
│   ├── baselines/              # Baseline comparison methods
│   └── run.py                  # Experiment runner
├── tests/
│   ├── test_df_quad.py
│   ├── test_qbaf.py
│   └── test_contestability.py
└── pyproject.toml
```

## DF-QuAD Semantics

The core mathematical foundation for computing argument strengths:

### Aggregation Function F

```python
F(v1, ..., vn) = 0 if n = 0, else 1 - ∏(1 - vi)
```

Represents disjunction of attacker/supporter strengths. If any attacker/supporter has high strength, the aggregated strength is high.

### Combination Function C

```python
C(v0, va, vs):
    if va = vs: return v0
    if va > vs: return v0 - (v0 * |vs - va|)
    if va < vs: return v0 + ((1 - v0) * |vs - va|)
```

Balances base score with aggregated attackers (va) vs supporters (vs):
- Supporters increase strength proportionally to `(1 - v0)`
- Attackers decrease strength proportionally to `v0`

### Final Strength σ

```python
σ(α) = C(τ(α), F(attacker_strengths), F(supporter_strengths))
```

## QBAF Structure

A Quantitative Bipolar Argumentation Framework is defined as:

- **A**: Set of arguments (strings)
- **R⁻**: Attack relations, R⁻ ⊆ A × A
- **R⁺**: Support relations, R⁺ ⊆ A × A
- **τ**: Base score function, τ: A → [0, 1]

Restricted to **tree structure** rooted at the claim.

## Experiment Variations

| Config | Depth | Base Score |
|--------|-------|------------|
| depth1_05base | 1 (claim + 1 attacker + 1 supporter) | 0.5 (neutral) |
| depth1_estbase | 1 | LLM estimates claim strength |
| depth2_05base | 2 (7 arguments total) | 0.5 |
| depth2_estbase | 2 | LLM estimates |

**Decision rule**: Claim is `True` if final strength > 0.5, else `False`

## Baselines

For comparison with standard approaches:

- **direct_question.py**: "Is this claim true or false?"
- **est_confidence.py**: "How confident are you on a scale of 0-1?"
- **chain_of_thought.py**: Step-by-step reasoning before answering

## Components

| Component | Function | Signature |
|-----------|----------|-----------|
| Argument Generation | Generate supporting/attacking arguments | Γ(x, G, θ) → BAF |
| Strength Attribution | Estimate base scores | E(B, E) → QBAF |
| Strength Calculation | Compute final verification | Σ(x, Q, σ) → σQ(x) |

## Running Experiments

```bash
# Run all experiments
python experiments/run.py --config experiments/configs/depth1_05base.yaml

# Run with specific config
python experiments/run.py --config experiments/configs/depth2_estbase.yaml
```

## Testing

```bash
pytest tests/ -v
```

## References

- ArgLLM: Argumentative Reasoning with Large Language Models (arXiv:2405.02079)
