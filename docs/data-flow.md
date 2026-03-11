# Data Flow

This document traces the journey of data through OpenEvolve, from the moment you run a command to the moment you get your optimized program back. Each section is a step in the pipeline.

## Overview: The Complete Data Journey

```
 ┌──────────────────────┐
 │  1. USER INPUT        │   initial_program.py + evaluator.py + config.yaml
 └──────────┬───────────┘
            ▼
 ┌──────────────────────┐
 │  2. INITIALIZATION    │   Config parsed → Components created → Initial program evaluated
 └──────────┬───────────┘
            ▼
 ┌──────────────────────────────────────────────────────────────┐
 │  3. EVOLUTION LOOP (repeats for N iterations)                │
 │                                                              │
 │   ┌──────────────┐                                          │
 │   │ 3a. SAMPLE    │  Pick parent program + inspirations      │
 │   └──────┬───────┘                                          │
 │          ▼                                                   │
 │   ┌──────────────┐                                          │
 │   │ 3b. PROMPT    │  Build context-rich prompt for the LLM   │
 │   └──────┬───────┘                                          │
 │          ▼                                                   │
 │   ┌──────────────┐                                          │
 │   │ 3c. GENERATE  │  LLM produces code mutation              │
 │   └──────┬───────┘                                          │
 │          ▼                                                   │
 │   ┌──────────────┐                                          │
 │   │ 3d. PARSE     │  Extract code from LLM response          │
 │   └──────┬───────┘                                          │
 │          ▼                                                   │
 │   ┌──────────────┐                                          │
 │   │ 3e. EVALUATE  │  Run the program and measure fitness     │
 │   └──────┬───────┘                                          │
 │          ▼                                                   │
 │   ┌──────────────┐                                          │
 │   │ 3f. STORE     │  Add to database if novel and fit        │
 │   └──────────────┘                                          │
 │                                                              │
 └──────────────────────────────────────────────────────────────┘
            ▼
 ┌──────────────────────┐
 │  4. OUTPUT            │   Best program + checkpoints + trace logs
 └──────────────────────┘
```

## Step 1: User Input

**What comes in:**
- **Initial program** — A code file (Python, Rust, R, etc.) containing `EVOLVE-BLOCK-START` / `EVOLVE-BLOCK-END` markers around the code to be evolved
- **Evaluator** — A Python file with an `evaluate(program_path)` function that returns a metrics dictionary
- **Config** — A YAML file with settings for LLMs, database, evaluation, etc.

**Example initial program:**
```python
import numpy as np

def optimize(x):
    # EVOLVE-BLOCK-START
    # Simple random search
    best = float('inf')
    for _ in range(100):
        candidate = x + np.random.randn(len(x)) * 0.1
        val = sum(c**2 for c in candidate)
        if val < best:
            best = val
            x = candidate
    return x
    # EVOLVE-BLOCK-END
```

**Example evaluator:**
```python
def evaluate(program_path):
    # Import and run the evolved program
    # Return metrics dict with 'combined_score' (higher = better)
    return {"combined_score": 0.85, "execution_time": 1.2, "complexity": 5}
```

**What happens to it:**
- The CLI (`cli.py`) or API (`api.py`) parses arguments
- Config YAML is loaded and environment variables like `${OPENAI_API_KEY}` are resolved
- CLI overrides (e.g., `--iterations 100`) are merged into the config

**Where the data goes next:** → Controller initialization

---

## Step 2: Initialization

**What the Controller does with the input:**

1. **Reads the initial program file** → stores the code string and detects the programming language
2. **Creates the LLM Ensemble** → initializes OpenAI clients for each configured model, sets API keys and base URLs
3. **Creates the Prompt Sampler** → loads prompt templates from `prompts/` directory
4. **Creates the Database** → initializes empty MAP-Elites grid with configured islands
5. **Creates the Evaluator** → loads the user's evaluation module
6. **Evaluates the initial program** → runs `evaluator.evaluate(initial_program)` to get baseline metrics
7. **Seeds the database** → adds the initial program to all islands as the starting point

**Data created during initialization:**
```python
# The initial Program object
Program(
    id="initial",
    code="import numpy as np\n...",
    language="python",
    parent_id=None,
    generation=0,
    metrics={"combined_score": 0.5, "complexity": 10},
    island_id=0,       # Seeded to all islands
    ...
)
```

**Where the data goes next:** → Evolution loop

---

## Step 3: The Evolution Loop

This is the heart of OpenEvolve. Each iteration follows the same six sub-steps.

### Step 3a: Sample

**What happens:** The database picks which programs to use as context for the LLM.

**Sampling strategy (configurable ratios):**
- **Elite selection** (default 10%): Pick from the top-scoring programs
- **Exploration** (default 20%): Pick random programs
- **Exploitation** (default 70%): Pick from the MAP-Elites archive (best per feature cell)

