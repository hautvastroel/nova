# VALIDATOR CODE ANALYSIS: STAGING AND SCORING PROCESS

## 📋 **EXECUTIVE SUMMARY**

The validator processes miner submissions through **5 distinct stages**, with scores calculated at multiple levels before determining winners based on PSICHIC and Boltz2 models.

---

## 🔄 **STAGE-BY-STAGE PROCESS FLOW**

### **STAGE 1: COMMITMENT RETRIEVAL & DECRYPTION**
**Location:** `neurons/validator/commitments.py`

**Process:**
1. **Retrieve Commitments** (lines 18-53)
   - Queries blockchain for commitments from all miners
   - Filters by block range (between `start_block` and `current_block - no_submission_blocks`)
   - Extracts commitment metadata (UID, block number, GitHub path)

2. **Fetch GitHub Content** (lines 87-121)
   - Downloads encrypted files from GitHub
   - Validates file format: `(int, bytes)` tuple
   - Retrieves commit timestamps for tie-breaking

3. **Decrypt Submissions** (lines 146-157)
   - Decrypts using timelock encryption (Boltz2 Drand)
   - Splits comma-separated molecule strings into arrays
   - **CRITICAL FILTER**: Only accepts submissions with exactly **100 molecules**

**Output:**
```python
uid_to_data = {
    uid: {
        "molecules": ["rxn:4:212581:225947", ...],  # 100 molecules
        "block_submitted": 123456,
        "push_time": "2025-10-28T14:21:17Z"
    }
}
```

**Rejection Criteria:**
- ❌ Wrong number of molecules (not exactly 100)
- ❌ Invalid encrypted format
- ❌ GitHub file not found
- ❌ Decryption failure

---

### **STAGE 2: MOLECULE VALIDATION**
**Location:** `neurons/validator/validity.py`

**Process:**
1. **Duplicate Check** (lines 41-45)
   - Verifies no duplicate molecule names in submission
   - **Rejects entire submission** if duplicates found

2. **Per-Molecule Validation** (lines 47-109)
   For each molecule in the submission:

   a. **Reaction Type Validation** (lines 60-67)
      - Checks if molecule is `rxn:4` or `rxn:5` format
      - **Currently hardcoded to only accept rxn:4 and rxn:5**
      - **Rejects entire submission** if any molecule fails

   b. **SMILES Retrieval** (lines 69-74)
      - Looks up actual molecular structure from database
      - Calls `get_smiles_from_reaction()` to perform chemical reaction
      - **Rejects entire submission** if SMILES retrieval fails

   c. **Heavy Atom Count** (lines 76-80)
      - Counts non-hydrogen atoms in molecule
      - Must be ≥ `min_heavy_atoms` (typically 20)
      - **Rejects entire submission** if any molecule fails

   d. **Rotatable Bonds** (lines 82-94)
      - Counts rotatable bonds using RDKit
      - Must be between `min_rotatable_bonds` (1) and `max_rotatable_bonds` (10)
      - **Rejects entire submission** if any molecule fails

   e. **Uniqueness Check** (lines 96-101)
      - Verifies molecule hasn't been used for weekly target protein
      - Queries Hugging Face dataset for uniqueness
      - **Rejects entire submission** if any molecule already used

3. **Chemical Identity Check** (lines 112-126)
   - Calculates InChIKey for all valid molecules
   - Groups chemically identical molecules
   - **Rejects entire submission** if any duplicates found

4. **Entropy Calculation** (lines 129-141)
   - Calculates MACCS molecular fingerprint entropy
   - Measures molecular diversity in submission
   - Stored for bonus scoring

**Output:**
```python
valid_molecules_by_uid = {
    uid: {
        "smiles": ["CN1CCN(...)", ...],  # Valid SMILES strings
        "names": ["rxn:4:212581:225947", ...]  # Original molecule names
    }
}
score_dict[uid]["entropy"] = 0.85  # MACCS entropy value
```

**Rejection Criteria:**
- ❌ Any duplicate molecule names
- ❌ Any molecule not rxn:4 or rxn:5
- ❌ Any molecule not in database
- ❌ Any molecule with insufficient heavy atoms
- ❌ Any molecule with invalid rotatable bonds
- ❌ Any molecule already used for target protein
- ❌ Any chemically identical molecules

---

### **STAGE 3: PSICHIC SCORING**
**Location:** `neurons/validator/scoring.py`

**Process:**
1. **Protein Selection** (validator.py lines 63-72)
   - Determines challenge proteins from block hash
   - Selects weekly target protein (e.g., "O15379")
   - Selects 8 antitarget proteins randomly

2. **PSICHIC Initialization** (validator.py lines 113-116)
   - Initializes PSICHIC model (protein-ligand binding affinity prediction)
   - Loads trained weights for specific protein

