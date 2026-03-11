# LLM & Agent Architecture

This document explains how OpenEvolve orchestrates Large Language Models as evolutionary agents — the ensemble strategy, model selection, prompt construction, manual mode, and the reasoning behind each design choice.

## The Core Idea

OpenEvolve uses LLMs not as chatbots, but as **mutation operators** in an evolutionary algorithm. Instead of random bit-flips (traditional genetic algorithms), the "mutations" are intelligent code modifications suggested by LLMs that understand programming.

This is the key insight from DeepMind's AlphaEvolve: **LLMs can generate higher-quality mutations than random search because they understand code semantics**.

```
Traditional Evolution:     OpenEvolve:
  Random mutation            LLM-guided mutation
  ┌──────────┐               ┌──────────┐
  │ 01101001 │               │ def f(x): │
  │    ↓     │               │   return  │
  │ 01100001 │  (bit flip)   │   x**2    │  (LLM suggests improvement)
  └──────────┘               └──────────┘
```

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    LLM Ensemble                              │
│                                                              │
│  ┌────────────────┐  ┌────────────────┐  ┌───────────────┐ │
│  │  Small Model    │  │  Large Model    │  │ Manual Mode   │ │
│  │  (80% weight)   │  │  (20% weight)   │  │ (Human LLM)  │ │
│  │                 │  │                 │  │               │ │
│  │  Fast, cheap    │  │  Smart, costly  │  │  Human input  │ │
│  │  Exploration    │  │  Exploitation   │  │  Guided steps │ │
│  └────────────────┘  └────────────────┘  └───────────────┘ │
│                                                              │
│  Model Selection: Weighted random sampling                   │
│  Each call picks ONE model based on weights                  │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                    OpenAI-Compatible API                      │
│                                                              │
│  Supports: OpenAI, Azure, Google AI Studio, local servers    │
│  Protocol: Chat completions API                              │
│  Features: Retry, timeout, reasoning model detection         │
└─────────────────────────────────────────────────────────────┘
```

## The Ensemble Strategy

### Why Multiple Models?

Different LLMs excel at different things:
- **Small models** (e.g., `gemini-2.0-flash-lite`, `gpt-4o-mini`) are fast and cheap — good for generating lots of diverse mutations quickly
- **Large models** (e.g., `gemini-2.0-flash`, `gpt-4o`) are slower but smarter — good for making targeted, high-quality improvements
- **Reasoning models** (e.g., `o1`, `o3`) think step-by-step — good for solving hard algorithmic problems

By combining models with weights, you get the best of both worlds.

### How Model Selection Works

**File:** `openevolve/llm/ensemble.py`

```python
class LLMEnsemble:
    def _sample_model(self) -> LLMInterface:
        """Pick a model using weighted random selection."""
        weights = [model.weight for model in self.models]
        return random.choices(self.models, weights=weights, k=1)[0]
```

**Example configuration:**
```yaml
llm:
  models:
    - name: "gemini-2.0-flash-lite"
      weight: 0.8        # Selected 80% of the time
      temperature: 0.8    # Higher temperature = more creative
      max_tokens: 8192
    - name: "gemini-2.0-flash"
      weight: 0.2        # Selected 20% of the time
      temperature: 0.7    # Lower temperature = more precise
      max_tokens: 16384
```

**Rationale:** This is inspired by population genetics. Most offspring come from "average" parents (small model, more random), but occasionally a "superior" parent contributes (large model, more targeted). The ratio is tunable — more exploration for new problems, more exploitation for fine-tuning.

### Dual Ensemble Architecture

OpenEvolve maintains **two separate** LLM ensembles:

1. **Evolution Ensemble** — Generates code mutations
2. **Evaluator Ensemble** — Provides LLM-based code review feedback

```yaml
llm:
  models:               # Evolution ensemble
    - name: "gpt-4o-mini"
      weight: 1.0
  evaluator_models:     # Evaluator ensemble (separate)
    - name: "gpt-4o"
      weight: 1.0
```

**Rationale:** Generation and evaluation are fundamentally different tasks. Generation benefits from creativity (higher temperature, smaller models for diversity). Evaluation benefits from precision (lower temperature, larger models for accuracy). Separating them lets you optimize each independently.

---

## The LLM Interface

### Abstract Base Class

**File:** `openevolve/llm/base.py`

```python
class LLMInterface(ABC):
    @abstractmethod
    async def generate(self, prompt: str, **kwargs) -> str:
        """Generate a response from a single prompt."""
        pass

    @abstractmethod
    async def generate_with_context(
        self,
        system_message: str,
        messages: List[Dict[str, str]],
        **kwargs
    ) -> str:
        """Generate with structured system + user messages."""
        pass
```

**Rationale:** The abstract interface allows swapping LLM backends without changing the rest of the system. Today it's OpenAI-compatible APIs; tomorrow it could be local models, Anthropic, or any other provider.

### OpenAI Implementation

**File:** `openevolve/llm/openai.py`

The `OpenAILLM` class handles all the complexity of talking to LLM APIs:

**1. Reasoning Model Detection:**
```python
REASONING_MODEL_PATTERNS = ["o1", "o3", "o4", "gpt-5", "gemini-2.5"]

def _is_reasoning_model(self, model_name: str) -> bool:
    return any(pattern in model_name.lower() for pattern in self.REASONING_MODEL_PATTERNS)
