# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A **ComfyUI custom-node package** implementing CLSS (Closed-Loop Streaming Synthesis) for long
audio-video generation on top of LTX-2. It is loaded by a ComfyUI install (this repo lives at
`ComfyUI/custom_nodes/ComfyUI-LTX-CLSS`; `../../main.py` is the ComfyUI entry point).

`__init__.py` prepends `Ltx-2-CLSS/packages/{ltx-core,ltx-pipelines}/src` to `sys.path`, then
exports `NODE_CLASS_MAPPINGS` / `NODE_DISPLAY_NAME_MAPPINGS` from [nodes.py](nodes.py). All node
code is in that single ~2700-line file.

### `Ltx-2-CLSS/` is a git submodule with its own history

It is a registered **submodule** (see `.gitmodules`, pointing at
`github.com:nazgut/Ltx-2-CLSS.git`) — a fork of Lightricks' LTX-2. Clone with
`--recurse-submodules` (or `git submodule update --init`). The outer repo records only its
commit pointer (that is what a `m Ltx-2-CLSS` in `git status` means). Edits inside `Ltx-2-CLSS/` are
committed **in that inner repo**, separately from the outer node repo. The CLSS algorithm itself
lives there at
[Ltx-2-CLSS/packages/ltx-pipelines/src/ltx_pipelines/streaming/clss.py](Ltx-2-CLSS/packages/ltx-pipelines/src/ltx_pipelines/streaming/clss.py)
(`CLSSConfig`, `CLSSState`); `nodes.py` imports and orchestrates it against ComfyUI's sampler/guider
infrastructure. Sub-package architecture is documented in
[Ltx-2-CLSS/packages/ltx-pipelines/CLAUDE.md](Ltx-2-CLSS/packages/ltx-pipelines/CLAUDE.md).

## The nodes and how they wire together (Stage 1)

```
LTXVideo Loader → MODEL, VAE, CLIP
CLSSScenePrompts(CLIP, prompts)          → CONDITIONING   (one entry per scene, split by a line of '---')
LTXVConditioning(positive, negative, fps)→ positive, negative
CLSSAVGuiderV2(model, pos, neg, ...)     → GUIDER         (split video/audio CFG + modality + STG; replaces CFGGuider→CLSSAVGuider)
EmptyLTXVLatentVideo + audio latent + LTXVConcatAVLatent → LATENT  (per-chunk AV template)
CLSSConfig                               → CLSS_CONFIG
CLSSStreamingSampler(GUIDER, SAMPLER, SIGMAS, NOISE, LATENT, CLSS_CONFIG, num_chunks, [image, vae]) → LATENT
```

Two-stage pipeline: Stage 1 (`CLSSStreamingSampler`) → `LTXVLatentUpsampler` (2× spatial) →
`CLSSStage2` (chunked distilled-LoRA refinement, same SLB continuity mechanism). See
[workflow/i2v_LTX_CLSS.json](workflow/i2v_LTX_CLSS.json) for the canonical wiring.

The nodes (`NODE_CLASS_MAPPINGS`): `CLSSConfig`, `CLSSScenePrompts`, `CLSSStreamingSampler`,
`CLSSStage2`, `CLSSAVGuider` (split-CFG patch over an existing guider), `CLSSAVGuiderV2`
(all-in-one Stage-1 guider). When the guider's
positive has N scene entries,
the sampler auto-unpacks one scene per chunk proportionally across `num_chunks`.

### Audio latent scale (2026-08-09)

The audio VAE's normalizer is calibrated so **real audio sits at unit variance** — measured over
four real tracks: overall latent std 0.98-1.26, per-freq-bin std flat at 1.00. Generated audio
does not get there, and the shortfall tracks how bad it sounds: the strongest run on these
measures (a ~26 s per-chunk run) reached std 0.84 / 6.1 kHz bandwidth / stereo, while a 104 s
single-window soundtrack draft came in at std 0.48 / 3.1 kHz / L/R correlation 1.00000 (literally
mono). Half-scale in a log-mel space = quiet (−31.6 dBFS vs −21…−16 for real tracks) and
low-passed well under the 8 kHz mel ceiling. Do **not** try to fix this by scaling the latent's
variance up — measured, that raises log-mel contrast past real music's while crushing the
loudness envelope, and it is what produced the deleted node below. **The VAE is innocent** — real music round-trips through it with bandwidth and stereo
intact; don't go looking for a decode or sample-rate bug.

