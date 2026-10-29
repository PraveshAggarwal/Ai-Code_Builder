# AppBuilder (LLM)

An AI-powered application builder that takes a plain-English project description and autonomously generates a fully working codebase. It uses a multi-agent pipeline — **Planner → Architect → Coder** — orchestrated with [LangGraph](https://github.com/langchain-ai/langgraph) and powered by the **Groq API**.

---

## How It Works

```
User Prompt
    │
    ▼
┌─────────┐     structured Plan      ┌───────────┐     TaskPlan       ┌───────┐
│ Planner │ ───────────────────────► │ Architect │ ─────────────────► │ Coder │ ──► generated_project/
└─────────┘                          └───────────┘                    └───────┘
                                                                          ▲  │
                                                                          └──┘ (loops until all tasks done)
```

### Agents

| Agent | Role |
|---|---|
| **Planner** | Converts the user prompt into a structured `Plan` — app name, description, tech stack, features, and a list of files (max 10). |
| **Architect** | Expands the `Plan` into a `TaskPlan` — an ordered list of `ImplementationTask` objects, each tied to a specific file with detailed instructions. |
| **Coder** | A ReAct agent that iterates over each `ImplementationTask`, reads existing file content for context, and writes the final implementation using file tools. Loops until all tasks are complete. |

---

## LLM Model

This project uses the **Groq API** via the `langchain-groq` integration.

- **Model:** `openai/gpt-oss-120b`
- **Provider:** [Groq](https://console.groq.com/)
- **Why Groq?** Groq's LPU inference engine delivers extremely low-latency responses, making the multi-step agentic loop significantly faster compared to standard GPU-based inference.

The model is initialized in [`agent/graph.py`](agent/graph.py):

```python
from langchain_groq.chat_models import ChatGroq

llm = ChatGroq(model="openai/gpt-oss-120b")
```

Structured outputs (Pydantic models) are used for the Planner and Architect agents to guarantee well-formed responses:

```python
llm.with_structured_output(Plan).invoke(...)
llm.with_structured_output(TaskPlan).invoke(...)
```

---

## Project Structure

```
AppBuilder(LLM)/
├── main.py                  # CLI entry point
├── .env                     # Environment variables (GROQ_API_KEY)
└── agent/
    ├── graph.py             # LangGraph pipeline definition
    ├── prompts.py           # System & user prompts for each agent
    ├── states.py            # Pydantic state models (Plan, TaskPlan, CoderState)
    └── tools.py             # File I/O tools for the Coder agent
```

### Key Files

- **`agent/graph.py`** — Defines the `StateGraph` with three nodes (`planner`, `architect`, `coder`) and a conditional loop edge on the coder node. Compiles to the `agent` runnable.
- **`agent/states.py`** — Pydantic models that act as typed state contracts between agents:
  - `File` — a file path + purpose
  - `Plan` — top-level project plan
  - `ImplementationTask` — a single coding task for one file
  - `TaskPlan` — ordered list of `ImplementationTask`
  - `CoderState` — tracks coder progress (current step index, file content)
- **`agent/prompts.py`** — Prompt factory functions for each agent role.
- **`agent/tools.py`** — LangChain `@tool`-decorated functions sandboxed to `generated_project/`:
  - `write_file` — writes content to a file
  - `read_file` — reads existing file content
  - `list_files` — lists all files in a directory
  - `get_current_directory` — returns the project root path
  - `run_cmd` — runs a shell command (available but not wired into the coder by default)
- **`main.py`** — CLI wrapper that reads a user prompt from stdin and invokes the agent graph.

---

## Prerequisites

- Python 3.10+
- A [Groq API key](https://console.groq.com/keys)

---

## Installation

```bash
git clone <repo-url>
cd "AppBuilder(LLM)"
pip install langchain-groq langgraph langchain-core python-dotenv pydantic
```

---

## Configuration

Create a `.env` file in the project root:

```env
GROQ_API_KEY=<your_groq_api_key>
```

The key is automatically loaded via `python-dotenv` in `agent/graph.py`.

---

## Usage

```bash
python main.py
```

You will be prompted:

```
Enter your project prompt:
```

**Example prompts:**

```
Build a colorful modern todo app in HTML, CSS, and JS
Build a REST API for a blog with Flask and SQLite
Build a CLI calculator in Python with add, subtract, multiply, divide
```

The generated project files are written to `generated_project/` in the current working directory.

### Optional CLI Flag

```bash
python main.py --recursion-limit 200
```

| Flag | Default | Description |
|---|---|---|
| `--recursion-limit` / `-r` | `100` | Max LangGraph recursion depth (increase for large projects with many files) |

---

## Output

All generated files are placed under:

```
generated_project/
└── <files as planned by the Planner and Architect agents>
```

The coder agent is sandboxed — it cannot write files outside `generated_project/` (enforced by `safe_path_for_project()` in `tools.py`).

---

## Dependencies

| Package | Purpose |
|---|---|
| `langchain-groq` | Groq LLM integration for LangChain |
| `langgraph` | Multi-agent graph orchestration |
| `langchain-core` | Core LangChain abstractions and tools |
| `pydantic` | Typed state models and structured outputs |
| `python-dotenv` | `.env` file loading |

---

## Limitations

- The model (`openai/gpt-oss-120b`) must be available on your Groq account.
- Very large projects (many files) may require increasing `--recursion-limit`.
- The `run_cmd` tool is defined but not included in the coder's tool list by default — add it to `coder_tools` in `graph.py` if you need the agent to execute shell commands.
- No retry logic on individual task failures; a failed LLM call will raise and exit.
