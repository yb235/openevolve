# Visualization & Manual Mode

This document explains the visualization server that lets you explore evolution results interactively, and the manual mode that enables human-in-the-loop evolution.

## Visualization Overview

OpenEvolve includes a Flask-based web UI for exploring evolution results. It reads checkpoint data and presents it as interactive graphs and lists.

```
┌──────────────────────────────────────────────┐
│  Browser (localhost:8080)                     │
│                                               │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐     │
│  │Branching │ │Performance│ │   List   │     │
│  │  (tree)  │ │  (chart)  │ │ (table)  │     │
│  └──────────┘ └──────────┘ └──────────┘     │
│                                               │
│  ┌────────────────────────────────────────┐  │
│  │              Graph Area                 │  │
│  │                                         │  │
│  │  ○──○──○                               │  │
│  │  │     └──○──○                         │  │
│  │  └──○       └──●  (selected)           │  │
│  │                                         │  │
│  └────────────────────────────────────────┘  │
│                                               │
│  ┌────────────┐  ┌───────────────────────┐  │
│  │ Metric     │  │ Highlight Filter      │  │
│  │ Select ▼   │  │ Select ▼              │  │
│  └────────────┘  └───────────────────────┘  │
│                                               │
│  ┌────────────────────────────────────────┐  │
│  │              Sidebar                    │  │
│  │  Program: prog_42                       │  │
│  │  Island: 2                              │  │
│  │  Generation: 5                          │  │
│  │  Score: 0.922                           │  │
│  │  Code: [expandable]                     │  │
│  │  Prompts: [expandable]                  │  │
│  └────────────────────────────────────────┘  │
└──────────────────────────────────────────────┘
```

## How to Launch

```bash
# Start the visualization server
python scripts/visualizer.py --path path/to/checkpoint/

# Server starts at http://localhost:8080

# Or export as standalone HTML (no server needed)
python scripts/visualizer.py --path path/to/checkpoint/ --static-output output.html
```

**Using Makefile:**
```bash
make visualizer
```

---

## Views

### 1. Branching View (Evolution Tree)

Shows the parent-child relationships as a tree graph using D3.js:

```
Initial ──○──○──○──○
           │        └──○──○──● (best!)
           └──○──○
                  └──○──○──○
```

- **Nodes** = Programs
- **Edges** = Parent-child relationships
- **Node color** = Metric value (configurable via metric selector)
- **Node size** = Can reflect score or generation

### 2. Performance View

Shows metric values over time/iterations as a chart:

```
Score ▲
 1.0  │                        ●
 0.8  │              ●    ● ●
 0.6  │       ●  ●
 0.4  │  ●
 0.2  │
      └───────────────────────▶ Iteration
```

### 3. List View

A sortable, filterable table of all programs:

| ID | Island | Generation | Score | Complexity | Parent |
|----|--------|-----------|-------|------------|--------|
| prog_42 | 2 | 5 | 0.922 | 7.0 | prog_37 |
| prog_89 | 0 | 8 | 0.910 | 5.5 | prog_78 |
| ... | ... | ... | ... | ... | ... |

---

## Interactive Features

### Node Selection (Sidebar)

The sidebar shows detailed program information and has two interaction modes:

**Hover mode:** Move your mouse over a node → sidebar appears with that program's details. Move away → sidebar hides.

**Sticky mode:** Click a node → sidebar stays visible ("sticky"). The selected node gets a red border. Click the background to deselect.

The selected node is **synchronized across all views** — clicking a node in the tree also highlights it in the list and performance chart.

### Metric Selector

A dropdown that controls which metric is used for:
- Node coloring in the tree view
- Y-axis in the performance chart
- Sorting in the list view

Available metrics are determined dynamically from the dataset.

### Highlight Filter

A dropdown that highlights multiple nodes based on criteria:
- **Top score** — Programs with the highest combined_score
- **First generation** — The initial program and its direct children
- **Failed** — Programs with score 0 or errors
- **Unset** — No highlighting

Highlighted nodes appear with a **blue shadow** in all views.

### Dark Mode

Toggle between light and dark themes.

---

## Data API

The visualization server exposes a REST API:

### `GET /api/data`

Returns the complete evolution dataset:

```json
{
    "nodes": [
        {
            "id": "prog_1",
            "island": 0,
            "generation": 0,
            "parent_id": null,
            "metrics": {
                "combined_score": 0.5,
                "complexity": 3.0,
                "diversity": 0.0
            },
            "code": "import numpy as np\n...",
            "prompts": {
                "system": "You are an expert...",
                "user": "Current program..."
            }
        },
        ...
    ],
    "edges": [
        {"source": "prog_1", "target": "prog_5"},
        {"source": "prog_1", "target": "prog_8"},
        ...
    ],
    "archive": ["prog_42", "prog_89", ...]
}
```

### `GET /program/<id>`

Returns an HTML page with detailed information about a specific program, including full code, all metrics, artifacts, and evolution lineage.

### Data Loading

The visualizer scans checkpoint directories recursively:

```python
def load_checkpoint_data(path):
    """
    Scan checkpoint directory for programs.
    Loads metadata.json and program.json files.
    Reconstructs parent-child relationships.
    Handles duplicate program IDs with "-copyN" suffixes.
    Sanitizes non-JSON-serializable metrics (infinity, NaN).
    """
```

**Rationale:** The visualizer reads from checkpoints (files on disk), not from the live database. This means you can visualize completed runs, share checkpoints with colleagues, and explore results offline.

---

## Static Export

For sharing results without running a server:

```bash
python scripts/visualizer.py --path path/to/checkpoint/ --static-output evolution_results.html
```

