---
name: research-deep
description: Read research outline, launch independent agent for each item for deep research. Disable task output.
---

# Research Deep - Deep Research

## Trigger
`/research-deep`

## Workflow

### Step 1: Auto-locate Outline
Find `*/outline.yaml` file in current working directory, read items list, execution config (including items_per_agent).

### Step 2: Resume Check
- Check completed JSON files in output_dir
- Skip completed items

### Step 3: Batch Execution
- Batch by batch_size from outline.yaml.
- Each child process handles items_per_agent items.
- Use `codex exec` child processes for true parallel execution when available.
- Keep the default Codex model and reasoning configuration. Do not pass `--model`
  or override model settings unless the user explicitly asks.
- Use the default `workspace-write` sandbox for child processes. Add
  `-c sandbox_workspace_write.network_access=true` only when the child process
  must run shell commands that access the network.
- Do not use deprecated or unsupported `codex exec` flags such as `--prompt-file`,
  `--mcp`, `--name`, `--output`, or `--log-level`. Run `codex exec --help`
  before generating the dispatcher if the local CLI version is unknown.

**Parallel Artifacts**:
- Create `{topic}/.research/` with these subdirectories:
  - `prompts/`: one prompt file per child process
  - `logs/`: stdout/stderr log for each child process
  - `status/`: one status file per child process with exit code and timestamps
  - `tmp/`: temporary dispatcher files
- Generate `{topic}/.research/run_children.sh`.
- The dispatcher must:
  1. Skip any output JSON that already exists and passes validation.
  2. Launch up to `batch_size` child processes in parallel.
  3. Use external timeouts: 5 minutes for small/default items, up to 15 minutes
     for large items.
  4. Write each child output JSON to the exact `{output_path}`.
  5. Capture logs under `{topic}/.research/logs/<item_slug>.log`.
  6. Capture exit code and elapsed time under `{topic}/.research/status/<item_slug>.status`.
  7. Continue running other children when one child fails.
  8. Produce a final summary of succeeded, skipped, failed, and timed-out items.

**Recommended `codex exec` shape**:
```bash
timeout 600 codex exec --full-auto --sandbox workspace-write \
  --output-last-message "$status_dir/$slug.last.md" \
  - <"$prompt_file" >"$log_file" 2>&1
```

If shell network access is required:
```bash
timeout 600 codex exec --full-auto --sandbox workspace-write \
  -c sandbox_workspace_write.network_access=true \
  --output-last-message "$status_dir/$slug.last.md" \
  - <"$prompt_file" >"$log_file" 2>&1
```

**Small-scale Verification Before Full Parallel Run**:
- Before launching all items, generate the first one or two prompt files and run
  them with parallelism 1.
- Inspect the generated prompt, output JSON, validation result, and log path.
- Only start the full dispatcher after the sample run produces valid JSON.

**Failure Handling**:
- If a child fails, do not discard its log. Mark the item as failed in
  `{topic}/.research/status/`.
- Retry failed items once when the error is transient, such as network timeout or
  temporary rate limiting.
- If retry still fails, keep the partial findings in the log and report the item
  in the final summary.

**Parameter Retrieval**:
- `{topic}`: topic field from outline.yaml
- `{item_name}`: item's name field
- `{item_related_info}`: item's complete yaml content (name + category + description etc.)
- `{output_dir}`: execution.output_dir from outline.yaml (default: ./results)
- `{fields_path}`: absolute path to {topic}/fields.yaml
- `{output_path}`: absolute path to {output_dir}/{item_name_slug}.json (slugify item_name: replace spaces with _, remove special chars)

**Hard Constraint**: The following prompt must be strictly reproduced, only replacing variables in {xxx}, do not modify structure or wording.

**Prompt Template**:
```python
prompt = f"""## Task
Research {item_related_info}, output structured JSON to {output_path}

## Field Definitions
Read {fields_path} to get all field definitions

## Output Requirements
1. Output JSON according to fields defined in fields.yaml
2. Mark uncertain field values with [uncertain]
3. Add uncertain array at the end of JSON, listing all uncertain field names
4. All field values must be in English

## Output Path
{output_path}

## Validation
After completing JSON output, run validation script to ensure complete field coverage:
python ~/.codex/skills/research/validate_json.py -f {fields_path} -j {output_path}
Task is complete only after validation passes.
"""
```

**One-shot Example** (assuming researching GitHub Copilot):
```
## Task
Research name: GitHub Copilot
category: International Product
description: Developed by Microsoft/GitHub, first mainstream AI coding assistant, ~40% market share, output structured JSON to {project_dir}/results/GitHub_Copilot.json

## Field Definitions
Read {project_dir}/fields.yaml to get all field definitions

## Output Requirements
1. Output JSON according to fields defined in fields.yaml
2. Mark uncertain field values with [uncertain]
3. Add uncertain array at the end of JSON, listing all uncertain field names
4. All field values must be in English

## Output Path
{project_dir}/results/GitHub_Copilot.json

## Validation
After completing JSON output, run validation script to ensure complete field coverage:
python ~/.codex/skills/research/validate_json.py -f {project_dir}/fields.yaml -j {project_dir}/results/GitHub_Copilot.json
Task is complete only after validation passes.
```

### Step 4: Wait and Monitor
- Wait for current batch to complete
- Launch next batch
- Display progress

### Step 5: Summary Report
After all complete, output:
- Completion count
- Failed/uncertain marked items
- Output directory

## Agent Config
- Background execution: Yes, through bounded `codex exec` child processes
- Task Output: Disabled where supported; child has explicit output file when complete
- Resume support: Yes
- Logs: Required under `{topic}/.research/logs/`
- Failure isolation: Required; one failed child must not stop the whole batch
