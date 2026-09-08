# Answer-blind solving pipeline

This directory publishes the controller, solver, compile-repair, reviewer, and
verification scripts used by the IChO 2026 answer-blind Lean experiments. The
files are an auditable source snapshot, not a bundle of campaign state.

## What is included

- `controller-tools/build_icho_answer_blind_bundles.py` splits source records
  into a question-only solver bundle and a controller-only grader bundle.
- `scripts/build_answer_blind_solver_seed.py` constructs and validates a clean
  problem-only workspace.
- `scripts/configure_answer_blind_workspace.py` selects the GPT or Kimi harness.
- `scripts/run_answer_blind_archon_campaign.py` runs the native full32
  formalize/review/prove/review lifecycle.
- `scripts/run_answer_blind_archon_isolated_campaign.py` adds Linux user and
  Landlock isolation around per-target workers.
- The remaining scripts implement isolated model access, compile feedback,
  independent review, and final verification.
- `prompts/` preserves the GPT-generation and Kimi-review prompt snapshots plus
  the shared chemistry rule registry.

The scripts import the `archon` package from the upstream `science-mango`
tree. `SOURCE_MANIFEST.json` pins every relevant source and historical run
commit. The large Archon package and sealed Lake dependency snapshot are not
duplicated here.

## Process

1. A controller that may see the official material creates two disjoint
   bundles: `questions_only.jsonl` for the solver and `grader.jsonl` for later
   evaluation.
2. The seed builder copies only the problem statement, permitted images, Lean
   project skeleton, and runtime code. It rejects answer fields, solution
   assets, credentials, prior logs, and generated results.
3. Each target is formalized, compiled, semantically reviewed, proved,
   compiled again, and proof-reviewed. The published runs allowed up to three
   compile-feedback attempts and three review/redraft attempts.
4. The final controller performs a clean Lake build, placeholder/axiom checks,
   and only then compares outputs with the controller-only grader.

The model never receives the grader or solution PDFs. Provider credentials
must live outside both this repository and the solver workspace; the broker
reads them from a controller-owned file and exposes only a loopback endpoint
with a public dummy token to the confined worker.

The runs were **filesystem answer-blind**, not network-disconnected. Online
access was available for allowed literature grounding, while the prompts
forbade searching for official IChO answers. Do not describe these runs as
network-level answer-blind.

## Running the public snapshot

Prerequisites are Linux with Landlock ABI 4 or newer, Lean/Lake, a read-only
Lake package snapshot, and an `archon` installation built from the pinned
source commit. The isolated launcher additionally needs root privileges and
dedicated Unix users.

Start from an already validated question-only seed. Keep every path below
outside this repository:

```bash
export SCIENCE_MANGO=/path/to/science-mango
export PIPELINE=$PWD/answer-blind-solving-pipeline
export SEED=/path/to/problem-only-seed
export LAKE_PACKAGES=/path/to/read-only-lake-packages
export CAMPAIGN=/path/to/new-empty-campaign

PYTHONPATH="$SCIENCE_MANGO/src" \
python "$PIPELINE/scripts/run_answer_blind_archon_campaign.py" \
  --campaign-root "$CAMPAIGN" \
  --seed-workspace "$SEED" \
  --lake-packages "$LAKE_PACKAGES" \
  --archon-bin "$SCIENCE_MANGO/.venv/bin/archon" \
  --variant gpt \
  --max-iterations 3 \
  --review-max-iterations 3 \
  --expected-items 32 \
  --max-parallel 32 \
  --target-lifecycle \
  --reuse-lake-packages \
  --dry-run
```

Change `--variant gpt` to `--variant kimi-k3` and use
`--max-parallel 4` for the public Kimi configuration. Replace `--dry-run`
with `--run` only after validating the seed, package snapshot, runtime, model
credential path, dedicated identities, and an absent/empty campaign root.

The controller-only bundle builder accepts official solution material because
it creates the grader split. Never run it inside the solver account or copy
its grader output into the seed. The released Hugging Face JSONL files also
contain post-run evaluation metadata and therefore are not solver inputs.

## Historical launchers

The final GPT and Kimi campaigns used sealed, host-specific wrappers around
these scripts. Those wrappers contained fixed UIDs, absolute runtime paths,
artifact inventory hashes, and one-run lock state. Publishing them verbatim
would not produce a portable launcher and could encourage unsafe reuse, so
their SHA-256 identities and effective settings are recorded in
`RUN_METADATA.json` instead. No API key, credential file, logs, sessions,
grader, solution asset, prior-result payload, or model transcript is included.

## Verification

```bash
sha256sum -c CHECKSUMS.sha256
python -m compileall -q scripts controller-tools
```

The matching upstream test suite is listed in `SOURCE_MANIFEST.json`. All
published Python files were syntax-checked, and the repository was scanned for
credential formats, private keys, answer assets, caches, logs, and private host
paths before publication.
