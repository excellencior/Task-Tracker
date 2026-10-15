# From Conversation History to Task State: Diagnosing Small Language Models under Incremental Observations

Apurbo Banik Turjo

**Collaborators:** Sushmita Paul, Dr. Ch. Md. Rakin Haider

Research progress updated 15 October 2026. Paper in preparation.

This repository presents the study's design and current results. The aggregate
analysis is public here; raw trajectories, full prompts, and run manifests are
not included.

## Research question

An agent acting over many turns must use each new observation without losing
the task state built so far. We ask whether externally maintained state,
conversation history, and state-report verification help small language models
complete interactive tasks while following the required algorithm.

We study breadth-first search (BFS) and depth-first search (DFS) on partially
observed trees. Each turn reveals only the neighbors of the node reached.
These tasks let us separate visiting all nodes from following the correct
traversal order. They are controlled diagnostics, not the full range of agents
we hope to study.

![An agent acts on partial observations while an external system maintains task state.](assets/research_setup.png)

## Experimental setup

The completed behavioral evidence contains **57,600 trajectories** across two
matched external-state campaigns and two model-managed-memory variants. Every
campaign uses Qwen3-1.7B, Qwen3.5-2B, Qwen3.5-4B, and Gemma-4-E4B-IT; normal
and hard BFS/DFS instances; 200 matched graphs per exact cell; greedy decoding;
a 4,096-token output cap; a 2,048-token thinking budget; and a 32,768-token
context limit.

| Campaign | Trajectories | State or memory supplied | History regimes | Prompt / execution |
| --- | ---: | --- | --- | --- |
| Initial external state | 25,600 | BFS: node + visited, no queue. DFS: node + visited + stack. RAW, PSR, SSR, VI. | Retain / clear injection | `p1_concise_legacy`; H100 NVL; up to 16 parallel trajectories. |
| Matched control-state follow-up | 25,600 | BFS: queue supplied. DFS: stack omitted. The same RAW, PSR, SSR, VI cells. | Retain / clear injection | Same task prompt, decoding, graphs, limits, and scorer; H100; 64 parallel trajectories. |
| Model memory + history | 3,200 | Model-chosen free-schema JSON memory, capped at 1,024 tokens; no verified queue or stack. | Full conversation plus latest memory | `p1_model_memory_injection`; H100; 64 parallel trajectories. |
| Model memory only | 3,200 | The same model-owned JSON store is the only cross-turn memory beyond current observation and feedback. | Earlier conversation omitted | `p1_model_memory_memory_only`; H100; 64 parallel trajectories. |

The two external-state campaigns form the intended symmetric control-state
comparison: together they cover BFS with and without its queue and DFS with
and without its stack while holding the remaining cell parameters fixed. The
model-managed variants ask whether the model can decide what to store and
preserve it, so they remain separate from the external-state averages.

![Four state-support conditions: RAW, PSR, SSR, and VI.](assets/state_conditions.png)

| Condition | State support |
| --- | --- |
| **RAW** | Current observation, with no injected state reminder. |
| **PSR: Prose State Reminder** | Verified state restated in ordinary language. |
| **SSR: Structured State Reminder** | The same task-specific facts in a tagged state block. |
| **VI: Verifiable Injection** | SSR plus a required state report, checked against the environment, with feedback after a missing, malformed, or incorrect report. |

