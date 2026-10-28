<p align="center">
  <img src="assets/banner.svg" alt="FieldOps: edge AI for orchard pest management" width="100%">
</p>

<p align="center">
  <img alt="Dell x NVIDIA AI Hackathon, Cambridge MA, invite-only" src="https://img.shields.io/badge/Dell%20x%20NVIDIA%20AI%20Hackathon-Cambridge%2C%20MA%20(invite--only)-76B900?style=flat-square&logo=nvidia&logoColor=white">
  <img alt="Team entry" src="https://img.shields.io/badge/team%20entry-4%20engineers-2d6a4f?style=flat-square">
  <img alt="Runs on Dell GB10 (NVIDIA Grace Blackwell)" src="https://img.shields.io/badge/hardware-Dell%20GB10%20(Grace%20Blackwell)-76B900?style=flat-square&logo=nvidia&logoColor=white">
  <img alt="Local inference" src="https://img.shields.io/badge/inference-local%20on--device-1f6feb?style=flat-square">
  <br>
  <img alt="Python" src="https://img.shields.io/badge/Python-3776AB?style=flat-square&logo=python&logoColor=white">
  <img alt="YOLO26s detector (Ultralytics)" src="https://img.shields.io/badge/vision-YOLO26s%20(Ultralytics)-111F68?style=flat-square">
  <img alt="Qwen3.6 on vLLM" src="https://img.shields.io/badge/agent%20LLM-Qwen3.6%20on%20vLLM-6f42c1?style=flat-square">
  <img alt="SQLite" src="https://img.shields.io/badge/store-SQLite-003B57?style=flat-square&logo=sqlite&logoColor=white">
  <a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-blue?style=flat-square"></a>
</p>

# FieldOps

**Edge AI for orchard pest management: count the insects, start the biological clock, and tell the grower when to act.**

FieldOps turns photos of sticky traps into species counts, tracks the pest flight curve across a season,
and calls the treatment window with a deterministic degree-day model. Vision, the agent LLM, speech and
timing all run locally on a single Dell GB10 (NVIDIA Grace Blackwell).

> **Hackathon entry.** Built by a four-person team in one day for the invite-only Dell x NVIDIA AI
> Hackathon in Cambridge, MA, on October 3, 2026.

**[Final presentation](pitch/2026-10-03-fieldops.html)** · [Two-minute demo film](pitch/video/fieldops-demo.mp4) · [Transcript](pitch/video/transcript.txt) · [Presenter runbook](pitch/DEMO.md)

---

## The problem

Growers inspect trap catches and combine those observations with weather to decide when a
block needs attention. Manual checks leave gaps between observations, and counts alone do not
show where a pest is in its development cycle.

FieldOps brings photo counting, season activity, and a deterministic degree-day clock into one
view. The demo uses a committed synthetic season: six blocks and eighteen traps. It demonstrates
the workflow; field accuracy and time savings still need validation with growers.

## What FieldOps does

- **Counts insects on a sticky trap** from a photo, by species. Send the Telegram bot a picture of
  a trap card and a YOLO26 detector on the GB10 counts codling moths, oriental fruit moths and
  spotted lanternflies, ignoring gnats and debris.
- **Tracks the flight curve** across the season and detects **biofix** — the first sustained catch,
  the moment the pest clock starts.
- **Accumulates degree-days** from daily high/low temperatures once biofix is set.
- **Calls the spray window** when accumulated degree-days cross the species threshold.
- **Buzzes the grower's phone** in plain language: *"Biofix reached on block C. Spray window opens
  Thursday."* — and answers follow-ups like *"why Thursday?"*

The counting is the party trick. The decision is the product.

## How it works

```text
 trap photo ──▶ vision ──▶ { trap_id, timestamp, species, count } ──▶ store (SQLite)
              (YOLO26, GPU)                                                 │
                                                                            ▼
 daily temps ──────────────────────────────────────────────────────▶    decide
                                                       biofix · degree-days · threshold
                                                                            │
                                                                            ▼
                                                           agent (local LLM) ──▶ phone alert
                                                           phrases it, answers questions
```

