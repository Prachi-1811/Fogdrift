# FogDrift

Code and results for the paper

> **FogDrift: Context-Conditional Tool-Use Drift Monitoring for Distributed LLM-Agent Fleets**
> Prachi Chaturvedi, Shahnawaz Ahmad, Ehsan Nowroozi, George Loukas, Lucas C. Cordeiro, Biju Issac.

FogDrift monitors the tool calls of an LLM-agent fleet arranged as edge → fog → cloud and
scores the conditional tool-use distribution P(t | c) rather than marginal tool frequencies.
Each edge keeps per-context tool histograms and reports only when they change or a heartbeat
is due. Fog nodes score per-agent and aggregated conditional KL divergence. The cloud
applies a corroboration rule before alerting the operator.
Thresholds at every tier are calibrated to a common in-control average run length (ARL₀).
All detectors are compared at the same operator-level false-alarm rate.

## Repository layout

```
fogdrift_core.py            simulator, FogDrift (HCDD) and all baselines
fogdrift_ext.py             shared helpers (byte-capped uplink, autocorrelated traffic,
                            recalibration on new traffic)
check_setup.py              quick sanity check of the installation
calibrate.py                edge parameters, matched-ARL thresholds, held-out validation
exp_*.py                    one script per experiment (see the table below)
agentdojo_collect.py        collects live AgentDojo traces with an LLM agent
agentdojo_runs.py           converts AgentDojo's published benchmark run logs to the same trace format
injecagent_asr.py           attack-success rates for InjecAgent live runs
injecagent_traces.py        InjecAgent ground-truth cases as episode traces for the fleet replay
edge_params.json            calibrated edge parameters        (written by calibrate.py)
calibration.json            matched-ARL thresholds            (written by calibrate.py)
validation.csv              held-out ARL validation, Table XII (written by calibrate.py)
data/                       InjecAgent toolkit map; AgentDojo traces (published runs of six models included)
results/                    tables and JSON produced by the experiment scripts
figures/                    plot_paper_figures.py and the generated figures
```

All detectors share one data generator, one detection endpoint and the same byte
accounting, because every experiment imports `fogdrift_core.py`.

## Setup

Python 3.10 or newer.

```bash
pip install -r requirements.txt
python check_setup.py
```

`check_setup.py` should finish in a few seconds and print `CHECK ...` followed by three numbers.

## Running the experiments

Run `calibrate.py` first. All later scripts read `edge_params.json` and `calibration.json`.
The calibrated files used in the paper are included, so this step can be skipped.

```bash
bash run_all.sh            # every simulation experiment, then the figures (~10 h)
bash run_all.sh quick      # short pipeline test of every script (~10 min)
```

Each script can also be run on its own, e.g. `python exp_dropout.py`. Every simulation
script accepts `quick` as an argument (for example `python exp_dropout.py quick`). A quick
run uses fewer seeds, so its numbers are noisy and are not the paper's values.
Quick runs also write to `results/`; `git checkout results/` restores the full-run files.
`run_all.sh` skips the InjecAgent replay when `InjecAgent/` has not been cloned, and skips
the AgentDojo replay when no trace file is in `data/`.

Each script writes its table as text to `results/`, plus JSON where a figure is built from it.

