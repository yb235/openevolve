# Database & MAP-Elites

This document explains how OpenEvolve stores and organizes programs using the MAP-Elites algorithm with island-based evolution. These are the core mechanisms that prevent the system from converging on a single solution and maintain diversity throughout the evolution process.

## Why Not Just Keep the Best Program?

The naive approach to optimization is: keep the best solution, mutate it, keep the new one if it's better. This is called **hill climbing**, and it has a fatal flaw: **local optima**.

```
Fitness
  ▲
  │        ╱╲
  │       ╱  ╲         ╱╲
  │      ╱    ╲       ╱  ╲
  │     ╱      ╲     ╱    ╲      ╱╲  ← Global optimum
  │    ╱        ╲   ╱      ╲    ╱  ╲
  │   ╱     X    ╲ ╱        ╲  ╱    ╲
  │  ╱  stuck!    ╲          ╲╱      ╲
  │ ╱              ╲                   ╲
  └──────────────────────────────────────▶ Solution space
```

Hill climbing gets stuck at the first peak it finds. OpenEvolve uses two complementary strategies to avoid this:
1. **MAP-Elites** — Maintain diversity across solution characteristics
2. **Island Evolution** — Run multiple independent populations

---

## MAP-Elites: Quality-Diversity Optimization

### The Concept

MAP-Elites (Multi-dimensional Archive of Phenotypic Elites) doesn't just find the best solution — it finds the **best solution for every type of solution**.

Imagine a 2D grid where:
- X-axis = code complexity (how many operations)
- Y-axis = execution speed (how fast it runs)

Each cell in the grid keeps the highest-scoring program that falls in that region:

```
Diversity ▲
(speed)   │
          │  ┌────┬────┬────┬────┬────┐
   Fast   │  │0.72│    │0.89│    │0.65│
          │  ├────┼────┼────┼────┼────┤
          │  │    │0.81│    │0.77│    │
          │  ├────┼────┼────┼────┼────┤
          │  │0.68│    │0.92│    │0.71│  ← Each cell keeps the
          │  ├────┼────┼────┼────┼────┤     BEST program for that
          │  │    │0.75│    │0.83│    │     complexity/speed combo
          │  ├────┼────┼────┼────┼────┤
   Slow   │  │0.61│    │0.78│    │0.69│
          │  └────┴────┴────┴────┴────┘
          └────────────────────────────▶ Complexity (simple → complex)
```

### How It Works in OpenEvolve

**File:** `openevolve/database.py`

**1. Feature Extraction:**
When a program is evaluated, its metrics include feature dimensions:
```python
metrics = {
    "combined_score": 0.85,     # Fitness (what we optimize)
    "complexity": 7.0,          # Feature dimension 1 (raw value)
    "diversity": 0.3            # Feature dimension 2 (raw value)
}
```

**2. Feature Binning:**
Raw values are mapped to grid cells using min-max normalization:
```python
def _calculate_feature_coords(self, program):
    coords = []
    for dim_name in self.feature_dimensions:
        raw_value = program.metrics.get(dim_name, 0)
        # Normalize to [0, 1] using observed min/max
        normalized = (raw_value - self.min_values[dim]) / (self.max_values[dim] - self.min_values[dim])
        # Map to bin index
        bin_index = int(normalized * self.num_bins)
        bin_index = max(0, min(bin_index, self.num_bins - 1))
        coords.append(bin_index)
    return tuple(coords)
```

**3. Cell Competition:**
When a new program maps to a cell that already has a program:
- If the new program has a **higher** `combined_score` → **replace** the existing one
- If the new program has a **lower** score → **discard** it
- If the cell is **empty** → **insert** directly

**Rationale:** MAP-Elites maintains a diverse archive of high-quality solutions. Even if a "simple and slow" program has a lower score than a "complex and fast" one, both are kept because they represent different trade-offs. This diversity helps the LLM explore multiple approaches.

### Configurable Feature Dimensions

You're not limited to `complexity` and `diversity`. Any metric returned by the evaluator can be a feature dimension:

```yaml
database:
  feature_dimensions: ["execution_time", "memory_usage"]  # Custom dimensions
  feature_bins: 10                                          # 10x10 grid = 100 cells
```

```yaml
database:
  feature_dimensions: ["accuracy", "model_size", "latency"]  # 3D grid
  feature_bins: [10, 5, 8]                                     # Different bins per dim
```

**Important rule:** Feature dimension values must be **raw continuous numbers** (e.g., `execution_time: 2.5`), **not** pre-binned indices (e.g., `execution_time_bin: 3`). OpenEvolve handles all binning internally.

---

## Island-Based Evolution

### The Concept

