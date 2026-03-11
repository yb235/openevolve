# Evaluation System

This document explains how OpenEvolve evaluates evolved programs — the cascade evaluation pattern, artifact capture, LLM-based feedback, and the scoring system.

## Overview

Evaluation is the **fitness function** of the evolutionary algorithm. It answers the question: "Is this mutated program better than its parent?"

```
Evolved Code ──▶ Evaluator ──▶ Metrics (scores) + Artifacts (diagnostics)
                    │
                    ├── Stage 1: Quick check (syntax, imports)
                    ├── Stage 2: Basic tests (correctness)
                    └── Stage 3: Full benchmark (performance)
```

## The Evaluator Contract

You (the user) write the evaluator. It's a Python file with an `evaluate()` function:

```python
# evaluator.py — YOUR code

def evaluate(program_path: str) -> dict:
    """
    Run the evolved program and return metrics.

    Args:
        program_path: Absolute path to the evolved program file

    Returns:
        dict with 'combined_score' key (required, float, higher = better)
        plus any additional metrics
    """
    # Your evaluation logic here
    return {
        "combined_score": 0.85,     # REQUIRED — primary fitness signal
        "accuracy": 0.92,           # Optional
        "execution_time": 1.2,      # Optional
    }
```

### Rules for Evaluators

| Rule | Why |
|------|-----|
| Must return `combined_score` | This is the primary fitness metric that drives evolution |
| Higher scores = better | The system maximizes `combined_score` |
| Handle exceptions gracefully | Return a low score instead of crashing |
| Feature values must be raw numbers | Don't pre-bin: return `execution_time: 2.5`, not `time_bin: 3` |
| Should be deterministic | Same input → same output (for reproducibility) |
| Accept `program_path` as argument | Even if you don't use it directly |

---

## Simple Evaluation

The simplest evaluation mode: call `evaluate()` once, get metrics back.

**File:** `openevolve/evaluator.py`

```python
class Evaluator:
    async def evaluate_program(self, program_code, program_id):
        # 1. Write code to temp file
        temp_path = self._write_temp_file(program_code, program_id)

        # 2. Run evaluation with timeout
        try:
            result = await asyncio.wait_for(
                self._run_evaluation(temp_path),
                timeout=self.config.timeout
            )
        except asyncio.TimeoutError:
            return {"combined_score": 0.0, "error": "timeout"}

        # 3. Process result (handle both dict and EvaluationResult)
        return self._process_evaluation_result(result)
```

**What happens under the hood:**
1. The evolved code is written to a temporary file
2. The evaluator module is imported and `evaluate()` is called
3. A timeout kills evaluation if it takes too long
4. The result is processed (backward compatibility with both dict and EvaluationResult)

**Rationale:** Writing to a temp file (instead of using `exec()`) lets the evaluator handle any language. The evaluator can compile Rust, run R scripts, or execute Metal shaders — it just needs the file path.

---

## Cascade Evaluation

Cascade evaluation is a **multi-stage filter** that saves compute by quickly rejecting bad programs.

### The Problem It Solves

Imagine your full benchmark takes 5 minutes to run. If 80% of mutations produce broken code, you're wasting 80% of your evaluation time on programs that fail basic syntax checks.

### How It Works

```
Stage 1: Quick filter (< 1 second)
  │
  ├── Score ≥ 0.5? ──▶ Continue to Stage 2
  └── Score < 0.5? ──▶ STOP (return low score)
  │
Stage 2: Basic tests (< 30 seconds)
  │
  ├── Score ≥ 0.75? ──▶ Continue to Stage 3
  └── Score < 0.75? ──▶ STOP (return partial score)
  │
Stage 3: Full benchmark (< 5 minutes)
  │
  └── Return complete metrics
```

### Configuration

```yaml
evaluator:
  cascade_evaluation: true
  cascade_thresholds: [0.5, 0.75, 0.9]  # Pass thresholds for each stage
```

### Evaluator Implementation

