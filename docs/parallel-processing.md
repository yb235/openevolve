# Parallel Processing

This document explains how OpenEvolve runs multiple evolution iterations simultaneously using process-based parallelism, and how checkpointing enables long-running evolution to be interrupted and resumed.

## Why Process-Based Parallelism?

OpenEvolve uses Python's `ProcessPoolExecutor` (true multiprocessing), not just `asyncio` (async I/O). Here's why:

| Approach | Good For | Bad For |
|----------|----------|---------|
| `asyncio` (async I/O) | Network calls (LLM API) | CPU-heavy evaluation |
| `threading` | I/O-bound tasks | Python's GIL blocks CPU work |
| `multiprocessing` | CPU-heavy evaluation | Shared state is hard |

**OpenEvolve's choice:** `multiprocessing` because evaluation (running benchmarks, executing programs) is CPU-bound. Each worker gets its own Python process, bypassing the GIL.

**Rationale:** In a typical evolution iteration, ~10% of time is spent calling the LLM API (I/O-bound) and ~90% is spent evaluating the program (CPU-bound). Async I/O would only help the 10%; multiprocessing helps the 90%.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                 Controller (Main Process)                 │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │         ProcessParallelController                 │   │
│  │                                                   │   │
│  │  ┌──────────────────────────────────┐            │   │
│  │  │    ProcessPoolExecutor            │            │   │
│  │  │                                   │            │   │
│  │  │  ┌─────────┐  ┌─────────┐       │            │   │
│  │  │  │Worker 1 │  │Worker 2 │  ...   │            │   │
│  │  │  │(Process)│  │(Process)│        │            │   │
│  │  │  └─────────┘  └─────────┘       │            │   │
│  │  └──────────────────────────────────┘            │   │
│  └──────────────────────────────────────────────────┘   │
│                                                          │
│  Database ◄── Results from workers                       │
│  Checkpointer ── Periodic state save                     │
└─────────────────────────────────────────────────────────┘
```

---

## How It Works

### 1. Worker Initialization

When the process pool starts, each worker process is initialized with its own components:

**File:** `openevolve/process_parallel.py`

```python
def _worker_init(config_dict):
    """
    Called once when a worker process starts.
    Reconstructs all necessary components from serialized config.
    """
    global _worker_config, _worker_evaluator, _worker_llm_ensemble, _worker_prompt_sampler

    # Reconstruct Config from dict (can't pickle complex objects)
    _worker_config = Config.from_dict(config_dict)

    # Initialize worker-local components
    _worker_llm_ensemble = LLMEnsemble(_worker_config.llm)
    _worker_prompt_sampler = PromptSampler(_worker_config.prompt)
    _worker_evaluator = Evaluator(_worker_config.evaluator)
```

**Key insight:** Workers don't share objects with the main process. Everything is reconstructed from a serialized config dict. This avoids the complexity of shared state and prevents race conditions.

**Rationale:** Python's `multiprocessing` requires all data passed between processes to be picklable (serializable). Complex objects like LLM clients and database connections can't be pickled, so they're reconstructed in each worker.

### 2. Database Snapshots

Workers need to read from the database (to sample parents and inspirations), but they can't access the main process's database directly. Solution: **snapshots**.

```python
# In the Controller (main process):
def _prepare_iteration(self, iteration):
    # Create a serializable copy of the database state
    snapshot = self.database.create_snapshot()
    # Contains: all programs, island assignments, feature maps, config
    return snapshot

# In the Worker (separate process):
def _run_iteration_worker(iteration, db_snapshot, parent_id, inspiration_ids):
    # Reconstruct a minimal database from the snapshot
    programs = {pid: Program.from_dict(pdata) for pid, pdata in db_snapshot["programs"].items()}
    parent = programs[parent_id]
    inspirations = [programs[iid] for iid in inspiration_ids]
    # ... proceed with iteration
