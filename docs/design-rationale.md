# Design Rationale

This document explains the **"why"** behind every major architectural decision in OpenEvolve. For each design choice, we explain the alternatives that were available, the trade-offs considered, and why this particular approach was chosen.

---

## 1. LLMs as Mutation Operators

### The Decision
Use LLMs to generate code mutations instead of traditional mutation operators (random token changes, syntax-tree transformations, etc.).

### Why This Works
Traditional genetic programming uses random mutations: swap a `+` for a `*`, change a constant, shuffle code blocks. These mutations are blind — they don't understand what the code does.

LLMs understand code semantics. When shown a sorting algorithm with poor performance, an LLM doesn't randomly change characters — it might suggest switching from bubble sort to quicksort. This is **orders of magnitude more efficient** than random search.

### The Trade-Off
- **Cost**: LLM API calls cost money (vs. free random mutations)
- **Speed**: API calls take seconds (vs. microseconds for random mutations)
- **Compensating benefit**: Each LLM mutation has a much higher chance of improving the program, so fewer iterations are needed

### Why It's Worth It
A single LLM-guided iteration can make improvements that would take thousands of random mutations to discover. The reduced iteration count more than compensates for the higher per-iteration cost.

---

## 2. MAP-Elites for Quality-Diversity

### The Decision
Use the MAP-Elites algorithm instead of standard evolutionary selection (tournament selection, roulette wheel, etc.).

### The Problem with Standard Selection
Standard selection keeps the best N programs. Over time, all programs converge to the same approach. This is fine for unimodal optimization but terrible for creative code evolution where the best approach isn't known in advance.

### Why MAP-Elites
MAP-Elites maintains a grid of programs across feature dimensions (e.g., complexity × speed). Each cell in the grid keeps only the best program for that particular characteristic combination. This guarantees:

1. **Diversity**: Programs with different characteristics coexist
2. **Quality**: Each cell contains the best-possible program for that type
3. **Stepping stones**: Mediocre programs in unusual regions can be ancestors of future breakthroughs

### Real-World Analogy
Think of it like preserving biodiversity. You don't want all animals to be lions (highest predator score). You want lions AND eagles AND dolphins — each best-in-class for their niche. A future environmental change might make the dolphin's traits suddenly valuable.

### The Trade-Off
- **Memory**: Storing a grid of programs uses more memory than storing just the top N
- **Complexity**: Feature extraction and binning add computational overhead
- **Compensating benefit**: Dramatically better exploration of the solution space

---

## 3. Island-Based Evolution

### The Decision
Maintain multiple isolated populations (islands) that evolve independently with periodic migration.

### The Problem It Solves
Even with MAP-Elites, a single population tends to develop a "culture" — a dominant coding style or algorithmic approach. Once this culture dominates, the LLM sees similar programs in every prompt and generates similar mutations.

### How Islands Help
Each island develops its own culture independently:
- Island 0 might converge on simulated annealing
- Island 1 might converge on genetic algorithms
- Island 2 might develop a novel approach nobody expected

Periodic migration shares breakthroughs without homogenizing the populations.

### Inspiration
This design comes from **island biogeography** in biology. Darwin's finches evolved different beak shapes on different Galápagos islands because each island had different food sources. When islands occasionally connect (land bridges, storms), beneficial traits spread.

### The Trade-Off
- **Compute**: More islands = more programs to evaluate per iteration
- **Communication**: Migration adds complexity
- **Convergence speed**: Each island has fewer programs, so individual island progress is slower
- **Compensating benefit**: Global solution quality is higher because more of the search space is explored

---

## 4. Lazy Migration (Generation-Based, Not Iteration-Based)

### The Decision
Trigger island migration based on **generation count** (successful additions to an island), not **iteration count** (total attempts).

### Why It Matters
Consider two islands:
- Island A: Very productive, 50 successful programs in 100 iterations
- Island B: Struggling, only 5 successful programs in 100 iterations

With iteration-based migration (every 100 iterations), both islands migrate at the same time. Island B's mediocre programs pollute Island A.

With generation-based migration (every 50 generations), Island A migrates when it has 50 new programs (productive sharing), and Island B migrates only when it has accumulated enough progress to share something worthwhile.

### The Trade-Off
- **Complexity**: Generation counting per island is more complex than a global iteration counter
- **Unpredictability**: Migration timing becomes less predictable
- **Compensating benefit**: Higher quality migrations, less pollution between islands

---

## 5. Diff-Based Evolution (SEARCH/REPLACE)

### The Decision
Default to diff-based code evolution (LLM returns SEARCH/REPLACE blocks) instead of full code rewrites.