![VI checks the model's state report and sends a correction on the next turn when needed.](assets/vi_state_matching.png)

Under **retain**, earlier injected reminders remain. Under **clear injection**,
only the older reminder text is removed; previous observations and visible
model replies remain. The current reminder is always present.

![Retaining and clearing earlier state reminders.](assets/history_policies.png)

## Research progress

This checklist changes as the study progresses. Completed items are crossed
out; active and planned work remains open.

### Completed

- [x] ~~Build and validate the long-horizon BFS/DFS evaluation pipeline.~~
- [x] ~~Run the original 25,600-trajectory state-support campaign.~~
- [x] ~~Audit the behavioral metrics and complete the 2,560-trajectory primary
  LLM-judge pass.~~
- [x] ~~Run the matched H100 follow-up with the BFS queue supplied and the DFS
  stack withheld.~~ This completed the symmetric control-state comparison.
- [x] ~~Run the first model-owned-memory pilot.~~ The pilot compared memory with
  full history against memory as the only cross-turn state and revealed that
  strict outer-JSON compliance was confounded with memory quality.
- [x] ~~Back up the completed follow-up trajectories and preserve the run
  manifests.~~

### In progress

- [ ] Run the 25,600-trajectory memory-responsibility study. It compares RAW,
  oracle state, free-form model memory, and fixed-schema model memory, each with
  and without conversation history. The outer JSON response is constrained so
  the experiment can measure memory behavior rather than Markdown formatting.
- [ ] Complete the matched analysis of the original and symmetric state-support
  campaigns, including model- and difficulty-level effects.
- [ ] Trace where autonomous memory first becomes inconsistent and separate
  memory-selection, memory-update, and action-selection failures.

### Next

- [ ] Complete judge repeatability and independent human calibration.
- [ ] Evaluate a time-aligned VI protocol.
- [ ] Test whether the findings transfer beyond graph traversal.

## What the metrics mean

Let `V` be all graph nodes, `U_K` the nodes visited after `K` accepted
moves, `L` the recorded node-history length, and `ell` the uninterrupted
initial sequence of algorithm-correct accepted moves.

| Measure | Definition | Why it matters |
| --- | --- | --- |
| **Graph completion** | `C = 1[U_K = V]` | Did the model visit every node, regardless of order? |
| **Correct traversal prefix** | `P = ell / L` | How long did it follow BFS or DFS before the first deviation? Later recovery does not restore the prefix. |
| **Algorithm-correct completion** | `S = 1[C = 1 and P = 1]` | Did it both finish and preserve the required traversal order? |
| **Zero-error FIFO success** | `S_0 = 1[S_FIFO = 1 and invalid attempts = 0 and format failures = 0]` | How often was strict traversal success achieved without recorded interface errors? |
| **Valid-move rate** | admissible attempts / all recorded attempts | Separates legal moves from algorithmically correct moves. |
| **Final decoverage** | `1 - |U_K| / |V|` | Fraction of nodes left undiscovered. |

AQA-Bench also records accumulated decoverage, the sum of uncovered fractions
after accepted moves. It is useful for following progress, but a short failed
episode can have a deceptively small sum. We read it alongside completion and
episode length.

The BFS campaign was originally scored with a strict FIFO-head checker.
We also replayed its recorded actions under the historical AQA-Bench BFS
frontier checker. The main **AQA success** values below use that replayed
checker; FIFO success answers the stricter prompt-specific question. DFS
scores coincide under the two rules. A valid move can still break either
traversal order.

## Behavioral results

The first table preserves the parameters of the **initial externally supplied
state campaign**. Each row pools 3,200 trajectories across models,
difficulties, and history policies.

| Task | Control state | Condition | Graph completion | AQA algorithm-correct completion |
| --- | --- | --- | ---: | ---: |
| BFS | Queue not supplied | RAW | 10.94% | 0.44% |
| BFS | Queue not supplied | PSR | 24.50% | 1.63% |
| BFS | Queue not supplied | SSR | 24.53% | 2.56% |
| BFS | Queue not supplied | VI | 16.41% | 3.03% |
| DFS | Stack supplied | RAW | 22.69% | 14.88% |
| DFS | Stack supplied | PSR | 72.84% | 58.13% |
| DFS | Stack supplied | SSR | 79.19% | 70.00% |
| DFS | Stack supplied | VI | 74.59% | 62.81% |

![Graph completion, recorded FIFO success, and replayed AQA success by task and condition.](assets/result_task_performance.png)

The matched follow-up preserves its complementary parameters. Each row again
contains 3,200 trajectories and pools the same models, difficulties, and two
history policies. Strict success means completion with a perfect recorded
traversal prefix.

| Task | Control state | Condition | Graph completion | Strict success |
| --- | --- | --- | ---: | ---: |
| BFS | Queue supplied | RAW | 10.00% | 0.28% |
| BFS | Queue supplied | PSR | 82.09% | 33.19% |
| BFS | Queue supplied | SSR | 87.75% | 45.91% |
| BFS | Queue supplied | VI | 80.25% | 35.31% |
| DFS | Stack omitted | RAW | 25.41% | 16.38% |
| DFS | Stack omitted | PSR | 51.16% | 40.53% |
| DFS | Stack omitted | SSR | 46.88% | 38.38% |
| DFS | Stack omitted | VI | 41.75% | 32.72% |

Supplying the missing BFS queue sharply raises externally assisted BFS
performance, while removing the DFS stack reduces the corresponding DFS
performance. RAW changes little. Together, the campaigns show that the
original task gap partly reflected which control state the system supplied;
they do not imply that BFS and DFS are intrinsically identical tasks.

The final 6,400 trajectories let the model choose its own memory contents.
Each row pools 1,600 trajectories across four models and two difficulties.
The JSON contract means these runs jointly test memory design, retention,
action choice, and format compliance.

| Task | Model-managed variant | Graph completion | Strict success |
| --- | --- | ---: | ---: |
| BFS | Memory injected alongside full history | 1.38% | 0.12% |
| BFS | Memory as the only cross-turn store | 3.25% | 0.19% |
| DFS | Memory injected alongside full history | 22.69% | 12.69% |
| DFS | Memory as the only cross-turn store | 2.69% | 1.00% |

The memory-only DFS drop shows that permission to create a store does not
guarantee preservation of sufficient task state when dialogue history is
removed. BFS remains weak in both model-managed regimes. These are distinct
variants, not replacements for RAW/PSR/SSR/VI or an isolated causal comparison
with externally verified state.

Within the initial campaign, SSR minus PSR completion is **+0.03 points in BFS
without a queue** and **+6.34 points in DFS with a stack**. Retaining older
reminders raises completion by **15.97 points for PSR** and **19.03 points for
SSR**. The new judge campaign will apply the same blinded rubric to 3,200
sampled follow-up trajectories before process-level conclusions are added.

## LLM judge and reasoning traces

A separate primary pass used **claude-opus-4-6-thinking** to judge **2,560**
paired trajectories, including 20 shared graph IDs per task-difficulty
stratum across all models, conditions, and histories. The judge received a
fixed rubric, visible responses and captured reasoning, actions, and
authoritative graph facts. Model, condition, history-policy labels, and
aggregate outcome scores were withheld. Prompt content still revealed some
state-support cues.

It scored four 1–5 criteria:

1. State tracking.
2. Algorithmic reasoning and action coherence.
3. History and feedback use.
4. Visible response quality.

![Mean judge ratings on the four criteria, separately for BFS and DFS.](assets/result_judge_dimensions.png)

The equal-weight mean of these four ordinal ratings is **1.82 for RAW**,
**2.85 for PSR**, **3.03 for SSR**, and **2.59 for VI**. These are
descriptive process ratings, not independently validated measurements of
hidden reasoning. The judge also recorded a dominant-failure label and
turn-cited rationale. Inspected traces show concrete state and action errors,
but selected examples do not establish how often a mechanism occurs.

All 2,560 primary judgments were extracted with no schema validation errors.
Seven semantic warning flags were inspected. The selected **246 repeat
judgments remain unrun**, and an independent blinded human-calibration study
is not complete.

## Limits and next steps

The study uses four checkpoints, selected trees, finite token limits, and one
execution per exact cell. The initial external-state campaign was asymmetric;
the matched follow-up covers the complementary queue/stack settings. The
model-managed variants use a different prompt and JSON response contract, so
their absolute results do not isolate memory ownership alone.

VI combines a state report, delayed feedback, and a prompt-format tension.
In **4,625 of 6,400 VI trajectories**, at least one correction about a
previous turn appears alongside different current-state values. Those
observations describe the implemented VI protocol; they do not identify the
causal effect of verification alone.

![Recorded VI report outcomes and the share of trajectories with conflicting temporal values.](assets/result_vi_protocol.png)

The live checklist above tracks the follow-up analyses and experiments. The
paper will keep behavioral outcomes primary and judge analysis exploratory
until the repeatability and human-calibration checks are complete.

---

**Apurbo Banik Turjo** · Dhaka, Bangladesh  
[GitHub](https://github.com/excellencior) · [Portfolio](https://abturjo.onrender.com/) · [LinkedIn](https://www.linkedin.com/in/apurbo-banik-turjo-86b5b3328/) · [Email](mailto:turjob44@gmail.com)
