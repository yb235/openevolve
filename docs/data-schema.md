# Data Schema

This document describes every data structure used in OpenEvolve — the dataclasses, dictionaries, and object shapes that flow through the system. If you need to understand what a piece of data looks like, this is the reference.

## Core Data Structures

### Program

The fundamental unit in OpenEvolve. Every piece of evolved code is wrapped in a `Program` object.

**Defined in:** `openevolve/database.py`

```python
@dataclass
class Program:
    # === Identity ===
    id: str                           # Unique identifier (e.g., "prog_42")
    code: str                         # The full source code

    # === Lineage ===
    parent_id: Optional[str]          # ID of the parent program (None for initial)
    generation: int                   # Number of ancestors since initial (0 = initial)

    # === Metadata ===
    language: str = "python"          # Programming language (python, rust, r, etc.)
    timestamp: float                  # When this program was created (Unix epoch)
    iteration_found: int              # Which iteration discovered this program

    # === Performance ===
    metrics: Dict[str, float]         # Evaluation results
                                      #   Required: "combined_score" (primary fitness)
                                      #   Optional: any additional metrics
    complexity: float                 # Derived feature (code complexity estimate)
    diversity: float                  # Derived feature (how different from others)

    # === Context ===
    changes_description: str = ""     # LLM-generated summary of accumulated changes
    metadata: Dict[str, Any]          # Free-form metadata (island_id, etc.)
    prompts: Optional[Dict[str, Any]] # The prompts used to generate this program
                                      #   {"system": "...", "user": "..."}

    # === Artifacts ===
    artifacts_json: Optional[str]     # Small artifacts stored inline (< 10KB JSON)
    artifact_dir: Optional[str]       # Path to directory for large artifact files

    # === Novelty ===
    embedding: Optional[List[float]]  # Code embedding vector for similarity checks
```

**Rationale for this design:**
- `id` and `parent_id` form a tree (evolution genealogy) — essential for visualization and analysis
- `metrics` is a flexible dict because different problems have different metrics
- `combined_score` is the required key — it's the primary fitness signal that drives selection
- `changes_description` is a compact LLM-generated summary used when code is too long to include in full in prompts
- `artifacts_json` vs `artifact_dir` is a size-based split: small debugging data stays in memory, large files (images, logs) go to disk
- `embedding` enables fast similarity checks without calling the LLM

### Example Program instance:

```python
Program(
    id="prog_42",
    code="import numpy as np\n\ndef optimize(x):\n    ...",
    parent_id="prog_37",
    generation=5,
    language="python",
    timestamp=1709234567.89,
    iteration_found=42,
    metrics={
        "combined_score": 0.85,
        "execution_time": 1.2,
        "memory_usage": 45.6,
        "complexity": 7.0,
        "diversity": 0.3
    },
    complexity=7.0,
    diversity=0.3,
    changes_description="Added simulated annealing with adaptive cooling schedule",
    metadata={"island_id": 2, "source": "diff_evolution"},
    prompts={"system": "You are an expert...", "user": "Current program..."},
    artifacts_json='{"test_output": "15/15 passed", "stderr": ""}',
    artifact_dir=None,
    embedding=[0.12, -0.34, 0.56, ...]  # 1536-dimensional vector
)
```

---

### EvaluationResult

Standardized output from evaluation functions. Separates metrics (scores) from artifacts (diagnostic data).

**Defined in:** `openevolve/evaluation_result.py`

```python
@dataclass
class EvaluationResult:
    metrics: Dict[str, float]                    # Evaluation scores
    artifacts: Dict[str, Union[str, bytes]]      # Diagnostic outputs (optional)
```

**Methods:**
| Method | Returns | Description |
|--------|---------|-------------|
| `from_dict(metrics)` | `EvaluationResult` | Wraps a plain dict (backward compatibility) |
| `to_dict()` | `Dict[str, float]` | Returns just metrics |
| `has_artifacts()` | `bool` | Whether any artifacts exist |
| `get_artifact_keys()` | `List[str]` | Names of all artifacts |
| `get_artifact_size(key)` | `int` | Size of a specific artifact in bytes |
| `get_total_artifact_size()` | `int` | Total size of all artifacts |

**Rationale:** Before `EvaluationResult`, evaluators returned plain dicts. The new type adds artifact support while maintaining backward compatibility — evaluators can still return plain dicts and the system wraps them automatically.