```

Reasoning models use different API parameters:
- Standard models: `temperature`, `top_p`, `max_tokens`
- Reasoning models: `max_completion_tokens`, `reasoning_effort` (low/medium/high)

**Rationale:** Reasoning models (o1, o3) don't support temperature or top_p — they have their own "thinking" process. The system auto-detects these models and adjusts parameters accordingly, so users don't need to know the difference.

**2. API Call with Retry:**
```python
async def generate_with_context(self, system_message, messages, **kwargs):
    for attempt in range(self.max_retries):
        try:
            response = await asyncio.wait_for(
                self.client.chat.completions.create(
                    model=self.model_name,
                    messages=formatted_messages,
                    **params
                ),
                timeout=self.timeout
            )
            return response.choices[0].message.content
        except Exception as e:
            if attempt < self.max_retries - 1:
                await asyncio.sleep(self.retry_delay * (2 ** attempt))
            else:
                raise
```

**Rationale:** LLM APIs are unreliable — they timeout, rate-limit, and occasionally return errors. Exponential backoff retry ensures transient failures don't crash the evolution run. The timeout prevents hanging on slow responses.

**3. Seed Management:**
```python
# Skip seed for Google AI Studio (not supported)
if "generativelanguage.googleapis.com" not in self.api_base:
    params["seed"] = self.random_seed or 42
```

**Rationale:** Reproducibility matters for scientific experiments. Seeds ensure the same model call produces the same output. However, Google's API doesn't support seeds, so the system adapts.

---

## Manual Mode (Human-in-the-Loop)

One of the most creative features: you can replace any LLM with a **human**. The system writes a task file, displays it in a web UI, and waits for the human to type a response.

### How It Works

```
┌──────────────┐      ┌───────────────┐      ┌──────────────┐
│ OpenEvolve    │      │  Task Queue    │      │  Web UI       │
│ (iteration)   │─────▶│  (JSON files)  │─────▶│  (browser)    │
│               │      │               │      │               │
│ Waiting...    │      │ task_42.json   │      │ "Here's the   │
│               │◀─────│               │◀─────│  program..."   │
│ Got answer!   │      │ task_42       │      │ [User types   │
│               │      │  .answer.json │      │  improvement] │
└──────────────┘      └───────────────┘      └──────────────┘
```

**File:** `openevolve/llm/openai.py` (manual mode section)

```python
if self._manual_queue_dir:
    # Write task file
    task = {
        "id": task_id,
        "system_message": system_message,
        "messages": messages,
        "model": self.model_name,
        "created_at": time.time()
    }
    with open(task_path, 'w') as f:
        json.dump(task, f)

    # Poll for answer
    while not os.path.exists(answer_path):
        await asyncio.sleep(1)

    with open(answer_path, 'r') as f:
        answer = json.load(f)["response"]
    return answer
```

**Rationale:** Manual mode enables:
- **Debugging** — See exactly what the LLM would see and provide a better answer
- **Guided evolution** — Steer the system toward specific approaches
- **Teaching** — Show the system patterns it hasn't discovered
- **Hybrid approaches** — Mix automated and manual iterations

---

## The Generation Pipeline (Per Iteration)

Here's the full pipeline for a single code generation:

```
1. Controller picks an island
         │
         ▼
2. Database samples parent + inspirations from that island
         │
         ▼
3. Prompt Sampler builds system + user messages
   (current code, metrics, history, top programs, inspirations)
         │
         ▼
4. LLM Ensemble selects a model (weighted random)
         │
         ▼
5. Selected model generates response
   ├── Standard model: uses temperature + top_p
   └── Reasoning model: uses reasoning_effort
         │
         ▼
6. Response is parsed
   ├── Diff mode: extract SEARCH/REPLACE blocks → apply to code
   └── Full rewrite mode: extract code block from response
         │
         ▼
7. New code is validated (length, non-empty, parseable)
         │
         ▼
8. Evaluator runs the new code → metrics
         │
         ▼
9. Database stores the result (if novel and passes quality checks)
```

---

## Error Handling & Resilience

The LLM layer is designed to be **failure-tolerant**:

| Failure | Recovery |
|---------|----------|
| API timeout | Retry with exponential backoff (up to 3 times) |
| Rate limiting | Wait and retry with increasing delays |
| Invalid response | Log error, skip iteration, continue evolution |
| Model unavailable | Fall back to other models in ensemble |
| Empty response | Return empty string, iteration skipped |
| Connection error | Retry with backoff |

**Rationale:** In a 1000-iteration evolution run, individual failures are expected. The system prioritizes resilience over perfection — it's better to skip one iteration than crash the entire run.

---

## Key Design Decisions

### 1. Why OpenAI-Compatible API?

Nearly every LLM provider (OpenAI, Google, Azure, local ollama, vLLM) supports the OpenAI chat completions format. By standardizing on this protocol, OpenEvolve works with any provider out of the box.

### 2. Why Weighted Ensemble Instead of Always Using the Best Model?

- **Cost efficiency** — The best model might cost 100x more per token
- **Diversity** — Different models produce different kinds of mutations
- **Speed** — Small models respond faster, enabling more iterations per hour
- **Exploration vs exploitation** — Small models explore widely, large models exploit deeply

### 3. Why Async Generation?

LLM API calls are I/O-bound (waiting for network responses). Async allows the system to make multiple API calls concurrently without blocking, improving throughput.

### 4. Why Separate Evolution and Evaluation Ensembles?

Different tasks benefit from different model characteristics:
- **Evolution** (generation): Benefits from creativity, diversity, and speed
- **Evaluation** (judgment): Benefits from precision, consistency, and analytical depth

### 5. Why Manual Mode?

Not all problems have clear automated evaluators. Manual mode lets domain experts:
- Guide evolution toward promising directions
- Provide feedback that's hard to automate (aesthetics, usability)
- Debug when automated evolution gets stuck
- Teach the system about domain-specific patterns