**Data flow:**
```
Database.sample_from_island(island_id, num_inspirations=3)
    │
    ├── parent: Program      # The program to mutate
    └── inspirations: List[Program]  # Other programs for context
```

**Rationale:** Using a mix of elite + random + archive programs ensures the LLM sees both high-quality and diverse examples, preventing premature convergence.

### Step 3b: Build Prompt

**What happens:** The Prompt Sampler assembles a context-rich prompt from multiple data sources.

**Data that goes into the prompt:**
```
┌─────────────────────────────────────────────┐
│              SYSTEM MESSAGE                  │
│  "You are an expert code optimizer..."       │
│  (From prompts/system_message.txt)           │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│              USER MESSAGE                    │
│                                              │
│  Current Program:                            │
│    [parent code with EVOLVE-BLOCK marked]    │
│                                              │
│  Current Metrics:                            │
│    combined_score: 0.72                      │
│    execution_time: 2.1s                      │
│                                              │
│  Evolution History:                          │
│    Attempt 1: score 0.65 → 0.68 (+0.03)     │
│    Attempt 2: score 0.68 → 0.72 (+0.04)     │
│                                              │
│  Top Performers:                             │
│    Program A (score: 0.89): [code snippet]   │
│    Program B (score: 0.85): [code snippet]   │
│                                              │
│  Diverse Inspirations:                       │
│    Program X (different approach): [code]    │
│    Program Y (novel strategy): [code]        │
│                                              │
│  Improvement Suggestions:                    │
│    "Fitness improved. Keep exploring..."     │
│                                              │
│  Artifacts (optional):                       │
│    [Test results, error logs, etc.]          │
└─────────────────────────────────────────────┘
```

**Two evolution modes:**
1. **Diff-based** (default): LLM returns SEARCH/REPLACE blocks to modify specific parts
2. **Full rewrite**: LLM returns the complete evolved code

**Rationale:** Showing the LLM both high-performing and diverse programs gives it signal about what works (exploitation) and what different approaches exist (exploration). This mirrors how researchers learn from both the best papers and unconventional approaches.

### Step 3c: Generate

**What happens:** The LLM ensemble picks a model and generates a response.

**Data flow:**
```
LLMEnsemble.generate_with_context(system_message, user_message)
    │
    ├── Model selection: weighted random (e.g., 80% small model, 20% large)
    ├── API call: OpenAI-compatible endpoint
    ├── Retry logic: up to 3 retries with exponential backoff
    │
    └── Returns: raw LLM response string
```

**Example LLM response (diff-based):**
```
I'll improve the optimization algorithm by using simulated annealing...

<<<SEARCH
    best = float('inf')
    for _ in range(100):
        candidate = x + np.random.randn(len(x)) * 0.1
===REPLACE
    best = float('inf')
    temperature = 1.0
    for i in range(200):
        candidate = x + np.random.randn(len(x)) * temperature
        temperature *= 0.995
>>>
```

### Step 3d: Parse

**What happens:** The raw LLM response is parsed to extract the actual code changes.

**For diff-based evolution:**
```
extract_diffs(response) → [(search_block, replace_block), ...]
apply_diff(parent_code, diffs) → new_code
```

**For full rewrite:**
```
parse_full_rewrite(response, language="python") → new_code
```

**Validation:**
- Code length must be under `max_code_length` (default: 10,000 chars)
- Code must be non-empty
- For diffs: SEARCH block must match existing code

**Rationale:** Diff-based evolution is preferred because it produces smaller, more focused changes. This is more reliable than full rewrites (which can lose working code) and is easier for the LLM to generate correctly.

### Step 3e: Evaluate

**What happens:** The evolved program is run through the user's evaluator to measure fitness.

**Simple evaluation:**
```
evaluator.evaluate(program_path) → {"combined_score": 0.78, "execution_time": 1.8}
```

**Cascade evaluation (if enabled):**
```
Stage 1: evaluate_stage1(path) → {"score": 0.6}     # Quick syntax/sanity check
    ↓ (score ≥ 0.5? Continue)
Stage 2: evaluate_stage2(path) → {"score": 0.7}     # Basic correctness tests
    ↓ (score ≥ 0.75? Continue)
Stage 3: evaluate_stage3(path) → {"score": 0.85}    # Full benchmark
    ↓
Combined metrics returned
```

**Artifact capture:**
Programs can return an `EvaluationResult` instead of a plain dict:
```python
return EvaluationResult(
    metrics={"combined_score": 0.85},
    artifacts={
        "test_output": "All 15 tests passed",
        "error_log": "",
        "visualization.png": <bytes>
    }
)
```

Small artifacts (< 10KB) are stored in the database. Larger ones are saved to disk.