| Paper item | Script | Output in `results/` | Approx. time |
|---|---|---|---|
| Table IV, Fig. 2, Fig. 3 — matched-ARL detection delay and bytes per round | `exp_matched_arl.py` | `matched_arl.txt/.json` | 5 min |
| Table V — partial compromise and localisation | `exp_localisation.py` | `localisation.txt` | 6 min |
| Table VI — agent dropout and burst loss (Sec. V-F) | `exp_dropout.py` | `dropout.txt/.json` | 20 min |
| Table VII(a), Fig. 6 — per-fog uplink byte cap | `exp_bandwidth_cap.py` | `bandwidth_cap.txt/.json` | 20 min |
| Table VII(b), VII(c) — fog outage, heterogeneous workloads, latency | `exp_deployment_stress.py` | `deployment_stress.txt` | 10 min |
| Table VIII — adaptive attacker; magnitude sweep (Sec. V-H); reference poisoning (Sec. V-K) | `exp_adaptive.py` | `adaptive.txt` | 8 min |
| Table IX — tier ablations (row V3 with Eq. (6): `exp_cloud_corroboration.py`) | `exp_ablation.py` | `ablation.txt`, `ablation_thresholds.json` | 20 min |
| Table X(a) — InjecAgent live LLM runs | `injecagent_asr.py` | printed | — |
| Table X(b), Fig. 7 — InjecAgent ground-truth replay, context pinned to the user task (`tool-context`: context taken from the invoked toolkit, Sec. V-L) | `exp_injecagent_replay.py` | `injecagent_replay.txt`, `injecagent_replay_toolcontext.txt` | 5 min |
| Table X(c) — InjecAgent replay with tools identified by function, function + argument bit, or toolkit bigram | `exp_injecagent_identity.py` | `injecagent_identity.txt/.json` (`tool-context`: `injecagent_identity_toolcontext.txt/.json`) | 25 min |
| Table XI — AgentDojo traces replayed through a 50-agent fleet: (a) our GPT-OSS-20B run, (b) AgentDojo's published runs of six models | `exp_agentdojo_replay.py` | `agentdojo_<model>.txt/.json` | 10 min |
| Sec. V-L — AgentDojo published runs: thresholds calibrated on half of each suite's benign user tasks, held-out ARL and delays on the other half (both directions) | `exp_agentdojo_heldout_tasks.py` | `agentdojo_heldout_tasks.txt/.json` | 40 min |
| Sec. V-L — AgentDojo published runs: response policies on a fog alert (global halt, random quarantine, fog halt, top-k quarantine), successful attacks in 100 rounds after onset | `exp_agentdojo_quarantine.py` | `agentdojo_quarantine_<model>.txt/.json` | 40 min |
| Table XII — held-out ARL validation (cloud row with Eq. (6): `exp_cloud_corroboration.py`) | `calibrate.py` | `validation.csv` (repo root) | 35 min |
| Table XII(b), Sec. V-M — cloud tier: calibration of τ_g, faulty fogs, one-fog outage, fleet-wide detection, null coincidence rate of the corroboration branch | `exp_cloud_corroboration.py` | `cloud_corroboration.txt/.json` | 15 min |
| Table XIII — autocorrelated benign traffic, recalibration | `exp_autocorrelation.py` | `autocorrelation.txt/.json` | 20 min |
| Table XIV — autocorrelation without knowing ρ: edge deduplication, ESS scaling, Markov-bootstrap calibration; benign ρ bursts; calibration variance over seed blocks (Sec. V-N) | `exp_autocorr_mitigation.py` | `autocorr_mitigation.txt/.json` | 60 min |
| Fig. 5 — workload-only shift (Sec. V-D) | `exp_workload_invariance.py` | `workload_invariance.txt` | 3 min |
| Sec. V-C — batch-phase timing | `exp_batch_phase.py` | `batch_phase.txt` | 10 min |
| Sec. V-J — fleet scaling | `exp_scaling.py` | `scaling.txt` | 10 min |
| Sec. V-K — warm-up length | `exp_warmup.py` | `warmup.txt` | 6 min |
| Sec. III-C, V-K — partly filled windows, minimum-sample rule, rare contexts and count-based warm-up | `exp_window_fill.py` | `window_fill.txt/.json` | 20 min |
| Sec. III-F — tracker reset rule: reset on every transmission vs. only on TV/tracker triggers, no tracker, no trigger | `exp_tracker_reset.py` | `tracker_reset.txt/.json` | 15 min |
| Sec. III-G — K_p persistence on cached summaries: counting rounds vs. new summaries, each calibrated to ARL₀ = 1000 | `exp_persistence_cache.py` | `persistence_cache.txt/.json` | 15 min |
| Sec. III-F — fog aggregate with silent members: full-coverage rule, partial coverage with the full-coverage τ_l, and τ_l per number of fresh members | `exp_partial_coverage.py` | `partial_coverage.txt/.json` | 25 min |
| Sec. V-H — every detector at m = 0.15 and with 5 or 1 compromised agents | `exp_baselines_small.py` | `baselines_small.txt/.json` | 10 min |
| Sec. V-D — episode-level contexts and Poisson call counts: window filling, ARL, delay | `exp_episode_workload.py` | `episode_workload.txt/.json` | 45 min |
| Sec. V-A — scalar baselines: input stream (conditional vs. unconditional), DDM mapping, CUSUM reference value, held-out ARL | `exp_scalar_parity.py` | `scalar_parity.txt/.json` | 15 min |
| Tables IV–VII, Figs. 2–3 — batched tuple baselines with onset at a random batch offset (second values, open markers) | `exp_batch_random_phase.py` | `batch_random_phase.txt/.json` | 30 min |
| Sec. V-F, V-G, Table VII(a), Fig. 6 — tuple baselines that keep only the W most recent unsent tuples per context (drop-oldest ring) under the byte cap, dropout with ACK, burst and fog outages; FIFO delay vs. time before onset | `exp_tuple_buffering.py` | `tuple_buffering.txt/.json` | 45 min |
| Sec. V-H, Tables VIII–IX — small persistent shifts: tracker reset rule (heartbeats reset or not, no tracker) on AA1/AA6/AA8 and the Table IX regimes; window size W = 20/50/100 and parallel W = 20 + 100: smallest detectable shift and delay cost | `exp_small_shift.py` | `small_shift.txt/.json` | 15 min |

Times are for a single CPU core of a recent laptop.

### Figures

```bash
python figures/plot_paper_figures.py
```

This draws Figs. 2–7 from the files in `results/` and writes PDF and PNG files to `figures/`.
Fig. 1 (architecture) is a drawing and is not generated by code.
Fig. 4 is the alert-timing schematic.

### InjecAgent ground-truth replay (Table X(b), Fig. 7)

No LLM is called. The attacker-tool sequences of InjecAgent are replayed into the detector
after a benign prefix.

