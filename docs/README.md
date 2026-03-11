# OpenEvolve Documentation

Welcome to the OpenEvolve documentation. OpenEvolve is an open-source implementation of Google DeepMind's AlphaEvolve system — an evolutionary coding agent that uses Large Language Models (LLMs) to optimize code through iterative evolution.

If you're new here, start with the **Architecture Overview**, then follow the **Data Flow** to understand how pieces connect.

## Table of Contents

| Document | What You'll Learn |
|----------|-------------------|
| [Architecture Overview](architecture-overview.md) | The big picture: what components exist and how they fit together |
| [Data Flow](data-flow.md) | Step-by-step journey of data through the system, from input to output |
| [Data Schema](data-schema.md) | Every data structure, dataclass, and dictionary shape used in the codebase |
| [API Reference](api-reference.md) | Public Python APIs and CLI interface for running evolution |
| [LLM & Agent Architecture](llm-agent-architecture.md) | How LLMs are orchestrated as an ensemble, manual mode, and retry logic |
| [Database & MAP-Elites](database-and-map-elites.md) | The MAP-Elites algorithm, island-based evolution, and program storage |
| [Evaluation System](evaluation-system.md) | Cascade evaluation, artifact capture, and scoring |
| [Configuration Guide](configuration-guide.md) | Every config option explained with rationale and examples |
| [Parallel Processing](parallel-processing.md) | Process workers, parallelism, checkpointing, and resume |
| [Prompt Engineering](prompt-engineering.md) | Template system, prompt building, stochasticity, and context assembly |
| [Visualization & Manual Mode](visualization-and-manual-mode.md) | Flask UI, D3 graphs, evolution tree, and human-in-the-loop mode |
| [Design Rationale](design-rationale.md) | Why each architectural decision was made — the "why" behind the "what" |

## Quick Orientation

```
openevolve/                  # Core Python package
├── controller.py            # Main orchestrator (start here to understand flow)
├── database.py              # MAP-Elites + island populations
├── evaluator.py             # Program evaluation with cascade stages
├── iteration.py             # Single evolution iteration logic
├── process_parallel.py      # Parallel worker management
├── config.py                # Configuration dataclasses
├── api.py                   # Public Python API
├── cli.py                   # Command-line interface
├── llm/                     # LLM integration (ensemble, OpenAI client)
├── prompt/                  # Prompt templates and assembly
├── utils/                   # Helpers (code parsing, async, metrics)
├── evolution_trace.py       # Evolution logging
├── novelty_judge.py         # Novelty assessment via LLM
└── embedding.py             # Code embeddings for similarity

configs/                     # YAML configuration examples
scripts/                     # Visualization server and manual mode
examples/                    # 18+ worked examples across languages
```

## How to Read These Docs

- **"I want to understand the system"** → Start with [Architecture Overview](architecture-overview.md), then [Data Flow](data-flow.md)
- **"I want to use the system"** → Start with [API Reference](api-reference.md) and [Configuration Guide](configuration-guide.md)
- **"I want to contribute"** → Read [Design Rationale](design-rationale.md) and [Data Schema](data-schema.md)
- **"I want to understand a specific part"** → Jump directly to the relevant document above
