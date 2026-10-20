# Frozen PHREEQC replay

Run from the repository root with `OPENAI_API_KEY` already exported:

```bash
PY=/home/user/miniconda3/envs/app_test/bin/python bash scripts/run_frozen_replay_gpt52.sh
```

This starts **GPT-5.2 only**, one repeat of paired inline/TOC continuations,
four concurrent pairs. The wrapper validates the saved checkpoints before making
API calls. Repeating the same command resumes completed/partially saved branches.
Results are under `result/custom/frozen_replay_gpt52/`:

- `checkpoints/manifest.json`: eligible items, exclusions and source provenance.
- `experiment.json`: frozen prompt, tool definitions, configuration and hashes.
- `runs/repeat_*/item_*/{full,toc}/`: durable call state and final answer/result.
- `results.jsonl`, `summary.json`: aggregated completed results and usage.
- `console.log`, `errors.json`: progress and errors.

Offline validation (no API calls):

```bash
DRY_RUN=1 PY=/home/user/miniconda3/envs/app_test/bin/python bash scripts/run_frozen_replay_gpt52.sh
```

The default estimated-cost stop is **USD 120**, including usage from previous
launches of the same experiment. In-flight calls may complete beyond this soft
threshold. Prices are GPT-5.2 standard input/output USD 1.75/14 per million tokens;
cached input is conservatively charged at the full input rate in the estimate.
Completion usage includes reasoning tokens; they are not charged twice.
Increase the threshold for a budget-paused experiment with `MAX_COST_USD=150`.
`WORKERS` can change on resume; prompts, model settings and checkpoint files cannot.
If additional replication is needed, use `REPEATS=3` with the same command to
increase the total target to three repeats. Completed branches in repeat 1 are
reused without API calls; only the two additional repeats run. A result-dependent
decision to add repeats should be documented when reporting the experiment.

There are no automatic API retries. Explicit request rejections (e.g. authentication
or rate-limit errors) can be retried by relaunching after the issue is fixed.
A timeout or interruption while a paid request is pending leaves `request_pending`.
The runner refuses to resend that request automatically because billing is uncertain.
Previously persisted responses are processed without another API call.

## Protocol

The source is fixed to the existing July GPT-5.2 inline run
`raw100k_repeat_gpt52`. Selection starts from the 137 verified
numeric items, read from the file passed with `--items-file` (`kind` only). It does
not select by original correctness, output value, truncation error or new results.
Each checkpoint ends at the first logged successful `execute_phreeqc` call, before
its result is delivered. Missing/unrecoverable checkpoints are reported, never
replaced with a more favorable run or later execution.

Preparation replays logged file writes and PHREEQC calls locally, in order, into
new directories. It does not invoke an LLM or copy the source's final workspace.
Thus later outputs, answers and transcripts cannot leak into the checkpoint.
Source logs, binary/database hashes, reconstruction checks and snapshot hashes are
recorded. Previously failed executions remain in the shared visible history.

Both arms receive identical reconstructed history, frozen files, a common neutral
system prompt, identical tools, and `original_max_steps - checkpoint_step` remaining
assistant turns. Only the pending result payload changes. `full` follows the
paper's 100,000-character inline cap; `toc` builds the normal section index at
every report size (up to 50 entries plus an omission marker). Arm order and item
order are shuffled with a fixed seed; each pair runs its arms consecutively.

The continuation can list/read files and submit `final_answer.txt`. It cannot
write simulation inputs or execute PHREEQC. Submission ends the continuation;
plain assistant text without a submitted answer is a non-commitment. Each API
response is capped at 4,096 completion tokens, identically in both arms. The
original GPT-5.2 sampling/reasoning defaults are retained.

This measures delivery effects conditional on a frozen, successfully executed
checkpoint. It does not recover provider hidden reasoning state, measure the
effect on constructing simulations, or reproduce the complete original agent.
The inline-fitting subset should be reported separately from capped outputs.
Summary effects average completed pairs within items and then across items;
interim summaries have incomplete coverage. Repeats are not independent questions.
Paper inference should account for item pairing and shared simulation setups.