Inspired by island biogeography, OpenEvolve runs multiple **isolated populations** that evolve independently. Periodically, top programs **migrate** between islands, sharing discoveries.

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Island 0    │     │  Island 1    │     │  Island 2    │
│              │     │              │     │              │
│  Population: │     │  Population: │     │  Population: │
│  200 progs   │     │  200 progs   │     │  200 progs   │
│              │     │              │     │              │
│  Evolving    │     │  Evolving    │     │  Evolving    │
│  approach A  │     │  approach B  │     │  approach C  │
│              │     │              │     │              │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                    │
       │    Migration       │    Migration       │
       │    (every 50       │    (every 50       │
       │     generations)   │     generations)   │
       │                    │                    │
       └──────▶ top 10% ──▶┘                    │
               ◀── top 10% ◀────────────────────┘
                            └──── top 10% ──────▶
```

### Why Islands?

Without islands, all programs share one population. If one approach gets a high score early, the entire population converges on that approach. Good alternatives never get a chance to develop.

With islands:
- Each island can discover a **different approach** independently
- Migration shares breakthroughs without overwhelming diversity
- Even if Island 0 converges on approach A, Island 1 might find a better approach B

### How Migration Works

**File:** `openevolve/database.py`

Migration happens every `migration_interval` **generations** (not iterations):

```python
def _maybe_migrate(self, island_idx):
    """Check if this island should trigger migration."""
    if self.generation_counts[island_idx] % self.migration_interval == 0:
        self._migrate_from(island_idx)

def _migrate_from(self, source_island):
    """Copy top programs from source to next island."""
    target_island = (source_island + 1) % self.num_islands  # Round-robin
    top_programs = self.get_top_programs(
        n=int(self.population_size * self.migration_rate),
        island_idx=source_island
    )
    for program in top_programs:
        copy = program.copy()  # Deep copy — original stays in source
        self._add_to_island(copy, target_island)
```

**Key detail:** Migration is based on **generation count**, not iteration count. This means migration is triggered by actual evolutionary progress on an island, not just by time passing. An island that hasn't produced new generations won't trigger migration.

**Rationale:** Generation-based migration (called "lazy migration") ensures that only active islands contribute to migration. If a particular island is producing lots of failed mutations (and thus not advancing generations), it won't pollute other islands with low-quality programs.

### Island Configuration

```yaml
database:
  num_islands: 5             # Number of separate populations
  migration_interval: 50     # Migrate every 50 generations
  migration_rate: 0.1        # Top 10% of programs migrate
  population_size: 1000      # Max programs per island
```

**Guidelines:**
| Problem Type | Islands | Migration | Rationale |
|-------------|---------|-----------|-----------|
| Simple optimization | 3 | Rare (100) | Fewer islands = faster convergence |
| Complex algorithm design | 5-7 | Medium (50) | Balance of diversity and focus |
| Open-ended exploration | 10+ | Frequent (25) | Maximum diversity |
| Quick prototyping | 2-3 | Very rare (200) | Fast iteration, minimal overhead |

---

## Sampling Strategy

When it's time to create a new program, the database needs to select a **parent** (to mutate) and **inspirations** (for the LLM's context).

### Three Sampling Modes

**File:** `openevolve/database.py`

```python
def sample_from_island(self, island_id, num_inspirations):
    roll = random.random()

    if roll < self.elite_selection_ratio:
        # ELITE: Pick from the best programs
        parent = random.choice(self.get_top_programs(n=10, island_idx=island_id))

    elif roll < self.elite_selection_ratio + self.exploration_ratio:
        # EXPLORATION: Pick a random program
        parent = random.choice(self.island_populations[island_id])

    else:
        # EXPLOITATION: Pick from the MAP-Elites archive
        parent = random.choice(list(self.feature_maps[island_id].values()))

    # Inspirations are always from top + diverse programs
    inspirations = self._get_diverse_inspirations(island_id, num_inspirations)
    return parent, inspirations
```

**Default ratios:**
- **Elite (10%):** Select from the top-scoring programs → exploitation of known good solutions
- **Exploration (20%):** Select random programs → discover unexplored areas
- **Exploitation (70%):** Select from MAP-Elites archive → refine diverse high-quality solutions

**Rationale:** The 10/20/70 split balances:
- **Not wasting time** on random mutations (only 20% exploration)
- **Not getting stuck** on the current best (only 10% elite)
- **Making the most of diversity** by exploiting the MAP-Elites archive (70% exploitation)

### Double-Selection Pattern

OpenEvolve uses a subtle but important pattern: the **parent** (the program being mutated) and the **inspirations** (programs shown to the LLM for context) are selected **independently**.

```
Parent selection:        Inspiration selection:
  Roll dice → pick one    Get top N programs + diverse M programs
  program to mutate        (from the same island)
