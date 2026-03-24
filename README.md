# Debate — AI-Powered Debate Engine

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

An autonomous debate system built with [CrewAI](https://docs.crewai.com/) that takes any debate motion as input, generates arguments **for** and **against** it, and renders a judge's verdict — all driven by large language models with no human intervention after the motion is entered.

---

## Table of Contents

1. [Overview](#overview)
2. [CrewAI Concepts](#crewai-concepts)
3. [Project Architecture](#project-architecture)
4. [Project Structure](#project-structure)
5. [Agents](#agents)
6. [Tasks](#tasks)
7. [Execution Flow](#execution-flow)
8. [Prerequisites](#prerequisites)
9. [Installation](#installation)
10. [Configuration](#configuration)
11. [Running the Project](#running-the-project)
12. [Outputs](#outputs)
13. [Customisation](#customisation)

---

## Overview

The **Debate** project orchestrates a three-stage autonomous debate:

| Stage | Actor | Result |
|-------|-------|--------|
| 1. Proposition | Debater Agent | A compelling argument **in favour** of the motion |
| 2. Opposition | Debater Agent | A compelling argument **against** the motion |
| 3. Judgement | Judge Agent | A verdict declaring the winning side with reasoning |

Each stage is a discrete CrewAI **Task**; two specialised agents collaborate inside a single **Crew** that runs in **sequential** order — ensuring the judge always has both arguments in full before delivering a verdict.

---

## CrewAI Concepts

Understanding these building blocks is essential to reading the code.

### Crew

A **Crew** is the top-level orchestrator. It holds references to every agent and task that belong to a run, defines the execution strategy (`sequential` or `hierarchical`), and exposes the `kickoff(inputs)` method that launches the entire pipeline. Think of it as the *director* who sits above the actors and ensures every scene is performed in order with the right context.

In code, a class decorated with `@CrewBase` and returning a `Crew` instance from a `@crew`-decorated method is a Crew.

### Agent

An **Agent** is an autonomous LLM-powered worker with a defined `role`, a `goal`, and a `backstory`. These three fields are injected into the system prompt that shapes the model's persona and behaviour for the entire run. An agent can be extended with tools (web search, code execution, etc.) but the persona itself is sufficient for pure reasoning tasks like debating.

| Field | Purpose |
|-------|---------|
| `role` | One-liner job title; anchors the persona |
| `goal` | What the agent is ultimately trying to achieve |
| `backstory` | Rich context that steers tone and decision-making style |
| `llm` | The model backend in `provider/model` format (e.g. `openai/gpt-4o-mini`) |

Agent definitions live in `config/agents.yaml` and are registered via the `@agent` decorator in `crew.py`.

### Task

A **Task** is a discrete unit of work assigned to a specific agent. It carries a `description` (the detailed prompt for this step), an `expected_output` (the acceptance criteria the model should satisfy), and an optional `output_file` path where the raw response is persisted to disk.

| Field | Purpose |
|-------|---------|
| `description` | Full instructions for this step, interpolated with runtime inputs |
| `expected_output` | Shape / content the agent should produce |
| `agent` | Which agent executes this task |
| `output_file` | Where to save the generated Markdown output |

Task definitions live in `config/tasks.yaml` and are registered via the `@task` decorator in `crew.py`.

### Process

A **Process** defines the order in which tasks are executed:

- **`Process.sequential`** *(used here)* — Tasks run one after another in declaration order. The output of every completed task is automatically passed as context to all subsequent tasks.
- **`Process.hierarchical`** — A manager agent dynamically orchestrates sub-agents (not used in this project).

### `@CrewBase` Decorator

The `@CrewBase` class decorator performs automatic wiring: it reads the YAML configuration files, discovers all methods annotated with `@agent`, `@task`, and `@crew`, and populates `self.agents` / `self.tasks` lists automatically — eliminating all manual list-management boilerplate.

### Input Interpolation

String placeholders in the format `{variable}` inside any YAML value are replaced at runtime by the `inputs` dictionary passed to `crew.kickoff(inputs=...)`. In this project, `{motion}` is the only input and it is wired into agent goals, backstories, and task descriptions alike.

---

## Project Architecture

```
User Input (motion)
        │
        ▼
┌───────────────────────────────────────────────────┐
│                    Debate Crew                    │
│                (Process.sequential)               │
│                                                   │
│  ┌─────────────────────────────────────────────┐  │
│  │  Task 1 — propose                           │  │
│  │  Agent  : Debater                           │  │
│  │  Goal   : Argue FOR the motion              │  │
│  │  Output : output/propose.md                 │  │
│  └──────────────────┬──────────────────────────┘  │
│                     │  output passed as context    │
│  ┌──────────────────▼──────────────────────────┐  │
│  │  Task 2 — oppose                            │  │
│  │  Agent  : Debater                           │  │
│  │  Goal   : Argue AGAINST the motion          │  │
│  │  Output : output/oppose.md                  │  │
│  └──────────────────┬──────────────────────────┘  │
│                     │  both outputs passed         │
│  ┌──────────────────▼──────────────────────────┐  │
│  │  Task 3 — decide                            │  │
│  │  Agent  : Judge                             │  │
│  │  Goal   : Pick the more convincing side     │  │
│  │  Output : output/decide.md                  │  │
│  └─────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────┘
        │
        ▼
   Final verdict printed to stdout
```

Key design decisions:
- The **same `debater` agent** is reused for both proposition and opposition — the task description alone redirects its stance, keeping the agent pool minimal.
- **Sequential context chaining** means the judge never operates blind; it always reads complete, model-generated arguments before ruling.
- Each task writes its own **Markdown output file**, giving a clean, human-readable audit trail of the full debate.

---

## Project Structure

```
debate/
├── pyproject.toml              # Project metadata, dependencies, entry-point scripts
├── README.md                   # This file
│
├── output/                     # Auto-generated at runtime — one file per task
│   ├── propose.md              # Proposition argument (Task 1 output)
│   ├── oppose.md               # Opposition argument (Task 2 output)
│   └── decide.md               # Judge's verdict     (Task 3 output)
│
└── src/
    └── debate/
        ├── __init__.py
        ├── crew.py             # @CrewBase class — agents, tasks, and crew assembly
        ├── main.py             # CLI entry point — reads motion, calls crew.kickoff()
        └── config/
            ├── agents.yaml     # Agent personas, goals, backstories, and LLM config
            └── tasks.yaml      # Task descriptions, expected outputs, and file mappings
```

---

## Agents

Defined in [src/debate/config/agents.yaml](src/debate/config/agents.yaml).

### `debater`

| Field | Value |
|-------|-------|
| Role | A compelling debater |
| Goal | Present a clear argument either in favour of or against the motion |
| Backstory | Experienced debater with a knack for concise but convincing arguments |
| LLM | `openai/gpt-4o-mini` |

The `debater` is reused for **both** the proposition and opposition tasks. The task `description` is what steers it towards the correct side each time — demonstrating how a single, well-crafted agent persona can serve multiple opposing roles.

### `judge`

| Field | Value |
|-------|-------|
| Role | Decide the winner of the debate |
| Goal | Determine which side is more convincing based purely on argument merit |
| Backstory | Fair judge known for weighing arguments without personal bias |
| LLM | `openai/gpt-4o-mini` |

The `judge` only acts in the final task and deliberately has no stake in either side, relying on the prior tasks' outputs that are automatically injected as context by the sequential process.

---

## Tasks

Defined in [src/debate/config/tasks.yaml](src/debate/config/tasks.yaml).

### `propose`
- **Agent**: `debater`
- **Description**: Argue *in favour* of the motion — make the case convincingly.
- **Expected output**: A concise, well-structured argument supporting the motion.
- **Output file**: `output/propose.md`

### `oppose`
- **Agent**: `debater`
- **Description**: Argue *against* the motion — make the counter-case convincingly.
- **Expected output**: A concise, well-structured argument opposing the motion.
- **Output file**: `output/oppose.md`

### `decide`
- **Agent**: `judge`
- **Description**: Review both the proposition and opposition arguments; decide which side is more convincing.
- **Expected output**: A verdict with clear, reasoned justification for the winning side.
- **Output file**: `output/decide.md`

> Because the Crew uses `Process.sequential`, the `judge` task automatically receives the full text of both prior task outputs as conversation context before it begins reasoning.

---

## Execution Flow

```
crewai run
    │
    ├── Prompts user: "Enter the debate motion:"
    │
    ├── Instantiates Debate() — YAML configs loaded, {motion} staged for interpolation
    │
    ├── crew.kickoff(inputs={"motion": <user input>})
    │   │
    │   ├── Task 1 — propose
    │   │   ├── {motion} interpolated into agent goal, backstory, and task description
    │   │   ├── debater LLM call: "You are proposing the motion: <motion>…"
    │   │   └── Response written to output/propose.md  ✓
    │   │
    │   ├── Task 2 — oppose   (Task 1 output appended to context)
    │   │   ├── debater LLM call: "You are in opposition to the motion: <motion>…"
    │   │   └── Response written to output/oppose.md  ✓
    │   │
    │   └── Task 3 — decide   (Task 1 + Task 2 outputs appended to context)
    │       ├── judge LLM call: "Review the arguments presented and decide…"
    │       └── Response written to output/decide.md  ✓
    │
    └── result.raw printed to stdout
```

---

## Prerequisites

| Requirement | Version |
|-------------|---------|
| Python | `>=3.13, <3.14` |
| [UV](https://docs.astral.sh/uv/) | latest |
| OpenAI API key | — |

---

## Installation

```bash
# 1. Clone the repository
git clone <repo-url>
cd debate

# 2. Install UV if not already present
pip install uv

# 3. Install project dependencies
crewai install
# or equivalently:
uv sync
```

---

## Configuration

Set your OpenAI API key as an environment variable before running:

```bash
# Windows PowerShell
$env:OPENAI_API_KEY = "sk-..."

# macOS / Linux
export OPENAI_API_KEY="sk-..."
```

Alternatively, create a `.env` file in the project root:

```
OPENAI_API_KEY=sk-...
```

The LLM used by both agents is `openai/gpt-4o-mini`. To switch models, edit the `llm` field in [src/debate/config/agents.yaml](src/debate/config/agents.yaml) — any model supported by LiteLLM (the underlying provider) can be used.

---

## Running the Project

```bash
crewai run
```

You will be prompted to enter a debate motion, for example:

```
Enter the debate motion: A country should aim to be a nuclear weapon state
```

The crew will run all three tasks sequentially, saving each to `output/`, and print the judge's full verdict to the terminal when complete.

---

## Outputs

After a successful run, three Markdown files are written (or overwritten) under `output/`:

| File | Contents |
|------|----------|
| `output/propose.md` | The debater's argument **for** the motion |
| `output/oppose.md` | The debater's argument **against** the motion |
| `output/decide.md` | The judge's verdict and full reasoning |

These files serve as a permanent, human-readable record of the full debate and can be committed to version control to compare runs across different motions or model configurations.

---

## Customisation

| What to change | Where |
|----------------|-------|
| Agent persona, tone, or model | [src/debate/config/agents.yaml](src/debate/config/agents.yaml) |
| Task instructions or expected output format | [src/debate/config/tasks.yaml](src/debate/config/tasks.yaml) |
| Add a new agent (e.g. a fact-checker) | [src/debate/crew.py](src/debate/crew.py) — add `@agent` method + new entry in `agents.yaml` |
| Add a new task (e.g. a rebuttal round) | [src/debate/crew.py](src/debate/crew.py) — add `@task` method + new entry in `tasks.yaml` |
| Switch to hierarchical orchestration | [src/debate/crew.py](src/debate/crew.py) — change `Process.sequential` → `Process.hierarchical` |
| Change output file locations | `output_file` values in [src/debate/config/tasks.yaml](src/debate/config/tasks.yaml) |
| Add multiple runtime inputs | Extend the `inputs` dict in [src/debate/main.py](src/debate/main.py) and reference new `{variables}` in YAML |

---

## License

This project is licensed under the [MIT License](LICENSE).

