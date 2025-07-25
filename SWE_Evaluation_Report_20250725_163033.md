# SWE Evaluation Flow Report (2025-07-25 16:30 UTC)

This report describes the data flow and control flow when running a SWE evaluation using the `swebench_tools_and_reasoning` flow. The flow applies structured tool calling with reasoning-focused models. The default model for this flow is `claude-sonnet-4-20250514` as noted in the README.

## Overview of `swebench_tools_and_reasoning`

The README lists the available flows and their default models. The entry for `swebench_tools_and_reasoning` shows that it uses the function-calling format and the `claude-sonnet-4-20250514` model:

```
| Flow ID | Format | Best Suited For | Default Model |
|---------|--------|-----------------|---------------|
| swebench_tools | Function calling | Models with native function calling support | gpt-4o-mini-2024-07-18 |
| swebench_tools_and_reasoning | Function calling | Reasoning models with native function calling support | claude-sonnet-4-20250514 |
| swebench_react | ReACT | Open source models without native function calling support | openrouter/mistralai/devstral-small |
| swebench_react_reasoning | ReACT | Reasoning models without function calling support | openrouter/deepseek/deepseek-r1-0528 |
```

(README lines 132‑140)

## Flow Configuration

The `.moatless/flows/swebench_tools_and_reasoning.json` file defines how the evaluation agent operates. Key settings include:

```
"id": "swebench_tools_and_reasoning",
"max_iterations": 100,
"flow_class": "moatless.flow.search_tree.SearchTree",
"agent": {
  "completion_model": {
    "model": "claude-sonnet-4-20250514",
    "temperature": 1.0,
    "max_tokens": 16000,
    "headers": {"anthropic-beta": "interleaved-thinking-2025-05-14"}
  },
  ...
  "actions": [
    "moatless.actions.append_string.AppendString",
    "moatless.actions.create_file.CreateFile",
    "moatless.actions.find_class.FindClass",
    "moatless.actions.find_code_snippet.FindCodeSnippet",
    "moatless.actions.find_function.FindFunction",
    "moatless.actions.glob.GlobTool",
    "moatless.actions.list_files.ListFiles",
    "moatless.actions.read_file.ReadFile",
    "moatless.actions.reject.Reject",
    "moatless.actions.run_tests.RunTests",
    "moatless.actions.semantic_search.SemanticSearch",
    "moatless.actions.string_replace.StringReplace",
    "moatless.actions.verified_finish.VerifiedFinish",
    "moatless.actions.bash.BashTool"
  ],
  "memory": {
    "include_file_context": true,
    "include_git_patch": true,
    "memory_class": "moatless.message_history.message_history.MessageHistoryGenerator"
  }
}
```

(flow JSON lines 1‑52 and 160‑171)

These settings show that the flow uses the `SearchTree` controller with a set of file manipulation and testing actions. The memory configuration keeps context from the repository and previous patches.

## Evaluation Script

The `run_evaluation.py` script illustrates the data path for each instance. It reads the SWE-bench instance from `/data/instance.json`, sets up a `SweBenchLocalEnvironment`, and invokes `swebench_evaluate`:

```
instance_path = Path("/data/instance.json")
...   
swebench_instance = json.loads(instance_path.read_text())
...
runtime = SweBenchLocalEnvironment(
    repo_path=Path(repo_path),
    swebench_instance=swebench_instance,
    storage=storage,
)
...
result = await runtime.swebench_evaluate(project_id, trajectory_id, patch)
```

(run_evaluation.py lines 20‑48)

`SweBenchLocalEnvironment` manages installation, patch application, and test execution within the repository. It constructs a shell script to apply the test patch and run the repository’s tests:

```
class SweBenchLocalEnvironment(RuntimeEnvironment, BaseEnvironment):
    ...
    self.test_spec = make_test_spec(self.swebench_instance)
    ...
    eval_script_list = self.make_eval_script_list_py()
    ...
```

(local.py lines 22‑60 and 240‑276)

## Data Flow

1. **Setup** – Environment variables and the selected dataset split determine which SWE-bench instances are evaluated. The CLI from `scripts/run_evaluation.py` sets the model, flow, and dataset split.
2. **Flow Initialization** – `setup_flow` loads the flow JSON and constructs an `AgenticFlow` (`SearchTree` controller). Workspace initialization (`setup_workspace`) attaches the repository and index.
3. **Instance Processing** – For each SWE-bench instance, the runtime loads `/data/instance.json`, sets up a local Git repository, and applies the provided patch when running the evaluation.
4. **Agent Execution** – The `ActionAgent` uses the list of tool-call actions to search for files, read them, modify code, run tests, and eventually call `VerifiedFinish` when satisfied.
5. **Result Storage** – Execution logs and evaluation reports are written to storage via `SweBenchLocalEnvironment._save_execution_log` and `swebench_evaluate`. The runner collects results, producing a resolution report for each instance.

## Control Flow

1. `scripts/run_evaluation.py` parses CLI arguments, setting the chosen flow (`swebench_tools_and_reasoning`), model, and dataset split.
2. A scheduler launches Docker jobs to run each instance in parallel.
3. Inside each job, `run_evaluation_async` sets up a `SweBenchLocalEnvironment` and invokes the flow’s agent using the `SearchTree` logic.
4. The agent iterates through actions up to `max_iterations` (100) or until it calls `VerifiedFinish`.
5. After each node expansion or flow event, trajectory data is persisted for later analysis.

## Conclusion

Running the `swebench_tools_and_reasoning` flow executes a structured search loop powered by the `claude-sonnet-4-20250514` function-calling model. The agent leverages file system tools and test execution to attempt to resolve each SWE-bench issue. Evaluation results are saved for review, providing a complete view of which patches successfully solved the benchmark instances.

