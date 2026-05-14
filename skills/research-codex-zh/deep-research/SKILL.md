---
name: deep-research
description: Run a Codex deep research workflow for open-ended research tasks. Split a research goal into parallel subgoals, use bounded codex exec child processes, keep logs and status under .research, then produce a polished report file with sources and a concise findings summary.
---

# Deep Research - Codex Parallel Workflow

## Trigger
`/deep-research <topic or question>`

Use this skill when the user asks for deep research, wide research, multi-agent
research, systematic web research, competitor research, technical research,
academic survey, market research, or a detailed sourced report.

If the current directory already contains a Weizhena `*/outline.yaml`, prefer
the structured `/research-deep` workflow for item-by-item JSON research. If no
outline exists, use this open-ended workflow.

## Workflow

### Step 1: Scope and Success Criteria
- Restate the research question and intended output format.
- Identify the audience, time range, source quality requirements, and known
  exclusions.
- If the task is ambiguous, ask at most three short clarification questions.
  Otherwise proceed with conservative assumptions.

### Step 2: Grounding Pass
- Get the current date.
- Perform a small initial search or source inspection before decomposition.
- Record 3-5 representative sources or datasets and what each contributes.
- Do not split work using only model knowledge when current or source-backed
  information matters.

### Step 3: Parallel Research Plan
- Create `.research/<name>/` where `<name>` is a semantic run id such as
  `<YYYYMMDD>-<topic-slug>`.
- Create these subdirectories:
  - `prompts/`
  - `child_outputs/`
  - `logs/`
  - `status/`
  - `raw/`
  - `cache/`
  - `tmp/`
- Split the goal into independent subgoals by source family, product/company,
  region, time period, claim cluster, or technical dimension.
- Keep subgoals non-overlapping enough that child outputs can be merged without
  heavy duplication.
- Default parallelism is 4-8 child processes, adjusted for task size and machine
  capacity.

### Step 4: Child Prompt Contract
For each subgoal, write a prompt file under `.research/<name>/prompts/`.
Each child prompt must require:
- Markdown output written to `.research/<name>/child_outputs/<id>.md`.
- Key findings first.
- Evidence with direct source links near the claims they support.
- Dates, versions, and uncertainty markers when relevant.
- A "Gaps" section when evidence is incomplete.
- No waiting for user input.
- No broad repo or filesystem edits outside `.research/<name>/`.

### Step 5: Bounded Codex Exec Dispatch
Generate `.research/<name>/run_children.sh` and run a one-child smoke test before
the full run.

Keep the default Codex model and reasoning settings. Do not pass `--model` or
override reasoning configuration unless the user explicitly asks.

Recommended command shape:
```bash
timeout 600 codex exec --full-auto --sandbox workspace-write \
  --output-last-message "$status_dir/$id.last.md" \
  - <"$prompt_file" >"$log_file" 2>&1
```

Use shell network access only when a child must run shell networking commands:
```bash
timeout 600 codex exec --full-auto --sandbox workspace-write \
  -c sandbox_workspace_write.network_access=true \
  --output-last-message "$status_dir/$id.last.md" \
  - <"$prompt_file" >"$log_file" 2>&1
```

Dispatcher requirements:
- Verify `codex exec --help` before relying on CLI flags.
- Do not use unsupported flags such as `--prompt-file`, `--mcp`, `--name`,
  `--output`, or `--log-level`.
- Skip child outputs that already exist and are non-empty.
- Run children in parallel with a bounded queue.
- Capture exit code, start time, end time, and elapsed seconds in `status/`.
- Continue other children when one fails.
- Retry transient failures once.

### Step 6: Aggregate and Polish
- Programmatically concatenate child outputs into
  `.research/<name>/aggregated_raw.md`.
- Read the aggregate and design `.research/<name>/polish_outline.md`.
- Write the final report by section into `.research/<name>/final_report.md`.
- Do not publish a raw concatenation as the final report.
- Keep citations close to the claims they support.
- Separate verified findings from plausible but unverified inferences.

### Step 7: Final Response
Return only:
- Final report path.
- 3-7 key findings or recommendations.
- Important gaps, failed child tasks, or verification limits.

Do not paste the full report into chat unless the user explicitly asks.