**Example:**
```python
# Simple evaluator (backward compatible)
def evaluate(program_path):
    return {"combined_score": 0.85, "accuracy": 0.92}

# Advanced evaluator with artifacts
def evaluate(program_path):
    return EvaluationResult(
        metrics={"combined_score": 0.85, "accuracy": 0.92},
        artifacts={
            "test_log": "Test 1: PASS\nTest 2: PASS\n...",
            "plot.png": open("result.png", "rb").read()
        }
    )
```

---

### EvolutionTrace

Records the details of a single evolution step for later analysis.

**Defined in:** `openevolve/evolution_trace.py`

```python
@dataclass
class EvolutionTrace:
    iteration: int                              # Iteration number
    timestamp: float                            # When this occurred
    parent_id: str                              # Source program ID
    child_id: str                               # New program ID
    parent_metrics: Dict[str, Any]              # Parent's scores
    child_metrics: Dict[str, Any]               # Child's scores
    parent_code: Optional[str]                  # Parent source code (if logging enabled)
    child_code: Optional[str]                   # Child source code (if logging enabled)
    parent_changes_description: Optional[str]   # Parent's change summary
    child_changes_description: Optional[str]    # Child's change summary
    code_diff: Optional[str]                    # Diff between parent and child
    prompt: Optional[Dict[str, str]]            # Prompts used {"system": ..., "user": ...}
    llm_response: Optional[str]                 # Raw LLM output
    improvement_delta: Optional[Dict[str, float]]  # Metric changes (child - parent)
    island_id: Optional[int]                    # Which island
    generation: Optional[int]                   # Generation number
    artifacts: Optional[Dict[str, Any]]         # Evaluation artifacts
    metadata: Optional[Dict[str, Any]]          # Additional metadata
```

**Rationale:** Evolution traces enable post-hoc analysis of the evolution process. You can answer questions like "what mutation led to the biggest improvement?" or "which island produced the best programs?" without re-running the experiment.

---

## Configuration Data Structures

All configuration lives in nested dataclasses. They form a hierarchy:

```
Config (master)
├── LLMConfig
│   ├── LLMModelConfig (for each model in ensemble)
│   └── LLMModelConfig (for evaluator models)
├── PromptConfig
├── DatabaseConfig
├── EvaluatorConfig
└── EvolutionTraceConfig
```

### Config (Master)

**Defined in:** `openevolve/config.py`

```python
@dataclass
class Config:
    # Sub-configs
    llm: LLMConfig
    prompt: PromptConfig
    database: DatabaseConfig
    evaluator: EvaluatorConfig
    evolution_trace: EvolutionTraceConfig

    # Top-level settings
    max_iterations: int = 100          # Total evolution iterations
    checkpoint_interval: int = 10      # Save checkpoint every N iterations
    output_dir: str = "openevolve_output"
    diff_based_evolution: bool = True  # Use SEARCH/REPLACE diffs vs full rewrite
    max_code_length: int = 10000       # Maximum allowed code length (chars)
    random_seed: Optional[int] = None  # For reproducibility

    # Early stopping
    early_stopping_patience: Optional[int] = None  # Stop after N iterations w/o improvement
    convergence_threshold: float = 0.01             # Minimum improvement to count
```

### LLMModelConfig (Single Model)

```python
@dataclass
class LLMModelConfig:
    name: str = ""                     # Model name (e.g., "gpt-4o", "gemini-2.0-flash")
    api_base: str = ""                 # API endpoint URL
    api_key: str = ""                  # API key (supports ${ENV_VAR} syntax)
    temperature: float = 0.7           # Randomness (0=deterministic, 1=creative)
    top_p: float = 0.95               # Nucleus sampling threshold
    max_tokens: int = 4096            # Maximum response length
    weight: float = 1.0               # Ensemble weight (higher = more likely to be selected)
    timeout: int = 60                 # API call timeout (seconds)
    max_retries: int = 3              # Retry count on failure
    retry_delay: float = 5.0          # Delay between retries (seconds)
    reasoning_effort: Optional[str] = None  # For reasoning models (o1, o3): "low"/"medium"/"high"
    random_seed: Optional[int] = None       # Per-model random seed
```

**Rationale:** Each model is independently configurable because different models need different parameters. Reasoning models (o1, o3) use `reasoning_effort` instead of `temperature`. The `weight` field enables ensemble weighting — a small cheap model gets weight 0.8 (80% of iterations), while a large expensive model gets weight 0.2 (20% of iterations).

### LLMConfig (Collection)

```python
@dataclass
class LLMConfig(LLMModelConfig):      # Inherits default model params
    models: List[LLMModelConfig]       # Evolution model ensemble
    evaluator_models: List[LLMModelConfig]  # Evaluation model ensemble (separate)
```

