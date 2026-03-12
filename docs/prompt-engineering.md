# Prompt Engineering

This document explains how OpenEvolve constructs the prompts sent to LLMs — the template system, context assembly, stochasticity, and the strategies that make LLM-guided evolution effective.

## Why Prompts Matter So Much

In OpenEvolve, the prompt **is** the mutation operator. The quality of mutations depends entirely on what context the LLM receives. A good prompt produces targeted improvements; a bad prompt produces random changes.

```
Bad prompt:   "Improve this code."        → Random changes, often worse
Good prompt:  "Here's the code, its scores, what top performers look like,
               what failed before, and what to focus on."  → Targeted improvement
```

---

## Prompt Structure

Every LLM call in OpenEvolve uses a **two-part prompt**: a system message and a user message.

```
┌─────────────────────────────────────────────┐
│              SYSTEM MESSAGE                  │
│                                              │
│  Role definition and behavioral guidance.    │
│  "You are an expert software engineer        │
│   tasked with evolving code..."              │
│                                              │
│  Consistent across iterations.               │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│              USER MESSAGE                    │
│                                              │
│  Dynamic content that changes each iteration:│
│  - Current program code                      │
│  - Current metrics                           │
│  - Evolution history                         │
│  - Top performers                            │
│  - Diverse inspirations                      │
│  - Improvement suggestions                   │
│  - Artifacts (test results, errors)          │
└─────────────────────────────────────────────┘
```

---

## The Template System

### How Templates Work

**File:** `openevolve/prompt/templates.py`

Templates are text files in the `prompts/` directory. They use `{variable}` placeholders:

```text
## Current Program
```{language}
{current_program}
```

## Current Metrics
{metrics}

## Evolution History
{evolution_history}
```

The `TemplateManager` loads these templates and renders them with context:

```python
class TemplateManager:
    def __init__(self, template_dir=None):
        # Load built-in templates from prompts/ directory
        # Optionally override with custom templates from template_dir
        self.templates = {}
        self._load_templates()

    def render(self, template_name, **kwargs):
        """Render a template with the given variables."""
        template = self.templates[template_name]
        return template.format(**kwargs)
```

### Built-In Templates

| Template Name | Purpose |
|--------------|---------|
| `system_message` | Main evolution guidance (role, task description) |
| `evaluator_system_message` | LLM code review guidance |
| `system_message_changes_description` | For compact change descriptions |
| `system_message_with_changes_description` | Combined evolution + changes |
| `diff_user` | User message for diff-based evolution |
| `full_rewrite_user` | User message for full code rewrite |
| `evolution_history` | Format for previous attempts |
| `previous_attempt` | Single previous attempt entry |
| `top_program` | High-performing example entry |
| `inspiration_program` | Diverse approach entry |

### Fragments

Small template pieces that are conditionally included:

| Fragment | When Included |
|----------|---------------|
| `fitness_improved` | Child scored higher than parent |
| `fitness_declined` | Child scored lower than parent |
| `fitness_stable` | No meaningful change |
| `no_feature_coordinates` | No MAP-Elites features available |
| `exploring_region` | Program is in a new feature region |
| `code_too_long` | Code exceeds simplification threshold |
| `no_specific_guidance` | No special guidance needed |

**Rationale:** Fragment-based templates enable **conditional prompting** — the LLM gets different guidance depending on the current situation. If fitness is declining, it's told to try a different approach. If code is too long, it's told to simplify.

---

## The Prompt Sampler

### How Context Is Assembled

**File:** `openevolve/prompt/sampler.py`

The `PromptSampler` is the "brain" that assembles all the pieces:

```python
class PromptSampler:
    def build_prompt(
        self,
        current_program,         # Code being evolved
        parent_program=None,     # Parent code (for diff context)
        program_metrics=None,    # Current performance scores
        previous_programs=None,  # Recent attempts (for history)
        top_programs=None,       # Best programs found so far
        inspirations=None,       # Diverse programs for cross-pollination
        language="python",       # Programming language
        evolution_round=0,       # Current iteration number
        diff_based_evolution=True,
        program_artifacts=None,  # Test results, error logs
        feature_dimensions=None, # MAP-Elites dimensions
        current_changes_description=None  # Compact change summary
    ) -> Dict[str, str]:
        """
        Build complete prompt from all available context.
        Returns {"system": "...", "user": "..."}
        """
```

