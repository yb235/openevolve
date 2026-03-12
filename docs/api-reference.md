# API Reference

This document covers all the public interfaces for using OpenEvolve — both the Python library API and the command-line interface.

## Python Library API

### Quick Start

```python
import openevolve

# Simplest possible usage
result = openevolve.run_evolution(
    initial_program="my_program.py",
    evaluator="my_evaluator.py",
    iterations=50
)

print(f"Best score: {result.best_score}")
print(f"Best code:\n{result.best_code}")
```

---

### `run_evolution()`

The main entry point for running code evolution.

```python
async def run_evolution(
    initial_program: Union[str, List[str]],   # File path(s) or code string(s)
    evaluator: Union[str, Callable],          # File path or callable
    config: Optional[Union[str, Config]] = None,  # YAML path or Config object
    iterations: int = 100,                    # Number of evolution iterations
    output_dir: Optional[str] = None,         # Where to save results
    cleanup: bool = True                      # Remove temp files after
) -> EvolutionResult
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `initial_program` | `str` or `List[str]` | Path to initial program file, or the code as a string. If a list, first element is used. |
| `evaluator` | `str` or `Callable` | Path to evaluator Python file, or a callable that takes a program path and returns metrics dict. |
| `config` | `str`, `Config`, or `None` | Path to YAML config, or a `Config` object. If `None`, uses defaults. |
| `iterations` | `int` | How many evolution iterations to run. Default: 100. |
| `output_dir` | `str` or `None` | Directory for checkpoints and logs. Default: `"openevolve_output"`. |
| `cleanup` | `bool` | Whether to clean up temp files when using code strings. Default: `True`. |

**Returns:** `EvolutionResult`

**Example with code strings (no files needed):**
```python
result = await openevolve.run_evolution(
    initial_program="""
# EVOLVE-BLOCK-START
def solve(x):
    return x * 2
# EVOLVE-BLOCK-END
""",
    evaluator="evaluator.py",
    iterations=50
)
```

**Rationale:** Accepting both file paths and code strings makes the API flexible. For quick experiments you can pass code directly; for production you use files. The function handles temp file creation/cleanup transparently.

---

### `evolve_function()`

Evolve a Python function based on test cases.

```python
async def evolve_function(
    func: Callable,                           # Function to evolve
    test_cases: List[Tuple[Any, Any]],       # [(input, expected_output), ...]
    iterations: int = 50,                     # Number of iterations
    **kwargs                                  # Additional arguments passed to run_evolution
) -> str  # Returns optimized function code
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `func` | `Callable` | The Python function to optimize |
| `test_cases` | `List[Tuple]` | List of `(input, expected_output)` pairs |
| `iterations` | `int` | Evolution iterations. Default: 50. |
| `**kwargs` | | Additional args forwarded to `run_evolution()` |

**Example:**
```python
def my_sort(arr):
    # EVOLVE-BLOCK-START
    return sorted(arr)  # Basic implementation
    # EVOLVE-BLOCK-END

result = await openevolve.evolve_function(
    func=my_sort,
    test_cases=[
        ([3, 1, 2], [1, 2, 3]),
        ([5, 4, 3, 2, 1], [1, 2, 3, 4, 5]),
        ([], []),
    ],
    iterations=100
)
```

**Rationale:** This is a convenience wrapper that auto-generates an evaluator from test cases. It's the fastest way to get started without writing a separate evaluator file.

---

### `evolve_algorithm()`

Evolve an algorithm class using a custom benchmark function.

```python
async def evolve_algorithm(
    algorithm_class: type,                    # Class to evolve
    benchmark: Callable,                      # Benchmark function
    iterations: int = 50,                     # Number of iterations
    **kwargs                                  # Additional arguments
) -> str  # Returns optimized class code
```