```python
# evaluator.py — with cascade stages

def evaluate_stage1(program_path):
    """Quick syntax and import check (< 1 second)."""
    try:
        code = open(program_path).read()
        compile(code, program_path, 'exec')
        return {"score": 1.0}
    except SyntaxError as e:
        return {"score": 0.0, "error": str(e)}

def evaluate_stage2(program_path):
    """Basic correctness tests (< 30 seconds)."""
    import subprocess
    result = subprocess.run(
        ["python", program_path, "--test"],
        capture_output=True, timeout=30
    )
    passed = result.stdout.count("PASS")
    total = passed + result.stdout.count("FAIL")
    return {"score": passed / max(total, 1), "tests_passed": passed}

def evaluate_stage3(program_path):
    """Full benchmark suite (< 5 minutes)."""
    # Run comprehensive benchmarks
    return {
        "combined_score": 0.85,
        "execution_time": 2.1,
        "memory_peak": 128.5
    }
```

**Rationale:**
- **Stage 1** catches syntax errors, missing imports, obvious bugs — 0.1 seconds vs 5 minutes
- **Stage 2** catches logical errors with quick tests — 30 seconds vs 5 minutes
- **Stage 3** runs only on programs that passed both earlier stages

In practice, cascade evaluation can reduce total evaluation time by **60-80%**, allowing many more iterations in the same wall-clock time.

---

## Artifacts System

Artifacts are **diagnostic outputs** from evaluation — test results, error logs, visualizations, or any data that helps the LLM understand why a program scored the way it did.

### Why Artifacts Matter

Without artifacts, the LLM only sees:
```
combined_score: 0.45
```

With artifacts, the LLM sees:
```
combined_score: 0.45
Artifacts:
  test_output: "Test 1: PASS, Test 2: FAIL (expected [1,2,3], got [1,3,2])"
  stderr: "Warning: array index out of bounds at line 42"
```

This context helps the LLM make **targeted fixes** instead of random mutations.

### Using Artifacts

```python
from openevolve.evaluation_result import EvaluationResult

def evaluate(program_path):
    # Run program...
    output = run_program(program_path)

    return EvaluationResult(
        metrics={"combined_score": 0.45, "tests_passed": 3},
        artifacts={
            "test_output": "Test 1: PASS\nTest 2: FAIL\nTest 3: PASS\nTest 4: FAIL",
            "stderr": captured_stderr,
            "visualization.png": open("plot.png", "rb").read(),  # Binary artifacts too
        }
    )
```

### Artifact Storage

Artifacts are handled differently based on size:

```
┌─────────────────────────────────────────────┐
│ Artifact Size < 10KB (default threshold)     │
│                                              │
│ Stored in: Program.artifacts_json (in-memory)│
│ Format: JSON string                          │
│ Access: Fast, always available               │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ Artifact Size ≥ 10KB                         │
│                                              │
│ Stored in: Program.artifact_dir (on disk)    │
│ Format: Individual files                     │
│ Access: Read from disk when needed           │
└─────────────────────────────────────────────┘
```

**Configuration:**
```yaml
database:
  artifact_size_threshold: 10240  # 10KB — above this, save to disk
  artifacts_base_path: "artifacts/"
  artifact_retention_days: 30     # Auto-clean old artifacts
```

**Rationale:** Small artifacts (error messages, test results) are cheap to keep in memory and are accessed frequently (included in prompts). Large artifacts (images, data files) would bloat memory — they're saved to disk and referenced by path.

### Artifact Security

Artifacts can contain sensitive information (file paths, API keys, stack traces). OpenEvolve includes a security filter:

```yaml
prompt:
  artifact_security_filter: true  # Sanitize potentially dangerous content
  max_artifact_bytes: 5000        # Limit artifact size in prompts
```

---

## LLM-Based Feedback

Optionally, OpenEvolve can use a separate LLM to review evolved code and provide quality feedback.

### How It Works

```
Evolved code ──▶ Evaluator (metrics) ──▶ Combined score
                       +
Evolved code ──▶ LLM Review (feedback) ──▶ LLM score

Final score = (1 - llm_weight) × evaluator_score + llm_weight × llm_score
```

### Configuration

```yaml
evaluator:
  use_llm_feedback: true       # Enable LLM code review
  llm_feedback_weight: 0.1    # 10% of final score comes from LLM

llm:
  evaluator_models:            # Separate model ensemble for evaluation
    - name: "gpt-4o"
      weight: 1.0
```