```bash
git clone https://github.com/uiuc-kang-lab/InjecAgent
python exp_injecagent_replay.py InjecAgent/data data/injecagent_toolkit_map.json
python exp_injecagent_identity.py InjecAgent/data data/injecagent_toolkit_map.json
```

The same cases also run through the 50-agent fleet pipeline (edge trigger, tracker,
heartbeats, fog tiers, matched ARL0 calibration) of the AgentDojo replay. One-call tasks fill
the per-context windows slowly, so these runs settle for 300 rounds before calibration and
onset:

```bash
python injecagent_traces.py InjecAgent/data data/injecagent_toolkit_map.json
python exp_agentdojo_replay.py data/injecagent_gt_dh.jsonl data/injecagent_gt_ds.jsonl settle=300
```

`data/injecagent_toolkit_map.json` maps each InjecAgent tool to its toolkit (the tool
identity) and to one of six context families. The context of a case is the family of its
user tool, fixed by the user request before any tool output is read; injected attacker
calls inherit it. Adding `tool-context` to `exp_injecagent_replay.py` or `exp_injecagent_identity.py` uses the family of the invoked
toolkit instead, which makes the context a function of the tool; these runs write the
`_toolcontext` result files.

### InjecAgent live runs (Table X(a))

The live runs used Qwen3.8-27B and GPT-OSS-20B (`qwen/qwen3.8-27b`, `openai/gpt-oss-20b`) through the Groq API, with InjecAgent's own
evaluation harness. They need an API key, and the raw outputs are not part of this
repository. Given InjecAgent result files, the attack-success rates are computed with:

```bash
python injecagent_asr.py results_dh.json results_ds.json
```

The script reports ASR-valid and ASR-all. When there are no successful attacks, it also
reports a one-sided 95% upper bound.

### AgentDojo (Table XI)

Collecting traces needs an OpenAI-compatible endpoint and an API key:

```bash
pip install "agentdojo==0.1.35" openai
export GROQ_API_KEY=...
python agentdojo_collect.py --model openai/gpt-oss-20b \
    --base-url https://api.groq.com/openai/v1 --api-key-env GROQ_API_KEY \
    --suites banking --pairs-per-suite 40 \
    --out data/agentdojo_gptoss20b_banking.jsonl
```

The collector is resumable: rerunning the same command skips finished episodes.
The detection part makes no API calls and needs only the trace file:

```bash
python exp_agentdojo_replay.py data/agentdojo_gptoss20b_banking.jsonl
```

To test the pipeline without a key, run:

```bash
python agentdojo_collect.py --dry-run --out traces_dryrun.jsonl
python exp_agentdojo_replay.py traces_dryrun.jsonl quick
```

The dry run uses AgentDojo's ground-truth actions in place of a model, so its numbers are
not model results.

Table XI(b) replays the run logs that AgentDojo publishes with its benchmark
(github.com/ethz-spylab/agentdojo, `runs/`, MIT licence): every episode's tool calls with
AgentDojo's own utility and attack-success labels, four suites, the important_instructions
attack on every (user task, injection task) pair. The converted traces of GPT-4o,
GPT-4o-mini, Llama-3-70B, Llama-3.3-70B, GPT-3.5-turbo and Command-R are in
`data/agentdojo_runs_<model>.jsonl` (no API calls needed). To rebuild them:

```bash
git clone --depth 1 --filter=blob:none --sparse https://github.com/ethz-spylab/agentdojo
cd agentdojo && git sparse-checkout set --no-cone \
    '/runs/gpt-4o-mini-2024-07-18/*/user_task_*/important_instructions/*' \
    '/runs/gpt-4o-mini-2024-07-18/*/user_task_*/none/*'        # likewise for the other models
cd .. && python agentdojo_runs.py agentdojo/runs gpt-4o-mini-2024-07-18 ...
python exp_agentdojo_replay.py data/agentdojo_runs_gpt-4o-mini-2024-07-18.jsonl
python exp_agentdojo_quarantine.py data/agentdojo_runs_gpt-4o-mini-2024-07-18.jsonl
```

## Reproducibility

- All randomness uses `numpy.random.default_rng` with fixed seeds.
- Calibration, held-out validation and each experiment use disjoint seed ranges, except that
  Tables IV, V, VIII and IX share the evaluation seeds 300–329, so a configuration that
  appears in several of them gives the same result in each.
- Delays are in rounds and do not depend on the machine.
- CPU times in `results/scaling.txt` (Sec. V-J) do depend on the machine and on the
  Python/NumPy build; only their trend with fleet size is meaningful.
- The result files in `results/` were produced with the full (non-quick) settings.

## Citation

```bibtex
@article{chaturvedi2026fogdrift,
  author  = {Chaturvedi, Prachi and Ahmad, Shahnawaz and Nowroozi, Ehsan and
             Loukas, George and Cordeiro, Lucas C. and Issac, Biju},
  title   = {{FogDrift}: Context-Conditional Tool-Use Drift Monitoring for
             Distributed {LLM}-Agent Fleets},
  journal = {IEEE Transactions on Dependable and Secure Computing},
  year    = {2026}
}
```

## License

MIT; see [LICENSE](LICENSE).