3. **Batch Scoring** (scoring.py lines 16-131)
   - **For each protein** (target + antitarget):
     - Collects all unique SMILES from all UIDs
     - Processes in batches of 32 molecules
     - Scores each batch using PSICHIC model
     - Distributes scores back to all UIDs using the molecule

4. **Score Storage** (scoring.py lines 116-129)
   - Stores scores per UID, per protein, per molecule
   - Structure: `target_scores[protein_idx][molecule_idx]`
   - Structure: `antitarget_scores[protein_idx][molecule_idx]`

**Output:**
```python
score_dict[uid] = {
    "target_scores": [
        [8.5, 7.2, 6.9, ...],  # Scores for target protein 1
        [8.1, 7.5, 6.7, ...]   # Scores for target protein 2 (if multiple)
    ],
    "antitarget_scores": [
        [4.2, 3.8, 3.5, ...],  # Scores for antitarget protein 1
        [4.1, 3.9, 3.6, ...],  # Scores for antitarget protein 2
        ...  # 8 antitarget proteins total
    ],
    "entropy": 0.85
}
```

**Scoring Details:**
- **Target proteins**: Positive scores (higher = better binding)
- **Antitarget proteins**: Positive scores (lower = better, avoid off-target binding)
- **Batch processing**: Efficiently scores unique molecules once, distributes to all UIDs

---

### **STAGE 4: BOLTZ2 SCORING**
**Location:** `boltz/wrapper.py` (referenced in validator.py lines 137-143)

**Process:**
1. **Boltz Initialization** (validator.py lines 138-141)
   - Cleans up PSICHIC model and resets CUDA context
   - Initializes Boltz2 model (structure prediction and scoring)

2. **Boltz Scoring** (validator.py line 143)
   - Scores molecules using Boltz2 model
   - Uses first molecule or random selection (`num_molecules_boltz = 1`)
   - Calculates Boltz entropy for diversity bonus

3. **Boltz Score Storage**
   - Stores Boltz score and entropy separately
   - Used for separate winner determination

**Output:**
```python
score_dict[uid]["boltz_score"] = 0.92
score_dict[uid]["entropy_boltz"] = 0.78
```

---

### **STAGE 5: FINAL SCORE CALCULATION & RANKING**
**Location:** `neurons/validator/ranking.py`

**Process:**

#### **5.1 Per-Molecule Score Calculation** (ranking.py lines 58-94)

For each molecule in each UID's submission:

a. **Average Target Score** (lines 63-69)
   ```python
   avg_target = sum(target_scores_for_mol) / len(target_scores_for_mol)
   ```
   - Averages scores across all target proteins

b. **Average Antitarget Score** (lines 71-77)
   ```python
   avg_antitarget = sum(antitarget_scores_for_mol) / len(antitarget_scores_for_mol)
   ```
   - Averages scores across all antitarget proteins

c. **Combined Score** (line 80)
   ```python
   mol_score = avg_target - (antitarget_weight * avg_antitarget)
   ```
   - Formula: `target - (0.9 × antitarget)`
   - **Example**: `8.5 - (0.9 × 4.2) = 4.72`

d. **Repetition Penalty** (lines 83-94)
   ```python
   if mol_score > molecule_repetition_threshold:
       mol_score = mol_score / (molecule_repetition_weight * count)
   else:
       mol_score = mol_score * (molecule_repetition_weight * count)
   ```
   - Penalizes molecules submitted by multiple miners
   - **Currently disabled** (weight = 0)

#### **5.2 Final Score Aggregation** (ranking.py lines 96-103)

a. **Sum Per-Molecule Scores** (line 99)
   ```python
   final_score = sum(molecule_scores_after_repetition)
   ```
   - Sums all 100 molecule scores

b. **Entropy Bonus** (lines 102-103)
   ```python
   if final_score > entropy_bonus_threshold and entropy is not None:
       final_score = final_score * (1 + (dynamic_entropy_weight * entropy))
   ```
   - Applies bonus if score exceeds threshold (currently 0, so always applies)
   - Formula: `score × (1 + (weight × entropy))`
   - **Dynamic weight**: Increases over epochs (starts at 0.3, increases by 0.007142857 per epoch)

c. **Boltz Entropy Bonus** (lines 105-118)
   ```python
   if boltz_score > threshold_boltz and entropy_boltz is not None:
       boltz_score = boltz_score * (1 + (dynamic_entropy_weight * entropy_boltz))
   ```
   - Similar entropy bonus for Boltz scores

**Output:**
```python
score_dict[uid] = {
    "combined_molecule_scores": [4.72, 3.85, 4.12, ...],  # Per-molecule scores
    "molecule_scores_after_repetition": [4.72, 3.85, 4.12, ...],
    "final_score": 425.8,  # Sum of all molecule scores with entropy bonus
    "boltz_score": 0.95,  # Boltz score with entropy bonus
    "entropy": 0.85,
    "entropy_boltz": 0.78
}
```

#### **5.3 Winner Determination** (ranking.py lines 144-241)