**Example:**
```python
class MyOptimizer:
    # EVOLVE-BLOCK-START
    def optimize(self, x):
        return x - 0.1 * gradient(x)
    # EVOLVE-BLOCK-END

def benchmark(program_path):
    # Run benchmark and return metrics
    return {"combined_score": 0.85, "convergence_rate": 0.92}

result = await openevolve.evolve_algorithm(
    algorithm_class=MyOptimizer,
    benchmark=benchmark,
    iterations=200
)
```

---

### `evolve_code()`

Generic code evolution with a custom evaluator.

```python
async def evolve_code(
    initial_code: str,                        # Code string to evolve
    evaluator: Union[str, Callable],         # Evaluator (file path or callable)
    iterations: int = 50,                     # Number of iterations
    **kwargs                                  # Additional arguments
) -> str  # Returns optimized code
```

---

### `EvolutionResult`

The return type from evolution functions.

```python
@dataclass
class EvolutionResult:
    best_program: Optional[Program]    # Full Program object (code, metrics, lineage)
    best_score: float                  # Highest combined_score achieved
    best_code: str                     # Source code string of best program
    metrics: Dict[str, Any]            # Complete metrics of best program
    output_dir: Optional[str]          # Path to output directory with checkpoints
```

**Usage:**
```python
result = await openevolve.run_evolution(...)

# Access the best code
print(result.best_code)

# Check the score
print(f"Score: {result.best_score}")

# Get all metrics
print(result.metrics)
# → {"combined_score": 0.922, "execution_time": 0.8, "accuracy": 0.95}

# Access the full Program object
program = result.best_program
print(f"Generation: {program.generation}")
print(f"Parent: {program.parent_id}")
```

---

### `Config`

Configuration management.

```python
from openevolve import Config

# Load from YAML file
config = Config.from_yaml("config.yaml")

# Load from dictionary
config = Config.from_dict({
    "llm": {"models": [{"name": "gpt-4o", "weight": 1.0}]},
    "max_iterations": 50
})

# Create with defaults
config = Config()

# Modify programmatically
config.max_iterations = 200
config.database.num_islands = 10
config.llm.models[0].temperature = 0.9
```

---

### `OpenEvolve` (Controller)

Direct access to the evolution controller for advanced usage.

```python
from openevolve import OpenEvolve, Config

# Create controller
config = Config.from_yaml("config.yaml")
controller = OpenEvolve(
    initial_program_path="program.py",
    evaluator_path="evaluator.py",
    config=config,
    output_dir="output/"
)

# Run evolution
best_program = await controller.run(
    iterations=100,
    target_score=0.95,        # Stop early if this score is reached
    checkpoint_path=None       # Or path to resume from
)

# Access internals
database = controller.database
all_programs = database.get_all_programs()
top_10 = database.get_top_programs(n=10)
```

---

## Command-Line Interface

### Basic Usage

```bash
python openevolve-run.py <initial_program> <evaluator> [options]
```

### Arguments

| Argument | Required | Description |
|----------|----------|-------------|
| `initial_program` | Yes | Path to the initial program file |
| `evaluation_file` | Yes | Path to the evaluator Python file |
| `--config` | No | Path to YAML configuration file |
| `--output` | No | Output directory (default: `openevolve_output`) |
| `--iterations` | No | Number of iterations (overrides config) |
| `--target-score` | No | Stop early when this score is reached |
| `--checkpoint` | No | Path to checkpoint directory to resume from |
| `--api-base` | No | Override LLM API base URL |
| `--primary-model` | No | Override primary LLM model name |
| `--secondary-model` | No | Override secondary LLM model name |

### Examples

**Basic run:**
```bash
python openevolve-run.py \
  examples/function_minimization/initial_program.py \
  examples/function_minimization/evaluator.py \
  --config examples/function_minimization/config.yaml \
  --iterations 50
```

**With model overrides:**
```bash
python openevolve-run.py program.py evaluator.py \
  --api-base "https://api.openai.com/v1" \
  --primary-model "gpt-4o" \
  --secondary-model "gpt-4o-mini" \
  --iterations 100
```