```

**Rationale:** If the parent and inspirations were always the same, the LLM would just copy the parent. By showing diverse inspirations alongside the parent, the LLM can combine ideas from multiple programs — a form of "crossover" in genetic algorithm terms.

---

## Population Management

### Population Limits

Each island has a maximum `population_size`. When it's exceeded:

```python
def _enforce_population_limit(self, island_idx):
    population = self.island_populations[island_idx]
    if len(population) > self.population_size:
        # Sort by timestamp (oldest first)
        population.sort(key=lambda p: p.timestamp)
        # Remove oldest programs that aren't in the archive
        while len(population) > self.population_size:
            candidate = population[0]
            if candidate.id not in self.feature_maps[island_idx].values():
                population.pop(0)  # Remove oldest non-archive program
            else:
                break  # Don't remove archive programs
```

**Rationale:** LRU (Least Recently Used) eviction keeps the population fresh while protecting MAP-Elites archive members. Archive programs represent the best in each feature cell — they're too valuable to evict.

### Best Program Tracking

The database tracks the absolute best program across all islands and all time:

```python
def add(self, program, iteration, target_island=None):
    # ... normal insertion logic ...

    # Track absolute best
    fitness = get_fitness_score(program.metrics, self.feature_dimensions)
    if fitness > self.best_fitness:
        self.best_fitness = fitness
        self.best_program = program
```

**Rationale:** Even though MAP-Elites maintains many programs, users ultimately want "the best program." Tracking it separately avoids an expensive search through all islands.

---

## Novelty Checking

Before adding a program to the database, OpenEvolve can check if it's "meaningfully different" from existing programs.

### Embedding-Based Novelty

```python
def _is_novel_embedding(self, program, island_idx):
    """Check novelty using code embedding similarity."""
    embedding = self.embedding_client.get_embedding(program.code)
    for existing in self.diversity_reference_set[island_idx]:
        similarity = cosine_similarity(embedding, existing.embedding)
        if similarity > self.novelty_threshold:
            return False  # Too similar to existing program
    return True
```

### LLM-Based Novelty

```python
def _is_novel_llm(self, program, island_idx):
    """Ask an LLM if this program is meaningfully different."""
    # Uses novelty_judge.py templates
    # LLM responds with "NOVEL" or "NOT_NOVEL"
```

**Rationale:** Without novelty checking, the database fills up with near-identical programs (same algorithm, slightly different variable names). Novelty checking ensures diversity is maintained in the population, which improves the quality of mutations.

---

## Checkpoint & Persistence

The database can serialize its entire state to disk and restore it later.

### What Gets Saved

```
checkpoint/
├── database.json          # All programs, islands, feature maps, generation counts
└── programs/
    ├── prog_1/
    │   ├── program.json   # Individual program (code, metrics, metadata)
    │   └── artifacts/     # Large artifact files
    └── prog_2/
        └── program.json
```

### What Gets Restored

On resume:
1. All programs are reconstructed from JSON
2. Island assignments are restored
3. MAP-Elites grid is rebuilt
4. Generation counts are restored (so migration timing continues correctly)
5. Best program tracking is restored

**Rationale:** Checkpointing enables:
- **Long runs** — Evolve for days, resuming after failures
- **Experimentation** — Save state, try different configs, compare results
- **Visualization** — Load checkpoints into the visualizer for analysis

---

## Key Design Decisions

### 1. Why MAP-Elites Instead of Simple Tournament Selection?

Tournament selection (pick N random programs, keep the best) loses diversity. MAP-Elites guarantees coverage across the feature space, which:
- Prevents premature convergence
- Provides diverse examples to the LLM
- Discovers surprising solutions in unexpected regions

### 2. Why Islands Instead of One Large Population?

One population converges on the dominant approach. Islands allow:
- Parallel exploration of different strategies
- Protection of minority approaches from being overwhelmed
- Migration provides controlled knowledge transfer

### 3. Why Generation-Based Migration Instead of Iteration-Based?

Iterations count all attempts (including failures). Generations count successful additions. Generation-based migration ensures migration is triggered by actual progress, not just the passage of time.

### 4. Why Store Programs Instead of Just Code?

The `Program` object captures the full evolutionary context: parent, generation, metrics, prompts, artifacts. This enables:
- Genealogy tracking (who descended from whom)
- Performance analysis (which mutations helped)
- Visualization (evolution tree)
- Debugging (what prompt produced this code)

### 5. Why Separate Feature Dimensions from Fitness?

Fitness (`combined_score`) is what we optimize. Feature dimensions define the diversity space. Keeping them separate means:
- Feature dimensions don't bias the fitness score
- You can change features without affecting optimization
- The system maintains diversity even when all programs have similar fitness