**PSICHIC Winner:**
1. Finds highest `final_score` across all UIDs
2. If tie: Uses earliest `block_submitted` as tiebreaker
3. If still tied: Uses earliest `push_time` (GitHub commit timestamp)
4. If still tied: Uses lowest UID

**Boltz Winner:**
1. Finds highest `boltz_score` across all UIDs
2. Same tie-breaking logic as PSICHIC

**Output:**
```python
winner_psichic = 229  # UID with highest final_score
winner_boltz = 42     # UID with highest boltz_score
```

---

## 📊 **SCORING FORMULA BREAKDOWN**

### **Level 1: Per-Molecule Score**
```
molecule_score = avg_target - (0.9 × avg_antitarget)
```

**Example:**
- Target scores: [8.5, 7.2] → avg = 7.85
- Antitarget scores: [4.2, 3.8, 3.5] → avg = 3.83
- Molecule score = 7.85 - (0.9 × 3.83) = **4.403**

### **Level 2: Submission Score (Before Entropy)**
```
submission_score = sum(all_molecule_scores)
```

**Example:**
- 100 molecules, average score of 4.5
- Submission score = **450.0**

### **Level 3: Final Score (With Entropy Bonus)**
```
final_score = submission_score × (1 + (dynamic_weight × entropy))
```

**Example:**
- Submission score: 450.0
- Entropy: 0.85
- Dynamic weight: 0.3 (starting epoch)
- Final score = 450.0 × (1 + (0.3 × 0.85)) = **564.75**

### **Entropy Weight Evolution**
- **Starting epoch**: 0.3
- **Per epoch increase**: 0.007142857
- **After 140 epochs**: 1.3 (30% bonus → 30% + 130% bonus)
- **Formula**: `weight = 0.3 + (epoch - 18703) × 0.007142857`

---

## 🎯 **KEY VALIDATION RULES**

### **All-or-Nothing Validation**
- **ANY** validation failure → **ENTIRE SUBMISSION REJECTED**
- No partial credit for valid molecules

### **Critical Filters**
1. **Exact molecule count**: Must be exactly 100
2. **Reaction type**: Only rxn:4 and rxn:5 allowed
3. **Database existence**: All molecule IDs must exist
4. **Chemical validity**: All molecules must pass chemical checks
5. **Uniqueness**: No molecules previously used for target protein
6. **No chemical duplicates**: All molecules must be chemically unique

### **Scoring Weights**
- **Antitarget weight**: 0.9 (strong penalty for off-target binding)
- **Entropy start weight**: 0.3 (increases over time)
- **Molecule repetition weight**: 0 (currently disabled)
- **Boltz weight**: 0.5 (50% of incentive to Boltz winner)

---

## 🔍 **SCORING METRICS**

### **PSICHIC Metrics**
- **Metric**: `predicted_binding_affinity` (binding affinity score)
- **Range**: Typically -∞ to positive values
- **Higher is better** for targets
- **Lower is better** for antitargets (penalized with 0.9× weight)

### **Boltz Metrics**
- **Metric**: `affinity_probability_binary` (probability of binding)
- **Range**: 0 to 1 (probability)
- **Higher is better**

### **Entropy Metrics**
- **MACCS Entropy**: Molecular diversity (0 = identical, 1 = maximally diverse)
- **Higher entropy** → Higher bonus
- **Encourages diverse molecule submissions**

---

## 📈 **PERFORMANCE OPTIMIZATIONS**

1. **Batch Processing**: Scores unique molecules once, distributes to all UIDs
2. **Lazy Model Loading**: PSICHIC and Boltz loaded only when needed
3. **GPU Memory Management**: Cleans up models between stages
4. **CUDA Context Reset**: Prevents memory leaks between PSICHIC and Boltz

---

## 🏆 **WINNER SELECTION PROCESS**

1. **Calculate final scores** for all UIDs with valid submissions
2. **Identify maximum scores** for PSICHIC and Boltz
3. **Resolve ties** using:
   - Earliest block number
   - Earliest GitHub push time
   - Lowest UID number
4. **Set weights** on blockchain for winners
5. **Distribute rewards** based on weights

---

## 📝 **SUMMARY**

**Submission Flow:**
```
Miner Submission → Encryption → GitHub → Blockchain Commitment
    ↓
Validator Retrieves → Decrypts → Validates → Scores → Ranks → Sets Weights
```

**Scoring Hierarchy:**
```
Per-Molecule Score → Submission Score → Final Score (with entropy) → Ranking
```

**Critical Success Factors:**
1. ✅ Exactly 100 valid molecules
2. ✅ All molecules exist in database
3. ✅ High target binding, low antitarget binding
4. ✅ High molecular diversity (entropy)
5. ✅ Early submission (tie-breaker)

The validator is **extremely strict** - a single validation failure results in complete submission rejection, making compliance critical for miners.
