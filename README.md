# How Boltz Configuration Parameters Affect Scores

This document explains how each parameter in `boltz/boltz_config.yaml` affects the Boltz score calculation.

## Overview

Boltz2 performs two main prediction tasks:
1. **Structure prediction** - Predicts the 3D structure of the protein-ligand complex
2. **Affinity prediction** - Predicts binding affinity (the score used for ranking)

Different parameters control these two stages, and they have different impacts on the final score.

---

## Structure Prediction Parameters

These parameters affect the quality of the predicted 3D structure, which indirectly influences affinity predictions.

### `recycling_steps: 3`

**What it does:**
- Number of times the model refines its predictions iteratively
- Each recycling step uses the previous prediction as input to improve accuracy

**Impact on score:**
- **Higher values** (e.g., 5-8): More accurate structure predictions, potentially better affinity estimates
- **Lower values** (e.g., 1-2): Faster computation but less refined structures
- **Current setting (3)**: Balanced trade-off between accuracy and speed
- **Direct impact**: Moderate - affects structure quality which influences affinity prediction

**Trade-off:** More recycling = better accuracy but slower computation

### `sampling_steps: 100`

**What it does:**
- Number of steps in the diffusion process for structure generation
- Controls how finely the model samples the conformational space

**Impact on score:**
- **Higher values** (e.g., 200-500): More thorough exploration of possible structures, potentially better predictions
- **Lower values** (e.g., 50-100): Faster but may miss optimal conformations
- **Current setting (100)**: Moderate sampling depth
- **Direct impact**: Moderate - affects structure diversity and quality

**Trade-off:** More steps = better coverage but slower

### `diffusion_samples: 1`

**What it does:**
- Number of independent structure predictions to generate
- Multiple samples can be averaged for better accuracy

**Impact on score:**
- **Higher values** (e.g., 3-5): Multiple predictions can be averaged, reducing variance
- **Lower values** (1): Single prediction, faster but potentially more variable
- **Current setting (1)**: Fastest option, single prediction
- **Direct impact**: Low for structure, but affects confidence in predictions

**Trade-off:** More samples = more stable predictions but slower

---

## Affinity Prediction Parameters

These parameters **directly** affect the affinity score that is used for ranking molecules.

### `sampling_steps_affinity: 100`

**What it does:**
- Number of diffusion steps specifically for affinity prediction
- Separate from structure prediction - this controls the affinity model's sampling

**Impact on score:**
- **Higher values** (e.g., 200-500): More thorough sampling for affinity prediction
- **Lower values** (e.g., 50-100): Faster but potentially less accurate affinity estimates
- **Current setting (100)**: Moderate sampling for affinity
- **Direct impact**: **HIGH** - directly affects the affinity prediction quality

**Note:** Affinity prediction uses its own separate model with `recycling_steps: 5` (hardcoded in the code)

### `diffusion_samples_affinity: 3`

**What it does:**
- Number of independent affinity predictions to generate
- Multiple samples are averaged to get the final affinity score

**Impact on score:**
- **Higher values** (e.g., 5-10): More samples averaged = more stable and accurate scores
- **Lower values** (e.g., 1-2): Faster but potentially more variable scores
- **Current setting (3)**: Moderate averaging for stability
- **Direct impact**: **HIGH** - directly affects score stability and accuracy

**How it works:**
```python
# The model generates multiple affinity predictions
# These are averaged to get the final score
final_affinity = mean([affinity_sample_1, affinity_sample_2, affinity_sample_3])
```

**Trade-off:** More samples = more stable scores but slower computation

### `affinity_mw_correction: true`

**What it does:**
- Applies a molecular weight (MW) correction to affinity predictions
- Accounts for the known correlation between molecular weight and binding affinity

**Impact on score:**
- **When enabled (true)**: Applies correction formula:
  ```
  corrected_affinity = 1.035 * raw_affinity - 0.600 * (MW^0.3) + 2.833
  ```
- **When disabled (false)**: Uses raw model predictions without correction
- **Current setting (true)**: Correction applied
- **Direct impact**: **HIGH** - directly modifies the final affinity score

**Why it matters:**
- Larger molecules tend to have higher binding affinities simply due to size
- This correction normalizes for molecular weight, making scores more comparable
- **This is a significant factor** - it can shift scores by several units

**Example:**
- Without correction: Large molecule might score 8.5
- With correction: Same molecule might score 7.2 (adjusted for size)

---

## Other Parameters

### `seed: 68`

**What it does:**
- Random seed for deterministic predictions
- Ensures reproducible results

**Impact on score:**
- **Direct impact**: None on score value, but ensures consistency
- Same molecule + same seed = same score (reproducibility)

### `no_kernels: true`

**What it does:**
- Disables optimized CUDA kernels
- Uses standard PyTorch operations instead

**Impact on score:**
- **Direct impact**: None on score value
- **Performance impact**: Slower computation but more compatible across hardware
- Ensures deterministic behavior across different GPU architectures

### `batch_predictions: true`

**What it does:**
- Processes multiple molecules in batches
- More efficient GPU utilization

**Impact on score:**
- **Direct impact**: None on score value
- **Performance impact**: Faster processing when scoring multiple molecules

### `output_format: "pdb"`

**What it does:**
- Format for output structure files

**Impact on score:**
- **Direct impact**: None on score value
- Only affects output file format

### `override: true`

**What it does:**
- Overwrites existing prediction results if they exist

**Impact on score:**
- **Direct impact**: None on score value
- Allows re-running predictions without manual cleanup

### `remove_files: true`

**What it does:**
- Cleans up temporary files after prediction

**Impact on score:**
- **Direct impact**: None on score value
- Disk space management

---

## Summary: Parameters That Directly Affect Scores

### High Impact (Directly modify scores):
1. **`diffusion_samples_affinity: 3`** - Number of affinity samples averaged
2. **`sampling_steps_affinity: 100`** - Quality of affinity sampling
3. **`affinity_mw_correction: true`** - MW normalization (can shift scores significantly)

### Moderate Impact (Affect structure quality, which influences affinity):
1. **`recycling_steps: 3`** - Structure refinement iterations
2. **`sampling_steps: 100`** - Structure sampling depth
3. **`diffusion_samples: 1`** - Structure prediction samples

### No Direct Impact (Performance/Output only):
- `seed`, `no_kernels`, `batch_predictions`, `output_format`, `override`, `remove_files`

---

## Recommendations for Tuning

### To improve score accuracy (slower):
```yaml
recycling_steps: 5
sampling_steps: 200
diffusion_samples: 3
sampling_steps_affinity: 200
diffusion_samples_affinity: 5
affinity_mw_correction: true  # Keep enabled
```

### To speed up computation (slightly less accurate):
```yaml
recycling_steps: 2
sampling_steps: 50
diffusion_samples: 1
sampling_steps_affinity: 50
diffusion_samples_affinity: 1
affinity_mw_correction: true  # Still recommended
```

### Current balanced configuration:
```yaml
recycling_steps: 3
sampling_steps: 100
diffusion_samples: 1
sampling_steps_affinity: 100
diffusion_samples_affinity: 3
affinity_mw_correction: true
```

---

## Code References

- Affinity MW correction: `boltz/src/boltz/model/models/boltz2.py` (lines 687-697)
- Affinity prediction parameters: `boltz/src/boltz/main.py` (lines 1123-1126)
- Structure prediction: `boltz/src/boltz/model/models/boltz2.py` (forward method)

