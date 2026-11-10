<p align="center">
  <a href="https://javgilavi.github.io/liminal/">
    <img src="docs/assets/liminal-header.png" alt="Liminal: dual-arm manipulation with FLUX 3 Action" width="100%" />
  </a>
</p>

<p align="center">
  <a href="https://javgilavi.github.io/liminal/">Project page</a> ·
  <a href="https://javgilavi.github.io/liminal/live.html">Live lab</a> ·
  <a href="https://drive.google.com/file/d/1et5ffQpXLLh4a1qOvociNB1weNlaCGPU/view">Video (2 min)</a> ·
  <a href="https://drive.google.com/file/d/1tG4ehYkKH0PK-AJg1K_wNPsWDiAQaoM3/view">Slides</a> ·
  <a href="https://huggingface.co/Oskrt">Models and datasets</a>
</p>

# Liminal

Liminal is an orchestration layer for long-horizon, dual-arm manipulation on top of FLUX 3 Action, a
vision-language-action (VLA) model. It was built at the ERH Physical AI Hackathon (18 October 2026, track
"Imagine, Then Act").

A VLA is good at short, text-prompted skills on one arm, such as *put the green pen in the green slot*. Real tasks are
longer and use both arms: one arm opens a box and keeps the lid up while the other places objects inside. Liminal adds
the layer around the model that decides:

- **what** runs next, on **which arm**, and **when** the other arm is ready;
- **which conditions must stay true** while a skill runs (the lid stays open while the other arm works);
- **whether a step actually succeeded**, judged from a fresh camera check and never from the skill's own report.

Every motor command goes through one safety supervisor with joint limits, step limits, arm ownership and an e-stop.
FLUX executes the manipulation; Liminal plans, verifies and recovers.

## Results

The robot is a green XLeRobot we call the *Haller*: two SO-101 arms on a torso, a RealSense D455 head camera, a wrist
camera on each arm and soft fin-ray fingers. The demo task is a four-step chain:

| Step | Arm | Executed by |
|---|---|---|
| Open the box and hold the lid | right | scripted replay of a recorded demo (the FLUX box LoRA did not start reliably from rest) |
| Put the green pen in the green slot | left | FLUX 3 Action + LoRA |
| Put the yellow cutter in the yellow slot | left | FLUX 3 Action + LoRA |
| Put the red pen in its slot | left | FLUX 3 Action + LoRA |