This generates a **single, self-contained HTML file** that includes:
- All JavaScript and CSS (inlined)
- The complete dataset (embedded as JSON)
- Interactive D3.js visualization (works offline)

**Rationale:** Static export makes results portable. You can email the HTML file, host it on a static site, or attach it to a paper/report. No server required.

---

## Manual Mode (Human-in-the-Loop)

### Overview

Manual mode replaces the LLM with a human. The system generates a "task" describing what it needs, displays it in a web UI, and waits for the human to provide a response.

```
Normal Mode:
  System → [LLM API] → Code mutation

Manual Mode:
  System → [Web UI] → Human types improvement → System continues
```

### How to Enable

Set a model's configuration to use manual mode:

```yaml
llm:
  models:
    - name: "manual"
      _manual_queue_dir: "manual_tasks_queue"
```

Or configure it through the controller, which automatically sets up the manual queue directory.

### How It Works

**File:** `scripts/manual.py`

```python
@manual_bp.route("/manual/api/tasks")
def list_tasks():
    """Return pending tasks (excluding answered ones)."""
    tasks = []
    for task_file in task_dir.glob("*.json"):
        if not task_file.name.endswith(".answer.json"):
            task = json.loads(task_file.read_text())
            if not (task_dir / f"{task['id']}.answer.json").exists():
                tasks.append(task)
    return jsonify(tasks)

@manual_bp.route("/manual/api/tasks/<task_id>/answer", methods=["POST"])
def answer_task(task_id):
    """Submit human answer for a task."""
    answer = request.json["response"]
    answer_path = task_dir / f"{task_id}.answer.json"
    with open(answer_path, 'w') as f:
        json.dump({"response": answer}, f)
    return jsonify({"status": "ok"})
```

### The Manual Mode Flow

```
1. Iteration starts normally
         │
         ▼
2. LLM call triggers manual mode
   └── Writes task_42.json to manual_tasks_queue/
       {
         "id": "task_42",
         "system_message": "You are an expert...",
         "messages": [{"role": "user", "content": "Current program..."}],
         "model": "manual",
         "created_at": 1709234567.89
       }
         │
         ▼
3. OpenEvolve PAUSES and polls for answer
   └── Checks for task_42.answer.json every 1 second
         │
         ▼
4. Human sees task in web UI (http://localhost:8080/manual)
   └── Reads the prompt, understands what's needed
   └── Types their code improvement
   └── Clicks "Submit"
         │
         ▼
5. Web UI writes task_42.answer.json
   {
     "response": "<<<SEARCH\ndef optimize(x):\n===REPLACE\ndef optimize(x, lr=0.01):\n>>>"
   }
         │
         ▼
6. OpenEvolve reads the answer and continues
   └── Parses the response as if it came from an LLM
   └── Evaluates the modified code
   └── Adds to database if good
```

### Task File Format

**Task (written by OpenEvolve):**
```json
{
    "id": "task_42",
    "system_message": "You are an expert software engineer...",
    "messages": [
        {
            "role": "user",
            "content": "## Current Program\n```python\n..."
        }
    ],
    "model": "manual",
    "created_at": 1709234567.89
}
```

**Answer (written by human via UI):**
```json
{
    "response": "Your code improvement here..."
}
```

### When to Use Manual Mode

| Use Case | Why Manual |
|----------|-----------|
| Debugging stuck evolution | Human can identify why the LLM isn't improving |
| Domain expertise | Some domains need human insight (physics, biology) |
| Teaching the system | Show it patterns it hasn't discovered |
| Evaluation without good metrics | Human judges quality subjectively |
| Hybrid approaches | Mix automated iterations with occasional human guidance |

---

## Visualization Architecture

### Technology Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Server | Flask (Python) | Serve pages, API endpoints |
| Visualization | D3.js | Interactive graphs and charts |
| Styling | CSS (custom) | Layout and theming |
| Data | JSON | Checkpoint → API → Browser |

### File Structure

```
scripts/
├── visualizer.py          # Flask server + data loading
├── manual.py              # Manual mode blueprint
├── templates/
│   ├── index.html         # Main visualizer page
│   ├── program_page.html  # Individual program view
│   └── manual_page.html   # Manual task submission UI
├── static/
│   ├── css/
│   │   └── main.css       # Styling
│   └── js/
│       └── *.js           # D3 visualization modules
└── requirements.txt       # Flask dependencies
```

---

## Key Design Decisions

### 1. Why Flask Instead of a SPA Framework?

Flask is minimal, Python-native, and doesn't require a build step. The visualization is a development/analysis tool, not a production web app. Simplicity > features.

### 2. Why D3.js for Graphs?

D3.js is the industry standard for data visualization in the browser. It handles:
- Force-directed graph layouts (evolution tree)
- SVG rendering (scalable, interactive)
- Event handling (hover, click, selection)
- Transitions and animations

### 3. Why Checkpoint-Based Instead of Live Data?

Reading from checkpoints (not the live database) means:
- Visualization works on completed runs
- No race conditions with the evolution process
- Checkpoints can be shared and visualized on any machine
- No additional server needed during evolution

### 4. Why Static Export?

Static export creates a portable, shareable snapshot of results:
- Email it to colleagues
- Attach it to publications
- Host on GitHub Pages
- No server infrastructure needed

### 5. Why Manual Mode Through Files Instead of WebSockets?

File-based communication is:
- Simple (no WebSocket server needed)
- Debuggable (you can inspect task files directly)
- Resumable (tasks survive server restarts)
- Process-safe (multiple processes can write/read atomically)

The trade-off is latency (1-second polling), which is irrelevant for human-in-the-loop interaction.