### Why Diffs Are Better
When you ask an LLM to rewrite an entire 500-line program, several things go wrong:
1. Working code gets accidentally changed
2. The LLM "forgets" parts of the original (context window limitations)
3. Small improvements get lost in large rewrites
4. It's hard to tell what actually changed

Diffs solve all of these:
1. Only the targeted section changes
2. Context outside the diff is preserved exactly
3. Changes are small and focused
4. It's obvious what changed

### When to Use Full Rewrite
Full rewrite mode is useful when:
- The initial code is very short (< 50 lines)
- You want fundamental restructuring
- The code has become unmaintainable and needs a clean slate

### The Trade-Off
- **Limitation**: Diffs can't make changes across distant code sections easily
- **Complexity**: SEARCH blocks must match the existing code exactly
- **Compensating benefit**: Much higher mutation success rate

---

## 6. Cascade Evaluation

### The Decision
Implement multi-stage evaluation where cheap checks run first and expensive benchmarks run only on promising programs.

### The Math
Assume:
- Stage 1 (syntax check): 0.1 seconds, rejects 40% of programs
- Stage 2 (basic tests): 10 seconds, rejects 30% of remaining
- Stage 3 (full benchmark): 300 seconds

**Without cascade**: Every program takes 300 seconds → 1000 programs = 83 hours

**With cascade**: 
- 1000 × 0.1s = 100s (Stage 1)
- 600 × 10s = 6000s (Stage 2, 60% pass Stage 1)
- 420 × 300s = 126,000s (Stage 3, 70% pass Stage 2)
- Total: 126,100s = 35 hours (58% savings)

### The Trade-Off
- **Complexity**: Users must implement multiple evaluation functions
- **Risk**: A program might pass Stage 1 but fail Stage 3 in a way Stage 1 could have caught
- **Compensating benefit**: Dramatic time savings, especially for expensive evaluations

---

## 7. Process-Based Parallelism (Not Async, Not Threading)

### The Decision
Use `ProcessPoolExecutor` (multiprocessing) instead of async I/O or threading.

### Why Not Async Only?
Async I/O excels at network calls (LLM API), but program evaluation is CPU-bound (running benchmarks, executing code). Async doesn't help with CPU work — Python's GIL blocks parallel CPU execution.

### Why Not Threading?
Python's Global Interpreter Lock (GIL) prevents threads from running Python code in parallel. Threading only helps with I/O waits, not CPU computation.

### Why Multiprocessing?
True process-based parallelism:
- Each worker gets its own Python interpreter → no GIL
- CPU-heavy evaluation runs truly in parallel
- Memory isolation prevents cross-worker interference

### The Trade-Off
- **Overhead**: Process creation and data serialization add overhead
- **Shared state**: Can't share objects directly; need serializable snapshots
- **Memory**: Each process duplicates the Python runtime
- **Compensating benefit**: True parallel execution of CPU-bound evaluation

---

## 8. Database Snapshots for Workers

### The Decision
Workers receive serialized database snapshots instead of accessing the live database.

### Why Not Shared Memory?
Shared memory (e.g., `multiprocessing.Manager`) introduces:
- Lock contention: Workers block each other when accessing the database
- Race conditions: What if a read happens during a write?
- Complexity: Debugging concurrent state mutations is extremely hard