**Rationale:** Cascade evaluation is a compute-efficiency optimization. If a program can't pass basic checks (Stage 1), there's no point running expensive benchmarks (Stage 3). This can save 80%+ of evaluation time.

### Step 3f: Store

**What happens:** The evaluated program is added to the database if it passes quality and novelty checks.

**Data created:**
```python
Program(
    id="prog_42",
    code="import numpy as np\n...",  # The evolved code
    parent_id="prog_37",             # Which program it came from
    generation=5,                     # How many ancestors since initial
    metrics={
        "combined_score": 0.78,
        "execution_time": 1.8,
        "complexity": 7
    },
    island_id=2,                      # Which island it belongs to
    artifacts_json='{"test_output": "..."}',
    prompts={"system": "...", "user": "..."},
    ...
)
```

**Database insertion logic:**
1. Calculate feature coordinates (map metrics to MAP-Elites grid bins)
2. Check novelty (is this program meaningfully different from existing ones?)
3. If the feature cell is empty → insert directly
4. If the feature cell has a program → replace only if new program has higher fitness
5. Enforce population limits (evict oldest programs if at capacity)

**Where the data goes next:** → Back to step 3a for the next iteration, or → Output if this is the last iteration

---

## Step 4: Output

**What gets produced:**

### Best Program
The highest-scoring program found across all iterations and islands:
```python
EvolutionResult(
    best_program=Program(...),
    best_score=0.922,
    best_code="import numpy as np\ndef optimize(x):\n...",
    metrics={"combined_score": 0.922, "execution_time": 0.8},
    output_dir="openevolve_output/"
)
```

### Checkpoints (saved every N iterations)
```
openevolve_output/
├── checkpoints/
│   ├── checkpoint_10/
│   │   ├── database.json          # Full database state
│   │   ├── metadata.json          # Iteration number, timestamp
│   │   └── programs/
│   │       ├── prog_1/
│   │       │   ├── program.json   # Code + metrics
│   │       │   └── artifacts/     # Saved artifacts
│   │       └── prog_2/
│   │           └── ...
│   └── checkpoint_20/
│       └── ...
├── logs/
│   └── evolution.log              # Detailed log file
└── evolution_trace.jsonl          # Per-iteration trace data
```

### Evolution Trace (optional)
Each iteration logs a trace entry:
```json
{
    "iteration": 42,
    "timestamp": 1709234567.89,
    "parent_id": "prog_37",
    "child_id": "prog_42",
    "parent_metrics": {"combined_score": 0.72},
    "child_metrics": {"combined_score": 0.78},
    "improvement_delta": {"combined_score": 0.06},
    "island_id": 2,
    "generation": 5
}
```

---

## Parallel Data Flow

When running with multiple workers, the data flow looks like this:

```
Controller
    │
    ├── Takes database snapshot (serializable copy)
    │
    ├── Worker 1: snapshot → sample → prompt → LLM → evaluate → result
    ├── Worker 2: snapshot → sample → prompt → LLM → evaluate → result
    └── Worker 3: snapshot → sample → prompt → LLM → evaluate → result
    │
    ├── Collects all results
    ├── Adds successful programs to database (sequentially, thread-safe)
    └── Continues to next batch of iterations
```

**Key insight:** Workers operate on a **snapshot** of the database, not the live database. This means:
- Workers don't block each other
- Workers don't see each other's results (until the next batch)
- The database is only modified by the Controller (thread-safe)

---

## Checkpoint & Resume Data Flow

**Saving a checkpoint:**
```
Controller._save_checkpoint(path)
    ├── Serialize database state (all programs, islands, feature maps)
    ├── Save as database.json
    ├── Save per-program files (code, metrics, artifacts)
    └── Save metadata (iteration number, config, timestamp)
```

**Resuming from a checkpoint:**
```
Controller._load_checkpoint(path)
    ├── Load database.json → reconstruct ProgramDatabase
    ├── Load per-program files → reconstruct Program objects
    ├── Restore island assignments, MAP-Elites grid
    └── Continue iteration from saved number
```

**Rationale:** Checkpointing enables long-running experiments to be interrupted and resumed. This is essential for evolution runs that may take hours or days.

---

## Island Migration Data Flow

Every `migration_interval` generations (not iterations), top programs migrate between islands:

```
Island 0 ──top 10%──▶ Island 1
Island 1 ──top 10%──▶ Island 2
Island 2 ──top 10%──▶ Island 3
Island 3 ──top 10%──▶ Island 4
Island 4 ──top 10%──▶ Island 0
```

**What gets migrated:** Copies of top-performing Program objects (the originals stay in their island).

**Rationale:** Islands evolve independently most of the time (diversity), but periodically share their best discoveries (knowledge transfer). This mirrors biological island biogeography — populations on separate islands develop unique traits, but occasional mixing introduces beneficial variation.
