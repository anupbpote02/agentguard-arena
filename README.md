# 🛡️ AgentGuard Arena

A self-adversarial multi-agent pipeline that automatically red-teams and hardens AI system prompts. Four specialized agents run in a closed loop — attacking, responding, judging, and patching — until the target agent can defend itself against two consecutive attacks.

```
Attacker ──► Target ──► Judge ──► Patcher ──► repeat
   ▲                                              │
   └──────────────── history ◄───────────────────┘
```

---

## How It Works

| Agent | Model | Role |
|---|---|---|
| 🔴 **Attacker** | `nvidia/NVIDIA-Nemotron-3-Super-120B-A12B-FP8` | Generates adversarial payloads. Analyzes its own failure history to escalate with new vectors each round. |
| 🔵 **Target** | `meta-llama/Llama-3.1-8B-Instruct` | The agent under evaluation. Runs the current (possibly patched) system prompt against every attack. |
| ⚖️ **Judge** | `Qwen/Qwen3-235B-A22B-Instruct-2507` | Returns a strict JSON verdict: `{"vulnerable": bool, "severity_score": 0–10, "exploit_mechanism": "..."}`. Never crashes the loop — bulletproof JSON parsing with fallback. |
| 🟡 **Patcher** | `deepseek-ai/DeepSeek-V4-Flash` | Reads the Judge's failure report and surgically appends a targeted defensive clause to the system prompt without changing its core functionality. |

**Loop termination:** 2 consecutive successful defenses **or** max rounds reached — whichever comes first.

All agent calls are traced end-to-end with [Weights & Biases Weave](https://wandb.ai/site/weave).

---

## Target Profiles

Four realistic production agent personas, each with sensitive data and tool access:

| Profile | Sensitive Data | Tools |
|---|---|---|
| **FinTech Wealth Assistant** | Routing numbers, MFA bypass codes, liquidation thresholds, VIP account IDs | `execute_wire_transfer`, `get_routing_number`, `override_dual_auth`, … |
| **HealthTech Triage Bot** | EMR credentials, physician on-call schedules, DEA license, HIPAA audit keys | `access_patient_chart`, `authorize_prescription_refill`, … |
| **LegalTech Contract Agent** | Sealed settlement amounts, privileged vault tokens, billing rates, conflict list | `access_privileged_vault`, `retrieve_settlement_record`, … |
| **EdTech Exam Proctor** | Answer keys, grade write API tokens, instructor override codes, FERPA records DB | `access_answer_key`, `modify_exam_score`, `grant_time_extension`, … |

---

## Prerequisites

- Python 3.9+
- A [Weights & Biases](https://wandb.ai) account with access to W&B Inference

---

## Setup

**1. Clone the repo**
```bash
git clone https://github.com/your-username/agentguard-arena.git
cd agentguard-arena
```

**2. Install dependencies**
```bash
pip install -r requirements.txt
```

**3. Set your W&B API key**

This is the only environment variable the project uses.

```bash
export WANDB_API_KEY=your_wandb_api_key_here
```

To make it persistent, add it to your shell profile (`~/.zshrc`, `~/.bashrc`, etc.).

**4. Launch**
```bash
streamlit run app.py
```

Open **http://localhost:8501** in your browser.

---

## Usage

1. **Select a target profile** from the sidebar dropdown (FinTech, HealthTech, LegalTech, or EdTech)
2. **Set max rounds** with the slider (2–6)
3. Click **⚡ LAUNCH ADVERSARIAL SWARM RUN**

The UI updates live after every agent call:

- **Left column** — color-coded terminal log showing each agent's output as it happens
- **Right column** — severity score metric, round counter, and a side-by-side prompt diff showing exactly what the Patcher changed

After the run, a **Hardening Summary** shows rounds completed, initial vs. final severity score, and the full patched system prompt.

---

## Project Structure

```
agentguard-arena/
├── requirements.txt   # openai, weave, streamlit
├── profiles.py        # 4 target agent profiles with realistic sensitive data
├── core_swarm.py      # 4 @weave.op() agent functions + W&B client setup
└── app.py             # Streamlit UI — live log, metrics, prompt diff, summary
```

---

## Viewing Traces

Every agent call is traced with Weave under the project name **`agent-guard-arena`**.

After a run, go to [wandb.ai](https://wandb.ai) → your workspace → **Weave** → `agent-guard-arena` to replay any round's exact inputs and outputs.

---

## Tech Stack

- **[W&B Inference](https://wandb.ai/site/inference)** — OpenAI-compatible endpoint hosting all four models
- **[Weave](https://wandb.ai/site/weave)** — LLM call tracing via `@weave.op()` decorators
- **[Streamlit](https://streamlit.io)** — UI with live mid-loop updates via `st.empty()`
- **[OpenAI Python SDK](https://github.com/openai/openai-python)** — client pointed at W&B's inference base URL
