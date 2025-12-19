---
name: data-analysis
description: Data analysis skill equipped with a mandatory prompt evaluation step. It enriches vague requests with schema discovery and context before execution, utilizing a specialized Python wrapper script.
---

# Data Analysis Skill

## Purpose
To prevent execution errors and hallucinations by enforcing a "Evaluate → Clarify → Execute" loop. This skill mandates the use of a pre-processor script that wraps user requests in an evaluation protocol, forcing the Agent to assess prompt clarity before writing analysis code.

## Phase 1: Prompt Evaluation & Wrapper Execution

**Step 1: Input Normalization**
All user interactions related to data analysis must first be processed through the `prompt-improver/improve-prompt.py` wrapper. This script handles bypass logic (shortcuts) and injects evaluation instructions.

**Invocation Command:**
The script expects a JSON object via `stdin`. Construct the command as follows:

```bash
echo '{"prompt": "USER_RAW_PROMPT_HERE"}' | python3 prompt-improver/improve-prompt.py


```

**Step 2: Interpreting Script Output**
The script outputs a JSON object containing a `wrapped_prompt`. The Agent must read this output and strictly follow the "PROMPT EVALUATION" instructions contained within.

### Bypass Logic (Handled by Script)

The script automatically detects and allows immediate execution for:

* **`*` prefix**: Explicit bypass (User says "I know what I'm doing").
* **`/` prefix**: Slash commands.
* **`#` prefix**: Memorization features.

### Evaluation Logic (The "Brain" of Phase 1)

When the script wraps the prompt, it asks the Agent: *"Is this prompt clear enough to execute?"*

The Agent must applies the following criteria to the **Data Context**:

1. **VAGUE Contexts (Trigger Phase 2):**

* "Analyze the data" (Which file? Which metric?)
* "Show me the trend" (Time range? Aggregation?)
* "Fix the error" (Which log file?)
* *Action:* Proceed to **Phase 2: Discovery & Clarification**.

2. **CLEAR Contexts (Skip to Execution):**

* "Plot daily active users from `users.csv` for Jan 2024."
* "Calculate the mean of column 'price' in `data.parquet`."
* *Action:* Proceed directly to **Execution Phase** (Part 2 of this skill).

## Phase 2: Discovery & Clarification (Invoked if Vague)

If the Evaluation in Phase 1 determines the prompt is **VAGUE**, the Agent effectively "invokes" this sub-routine to gather context.

**Workflow:**

1. **Acknowledge & Notify:**

* Inform the user: "Hey! The Prompt Improver flagged your request as a bit vague regarding [dataset/metric/scope]. Let me check the available data first."

2. **Systematic Research (Data Discovery):**

* **Scan Directory:** Use `ls -lh` or `find` to locate potential data files (`.csv`, `.json`, `.parquet`, `.xlsx`, `.log`).
* **Inspect Schema:** Use `head -n 5 <filename>` or a Python snippet to read column names. **Do not assume column names.**
* **Check History:** Review previous conversation turns for loaded dataframes.

3. **Generate Clarifying Questions:**

* Based *only* on the file headers found in step 2, formulate 1-3 specific questions.
* **Invoke Tool:** You MUST use the `AskUserQuestion` tool to present these questions to the user. Do not use free text.
* **Do not ask open-ended questions.** Offer options found in the data.

*Example:*

> **Files Found:** `sales_2023.csv` (cols: date, amount), `sales_2024.csv` (cols: date, amount)
> **Question:** "Which year's data should I analyze: 2023 or 2024?"

4. **Wait for User Input:**

* Once the user answers, combine their answer with the original prompt and proceed to Execution.

## Examples of Behavior

### Scenario A: The Vague Request

**User:** "Analyze the logs."

**1. Execution:**

```bash
echo '{"prompt": "Analyze the logs"}' | python3 prompt-improver/improve-prompt.py


```

**2. Agent Internal Thought (triggered by script output):**
"The wrapper asks if this is clear. It is NOT. 'Logs' is ambiguous. I need to find what logs exist."

**3. Agent Action (Phase 2):**

* Runs `ls *.log` -> Finds `access.log`, `error.log`.
* Runs `head -n 1 access.log` -> Sees specific columns.
* **Reply to User:** "The prompt improver flagged this. I found `access.log` and `error.log`. Which one would you like to analyze, and are you looking for error rates or traffic volume?"

### Scenario B: The Bypass (Expert Mode)

**User:** "*Count rows in data.csv"

**1. Execution:**

```bash
echo '{"prompt": "*Count rows in data.csv"}' | python3 prompt-improver/improve-prompt.py


```

**2. Script Logic:**
Detects `*`. Strips it. Returns `"Count rows in data.csv"` as a raw command without the evaluation wrapper.

**3. Agent Action:**
Proceeds immediately to write Python code to count rows.

## Progressive Disclosure

This SKILL.md contains the core workflow and essentials. For deeper guidance:

* Research strategies: prompt-improver/research-strategies.md
* Question patterns: prompt-improver/question-patterns.md
* Comprehensive examples: prompt-improver/examples.md
* Load these references only when detailed guidance is needed on specific aspects of prompt improvement.