The store is the single source of truth `decide` reads, and it holds both halves of the
decision: trap counts and the daily temperatures the degree-day clock runs on. It is rebuilt from
the committed `data/season.csv`, so a reload is deterministic and the replay comes out identical
every time. Live counts from the vision layer can append to the same store (opt-in, so a demo
photo never moves the spray dates). With no store present,
`decide` falls back to reading the CSV directly.

Two deliberate splits:

- **The decision is plain Python, not a model.** Biofix and degree-day math are deterministic and
  auditable. An LLM that hallucinates a spray date costs someone a harvest.
- **The LLM only phrases and explains.** It reads the decision, writes the grower-facing message,
  and answers questions about it. It never computes the answer.

## Why local

Vision inference, the local agent model, speech generation, and deterministic timing run on
the GB10. The committed season replay also works without internet. Telegram photo transport and
message delivery require connectivity; using Telegram sends those messages through its service.
Cloud tools used to produce the narrated film are separate from product inference.

## Results

The detector is YOLO26s, trained entirely on synthetic trap cards with exact labels. Our first model scored
mAP50 0.995 on its own validation set and still failed on phone-style photos, so we added a held-out
phone-photo stress test and fine-tuned on it (v2). Held-out, seed-separated phone-style photos (200):

| | v1 (clean cards) | **v2 (+ phone-style fine-tune)** |
|---|---|---|
| Codling moth, mean abs. error | 3.67 | **0.07** (189/200 exact) |
| Oriental fruit moth, mean abs. error | 4.65 | **0.04** (192/200 exact) |
| Spotted lanternfly, mean abs. error | 0.12 | **0.00** |

These are best-case numbers: the test uses the same kinds of image effects v2 trained on. Accuracy on real
traps is unmeasured and is the next step. Method, data splits and the overfitting analysis are in
[fieldops/yolo/README.md](fieldops/yolo/README.md).

## Components

Working end to end: season data, the decision engine, the replay dashboard, and the alert.
`fieldops/season.py` generates a season, `fieldops/decide.py` finds biofix, accumulates
degree-days and calls the spray window, and `--replay` writes the timeline the dashboard plays.
`fieldops/agent.py` turns an event into a sentence for the grower and `fieldops/alert.py` sends it
to a phone, and `fieldops/api.py` serves both the dashboard trigger and the OpenClaw tools. Also
here: a synthetic trap-image generator with ground-truth counts
(`fieldops/synth_traps.py`), the deck (`pitch/index.html`) and the presenter runbook
(`pitch/DEMO.md`).

**Vision has landed.** `fieldops/vision.py` counts pests in a trap photo with a YOLO26 detector
trained on synthetic cards (`models/fieldops-yolo26s-v2.pt`). The OpenClaw agent on Telegram sends
photos to it through `POST /count` on the tools API. How it was trained, how accurate it is, and
the overfitting we caught and fixed are in [fieldops/yolo/README.md](fieldops/yolo/README.md).

**Models on the GB10:** the agent's LLM is Qwen3.6-35B-A3B (NVFP4) on the local vLLM server at
`127.0.0.1:8000`, and counting is YOLO26s in a GPU container. Both run offline.

**On the GB10 everything starts at boot:** `bash deploy/install.sh` installs `fieldops.target`
(OpenClaw sandbox + Telegram, vision service, tools API) as systemd user services. See
[deploy/README.md](deploy/README.md). See [PLAN.md](PLAN.md) for the order of work and open decisions.

## Quick start

