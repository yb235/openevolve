# Configuration Guide

This document explains every configuration option in OpenEvolve — what it does, what values to use, and why.

## Configuration Basics

OpenEvolve uses YAML configuration files. You can also configure it programmatically via Python.

### Loading Config

```bash
# From CLI
python openevolve-run.py program.py evaluator.py --config config.yaml

# From Python
from openevolve import Config
config = Config.from_yaml("config.yaml")
```

### Environment Variables

Config values support `${ENV_VAR}` syntax:

```yaml
llm:
  api_key: "${OPENAI_API_KEY}"        # Resolved at load time
  api_base: "${API_BASE_URL}"         # Keeps secrets out of files
```

**Rationale:** API keys should never be hardcoded in config files. Environment variable resolution makes it easy to use different keys in development vs production.

### Override Priority

```
Defaults  <  YAML Config  <  CLI Arguments  <  Programmatic Changes
(lowest)                                       (highest)
```

---

## Full Configuration Reference

### Top-Level Settings

```yaml
# How many evolution iterations to run
max_iterations: 100
# Rationale: More iterations = better results, but takes longer.
# Start with 50 for quick experiments, use 500-1000 for production.

# Save checkpoint every N iterations
checkpoint_interval: 10
# Rationale: Frequent checkpoints protect against crashes.
# 10 is good for short runs, 50 for long runs (reduces I/O overhead).

# Output directory for checkpoints, logs, and results
output_dir: "openevolve_output"

# Evolution mode: diff-based (SEARCH/REPLACE) or full rewrite
diff_based_evolution: true
# Rationale: Diffs are more reliable — they make small, focused changes.
# Full rewrites risk losing working code. Use full rewrite only when
# you want the LLM to completely reimagine the solution.

# Maximum allowed code length in characters
max_code_length: 10000
# Rationale: Prevents code from growing unboundedly. LLMs sometimes
# add unnecessary complexity. This forces simplification.

# Random seed for reproducibility (null = random)
random_seed: null

# Early stopping: stop after N iterations without improvement
early_stopping_patience: null    # null = disabled, e.g., 30 = stop after 30 stale iterations
convergence_threshold: 0.01      # Minimum improvement to count as "not stale"
# Rationale: Saves compute when evolution has plateaued.
# Be careful with small patience values — evolution can have long flat periods
# before making breakthroughs.
```

---

### LLM Configuration

```yaml
llm:
  # Shared defaults (inherited by all models unless overridden)
  api_base: "https://generativelanguage.googleapis.com/v1beta/openai/"
  api_key: "${OPENAI_API_KEY}"
  temperature: 0.7
  top_p: 0.95
  max_tokens: 4096
  timeout: 60
  max_retries: 3
  retry_delay: 5.0

  # Evolution model ensemble
  models:
    - name: "gemini-2.0-flash-lite"
      weight: 0.8              # Selected 80% of the time
      temperature: 0.8         # Higher temperature = more creative mutations
      max_tokens: 8192
    - name: "gemini-2.0-flash"
      weight: 0.2              # Selected 20% of the time
      temperature: 0.7         # Lower temperature = more precise mutations
      max_tokens: 16384

  # Evaluator model ensemble (separate, optional)
  evaluator_models:
    - name: "gemini-2.0-flash"
      weight: 1.0
      temperature: 0.3         # Low temperature for consistent evaluation
      max_tokens: 4096
```

#### Model Parameters Explained

| Parameter | Default | Description |
|-----------|---------|-------------|
| `name` | (required) | Model identifier (e.g., "gpt-4o", "gemini-2.0-flash") |
| `api_base` | `""` | API endpoint URL |
| `api_key` | `""` | API key (supports `${ENV_VAR}`) |
| `temperature` | `0.7` | Randomness: 0 = deterministic, 1 = very creative |
| `top_p` | `0.95` | Nucleus sampling: only consider tokens in top 95% probability |
| `max_tokens` | `4096` | Maximum response length |
| `weight` | `1.0` | Ensemble weight (relative probability of selection) |
| `timeout` | `60` | API call timeout in seconds |
| `max_retries` | `3` | Number of retry attempts on failure |
| `retry_delay` | `5.0` | Seconds between retries |
| `reasoning_effort` | `null` | For reasoning models (o1, o3): "low", "medium", "high" |
| `random_seed` | `null` | Per-model seed for reproducibility |

#### Temperature Guidelines