### What Gets Included (Step by Step)

**1. System Message:**
```
You are an expert software engineer tasked with improving code through
evolutionary optimization. Your goal is to make targeted modifications
that improve the program's performance metrics.

Key guidelines:
- Focus on the code between EVOLVE-BLOCK-START and EVOLVE-BLOCK-END
- Make meaningful changes that could improve the combined_score
- Consider both correctness and efficiency
- Learn from the top-performing examples provided
```

**2. Current Program:**
```python
## Current Program (Python)
```python
import numpy as np

# EVOLVE-BLOCK-START
def optimize(x):
    best = float('inf')
    for _ in range(100):
        candidate = x + np.random.randn(len(x)) * 0.1
        if sum(c**2 for c in candidate) < best:
            best = sum(c**2 for c in candidate)
            x = candidate
    return x
# EVOLVE-BLOCK-END
```

**3. Current Metrics:**
```
## Current Metrics
combined_score: 0.72
execution_time: 2.34
complexity: 5.00
```

**4. Evolution History:**
```
## Evolution History (Recent Attempts)

Attempt 1 (iteration 38):
  Score: 0.65 → 0.68 (+0.03)
  Summary: Added momentum term to search

Attempt 2 (iteration 40):
  Score: 0.68 → 0.72 (+0.04)
  Summary: Increased search iterations to 200
```

**5. Top Performers:**
```
## Top-Performing Programs

Program prog_89 (score: 0.91):
  Used simulated annealing with adaptive cooling
  [code snippet or changes description]

Program prog_67 (score: 0.88):
  Implemented CMA-ES optimization strategy
  [code snippet or changes description]
```

**6. Diverse Inspirations:**
```
## Diverse Inspirations

Program prog_23 (score: 0.62, island 3):
  Novel approach using genetic algorithm crossover
  [code snippet]

Program prog_45 (score: 0.55, island 1):
  Bayesian optimization with Gaussian processes
  [code snippet]
```

**7. Improvement Suggestions:**
```
## Suggestions
- Fitness has been improving. Continue exploring this direction.
- Consider combining ideas from the top performers.
- Current code complexity is moderate (5.0). Room for more sophisticated approaches.
```

**8. Artifacts (if available):**
```
## Evaluation Artifacts

Test Output:
  Test 1 (basic): PASS
  Test 2 (edge case): FAIL - expected 0.0, got 0.001
  Test 3 (performance): PASS - 1.2s (target: <2.0s)

Error Log:
  Warning: numerical instability at iteration 87
```

---

## Two Evolution Modes

### Diff-Based Evolution (Default)

The LLM returns SEARCH/REPLACE blocks that modify specific parts of the code:

**Prompt instruction (in system message):**
```
Respond with SEARCH/REPLACE blocks to modify the code:

<<<SEARCH
exact code to find
===REPLACE
new code to replace it with
>>>
```

**Example LLM response:**
```
I'll improve convergence by adding adaptive step sizes.

<<<SEARCH
    for _ in range(100):
        candidate = x + np.random.randn(len(x)) * 0.1
===REPLACE
    step_size = 1.0
    for i in range(200):
        candidate = x + np.random.randn(len(x)) * step_size
        step_size *= 0.995  # Adaptive cooling
>>>
```

**Rationale:** Diff-based evolution has several advantages:
- **Focused changes** — The LLM targets specific code sections
- **Preserves working code** — Unchanged parts aren't touched
- **Easier to validate** — Small diffs are easier to review
- **Better for large codebases** — Only the relevant part needs to change

### Full Rewrite Mode

The LLM returns the complete evolved code:

**Prompt instruction:**
```
Return the complete improved code in a code block.
```

**When to use full rewrite:**
- The initial code is very short (< 50 lines)
- You want radical restructuring
- Diff-based evolution is getting stuck

---

## Template Stochasticity

OpenEvolve can randomly vary prompts to produce more diverse LLM outputs:

```yaml
prompt:
  use_template_stochasticity: true
```

**How it works:**
```python
def _apply_template_variations(self, template):
    """Apply random variations to the template for diversity."""
    variations = self.config.template_variations  # Dict of {placeholder: [options]}
    for placeholder, options in variations.items():
        if placeholder in template:
            chosen = random.choice(options)
            template = template.replace(placeholder, chosen)
    return template