```bash
python -m fieldops.check_models                  # do the local LLM and VLM load? run this first
python -m fieldops.telegram_setup                # one-time: connect the phone alert to your Telegram bot
python -m fieldops.season                        # regenerate data/season.csv
python -m fieldops.weather --normals             # refetch the average temperatures behind the spray-date estimate (needs network)
python -m fieldops.store --load data/season.csv  # build the store: counts + temperatures
python -m fieldops.decide                        # self-test, then print each block's milestones
python -m fieldops.decide --replay               # ...and write the dashboard timeline
python -m fieldops.agent --status                # what the agent's tools return right now
python -m fieldops.api                           # tool server: dashboard buzz + OpenClaw tools + /count

# vision (GPU container; details in fieldops/yolo/README.md)
bash fieldops/yolo/run.sh serve                  # YOLO counting service on 127.0.0.1:8767
curl --data-binary @data/traps/trap_05_codling050.jpg "localhost:8765/count?trap_id=block-c-04"
bash fieldops/yolo/run.sh eval                   # score the 6 reference cards
bash fieldops/yolo/run.sh phone-test --n 200     # accuracy on phone-style photos (overfitting check)
open dashboard/index.html                        # the replay, straight from disk, no server
open pitch/index.html                            # the deck; arrow keys to advance

python -m fieldops.decide --csv                  # bypass the store and read the CSV directly

# ground-truth trap counts as test data. Use a scratch --db: these traps sit in block C on a
# real season date, so loading them into the demo store spikes its flight curve.
python -m fieldops.store --db data/test.db --load-traps data/traps/manifest.json

# two seasons of multi-species history in its own store, for exercising queries and the
# vision path. 4,665 readings, 30 traps, 10 species, irregular check times.
python -m fieldops.history
```

## Repo map

| Path | What's in it |
|---|---|
| [PLAN.md](PLAN.md) | Implementation plan — phases, tasks, interfaces, risks, definition of done. |
| [ORCHESTRATION.md](ORCHESTRATION.md) | How the pieces run together: processes, ports, data flow, start order, failure modes. |
| [AGENTS.md](AGENTS.md) | Build conventions for coding agents and teammates. Read before writing code. |
| [FieldOps_Hackathon_Plan.md](FieldOps_Hackathon_Plan.md) | The pitch: story beats, demo script, scope calls. |
| [dashboard/TIMELINE_CONTRACT.md](dashboard/TIMELINE_CONTRACT.md) | The `decide → dashboard` timeline shape and the decision rules. Frozen. |
| [fieldops/synth_traps.py](fieldops/synth_traps.py) | Generates synthetic trap photos with ground-truth counts into [data/traps/](data/traps/). |
| [fieldops/vision.py](fieldops/vision.py) | The vision lane: YOLO counting service and CLI, contract records out. |
| [fieldops/yolo/](fieldops/yolo/README.md) | Dataset generator, training, counting, phone-photo stress test, GPU container. |
| [models/](models/README.md) | Trained detector weights and their model card. |
| [openclaw/](openclaw/) | The OpenClaw skill (tools + photo routing) and the sandbox network policy. |
| [deploy/](deploy/README.md) | systemd units for the GB10, plus the CI deploy unit. |

## Stack

Python on a Dell/NVIDIA GB10. A YOLO26 detector for counting, Qwen3.6 on vLLM for the agent
layer, Kokoro for local speech, NemoClaw + OpenClaw + OpenShell for the sandboxed Telegram agent,
SQLite for counts and weather, and a single-page dashboard. No cloud services anywhere in the
product's inference path.

## Limitations

- The demo runs on a committed **synthetic season** (six blocks, eighteen traps). Field accuracy,
  yield impact and time savings are not claimed and still need validation with growers.
- Detector accuracy is measured on synthetic and phone-style synthetic photos, not on real traps.
- The 3D drone survey is a simulation, and the drone footage in the film is labeled concept footage.
- Telegram photo transport and alert delivery need a network connection; everything else is local.

## Team

A four-person team, working in parallel across vision, decision logic, agent and alerting, and the
dashboard and presentation.

| Name | GitHub |
|---|---|
| Himanshu | [@hnagpa](https://github.com/hnagpa) |
| Jen Kelleman | [@jkelleman](https://github.com/jkelleman) |
| Piyush Dubey | [@piyushdubey](https://github.com/piyushdubey) |
| Rishab Nayak | [@rishabnayak](https://github.com/rishabnayak) |

## License

MIT — see [LICENSE](LICENSE).