**Resume from checkpoint:**
```bash
python openevolve-run.py program.py evaluator.py \
  --config config.yaml \
  --checkpoint openevolve_output/checkpoints/checkpoint_50 \
  --iterations 100  # Will run 50 more iterations (50→100)
```

**With target score (early stopping):**
```bash
python openevolve-run.py program.py evaluator.py \
  --config config.yaml \
  --iterations 1000 \
  --target-score 0.99  # Stop as soon as 0.99 is reached
```

### Installed command

If OpenEvolve is installed via pip, you can use the `openevolve-run` command directly:

```bash
pip install openevolve
openevolve-run program.py evaluator.py --config config.yaml --iterations 50
```

---

## Evaluator API

The evaluator is a Python file that you write. It must export an `evaluate()` function.

### Simple Evaluator

```python
def evaluate(program_path: str) -> dict:
    """
    Evaluate an evolved program.

    Args:
        program_path: Path to the program file to evaluate

    Returns:
        Dictionary with 'combined_score' (required, higher = better)
        and any additional metrics
    """
    # Import and run the program
    import importlib.util
    spec = importlib.util.spec_from_file_location("program", program_path)
    module = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(module)

    # Test it
    result = module.solve([1, 2, 3])
    expected = [1, 2, 3]  # sorted

    score = 1.0 if result == expected else 0.0
    return {"combined_score": score}
```

### Cascade Evaluator

For cascade evaluation, export stage-specific functions:

```python
def evaluate_stage1(program_path: str) -> dict:
    """Quick syntax and import check."""
    try:
        compile(open(program_path).read(), program_path, 'exec')
        return {"score": 1.0}
    except SyntaxError:
        return {"score": 0.0}

def evaluate_stage2(program_path: str) -> dict:
    """Basic correctness tests."""
    # Run a few quick tests
    return {"score": 0.75, "tests_passed": 3, "tests_total": 4}

def evaluate_stage3(program_path: str) -> dict:
    """Full benchmark suite."""
    # Run comprehensive benchmarks
    return {"combined_score": 0.85, "execution_time": 2.1}
```

### Evaluator with Artifacts

```python
from openevolve.evaluation_result import EvaluationResult

def evaluate(program_path: str) -> EvaluationResult:
    """Evaluate with diagnostic artifacts."""
    # Run tests...
    return EvaluationResult(
        metrics={"combined_score": 0.85},
        artifacts={
            "test_output": "Test 1: PASS\nTest 2: FAIL (expected 5, got 4)",
            "visualization.png": open("result_plot.png", "rb").read()
        }
    )
```

### Evaluator Requirements

| Requirement | Details |
|-------------|---------|
| Must return `combined_score` | This is the primary fitness metric (higher = better) |
| Feature dimensions are raw values | Return continuous numbers, not pre-binned indices |
| Handle exceptions gracefully | Catch errors and return low scores instead of crashing |
| Must accept `program_path` argument | Even if you don't use it directly |
| Should be deterministic | Same input → same output (for reproducibility) |

---

## Visualization API

The visualization server provides a REST API:

### `GET /`
Returns the main visualization page (HTML).

### `GET /api/data`
Returns the full evolution dataset as JSON:

```json
{
  "nodes": [
    {
      "id": "prog_1",
      "island": 0,
      "generation": 0,
      "parent_id": null,
      "metrics": {"combined_score": 0.5},
      "code": "...",
      "prompts": {"system": "...", "user": "..."}
    },
    ...
  ],
  "edges": [
    {"source": "prog_1", "target": "prog_5"},
    ...
  ],
  "archive": ["prog_42", "prog_89", ...]
}
```

### `GET /program/<id>`
Returns detailed information about a specific program.

### Launch Visualization

```bash
# From command line
python scripts/visualizer.py --path path/to/checkpoint/

# Or with static export (no server needed)
python scripts/visualizer.py --path path/to/checkpoint/ --static-output output.html
```
