# Architecture Overview

This document explains the high-level architecture of OpenEvolve — what components exist, what each one does, and how they connect.

## The Big Picture

OpenEvolve is an **evolutionary coding agent**. It takes a program, uses LLMs to generate mutations of that program, evaluates the mutations, keeps the best ones, and repeats. Over hundreds or thousands of iterations, the program improves.

Think of it like biological evolution, but for code:
- **Organisms** = Programs (code files)
- **DNA** = The source code itself
- **Mutations** = LLM-generated code changes
- **Fitness** = Evaluation scores from running the code
- **Natural Selection** = Keeping high-scoring programs, discarding low ones
- **Islands** = Separate populations that evolve independently (preventing inbreeding)

## System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         User Input                              │
│   initial_program.py  +  evaluator.py  +  config.yaml           │
└─────────────────────┬───────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                    CLI / Python API                              │
│  cli.py (command-line)  or  api.py (library interface)          │
│  Parses arguments, loads config, creates the Controller         │
└─────────────────────┬───────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Controller (controller.py)                  │
│  The "brain" — orchestrates the entire evolution loop           │
│                                                                 │
│  ┌──────────┐  ┌──────────────┐  ┌──────────┐  ┌────────────┐ │
│  │ Database  │  │ LLM Ensemble │  │Evaluator │  │  Prompt    │ │
│  │(database  │  │(llm/ensemble │  │(evaluator│  │  Sampler   │ │
│  │  .py)     │  │  .py)        │  │  .py)    │  │(prompt/    │ │
│  │          │  │              │  │          │  │ sampler.py)│ │
│  └──────────┘  └──────────────┘  └──────────┘  └────────────┘ │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │        ProcessParallelController (process_parallel.py)     │ │
│  │   Manages a pool of worker processes for parallel runs     │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────┬───────────────────────────────────────────┘
                      │
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
┌──────────────┐┌──────────────┐┌──────────────┐
│  Worker #1   ││  Worker #2   ││  Worker #N   │
│  (iteration  ││  (iteration  ││  (iteration  │
│   .py)       ││   .py)       ││   .py)       │
│              ││              ││              │
│ 1. Sample    ││ 1. Sample    ││ 1. Sample    │
│ 2. Prompt    ││ 2. Prompt    ││ 2. Prompt    │
│ 3. LLM call  ││ 3. LLM call  ││ 3. LLM call  │
│ 4. Evaluate  ││ 4. Evaluate  ││ 4. Evaluate  │
│ 5. Return    ││ 5. Return    ││ 5. Return    │
└──────┬───────┘└──────┬───────┘└──────┬───────┘
       │               │               │
       └───────────────┼───────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                  Database (database.py)                          │