**Rationale:** Separate evolution and evaluator ensembles because generation (creative) and evaluation (analytical) benefit from different model characteristics. You might use a creative model for generating mutations but a precise model for code review.

### DatabaseConfig

```python
@dataclass
class DatabaseConfig:
    population_size: int = 1000         # Max programs per island
    archive_size: int = 100             # Elite archive size per island
    num_islands: int = 5                # Number of separate populations

    # MAP-Elites feature space
    feature_dimensions: List[str] = ["complexity", "diversity"]
    feature_bins: Union[int, List[int]] = 10  # Bins per dimension (uniform or per-dim)

    # Selection ratios (must sum to 1.0)
    elite_selection_ratio: float = 0.1   # Pick from top programs
    exploration_ratio: float = 0.2       # Pick random programs
    exploitation_ratio: float = 0.7      # Pick from MAP-Elites archive

    # Island migration
    migration_interval: int = 50         # Generations between migrations
    migration_rate: float = 0.1          # Fraction of top programs to migrate

    # Artifact storage
    artifacts_base_path: Optional[str] = None
    artifact_size_threshold: int = 10240  # 10KB — above this, save to disk
    artifact_retention_days: Optional[int] = None
```

**Rationale:** The three selection ratios (elite/exploration/exploitation) control the exploration-exploitation tradeoff. Too much exploitation → stuck in local optima. Too much exploration → wasting compute on random programs. The default 10/20/70 split is tuned for most problems.

### PromptConfig

```python
@dataclass
class PromptConfig:
    template_dir: Optional[str] = None  # Custom prompt template directory
    num_top_programs: int = 3           # Top-scoring programs shown to LLM
    num_diverse_programs: int = 2       # Diverse programs shown to LLM
    use_template_stochasticity: bool = True  # Random template variations

    # Compact representation for large codebases
    programs_as_changes_description: bool = False  # Use change summaries instead of full code

    # Artifact rendering in prompts
    include_artifacts: bool = True      # Include evaluation artifacts in prompts
    max_artifact_bytes: int = 5000      # Max artifact size in prompt
    artifact_security_filter: bool = True  # Sanitize potentially dangerous content

    # Code length management
    simplification_suggestion_threshold: int = 500   # Suggest simplification above this
    comprehensive_code_char_threshold: int = 50       # Lines threshold for code detail level
```

### EvaluatorConfig

```python
@dataclass
class EvaluatorConfig:
    timeout: int = 300                   # Max seconds per evaluation
    cascade_evaluation: bool = False     # Enable multi-stage evaluation
    cascade_thresholds: List[float] = [0.5, 0.75, 0.9]  # Stage pass thresholds
    parallel_evaluations: int = 4        # Concurrent evaluations
    use_llm_feedback: bool = False       # LLM-based code review
    llm_feedback_weight: float = 0.1     # Weight of LLM feedback in score
```

### EvolutionTraceConfig

```python
@dataclass
class EvolutionTraceConfig:
    enabled: bool = False               # Enable trace logging
    format: str = "jsonl"               # Output format: jsonl, json, hdf5
    include_code: bool = True           # Include source code in traces
    include_prompts: bool = False       # Include prompts in traces
    output_path: Optional[str] = None   # Custom output path
    compress: bool = False              # gzip compression for JSONL
    buffer_size: int = 100              # Flush buffer every N entries
```

---

## Internal Data Structures

### SerializableResult (Worker Output)

Workers run in separate processes, so their output must be serializable.

**Defined in:** `openevolve/process_parallel.py`

```python
@dataclass
class SerializableResult:
    child_program_dict: Optional[Dict[str, Any]]  # Program as dictionary
    parent_id: Optional[str]                       # Parent program ID
    iteration_time: float                          # How long the iteration took
    prompt: Optional[Dict[str, str]]               # Prompts used
    llm_response: Optional[str]                    # Raw LLM output
    artifacts: Optional[Dict[str, Any]]            # Evaluation artifacts
    iteration: int                                 # Iteration number
    error: Optional[str]                           # Error message if failed
    target_island: Optional[int]                   # Which island to store in
```

**Rationale:** Python's `ProcessPoolExecutor` requires all data to be picklable. The `Program` dataclass is converted to a plain dict (`child_program_dict`) for serialization, then reconstructed on the Controller side.

### Database Snapshot

When workers need to read from the database, they get a serializable snapshot:

```python
# Snapshot structure (from database.py)
{
    "programs": {
        "prog_1": {
            "id": "prog_1",
            "code": "...",
            "metrics": {...},
            "parent_id": "...",
            "generation": 3,
            "island_id": 0,
            ...
        },
        "prog_2": {...},
        ...
    },
    "islands": {
        0: ["prog_1", "prog_5", ...],  # Program IDs per island
        1: ["prog_2", "prog_7", ...],
        ...
    },
    "best_program_id": "prog_42",
    "feature_maps": {
        0: {  # Island 0's MAP-Elites grid
            (0, 3): "prog_1",    # Feature bin (0,3) → best program ID
            (2, 1): "prog_5",
            ...
        },
        ...
    },
    "config": {...}
}
```

### Prompt Output

The prompt sampler returns a dictionary with system and user messages:

```python
{
    "system": "You are an expert software engineer...",
    "user": "## Current Program\n```python\n...\n```\n\n## Metrics\n..."
}
```

### Metrics Dictionary

The standard format for evaluation metrics:

```python
{
    "combined_score": 0.85,         # REQUIRED — primary fitness (higher = better)
    "accuracy": 0.92,               # Optional — problem-specific
    "execution_time": 1.2,          # Optional — performance metric
    "memory_usage": 45.6,           # Optional — resource metric
    "complexity": 7.0,              # Used as MAP-Elites feature dimension
    "diversity": 0.3                # Used as MAP-Elites feature dimension
}
```

**Rules:**
- `combined_score` is the primary metric — it drives selection and fitness
- Feature dimensions (e.g., `complexity`, `diversity`) must be **raw continuous values**, not pre-binned indices
- All numeric values should be `float` or `int`
- Non-numeric values (strings) are stored but ignored for fitness calculations

---

## Checkpoint Data Structure

A checkpoint directory contains:

```
checkpoint_100/
├── metadata.json           # Checkpoint metadata
│   {
│     "iteration": 100,
│     "timestamp": 1709234567.89,
│     "best_score": 0.922,
│     "num_programs": 347,
│     "config_hash": "abc123..."
│   }
│
├── database.json           # Full database state
│   {
│     "programs": [...],
│     "islands": {...},
│     "feature_maps": {...},
│     "generation_counts": {...},
│     "best_program_id": "prog_42"
│   }
│
└── programs/               # Per-program files
    ├── prog_1/
    │   ├── program.json    # Code + metrics + metadata
    │   └── artifacts/      # Large artifact files
    │       └── plot.png
    ├── prog_2/
    │   └── program.json
    └── ...
```

---

## EvolutionResult (API Output)

The final result returned by the Python API:

```python
@dataclass
class EvolutionResult:
    best_program: Optional[Program]    # Full Program object of best solution
    best_score: float                  # Highest combined_score achieved
    best_code: str                     # Source code of best program
    metrics: Dict[str, Any]            # Best program's full metrics
    output_dir: Optional[str]          # Path to output directory
```

---

## YAML Config File Format

Example showing all major sections:

```yaml
llm:
  api_base: "https://generativelanguage.googleapis.com/v1beta/openai/"
  api_key: "${OPENAI_API_KEY}"
  models:
    - name: "gemini-2.0-flash-lite"
      weight: 0.8
      temperature: 0.8
      max_tokens: 8192
    - name: "gemini-2.0-flash"
      weight: 0.2
      temperature: 0.7
      max_tokens: 16384

database:
  population_size: 1000
  archive_size: 100
  num_islands: 5
  feature_dimensions: ["complexity", "diversity"]
  feature_bins: 10
  migration_interval: 50
  migration_rate: 0.1

evaluator:
  timeout: 300
  cascade_evaluation: false
  parallel_evaluations: 4

prompt:
  num_top_programs: 3
  num_diverse_programs: 2
  use_template_stochasticity: true

diff_based_evolution: true
max_iterations: 100
checkpoint_interval: 10

evolution_trace:
  enabled: false
  format: "jsonl"
```

---

## Summary of Key Types

| Type | Location | Purpose |
|------|----------|---------|
| `Program` | `database.py` | Individual evolved program with code, metrics, lineage |
| `EvaluationResult` | `evaluation_result.py` | Evaluation output (metrics + artifacts) |
| `EvolutionTrace` | `evolution_trace.py` | Single iteration log entry |
| `Config` | `config.py` | Master configuration |
| `LLMModelConfig` | `config.py` | Single LLM model settings |
| `LLMConfig` | `config.py` | LLM ensemble settings |
| `DatabaseConfig` | `config.py` | MAP-Elites and island settings |
| `EvaluatorConfig` | `config.py` | Evaluation settings |
| `PromptConfig` | `config.py` | Prompt assembly settings |
| `EvolutionTraceConfig` | `config.py` | Trace logging settings |
| `SerializableResult` | `process_parallel.py` | Worker process output |
| `EvolutionResult` | `api.py` | Final API output |