```

**Rationale:** Snapshots provide **isolation** — workers see a consistent view of the database at the time the iteration started. They don't see results from other concurrent workers. This eliminates race conditions and makes reasoning about correctness straightforward.

**Trade-off:** Workers don't see each other's results within a batch. This means if Worker 1 discovers a breakthrough, Worker 2 (running concurrently) won't know about it until the next batch. In practice, this is a minor cost because the breakthrough will be available in the next batch's snapshot.

### 3. Iteration Execution

Each worker runs one complete iteration:

```python
def _run_iteration_worker(iteration, db_snapshot, parent_id, inspiration_ids):
    """
    Execute a single evolution iteration in a worker process.
    Returns a SerializableResult.
    """
    try:
        # 1. Reconstruct programs from snapshot
        parent = reconstruct_program(db_snapshot, parent_id)
        inspirations = [reconstruct_program(db_snapshot, iid) for iid in inspiration_ids]

        # 2. Build prompt
        prompt = _worker_prompt_sampler.build_prompt(
            current_program=parent.code,
            program_metrics=parent.metrics,
            top_programs=get_top_from_snapshot(db_snapshot),
            inspirations=inspirations,
            ...
        )

        # 3. Call LLM (async inside sync worker using event loop)
        loop = asyncio.new_event_loop()
        llm_response = loop.run_until_complete(
            _worker_llm_ensemble.generate_with_context(prompt["system"], prompt["user"])
        )

        # 4. Parse response → new code
        new_code = parse_llm_response(llm_response, parent.code)

        # 5. Evaluate
        metrics = loop.run_until_complete(
            _worker_evaluator.evaluate_program(new_code, f"prog_{iteration}")
        )

        # 6. Return serializable result
        return SerializableResult(
            child_program_dict=create_program_dict(new_code, metrics, parent),
            parent_id=parent_id,
            iteration_time=elapsed,
            ...
        )
    except Exception as e:
        return SerializableResult(error=str(e), iteration=iteration, ...)
```

### 4. Result Collection

The Controller collects results from all workers and updates the database:

```python
# In the Controller:
async def _run_evolution_with_checkpoints(self, start_iter, max_iter):
    for batch_start in range(start_iter, max_iter, self.batch_size):
        # Submit batch of iterations to worker pool
        futures = []
        for i in range(self.batch_size):
            snapshot = self.database.create_snapshot()
            parent, inspirations = self.database.sample()
            future = self.pool.submit(
                _run_iteration_worker,
                iteration=batch_start + i,
                db_snapshot=snapshot,
                parent_id=parent.id,
                inspiration_ids=[insp.id for insp in inspirations]
            )
            futures.append(future)

        # Collect results (sequential — database modifications are thread-safe)
        for future in futures:
            result = future.result()
            if result.child_program_dict and not result.error:
                program = Program.from_dict(result.child_program_dict)
                self.database.add(program, result.iteration, result.target_island)

        # Checkpoint if needed
        if batch_start % self.checkpoint_interval == 0:
            self._save_checkpoint(batch_start)
```

---

## The Worker Lifecycle

```
Worker Process Created
    │
    ▼
_worker_init() called
    │ Reconstruct Config
    │ Create LLMEnsemble (new API clients)
    │ Create PromptSampler (load templates)
    │ Create Evaluator (load evaluation module)
    │
    ▼
Ready for iterations
    │
    ├──▶ _run_iteration_worker(iter_1, snapshot, ...)
    │        │ Sample from snapshot
    │        │ Build prompt
    │        │ Call LLM API
    │        │ Parse response
    │        │ Evaluate program
    │        │ Return SerializableResult
    │        ▼
    ├──▶ _run_iteration_worker(iter_5, snapshot, ...)
    │        │ (same process, reuses components)
    │        ▼
    └──▶ ... (reused for multiple iterations)
    │
    ▼
Pool shutdown → Worker process terminated
```

**Rationale:** Worker processes are **reused** across iterations (they're not created and destroyed each time). This amortizes the initialization cost (loading models, creating API clients) across many iterations.

---

## Graceful Shutdown

The Controller handles `Ctrl+C` (SIGINT) and `SIGTERM` gracefully:

```python
class OpenEvolve:
    def __init__(self):
        signal.signal(signal.SIGINT, self._signal_handler)
        signal.signal(signal.SIGTERM, self._signal_handler)
        self._shutdown_requested = False

    def _signal_handler(self, signum, frame):
        if self._shutdown_requested:
            # Second signal → force quit
            sys.exit(1)
        self._shutdown_requested = True
        # First signal → graceful shutdown
        # Finish current iteration batch, save checkpoint, then exit