**Rationale:** Some quality aspects are hard to capture in automated tests:
- Code readability
- Algorithm elegance
- Potential edge cases
- Style and best practices

LLM feedback adds a "soft" quality signal that complements hard metrics from automated evaluation.

---

## Parallel Evaluation

Multiple programs can be evaluated simultaneously:

```yaml
evaluator:
  parallel_evaluations: 4  # Run 4 evaluations concurrently
```

**Implementation:** Uses `TaskPool` (from `utils/async_utils.py`) with semaphore-based concurrency control:

```python
class TaskPool:
    async def run(self, evaluation_coro, program_code, program_id):
        async with self.semaphore:  # Limits to N concurrent
            return await evaluation_coro(program_code, program_id)
```

**Rationale:** Evaluation is often the bottleneck (running tests, benchmarks, etc.). Parallel evaluation multiplies throughput. The semaphore prevents overwhelming the machine with too many concurrent evaluations.

---

## Timeout Protection

Every evaluation has a timeout:

```yaml
evaluator:
  timeout: 300  # 5 minutes maximum per evaluation
```

**What happens on timeout:**
1. The evaluation coroutine is cancelled
2. A default "failed" result is returned: `{"combined_score": 0.0, "error": "timeout"}`
3. The iteration continues (the program is discarded)
4. A warning is logged

**Rationale:** Evolved programs can contain infinite loops, exponential algorithms, or resource-hungry operations. Without timeouts, a single bad program could block the entire evolution run indefinitely.

---

## The Evaluation Pipeline (Complete)

Here's the complete flow from evolved code to stored metrics:

```
1. Worker generates new code via LLM
         │
         ▼
2. Evaluator writes code to temp file
         │
         ▼
3. Cascade evaluation (if enabled)
   ├── Stage 1: Quick filter
   │   └── Fail? → Return {"combined_score": 0.0}
   ├── Stage 2: Basic tests
   │   └── Fail? → Return partial metrics
   └── Stage 3: Full benchmark
         │
         ▼
4. Process result
   ├── Convert dict → EvaluationResult (if needed)
   ├── Extract metrics
   └── Extract artifacts
         │
         ▼
5. LLM feedback (if enabled)
   ├── Send code to evaluation LLM
   ├── Get quality score
   └── Blend with evaluator score
         │
         ▼
6. Artifact handling
   ├── Small artifacts → stored in Program.artifacts_json
   └── Large artifacts → saved to disk in Program.artifact_dir
         │
         ▼
7. Return to worker
   ├── Metrics dict (for database)
   └── Artifacts dict (for prompt context in future iterations)
```

---

## Key Design Decisions

### 1. Why User-Defined Evaluators Instead of Built-In Metrics?

Every optimization problem has different quality criteria. A sorting algorithm evaluator measures speed and correctness; a neural network evaluator measures accuracy and model size. User-defined evaluators keep the framework general-purpose.

### 2. Why Cascade Instead of Just Quick Evaluation?

Cascade evaluation gives fine-grained control:
- Stage 1 can be a cheap syntax check
- Stage 2 can run a subset of tests
- Stage 3 can run the full suite

Without cascade, you'd either run everything (slow) or only run quick checks (miss subtle bugs). Cascade gives you both: quick rejection of bad programs AND thorough evaluation of promising ones.

### 3. Why Artifacts Are Separate from Metrics?

Metrics are **numbers** used for selection (fitness, feature dimensions). Artifacts are **diagnostic data** used for context (test output, error logs). Keeping them separate means:
- Metrics stay clean (no mixing numbers and strings)
- Artifacts don't pollute the MAP-Elites grid
- Artifacts can be large without affecting database performance

### 4. Why LLM Feedback Weight Is Low (10%)?

LLM feedback is subjective and sometimes inaccurate. A 10% weight means it can nudge scores but can't override clear evaluation results. This prevents the LLM from "gaming" its own feedback to inflate scores.

### 5. Why Write to Temp Files Instead of Using exec()?

- **Security:** Temp files can be sandboxed; `exec()` runs in the current process
- **Multi-language:** Temp files work for Python, Rust, R, Metal, anything
- **Isolation:** Each evaluation runs in its own process space
- **Debugging:** Temp files can be inspected if evaluation fails