| Temperature | Use Case |
|-------------|----------|
| 0.0 - 0.3 | Evaluation, code review (consistent, precise) |
| 0.4 - 0.6 | Exploitation (targeted improvements) |
| 0.7 - 0.8 | Balanced (default for most evolution tasks) |
| 0.9 - 1.0 | Exploration (creative, diverse mutations) |

**Rationale for ensemble weights:** Most iterations should use a fast, cheap model for broad exploration. A fraction should use a powerful model for deep improvements. The 80/20 split balances cost, speed, and quality.

---

### Database Configuration

```yaml
database:
  # Population management
  population_size: 1000      # Max programs per island
  archive_size: 100           # Elite archive size per island
  num_islands: 5              # Number of separate populations

  # MAP-Elites feature space
  feature_dimensions:         # What characteristics define the diversity grid
    - "complexity"            # Built-in: code complexity estimate
    - "diversity"             # Built-in: how different from other programs
  feature_bins: 10            # Number of bins per dimension (10x10 = 100 cells)

  # Selection strategy (must sum to 1.0)
  elite_selection_ratio: 0.1   # 10% chance: pick from top programs
  exploration_ratio: 0.2       # 20% chance: pick random program
  exploitation_ratio: 0.7      # 70% chance: pick from MAP-Elites archive

  # Island migration
  migration_interval: 50       # Migrate every 50 generations
  migration_rate: 0.1          # Top 10% of programs migrate

  # Artifact storage
  artifact_size_threshold: 10240  # Bytes — above this, save to disk (10KB)
  artifact_retention_days: null   # null = keep forever
```

#### Island Configuration Guidelines

| Scenario | Islands | Migration Interval | Migration Rate |
|----------|---------|-------------------|----------------|
| Simple problem, fast iteration | 2-3 | 100 | 0.05 |
| Medium complexity | 5 | 50 | 0.10 |
| Complex, open-ended | 7-10 | 25 | 0.20 |
| Maximum diversity | 10-15 | 25 | 0.15 |

**Rationale:**
- More islands = more diversity but more compute per iteration
- Shorter migration intervals = faster knowledge sharing but less independent exploration
- Higher migration rates = more mixing but risk homogenizing islands

#### Feature Dimensions

You can use **custom feature dimensions** from your evaluator metrics:

```yaml
# Example: optimize for both speed and accuracy
database:
  feature_dimensions:
    - "execution_time"      # From evaluator: return {"execution_time": 2.5, ...}
    - "accuracy"            # From evaluator: return {"accuracy": 0.92, ...}
  feature_bins: [10, 10]    # 10 bins for time, 10 bins for accuracy
```

```yaml
# Example: 3D feature space
database:
  feature_dimensions:
    - "model_size"
    - "inference_speed"
    - "accuracy"
  feature_bins: [5, 8, 10]  # Different resolution per dimension
```

---

### Evaluator Configuration

```yaml
evaluator:
  # Timeout per evaluation (seconds)
  timeout: 300
  # Rationale: Long enough for complex benchmarks, short enough to
  # catch infinite loops. Adjust based on your evaluator's expected runtime.

  # Multi-stage cascade evaluation
  cascade_evaluation: false   # Enable cascade filtering
  cascade_thresholds:
    - 0.5                     # Stage 1 minimum score to continue
    - 0.75                    # Stage 2 minimum score to continue
    - 0.9                     # Stage 3 minimum score (informational)

  # Parallel evaluation
  parallel_evaluations: 4     # Run 4 evaluations concurrently
  # Rationale: Matches typical CPU core count. Increase for I/O-bound
  # evaluators, decrease for CPU-bound ones.

  # LLM-based feedback
  use_llm_feedback: false     # Optional LLM code review
  llm_feedback_weight: 0.1   # Weight of LLM feedback (10%)
```

#### Cascade Threshold Guidelines

| Stage | Purpose | Typical Threshold | Example Check |
|-------|---------|-------------------|---------------|
| 1 | Quick filter | 0.3 - 0.5 | Syntax check, imports work |
| 2 | Basic validation | 0.6 - 0.8 | Core tests pass |
| 3 | Full evaluation | 0.8 - 0.95 | Complete benchmark |

---

### Prompt Configuration