**The wall is on the per-chunk WINDOW, not on output length.** `CLSSStreamingSampler` chunks
audio exactly as it chunks video, so long audio just needs `overlap_af + new_af` < 20 s. At 24 fps
that is 409 px per chunk (17.1 s new + 2.7 s overlap = 20.0 s); 441 px is already over. The
sampler now prints an `audio window:` line every run and warns when it exceeds the wall. A
6 x 17.3 s chained run measured latent std 0.63 and was judged ok by ear (2026-08-10).

This retires the soundtrack draft: its only justification was that one pass saw the whole
timeline, so once it has to be chunked it duplicates the main sampler.  The two canonical
workflows in `workflow/` (i2v + t2v) are the single-pass per-chunk configuration.

Measure any take with `python simulations/audio_latent_probe.py FILE --reference` (audio VAE only,
CPU, runs while ComfyUI holds the GPU). **Gate on dynR, not bandwidth** — an ear-validated A/B
(2026-08-10) had the better-sounding take at LOWER ro99 (3023 vs 3516); the only separating
metric was the 0.25 s envelope range, 9.8 dB vs 3.3 dB. The drone is the flat envelope.
The probe runs in ~2 min with window length as the knob.

**Do not try to fix this after the fact.** A `CLSSAudioRestore` node that shifted the audio
latent's per-(channel,bin) mean toward real-music statistics was built and DELETED (2026-08-10):
it improved every metric being watched (ro99 3141->4477 Hz, +4 dB level) and the live run was
noise — the "recovered bandwidth" was a broadband noise bed (>4 kHz energy 0.22%->1.31%) and the
loudness envelope collapsed to 2.4 dB. Textbook case of the rule below: a latent metric moving
the right way is not a quality win.

### CLSS in one paragraph

Video is generated in short temporal **chunks** sharing an **SLB** (streaming latent buffer)
overlap, keeping latent memory O(overlap) instead of O(length). Four closed-loop corrections fight
exposure-bias drift: **§2.1** calibrated context re-noising (`tau_c`), **§2.3** EMA per-channel
AdaIN drift correction (`beta`), **§2.4** frequency-band soft shrinkage, **§2.5** dynamic anchor
bank. Audio and video are jointly modelled; audio needs higher CFG (~7) than video (~4), and audio
drift is a recurring failure mode (see the extensive per-input tooltips).

## Commands

There is **no build/lint/test step at the custom-node layer** — ComfyUI imports the package
directly. Ways to exercise the code:

- **Run in ComfyUI**: `cd ../.. && python main.py`, then load `workflow/i2v_LTX_CLSS.json`. This is
  the ground-truth path — the generation path can only be validated by a live run.
- **Standalone 16 GB-VRAM CLI** (no ComfyUI; loads GGUF transformer + Gemma):
  `python Ltx-2-CLSS/generate_clss.py --gguf-path ... --prompt "..."` (see its module docstring for
  the full flag set and the block-streaming / CLSS memory strategy).
- **Offline math validation** (no model, no ComfyUI): `python simulations/<name>.py`. These replay
  *measured* failure trajectories from live runs through the exact correction math to sanity-check a
  control law **before** paying for a live generation. They validate the math, never perception.
- **Paper**: `cd paper && latexmk -pdf clss_paper.tex` (auxiliary `.aux/.fls/.log/.fdb_latexmk` are
  committed build artifacts).
- **Inner repo** (`Ltx-2-CLSS/`) has ruff + pytest configured in its `pyproject.toml`; the node
  layer does not.

## Conventions specific to this codebase

- **Node inputs are experiment knobs, not user settings.** Most `optional` inputs on the sampler /
  Stage 2 / guiders exist because a specific failure was measured live; the tooltips record the
  evidence (e.g. "RMS +58% over 7 chunks"). Defaults are baked to the validated production config.
  Read the tooltip before changing a default.
- **Removing a failed experiment means deleting its input + code**, not defaulting it off. A knob's
  presence implies it is still a live lever.
- **Latent metrics (cosine sims, RMS, band energies logged per chunk) measure structure only.** They
  localize failures; they never prove a quality win. The user's eyes/ears on a live decode are the
  only ground truth.
- **The denoising/generation path is high-risk.** Never ship a change to the chunk loop, noise
  construction, or correction math without a user-validated live run. Noise edits are only seed-safe
  if they preserve the exact N(0,1) marginal (that is the design constraint behind
  `noise_temporal_corr` and the audio-noise simulations).
- `t2v_logs.txt` / `i2v_logs.txt` in `paper/` are captured per-chunk telemetry from real runs, used
  to calibrate the simulations and corrections.
