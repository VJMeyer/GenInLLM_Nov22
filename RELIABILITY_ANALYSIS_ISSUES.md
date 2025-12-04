# Reliability Analysis Issues: Deep Investigation

## Executive Summary

After thorough investigation of the reliability analysis pipeline, I've identified several fundamental methodological issues that explain why both real and simulated data show high reliabilities in some scales.

## Issue 1: Inverted Unit of Analysis (Critical)

### The Problem

In `analysis_reliability.ipynb`, Cronbach's alpha is computed with:

```python
df_wide = exp_data.pivot_table(
    index="model", columns="item", values=score)
alpha, ci = pg.cronbach_alpha(df_wide)
```

This creates a matrix where:
- **Rows** = LLM models (46 different models)
- **Columns** = Items (e.g., 11 items for AUDIT)

### Why This Is Wrong

Standard psychometric reliability analysis:
- Rows = Participants (individuals being assessed)
- Columns = Items
- α measures whether items correlate *within* individuals across items

Current analysis:
- Rows = Different LLM models (not samples from a population)
- Columns = Items
- α measures whether **different LLMs agree on items**

### Consequence

High Cronbach's alpha here means: "All LLMs tend to give similar responses to the same items."

This does NOT mean: "Each individual LLM has internally consistent personality traits."

All LLMs showing high agreement is expected because they:
1. Share similar training data (internet text)
2. Are optimized for coherent, helpful responses
3. Converge on socially desirable or "obvious" answers
4. Inherit similar biases from web-scale training

## Issue 2: Incomplete Simulation Pipeline

### Observation

Looking at the reliability outputs:

| Task | Real Data α | Simulated Data α | Status |
|------|-------------|------------------|--------|
| AUDIT | 0.739466 | 0.739466 | IDENTICAL |
| BARRAT BISa | 0.830566 | 0.830566 | IDENTICAL |
| BART | 0.984716 | 0.984716 | IDENTICAL |
| CCT | 0.984565 | -0.233085 | DIFFERENT |
| DFD gain | 0.971150 | 0.022746 | DIFFERENT |

Survey tasks and BART show identical reliability in both conditions. Behavioral tasks (CCT, DFD, etc.) show properly different values.

### Root Cause

The simulation pipeline is incomplete:

1. `random_simulation_data_generation.py` randomizes raw LLM data files
2. But there's NO script to reprocess the randomized raw data into `items_per_LLM_random_simulation.csv`
3. The behavioral task processing notebook (`process_behav_tasks_itemlevel.ipynb`) starts by loading `items_per_LLM.csv`, which already contains survey data
4. Result: Survey data was copied unchanged from real data into the simulation file

### Missing Processing Step

To properly simulate survey tasks, need to:
1. Run randomization script on raw data
2. Re-run `process_survey_tasks_itemlevel.ipynb` on randomized data
3. Re-run `process_behav_tasks_itemlevel.ipynb` on randomized data
4. Save the combined result as `items_per_LLM_random_simulation.csv`

## Issue 3: Score Computation Embeds Human Data Structure

### How Scores Are Computed

From `process_survey_tasks_itemlevel.ipynb`:

```python
score = Σ(human_answer × LLM_probability) / Σ(LLM_probability)
```

This is a weighted average of HUMAN responses, weighted by LLM probabilities.

### Implications

1. If LLM probabilities are randomized uniformly, all human responses get equal weight
2. Score converges to the **mean of human responses** for that item
3. This mean is constant across all models
4. Result: Near-zero between-model variance → unstable alpha

The human data distribution creates "structural patterns" that persist even with random LLM weights.

## Issue 4: Conceptual Problems with "LLM Personality"

### What Would Valid Reliability Analysis Look Like?

To measure whether an individual LLM has "reliable personality traits":

1. **Test-retest reliability**: Run the same LLM multiple times with different random seeds/temperatures
2. **Within-model consistency**: Compute alpha across trials/participants for a single model
3. **Parallel forms**: Present same construct with different phrasings

### What Current Analysis Measures

The current analysis measures **inter-model agreement**, which tells us:
- LLMs are similar to each other (expected)
- NOT that individual LLMs have stable traits

### Sample Size Concern

With only 46 LLM models as "subjects," the sample size is marginal for stable alpha estimates, especially for scales with many items.

## Recommendations

### 1. Correct the Unit of Analysis

Option A: Within-model reliability
```python
# For each model, compute alpha across participants
for model in models:
    model_data = data[data['model'] == model]
    df_wide = model_data.pivot_table(
        index="participant", columns="item", values="score")
    alpha = pg.cronbach_alpha(df_wide)
```

Option B: Treat models as separate experiments
- Compute descriptive statistics per model
- Don't aggregate alpha across models

### 2. Fix the Simulation Pipeline

Create a new script that:
1. Loads randomized raw data
2. Re-runs full processing pipeline for both survey and behavioral tasks
3. Saves properly randomized `items_per_LLM_random_simulation.csv`

### 3. Appropriate Null Model

Randomize at the level that should destroy construct validity:
- Shuffle `human_number` values within items
- OR shuffle participant mappings between items
- OR use pure random scores (not weighted averages)

### 4. Consider Alternative Metrics

- ICC (Intraclass Correlation Coefficient) with appropriate model
- Test-retest correlation within single LLMs
- Convergent/discriminant validity with clear hypotheses

## Technical Details

### Files Examined

- `data_analysis/analysis_reliability.ipynb` - Main reliability analysis
- `data_analysis/analysis_reliability_simulation_data.ipynb` - Simulation analysis
- `data_analysis/random_simulation_data_generation.py` - Randomization script
- `data_analysis/process_survey_tasks_itemlevel.ipynb` - Survey task processing
- `data_analysis/process_behav_tasks_itemlevel.ipynb` - Behavioral task processing
- `data_generation/base_task.py` - Base task class with score computation

### Key Code Locations

- Cronbach's alpha computation: `analysis_reliability.ipynb` cells 887b0ff9, 593fc36f
- Score computation: `process_survey_tasks_itemlevel.ipynb` cells 72-97
- Randomization: `random_simulation_data_generation.py` lines 28-57

## Conclusion

The high reliability values in both real and simulated data are artifacts of:
1. Computing alpha with models as "subjects" rather than individuals
2. An incomplete simulation that preserved original survey data
3. Score computation that embeds human data structure

These issues don't invalidate the entire project, but the reliability analyses need to be reconceptualized with appropriate units of analysis to answer meaningful questions about LLM "personality" consistency.