```yaml
prompt:
  # Custom template directory (null = use built-in templates)
  template_dir: null

  # How many example programs to show the LLM
  num_top_programs: 3          # Best-scoring programs for inspiration
  num_diverse_programs: 2      # Different-approach programs for diversity

  # Template variation for diversity
  use_template_stochasticity: true
  # Rationale: Slight random variations in prompts produce more diverse
  # LLM outputs. Without this, the same prompt produces similar mutations.

  # Compact representation for large codebases
  programs_as_changes_description: false
  # Rationale: When code is very long (>5000 chars), showing full code
  # in prompts wastes tokens. Instead, show a compact LLM-generated
  # summary of changes.

  # Artifact rendering in prompts
  include_artifacts: true       # Show evaluation artifacts to LLM
  max_artifact_bytes: 5000      # Max artifact size in prompt
  artifact_security_filter: true # Sanitize sensitive content

  # Code length management
  simplification_suggestion_threshold: 500   # Suggest simplification above this many chars
  comprehensive_code_char_threshold: 50      # Lines threshold for code detail level
```

---

### Evolution Trace Configuration

```yaml
evolution_trace:
  enabled: false               # Enable trace logging
  format: "jsonl"              # Output format: jsonl, json, or hdf5
  include_code: true           # Include source code in traces
  include_prompts: false       # Include full prompts (verbose!)
  output_path: null            # Custom output path (null = auto)
  compress: false              # gzip compression for JSONL format
  buffer_size: 100             # Flush to disk every N entries
```

**Format comparison:**

| Format | Size | Speed | Use Case |
|--------|------|-------|----------|
| `jsonl` | Medium | Fast write | Default, streaming analysis |
| `json` | Large | Slow write | Human-readable, small runs |
| `hdf5` | Small | Fast both | Scientific computing, large runs |

---

## Example Configurations

### Quick Experiment

```yaml
llm:
  api_key: "${OPENAI_API_KEY}"
  models:
    - name: "gpt-4o-mini"
      weight: 1.0
      temperature: 0.8

database:
  num_islands: 2
  population_size: 100

evaluator:
  timeout: 60

max_iterations: 50
checkpoint_interval: 25
```

### Production Run

```yaml
llm:
  api_key: "${OPENAI_API_KEY}"
  models:
    - name: "gpt-4o-mini"
      weight: 0.7
      temperature: 0.8
      max_tokens: 8192
    - name: "gpt-4o"
      weight: 0.3
      temperature: 0.7
      max_tokens: 16384

database:
  num_islands: 7
  population_size: 1000
  archive_size: 100
  feature_dimensions: ["complexity", "diversity"]
  migration_interval: 50
  migration_rate: 0.1

evaluator:
  timeout: 300
  cascade_evaluation: true
  cascade_thresholds: [0.5, 0.75, 0.9]
  parallel_evaluations: 4

prompt:
  num_top_programs: 3
  num_diverse_programs: 2
  use_template_stochasticity: true

max_iterations: 500
checkpoint_interval: 25
diff_based_evolution: true

evolution_trace:
  enabled: true
  format: "jsonl"
```

### Maximum Diversity

```yaml
database:
  num_islands: 10
  population_size: 500
  migration_interval: 25
  migration_rate: 0.2
  elite_selection_ratio: 0.05
  exploration_ratio: 0.35
  exploitation_ratio: 0.60

llm:
  models:
    - name: "gpt-4o-mini"
      weight: 0.9
      temperature: 0.95  # Very creative
    - name: "gpt-4o"
      weight: 0.1
```

### Early Stopping

```yaml
max_iterations: 1000
early_stopping_patience: 30
convergence_threshold: 0.01

database:
  num_islands: 3
  population_size: 200
```

---

## Configuration Anti-Patterns

| ❌ Don't | ✅ Do Instead | Why |
|----------|--------------|-----|
| `feature_dimensions: 2` | `feature_dimensions: ["complexity", "diversity"]` | Must be a list of names |
| `"total_score": 0.85` | `"combined_score": 0.85` | The key must be `combined_score` |
| Return `bin_index: 3` | Return `raw_value: 2.5` | Feature dims must be raw values |
| Multiple EVOLVE-BLOCKs | Exactly one EVOLVE-BLOCK | System expects a single block |
| `temperature: 2.0` | `temperature: 0.8` | Valid range is 0.0-1.0 (2.0 = garbage output) |
| `num_islands: 1` | `num_islands: 3+` | 1 island = no island benefit |
| `migration_interval: 1` | `migration_interval: 25+` | Too frequent = homogenized islands |