```

**Example variations:**
- "Make the code faster" vs "Optimize for execution speed" vs "Reduce computational complexity"
- "Focus on correctness" vs "Ensure all test cases pass" vs "Prioritize robustness"

**Rationale:** Without stochasticity, the same parent + same prompt → same LLM output. Template variations introduce diversity in the mutation process, similar to how different environmental pressures drive different evolutionary adaptations.

---

## Compact Representation (Changes Description)

For large codebases (thousands of lines), showing the full code in every prompt wastes tokens. OpenEvolve can use compact summaries instead:

```yaml
prompt:
  programs_as_changes_description: true
```

**Instead of showing full code:**
```
Program prog_89 (score: 0.91):
```python
[2000 lines of code]
```
```

**Shows a compact summary:**
```
Program prog_89 (score: 0.91):
  Changes from parent: Added simulated annealing with adaptive cooling schedule.
  Modified the main loop to use temperature-based acceptance criteria.
  Added early stopping when temperature drops below 0.001.
```

The changes description is maintained automatically by the LLM:
1. When a new program is created, the LLM describes what changed
2. This description is stored in `Program.changes_description`
3. In future prompts, the description is shown instead of full code

**Rationale:** LLMs have limited context windows. For large codebases, token efficiency matters. Compact descriptions preserve the essential information (what changed and why) while dramatically reducing prompt size.

---

## Improvement Suggestions

The prompt sampler analyzes the current evolution state and provides targeted suggestions:

```python
def _identify_improvement_areas(self, metrics, previous_metrics, feature_dimensions):
    suggestions = []

    # Fitness trend analysis
    if previous_metrics:
        delta = metrics["combined_score"] - previous_metrics["combined_score"]
        if delta > 0:
            suggestions.append("Fitness improved. Continue exploring this direction.")
        elif delta < 0:
            suggestions.append("Fitness declined. Try a different approach.")
        else:
            suggestions.append("Fitness is stable. Consider making bolder changes.")

    # Feature space exploration
    if feature_dimensions:
        for dim in feature_dimensions:
            if dim in metrics:
                suggestions.append(f"Current {dim}: {metrics[dim]:.2f}")

    # Code length management
    if len(current_code) > self.config.simplification_suggestion_threshold:
        suggestions.append("Code is getting long. Consider simplifying.")

    return "\n".join(suggestions)
```

**Rationale:** Suggestions give the LLM direction without constraining it. They're not commands — they're hints. This helps the LLM make more productive mutations, especially in later iterations when improvements become harder to find.

---

## Prompt for Different Components

### Evolution Prompts (Main)
- Goal: Generate improved code
- Temperature: Higher (0.7-0.9) for creativity
- Context: Full evolution history, top programs, inspirations

### Evaluation Prompts (Code Review)
- Goal: Assess code quality
- Temperature: Lower (0.2-0.4) for consistency
- Context: Just the code to review

### Novelty Prompts
- Goal: Determine if code is meaningfully different
- Temperature: Low (0.1-0.3) for reliability
- Context: Two programs to compare

---

## Key Design Decisions

### 1. Why Two-Part Prompts (System + User)?

- **System message** stays constant → establishes consistent behavior
- **User message** changes each iteration → provides fresh context
- Separating them allows caching of system message on some APIs

### 2. Why Show Top Programs AND Diverse Programs?

- **Top programs** show what works → exploitation signal
- **Diverse programs** show what else is possible → exploration signal
- Together, they enable the LLM to combine good ideas from different approaches

### 3. Why Include Evolution History?

Without history, the LLM might:
- Repeat failed mutations
- Undo successful changes
- Not know what's already been tried

History provides a learning signal: "this worked, this didn't, try something new."

### 4. Why Template-Based Instead of Hardcoded?

Templates enable:
- Custom prompts for specific domains
- A/B testing of prompt strategies
- Community contribution of effective templates
- Language-specific adaptations

### 5. Why Artifacts in Prompts?

Artifacts give the LLM **concrete feedback** about what went wrong:
- "Test 2 failed: expected 5, got 4" → LLM knows exactly what to fix
- Without artifacts: LLM only knows "score was low" → random guessing