### Why Snapshots Work
Each worker sees a **consistent, frozen view** of the database at iteration start time. This means:
- No locks needed (read-only access)
- No race conditions (can't modify the snapshot)
- Reasoning is simple (snapshot is immutable)

### The Trade-Off
- **Staleness**: Workers don't see each other's results within a batch
- **Memory**: Each snapshot duplicates the database
- **Compensating benefit**: Simplicity, correctness, debuggability

---

## 9. Artifact System (Side-Channel Diagnostics)

### The Decision
Separate diagnostic outputs (artifacts) from evaluation metrics, with size-based storage split (inline JSON for small, disk files for large).

### Why Separate from Metrics?
Metrics are numbers used for selection. Artifacts are diagnostic data for debugging. Mixing them causes problems:
- Non-numeric data in metrics breaks sorting/averaging
- Large artifacts bloat the database
- Artifacts don't contribute to fitness

### The Size-Based Split
Small artifacts (< 10KB): Stored inline in `Program.artifacts_json`. Fast access, always available.
Large artifacts (≥ 10KB): Stored as files on disk. Don't bloat memory.

### The Trade-Off
- **Complexity**: Two storage paths for artifacts
- **Consistency**: Disk artifacts might get deleted or moved
- **Compensating benefit**: Memory-efficient, fast for common case (small artifacts)

---

## 10. Dual LLM Ensembles (Evolution + Evaluation)

### The Decision
Maintain separate LLM ensembles for code generation and code evaluation.

### Why Separate?
Code generation and code evaluation are fundamentally different tasks:
- **Generation** benefits from creativity (high temperature, diverse models)
- **Evaluation** benefits from consistency (low temperature, precise models)

Using the same model for both would force a compromise on temperature and model selection.

### The Trade-Off
- **Cost**: Two sets of API calls
- **Complexity**: Two configurations to maintain
- **Compensating benefit**: Better generation quality AND better evaluation quality

---

## 11. Template-Based Prompt System

### The Decision
Use text templates with `{placeholders}` instead of hardcoded prompt strings.

### Why Templates?
1. **Customization**: Users can provide domain-specific prompt templates
2. **Experimentation**: Easy to A/B test different prompt strategies
3. **Maintenance**: Changing prompt wording doesn't require code changes
4. **Language support**: Different templates for Python, Rust, R, etc.

### The Trade-Off
- **Indirection**: Templates are harder to trace than inline strings
- **Error handling**: Missing placeholders cause runtime errors
- **Compensating benefit**: Flexibility and separation of concerns

---

## 12. YAML Configuration with Environment Variable Resolution

### The Decision
Use YAML files for configuration with `${ENV_VAR}` syntax for secrets.

### Why YAML?
- Human-readable and writable
- Supports nested structures (perfect for hierarchical config)
- Comments for documentation
- Industry standard for configuration

### Why Environment Variables?
API keys must not be in config files (security). `${ENV_VAR}` syntax lets users reference secrets without embedding them:

```yaml
llm:
  api_key: "${OPENAI_API_KEY}"  # Resolved at runtime
```

### The Trade-Off
- **Parsing**: YAML has subtle gotchas (e.g., `yes` is a boolean)
- **Validation**: Typos in YAML aren't caught until runtime
- **Compensating benefit**: Familiarity, readability, widespread tooling support

---

## 13. Manual Mode (Human-in-the-Loop via File Queue)

### The Decision
Implement human-in-the-loop through a file-based task queue, not WebSockets or direct API calls.

### Why Files?
- **Simple**: No WebSocket server, no real-time protocol
- **Debuggable**: Task files can be inspected directly
- **Resilient**: Tasks survive server restarts
- **Process-safe**: Multiple processes can read/write atomically

### Why Not WebSockets?
WebSockets add:
- Connection management complexity
- State synchronization issues
- Server restart = lost connections
- More dependencies

### The Trade-Off
- **Latency**: 1-second polling delay (vs. instant WebSocket)
- **Not real-time**: Can't push updates to the browser
- **Compensating benefit**: Simplicity, reliability, zero extra dependencies

---

## 14. Two-Stage Graceful Shutdown

### The Decision
First `Ctrl+C` triggers graceful shutdown (finish current batch, save checkpoint). Second `Ctrl+C` forces immediate exit.

### Why Two Stages?
Evolution runs invest significant compute in each iteration. Killing mid-iteration wastes that compute. But sometimes graceful shutdown hangs (network issue, infinite loop in evaluation), so a forced exit must be available.

### The Trade-Off
- **Complexity**: Signal handling adds code
- **User confusion**: Some users don't know to press Ctrl+C twice
- **Compensating benefit**: No wasted compute, no lost progress

---

## 15. Changes Description (Compact Program Representation)

### The Decision
Optionally represent programs as LLM-generated change summaries instead of full code in prompts.

### The Problem
When evolving large codebases (thousands of lines), including full code for 3 top programs + 2 inspirations in every prompt uses enormous amounts of tokens. This is:
- Expensive (more tokens = more cost)
- Slow (more tokens = more latency)
- Sometimes impossible (exceeds context window)

### The Solution
Instead of:
```
Top Program (2000 lines of Python code)
```

Show:
```
Top Program: Added simulated annealing with adaptive cooling. Modified main loop
to use temperature-based acceptance criteria. Added early stopping.
```

### The Trade-Off
- **Information loss**: Summaries omit implementation details
- **Dependency on LLM quality**: Bad summaries mislead evolution
- **Compensating benefit**: 90%+ token reduction, works within context windows

---

## Summary: Design Philosophy

OpenEvolve's architecture is guided by these principles:

1. **Diversity over convergence** — Multiple islands, MAP-Elites, template stochasticity
2. **Resilience over perfection** — Skip failed iterations, retry API calls, graceful shutdown
3. **Simplicity over cleverness** — File-based queues, snapshots instead of shared memory, YAML config
4. **Flexibility over opinion** — User-defined evaluators, customizable templates, multiple LLM providers
5. **Efficiency over completeness** — Cascade evaluation, compact representations, weighted ensembles
