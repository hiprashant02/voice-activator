# AGENTS.md

## Scope and mission

These instructions own this voice-activator subtree. The parent ocean project's mission, architecture and no-training rule do not apply here. Preserve all files outside this subtree.

Build WakeTrace (working product name, not the wake word): a custom-trained MCU keyword detector followed by prompt, bandwidth-efficient streaming to an open-source remote ASR server. The user's provisional hardware is ESP32-WROOM-32 plus INMP441; verify the board before pin assignments or flashing.

## Read first

Read prd.md, plan.md, task.md and research.md completely before starting a phase. README.md owns usable commands; do not present planned commands as existing software. prd.md owns requirements; plan.md owns designs; task.md owns progress/evidence; research.md owns sources and originality boundaries.

## Non-negotiables

- Train the edge keyword model on the team's custom keyword. Do not deploy pretrained global-assistant keyword weights or proprietary voice SDKs. Training is required for this project.
- Respect prd.md resource definitions: total edge RAM, not just model tensors; continuous-listening CPU, not silent-room inference time. No external PSRAM or off-device KWS loopholes.
- A proposed model, arithmetic estimate, desktop simulation or paper benchmark cannot pass a physical-device gate.
- No pre-activation audio egress. A bounded local pre-roll may be sent only after activation, with the privacy behavior declared.
- Do not drop first-command samples silently, fabricate transcripts, hide false wakes, or use unsynchronized clocks for latency claims.
- Keep neural weights, runtime code and dataset licenses separate in the inventory. Cloud infrastructure credits do not authorize a closed-source voice service.
- Freeze speaker/source-disjoint validation and test sets before augmentation. Never tune thresholds or mine negatives on the final test set.
- Real board measurements, microphone checks and consent collection require human action. AI can implement, inspect logs and reproduce calculations; it cannot attest to a physical test it did not observe.
- Do not burn security eFuses, erase user hardware, publish recordings, spend cloud credits or deploy a public service without corresponding user authorization.
- No winning, uniqueness, zero-false-wake or universal multilingual claims unsupported by evidence.

## Work method

1. Select an eligible item in task.md; state its requirement ID, dependency and file scope.
2. Inspect contracts/tests before editing. Keep C/C++ bounds explicit and Python/TypeScript typed.
3. Implement one coherent change and its normal, failure and resource tests.
4. Run narrow checks, then the applicable gate. On unfamiliar toolchain failures, inspect current primary documentation before retrying.
5. Record commands, versions, hashes, measured hardware/configuration and artifact paths in task.md. Check boxes only with evidence.
6. For ML changes compare float, INT8 host, streaming host and MCU outputs with declared tolerances; measure the full firmware again.
7. Request only the specific physical action needed from the ECE teammate; do all available software preparation first.

Use explicit states: NOT STARTED, IN PROGRESS, BLOCKED, PASSED, FAILED. Failed requirements remain visible. Technical review may be a clean-context AI reproduction, labelled as such; physical and participant evidence must come from people.

## Architecture boundaries

- firmware/: ESP-IDF, I2S/DMA, fixed-point frontend, TFLM/ESP-NN, trigger state machine, bounded stream client, resource instrumentation.
- training/: consent manifests, dataset splits, scratch training, calibration, INT8 export, model cards.
- server/: portable ASR gateway, model adapters, session lifecycle, evaluation ingress timestamps.
- dashboard/: browser evidence console; never the hidden microphone/KWS source in the real demo.
- contracts/: versioned protocol, model bundle and measurement schemas.
- tests/: host unit/contract tests, golden signals, hardware runner, accuracy/latency/resource analysis.
- infra/: Modal GPU containers for ASR; Cloudflare Workers/D1/R2 for dashboard and metadata only; no live audio through Cloudflare. No embedded credentials. LAN fallback adapters preserved.

Keep only these six hand-authored planning Markdown files unless a concrete need is approved. Machine-readable evidence, test fixtures and release-generated attribution/SBOM files are appropriate additional artifacts.