│                                                                 │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ │
│  │Island 0 │ │Island 1 │ │Island 2 │ │Island 3 │ │Island 4 │ │
│  │         │ │         │ │         │ │         │ │         │ │
│  │Programs │ │Programs │ │Programs │ │Programs │ │Programs │ │
│  │Archive  │ │Archive  │ │Archive  │ │Archive  │ │Archive  │ │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘ │
│                                                                 │
│  MAP-Elites Grid: programs mapped to feature dimensions         │
│  Migration: top programs periodically shared between islands    │
└─────────────────────────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Output                                     │
│  Best program  +  Checkpoints  +  Evolution trace logs          │
└─────────────────────────────────────────────────────────────────┘
```

## Component Breakdown

### 1. Controller (`controller.py`)

**What it does:** The main orchestrator. It initializes all other components, runs the evolution loop, saves checkpoints, and handles graceful shutdown.

**Rationale:** Having a single orchestrator avoids circular dependencies between components. Each component is initialized once and passed around — the Controller owns the lifecycle of everything.

**Key responsibilities:**
- Load the initial program file
- Initialize the LLM ensemble, database, evaluator, and prompt sampler
- Run iterations (delegating to parallel workers)
- Save/load checkpoints for resuming evolution
- Handle `Ctrl+C` gracefully (two-stage: first graceful, second force-quit)

### 2. Database (`database.py`)

**What it does:** Stores all programs ever created, organized into islands and a MAP-Elites feature grid. Provides sampling methods for selecting parent programs and inspirations.

**Rationale:** The MAP-Elites algorithm maintains diversity by ensuring the solution space is covered evenly. Without it, evolution would converge on a single local optimum. Islands add another layer of diversity by running parallel, independent populations.

**Key responsibilities:**
- Store programs with their metrics, code, and metadata
- Organize programs into islands (separate populations)
- Maintain MAP-Elites grid (programs mapped to feature bins)
- Sample parents and inspirations for new iterations
- Migrate top programs between islands periodically
- Track the absolute best program found so far

### 3. Evaluator (`evaluator.py`)

**What it does:** Runs the user's evaluation function on evolved programs. Supports cascade evaluation (multi-stage filtering) and parallel evaluation.

**Rationale:** Evaluation is often the bottleneck. Cascade evaluation saves compute by quickly rejecting bad programs in early stages. Parallel evaluation lets multiple programs be tested simultaneously.

**Key responsibilities:**
- Execute the evaluation function with timeout protection
- Run cascade evaluation (stage 1 → 2 → 3) with early-exit thresholds
- Capture artifacts (diagnostic outputs) from evaluations
- Optionally integrate LLM-based code review feedback

### 4. LLM Ensemble (`llm/ensemble.py`)

**What it does:** Manages multiple LLM models with weighted random selection. When the system needs to generate code, it picks a model based on configured weights.

**Rationale:** Different LLMs have different strengths. A smaller, faster model handles most iterations (exploration), while a larger, smarter model handles a fraction (exploitation of harder mutations). The ensemble approach mirrors biological genetic diversity.

**Key responsibilities:**
- Select which LLM to use for each generation (weighted random)
- Make API calls with retry logic and timeout handling
- Support both standard and reasoning models (o1, o3, etc.)
- Enable manual/human-in-the-loop mode

### 5. Prompt Sampler (`prompt/sampler.py`)

**What it does:** Assembles the prompt sent to the LLM. It combines the current program, top-performing examples, diverse inspirations, evolution history, and improvement suggestions into a structured message.

**Rationale:** The quality of the LLM's output depends heavily on what context it receives. The prompt sampler carefully curates this context to guide evolution in productive directions while maintaining diversity.

**Key responsibilities:**
- Build system messages explaining the evolution task
- Format the current program and its metrics
- Include top-performing and diverse example programs
- Format evolution history (previous attempts and outcomes)
- Render artifacts (test results, error logs)
- Apply template stochasticity for variation

### 6. Iteration Worker (`iteration.py`)

**What it does:** Executes a single evolution step: sample a parent, build a prompt, call the LLM, parse the response, evaluate the result, and return it.

**Rationale:** Isolating a single iteration into its own module makes it easy to run iterations in parallel across separate processes. Each worker is stateless — it receives a database snapshot and returns a result.

### 7. Process Parallel Controller (`process_parallel.py`)

**What it does:** Manages a pool of worker processes using Python's `ProcessPoolExecutor`. Each worker runs an independent iteration.

**Rationale:** True multiprocessing (not just async) because each iteration may include CPU-intensive evaluation. Process isolation also prevents memory leaks from accumulating across iterations.

## Supporting Components

| Component | File | Purpose |
|-----------|------|---------|
| Config | `config.py` | Hierarchical YAML-based configuration with environment variable support |
| Evolution Trace | `evolution_trace.py` | Logs every iteration's details for later analysis |
| Novelty Judge | `novelty_judge.py` | Uses LLM to determine if a program is meaningfully different |
| Embeddings | `embedding.py` | Code embeddings for fast similarity checks |
| Code Utils | `utils/code_utils.py` | Parse EVOLVE-BLOCK markers, apply diffs, detect languages |
| Async Utils | `utils/async_utils.py` | Timeout handling, concurrency control, retry logic |
| Evaluation Result | `evaluation_result.py` | Standardized evaluation output with artifact support |

## How Components Connect

1. **User provides:** initial program + evaluator + config
2. **CLI/API** creates a **Controller**
3. **Controller** initializes **Database**, **LLM Ensemble**, **Evaluator**, **Prompt Sampler**
4. **Controller** starts the evolution loop via **ProcessParallelController**
5. Each **Worker** receives a database snapshot and runs one iteration:
   - **Database.sample()** → parent program + inspirations
   - **PromptSampler.build_prompt()** → structured prompt
   - **LLMEnsemble.generate()** → mutated code
   - **Evaluator.evaluate_program()** → metrics + artifacts
6. **Controller** adds successful results back to the **Database**
7. **Database** updates MAP-Elites grid and island populations
8. Every N iterations, **Controller** saves a checkpoint
9. After all iterations, **Controller** returns the best program

## File-by-File Map

```
openevolve/
├── __init__.py              # Package exports (Config, OpenEvolve, run_evolution, etc.)
├── _version.py              # Version string
├── api.py                   # Python library API (run_evolution, evolve_function, etc.)
├── cli.py                   # Command-line interface (argparse → Controller)
├── config.py                # Configuration dataclasses (LLM, Database, Evaluator, etc.)
├── controller.py            # Main orchestrator (evolution loop, checkpoints)
├── database.py              # Program storage (MAP-Elites + islands)
├── embedding.py             # Code embeddings (OpenAI text-embedding models)
├── evaluation_result.py     # Standardized evaluation output dataclass
├── evaluator.py             # Program evaluation (cascade, parallel, artifacts)
├── evolution_trace.py       # Evolution logging (JSONL/JSON/HDF5)
├── iteration.py             # Single iteration logic (sample→prompt→LLM→evaluate)
├── novelty_judge.py         # LLM-based novelty assessment
├── process_parallel.py      # Worker process pool management
├── llm/
│   ├── __init__.py          # LLM module exports
│   ├── base.py              # Abstract LLM interface
│   ├── ensemble.py          # Weighted model ensemble
│   └── openai.py            # OpenAI-compatible API client
├── prompt/
│   ├── __init__.py          # Prompt module exports
│   ├── sampler.py           # Prompt assembly logic
│   └── templates.py         # Template loading and rendering
├── prompts/
│   └── *.txt                # Actual prompt template text files
└── utils/
    ├── __init__.py           # Utils exports
    ├── async_utils.py        # Async helpers (timeout, retry, concurrency)
    ├── code_utils.py         # Code parsing (EVOLVE-BLOCK, diffs, language detection)
    ├── format_utils.py       # Safe metric formatting
    ├── metrics_utils.py      # Fitness scoring, feature formatting
    └── trace_export_utils.py # Trace export (JSONL, JSON, HDF5)
```