[![Full four-skill chain on the Haller, one run at 5.71x speed with no cuts](media/full_chain_60s_speedup.webp)](https://drive.google.com/file/d/1tK71kTtUIeYPc1cwt6Jqc6ZbzJMnnCIB/view)

*One complete run, sped up 5.71x with no cuts. The operator console shows the plan, Claude's step checks, the head and
wrist cameras, and the frames FLUX predicts next to what actually happens.*

- **Full chain: 2 of 3 runs completed all four steps with no human intervention** (208.6 s and 302.2 s). Claude
  planned the chain and checked every step on the head camera. When the box replay missed twice, it replanned and the
  third attempt opened the box. In both successful runs the green-pen policy also slotted the red pen on its way home,
  so the final check found it already in place.
- **Fine-tuning made the difference.** On the red-pen insert, the released FLUX checkpoint went 0/16. Our LoRA (rank
  32, trained on the skill's own teleoperated demos) went 3/5, then 2/3 in a later session.
- **The behavior-tree harness ran on the real robot** through the same safety stack. It skipped the box step when it
  saw the lid already open and placed the green pen with its FLUX LoRA, with Claude Opus 5.5 verifying each step.
  The Opus 5.5 hybrid gate ran in observation mode.

During the demo runs, keep-out zones and stall/load stops were disabled; joint limits, a 1° per tick step limit, a 60 %
torque cap and the e-stop stayed on. Sample sizes are small. The full write-up with every trial is in
[ws4/pitch/README_submission.md](ws4/pitch/README_submission.md).

## How it works

[![Liminal architecture: a behavior-tree harness coordinates independent left and right FLUX 3 Action policies, with perception updating verified world-state predicates and safety gating every command](docs/assets/liminal-architecture.png)](docs/assets/liminal-architecture.png)

**Behavior-tree harness** (`src/liminal/orchestration`). A task is a YAML behavior tree built from `sequence`,
`parallel`, `fallback`, `while`/`until`, `retry`, `timeout` and `check` nodes. Each skill node names an arm, a text
prompt and a `success` predicate. A skill counts as done only when that predicate holds in the world state, and a
`while` node cancels its body as soon as the maintained condition breaks. Nodes claim the arms they use, so two skills
can never command the same arm.

```yaml
tree:
  sequence:
    - skill: {arm: left, name: open_lid, prompt: "Lift the lid open", success: lid_open}
    - while:
        arm: left
        maintain: lid_open            # must stay true, or the body is cancelled
        until: object_inserted
        do:
          skill: {arm: right, name: pick_place, prompt: "Place the block inside the open box",
                  success: object_inserted}
    - check: object_inserted
```

Task files are written by hand or generated by a planner (template, Claude or Gemini). Planner output is compiled and
validated by the same grammar (`orchestration/task_spec.py`) before anything runs.

**Per-arm FLUX skills** (`src/liminal/policy`). Each arm has its own FLUX 3 Action instance, observation history and
inference lock. A skill registry maps each skill to a LoRA bundle, so changing skill means changing adapter, not model.
Holding and releasing use a local position controller and skip inference.

**Perception** (`src/liminal/perception`). Predicates such as `lid_open` are evaluated from camera frames: cheap colour
and zone checks first, a VLM when needed (Claude on the Haller, Qwen3-VL-4B as a local option). Every run writes a
JSONL event log of facts, node transitions, retries and violations under `runs/`.

**Hybrid gate** (`src/liminal/policy/hybrid.py`). An optional Claude gate reviews each FLUX action chunk and chooses
to execute it, edit it, replace it with a bounded end-effector target, or stop. Edits go through local IK and the
safety supervisor. The gate is opt-in and refuses to move the robot until the kinematic mapping has been verified.

**Haller stack** (`ws4/`). The FLUX skill pipeline, LoRA training, safety runner and Claude orchestrator used for the
demo. The behavior-tree harness drives the Haller through this stack via `src/liminal/ws4_bridge`; see
[ws4/HARNESS_BRIDGE.md](ws4/HARNESS_BRIDGE.md).

## Quick start

No robot or GPU required: tests and demos run on mock arms.

```bash
uv sync --extra dev
uv run pytest

uv run liminal validate configs/tasks/*.yaml               # compile every task file
uv run liminal run configs/tasks/lid_hold_insert.yaml      # cooperative holding on mock arms
uv run liminal plan "hold the lid and insert the block"    # goal -> task tree (template planner)
```

`--planner claude` or `--planner gemini` needs `--extra llm` and an API key. For real hardware:

| Guide | Covers |
|---|---|
| [Robot control](docs/robot-control.md) | XLeRobot setup, calibration, limits, operator UI |
| [Official inference](docs/official-inference.md) | FLUX inference on the real arm with a remote GPU |
| [Skill runtime](docs/showcase-runtime.md) | per-arm LoRA bundles and the checkpoint registry |
| [Haller bridge](ws4/HARNESS_BRIDGE.md) | running the behavior tree on the Haller |
| [Research UI](docs/research-ui.md) | project page, live lab and MuJoCo digital twin |
| [LoRA how-to](ws4/LORA_HOWTO.md) | training a skill adapter on your own demos |

## Models and datasets

All public on Hugging Face. Each model repo holds every checkpoint under `checkpoints/<run>/<step>/{raw,ema}`.

| Skill | LoRA | Dataset (LeRobot v3, VR teleoperation) |
|---|---|---|
| Red pen | [haller-pen-lora](https://huggingface.co/Oskrt/flux-3-action-so101-haller-pen-lora) (demo: `lora_pen84/005000/raw`) | [put the red pen in its slot](https://huggingface.co/datasets/Oskrt/haller_put_the_red_pen_in_its_slot_in_the_cardboard_box) |
| Green pen | [haller-green-pen-lora](https://huggingface.co/Oskrt/flux-3-action-so101-haller-green-pen-lora) | [put the green pen in the green slot](https://huggingface.co/datasets/Oskrt/haller_put_the_green_pen_in_the_green_slot) |
| Yellow cutter | [haller-yellow-cutter-lora](https://huggingface.co/Oskrt/flux-3-action-so101-haller-yellow-cutter-lora) | [put the yellow cutter in the yellow slot](https://huggingface.co/datasets/Oskrt/haller_put_the_yellow_cutter_in_the_yellow_slot) |
| Open box | [haller-open-box-lora](https://huggingface.co/Oskrt/flux-3-action-so101-haller-open-box-lora) (not used in the demo) | [open the box](https://huggingface.co/datasets/Oskrt/haller_open_the_box) |
| Take the red pen out | – | [take the red pen out](https://huggingface.co/datasets/Oskrt/haller_take_the_red_pen_out_of_its_slot) |

## Repository layout

```
src/liminal/
  orchestration/  behavior tree, task-file compiler, executor, planners
  perception/     world state, event log, colour/zone detectors, VLM verifier
  policy/         FLUX 3 Action backend, hybrid gate, scripted fallback, mock
  safety/         arm ownership, joint limits, zones, e-stop
  hardware/       SO-101 via LeRobot, cameras, kinematics, mock arm
  ws4_bridge/     runs the behavior tree on the Haller stack
  sim/ bimanual/ ui/ core/
configs/          robots, policies, skills, perception, task trees
ws4/              Haller stack: FLUX skill pipeline, LoRA training, safety runner, orchestrator
docs/             guides, validation reports, architecture
tests/            run on mock arms, no GPU or network
```

Design and decisions: [docs/architecture.md](docs/architecture.md), [PLAN.md](PLAN.md). Contribution rules:
[AGENTS.md](AGENTS.md).

## Limitations

- Sample sizes are small; the numbers above are hackathon results, not benchmarks.
- The box step in the demo is a scripted replay, not a learned policy.
- Learned skills running on both arms at the same time have not been verified on hardware.
- SO-101 holding is position control, not torque control; keep loads light.
- In simulation the LoRA improved grasping but completed 0/5 placements
  ([validation report](docs/validation/single-arm-v1.md)).

## Team

Oscar Devos, Javier Gil, Tomas Leggeat, Simone Utili, Stanisław Sufin.
