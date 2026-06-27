# Customer Care DSPy Agent

An agentic customer support chatbot built with [DSPy](https://dspy.ai) and local Ollama models — demonstrated as a fully runnable Jupyter notebook.

DSPy replaces hand-written prompts with automatically optimized ones. You define the structure of each LLM step and a scoring metric. The optimizer finds the best prompt for you.

---

## What's in this repo

```
customer-care-dspy/
│
├── customer_care_dspy.ipynb     ← the notebook — run this top to bottom
├── setup.ps1                    ← one-click environment setup (Windows)
├── requirements.txt             ← Python dependencies
├── .gitignore
│
├── .vscode/
│   ├── settings.json            ← auto-selects .venv interpreter in VS Code
│   └── extensions.json          ← recommends Python + Jupyter extensions
│
└── optimized/
    ├── .gitkeep
    ├── miprov2_agent.json       ← compiled prompt after MIPROv2 (auto-created)
    └── bootstrap_agent.json     ← compiled prompt after BootstrapFewShot (auto-created)
```

---

## Stack

| Component     | Choice                                                |
| ------------- | ----------------------------------------------------- |
| Framework     | DSPy 2.5+                                             |
| Student model | `qwen2.5:14b` — runs in production                    |
| Teacher model | `qwen2.5:32b` — only used during MIPROv2 optimization |
| LLM provider  | Ollama (fully local, no API key needed)               |
| Optimizers    | MIPROv2 (best quality) · BootstrapFewShot (fast)      |
| Environment   | Python `.venv` inside project folder                  |

---

## Prerequisites

Before running anything, make sure you have these installed:

- [Python 3.10+](https://www.python.org/downloads/)
- [VS Code](https://code.visualstudio.com/)
- [Ollama](https://ollama.com/download) — local LLM runner

---

## Setup

### Step 1 — Clone the repo

```bash
git clone https://github.com/your-username/customer-care-dspy.git
cd customer-care-dspy
```

### Step 2 — Run the setup script

Open PowerShell in the project folder and run:

```powershell
.\setup.ps1
```

if this fails then try

```powershell
PowerShell -ExecutionPolicy Bypass -File .\setup.ps1
```

This script does three things automatically:

1. Creates a `.venv` virtual environment inside the project folder
2. Installs all Python dependencies (`dspy-ai`, `jupyter`, `ipykernel`)
3. Registers the Jupyter kernel as `customer-care-dspy` so VS Code can find it

> **If you get a script execution error**, run this first to allow local scripts:
>
> ```powershell
> Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
> ```
>
> Then re-run `.\setup.ps1`

### Step 3 — Pull Ollama models

Open a terminal and run:

```bash
# Start the Ollama server (if not already running)
ollama serve

# Pull the student model — always required
ollama pull qwen2.5:14b

# Pull the teacher model — only needed for MIPROv2 optimization
ollama pull qwen2.5:32b
```

> If you only want to run the fast BootstrapFewShot optimizer, you only need `qwen2.5:14b`.

### Step 4 — Open in VS Code

```bash
code .
```

VS Code will prompt you to **install recommended extensions** (Python + Jupyter). Click **Install**.

---

## Running the Notebook

### Step 1 — Open the notebook

Open `customer_care_dspy.ipynb` in VS Code.

### Step 2 — Select the kernel

In the top-right corner of the notebook, click the kernel selector and choose:

```
customer-care-dspy
```

If it doesn't appear, run this in PowerShell:

```powershell
.\.venv\Scripts\Activate.ps1
python -m ipykernel install --user --name customer-care-dspy --display-name "customer-care-dspy"
```

Then reload VS Code.

### Step 3 — Configure the optimizer (optional)

In **Section 1** of the notebook, find these two lines and set them before running:

```python
OPTIMIZER  = "bootstrap"   # "bootstrap" (fast) or "miprov2" (best quality)
MIPRO_AUTO = "light"       # "light" (~10 trials) | "medium" (~25) | "heavy" (~50+)
```

Use `bootstrap` for a quick first run. Switch to `miprov2` + `medium` for production-quality results.

### Step 4 — Run all cells

Use **Run All** (`Ctrl+Alt+Enter`) or run each section in order.

The notebook is structured as a live walkthrough:

| Section       | What happens                                        |
| ------------- | --------------------------------------------------- |
| 1. Setup      | Installs DSPy, configures models                    |
| 2. Signatures | Defines the 3 LLM steps                             |
| 3. Agent      | Chains signatures into a pipeline                   |
| 4. Trainset   | Loads 31 labeled examples                           |
| 5. Metrics    | Defines the scoring function                        |
| 6. Baseline   | Runs the agent before optimization — shows accuracy |
| 7. Optimize   | Runs the optimizer — watch it work live             |
| 8. Compare    | Before vs after accuracy + prompt inspection        |
| 9. Inference  | Loads compiled prompt, runs 5 test messages         |
| 10. Try it    | Type your own message and run                       |

---

## DSPy concepts in this notebook

### What you write vs what DSPy generates

| You write                             | DSPy generates                      |
| ------------------------------------- | ----------------------------------- |
| Signature docstring                   | Task instruction line in the prompt |
| `InputField` / `OutputField` + `desc` | Full prompt format block            |
| Metric function                       | The goal the optimizer maximizes    |
| 30 labeled examples                   | Best few-shot demos (auto-selected) |

You never write `"You are a helpful agent..."` by hand.

### The 5 prompt parts — where they live

| Prompt part | In DSPy                                    |
| ----------- | ------------------------------------------ |
| Role        | Inferred by the LLM / generated by MIPROv2 |
| Task        | Your **docstring**                         |
| Context     | `InputField` — passed as input at runtime  |
| Examples    | Auto-injected by the optimizer             |
| Format      | `OutputField` names + types + `desc`       |

### Optimizer comparison

|                       | BootstrapFewShot       | MIPROv2                             |
| --------------------- | ---------------------- | ----------------------------------- |
| What it optimizes     | Few-shot examples only | Instructions + examples jointly     |
| Rewrites docstring?   | No                     | Yes (teacher LLM proposes variants) |
| Teacher model needed? | No                     | Yes                                 |
| Speed                 | Fast (~5 min)          | Slower (~20–40 min on `medium`)     |
| Quality               | Good                   | Best                                |

### Teacher / Student split (MIPROv2 only)

- **Teacher** (`qwen2.5:32b`) runs once during optimization to propose better instruction text
- **Student** (`qwen2.5:14b`) gets optimized with those instructions — this is what deploys
- Pay the compute cost of 32b once; run 14b forever in production

### Pipeline flow

```
user_message
    │
    ├─→ ClassifyIntent       → intent, confidence
    ├─→ ExtractEntities      → order_id, email, issue_summary
    │
    └─→ GenerateResponse     ← intent + entities + policy_context
            │
            └─→ response (sent to customer)
```

---

## Development lifecycle

```
Notebook Section 7          Notebook Section 9
─────────────────────       ──────────────────────────────
Run optimizer (once)   →    Load compiled prompt + call LLM
        ↓
optimized/*.json
(static, saved to disk)
```

The optimizer runs only during development. In production (Section 9), it's just one LLM call — no optimizer, no training loop.

---

## Extending this project

**Add more intents**
Add rows to `RAW_EXAMPLES` in Section 4 of the notebook, update the `desc` in `ClassifyIntent` (Section 2), and re-run Section 7.

**Add RAG**
In Section 9, replace the `POLICY_SNIPPETS` dict lookup in `predict_with_rag` with a vector DB call (ChromaDB, pgvector, etc.).

**Better metric**
Once intent accuracy is consistently above 85%, switch `intent_accuracy` to `combined_metric` in Section 7. The combined metric adds an LLM-judge score for response quality (60% intent + 40% quality).

**Re-optimize**
Re-run Section 7 any time you add examples, change the metric, or upgrade models. Each run saves a fresh JSON to `optimized/`.

---

## Troubleshooting

**`ollama: command not found`**
Ollama is not installed or not on your PATH. Download from [ollama.com](https://ollama.com/download) and restart your terminal.

**Kernel not appearing in VS Code**
Re-run the kernel registration step:

```powershell
.\.venv\Scripts\Activate.ps1
python -m ipykernel install --user --name customer-care-dspy --display-name "customer-care-dspy"
```

**`dspy.LM` connection error**
Ollama server is not running. Start it with:

```bash
ollama serve
```

**MIPROv2 is slow**
Switch to `MIPRO_AUTO = "light"` in Section 1 for ~10 trials instead of ~25. Or use `OPTIMIZER = "bootstrap"` for the fastest run.

**Out of VRAM**
If `qwen2.5:32b` doesn't fit in your 24GB VRAM alongside `qwen2.5:14b`, run optimization in two separate passes — or use `qwen2.5:14b` as both teacher and student by changing `TEACHER_MODEL_ID` in Section 1.