```

**Rationale:** Evolution runs can take hours. Graceful shutdown ensures:
- Current iterations complete (not wasted compute)
- A checkpoint is saved (progress is preserved)
- Resources are cleaned up (file handles, API connections)
- A second Ctrl+C force-quits if the graceful shutdown hangs

---

## Checkpointing

### What Gets Saved

```
openevolve_output/
├── checkpoints/
│   ├── checkpoint_10/
│   │   ├── metadata.json       # Iteration number, timestamp, scores
│   │   ├── database.json       # Complete database state
│   │   └── programs/           # Per-program files
│   │       ├── prog_1/
│   │       │   ├── program.json
│   │       │   └── artifacts/
│   │       └── ...
│   ├── checkpoint_20/
│   │   └── ...
│   └── latest -> checkpoint_20  # Symlink to most recent
├── logs/
│   └── evolution.log
└── evolution_trace.jsonl
```

### Checkpoint Metadata

```json
{
    "iteration": 100,
    "timestamp": 1709234567.89,
    "best_score": 0.922,
    "best_program_id": "prog_42",
    "num_programs": 347,
    "num_islands": 5,
    "config_hash": "abc123..."
}
```

### Resume from Checkpoint

```bash
python openevolve-run.py program.py evaluator.py \
    --config config.yaml \
    --checkpoint openevolve_output/checkpoints/checkpoint_50 \
    --iterations 100
```

**What happens on resume:**
1. Load `database.json` → reconstruct all programs, islands, feature maps
2. Load per-program files → restore code, metrics, artifacts
3. Restore generation counts → migration timing continues correctly
4. Set starting iteration to checkpoint iteration + 1
5. Continue evolution as if nothing happened

**Rationale:** Long evolution runs (hours/days) need resilience against:
- Machine crashes
- Out-of-memory errors
- User interrupts (Ctrl+C)
- Power outages
- Experimentation (save state, try different configs, compare)

---

## Error Resilience

Individual iteration failures don't crash the system:

```python
for future in futures:
    result = future.result()
    if result.error:
        logger.warning(f"Iteration {result.iteration} failed: {result.error}")
        continue  # Skip this iteration, try the next one
    # ... process successful result
```

**Types of failures handled:**
| Failure | Response |
|---------|----------|
| LLM API timeout | Retry in worker, then return error result |
| LLM returns garbage | Parse failure → skip iteration |
| Evaluation timeout | Return score 0.0, skip iteration |
| Evaluation crash | Catch exception, return error result |
| Worker process crash | ProcessPoolExecutor handles restart |
| Out-of-memory | Worker process dies, pool creates new one |

**Rationale:** In a 1000-iteration run, some failures are inevitable (API hiccups, edge-case programs). The system prioritizes **availability** (keep running) over **consistency** (every iteration must succeed). A 5% failure rate barely affects evolution quality.

---

## Key Design Decisions

### 1. Why ProcessPoolExecutor Instead of Custom Process Management?

Python's `ProcessPoolExecutor` is battle-tested, handles worker lifecycle, and integrates with `concurrent.futures`. Building custom process management would add complexity without benefit.

### 2. Why Snapshots Instead of Shared Memory?

Shared memory (e.g., `multiprocessing.Manager`) introduces:
- Lock contention (workers block each other)
- Complexity (what if a read happens during a write?)
- Hard-to-debug race conditions

Snapshots are **simple and correct** at the cost of slightly stale data.

### 3. Why Batch Processing Instead of Continuous Submission?

Batches provide natural checkpointing boundaries. After each batch:
- All results are collected
- Database is updated
- Checkpoint can be saved
- Early stopping can be checked

Continuous submission would need more complex checkpoint logic.

### 4. Why Reconstruct Components in Each Worker?

Python's `multiprocessing` uses `fork()` or `spawn()` to create workers. Neither cleanly handles:
- Network connections (sockets can't be shared)
- File handles (not portable across processes)
- Event loops (each process needs its own)

Reconstruction is the clean, reliable approach.

### 5. Why Two-Stage Shutdown?

First `Ctrl+C` = "finish gracefully." Second `Ctrl+C` = "stop now." This matches user expectations:
- If you want to save progress: press Ctrl+C once, wait
- If something is hung: press Ctrl+C twice, force quit
