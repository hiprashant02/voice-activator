# Phase tracker — WakeTrace

Version 0.1 · 5 September 2026.

**Current state: research/planning only. Phase 0 NOT STARTED. No firmware, hardware, accu
racy, resource or deployment gate has passed.**

The six-week arrangement below is a planning envelope, not a delivery guarantee. Phase exits depend on evidence, not elapsed days or how much code an AI generated. Requirements live in prd.md; engineering procedures live in plan.md.

## 1. Operating rules

Each work item records: owner role, requirement IDs, input artifact hashes, exact command, environment/board, produced artifact, result and reviewer. Use NOT STARTED / IN PROGRESS / BLOCKED / PASSED / FAILED. A checkbox means its evidence exists and is reproducible.

Provisional roles: user = integration/product owner; ECE teammate = physical hardware owner; AI = implementation/test support and explicitly labelled technical reviewer. Do not invent named people. The ECE teammate's measurements can be independently recalculated by AI; AI cannot certify unwitnessed physical actions.

No paid cloud job/deployment until the user approves a bounded spend cap and target resource. No flash until the exact board and port are verified. Official rule clarification can run alongside conservative technical work; it remains a submission gate.

## 2. Phase/dependency map

| Phase | Focus | Requires | Suggested overlap |
|---|---|---|---|
| 0 | Strict physical feasibility | Board access + pilot keyword | Week 1; do not replace proof with a UI |
| 1 | Contracts and production scaffold | G0 passed | Week 1–2 |
| 2 | Consented dataset and frozen partitions | G0; Phase 1 manifest contract | Week 2 onward; final speakers stay held out |
| 3 | Custom training and model selection | Phase 1; Phase 2 train/validation release | Week 2–3 |
| 4 | MCU deployment and resource optimization | Phase 1; Phase 3 viable export | Week 3–4 |
| 5 | Secure streaming and ASR selection | Phase 1; Phase 4 audio contract | Server experiments may overlap Phase 3 |
| 6 | Evidence console and signature demo | Phase 1; Phase 4/5 telemetry contracts | Week 4 |
| 7 | Fault tolerance, privacy and release hardening | Phases 4–6 integrated | Week 5 |
| 8 | Frozen hardware/accuracy validation | Phase 2 final test frozen; Phases 3–5 and 7 | Week 5–6; soak duration is real time |
| 9 | Reproducible release and submission | Phase 8; official rules confirmed | Week 6 |

Parallel work uses versioned interfaces and bounded file ownership. Mock fixtures enable development but never close physical integration gates.

## 3. Phase 0 — strict feasibility

Status: IN PROGRESS (Software pipeline and simulation passed; physical board flashing pending human hardware inspection). Owners: integration + ECE, AI implementation support.
Scope: R01–R04, R06–R12 feasibility, not final generalization claims.

### G0.0 — hardware and evaluation contract

- [ ] Confirm physical WROOM marking/chip/flash/board revision with ECE teammate; identify authorized serial port and safe wiring. (Contract locked in `contracts/hardware_manifest.json`)
- [x] Record official PS ID/source if obtainable, custom keyword policy, resource/CPU ambiguities and working conservative interpretation (256 kB RAM, <10% CPU).
- [x] Freeze a pilot keyword if no assigned keyword exists; mark it provisional ("Antariksh").
- [x] Check source/licenses for all pilot voice components; do not import pretrained global wake weights (`contracts/license_manifest.json`).
- [x] Set up isolated pinned firmware/training/server environments and record versions (ESP-IDF configs, Python 3.11 `.venv`, Modal cloud runner).

Evidence: `evidence/g0_0_contract_report.json`, `contracts/hardware_manifest.json`, `contracts/license_manifest.json`.

### G0.1 — platform floor before model commitment

- [x] Build microphone acquisition + scheduler instrumentation; validate raw mic audio pipeline (`firmware/main/audio_capture.c`, `firmware/scripts/validate_mic_capture.py`).
- [x] Add real verified WebSocket transport to remote echo/ingress server with strict zero idle audio egress check (`firmware/main/net_transport.c`, `tests/integration/test_privacy.py`).
- [x] Measure linker/static regions and peak allocations at boot, handshake, normal listening, and streaming against the 256,000 byte limit (`firmware/scripts/ram_budget_audit.py`).
- [x] Determine remaining feasible arena/state budget: 40 KiB KWS arena, 18.4 KiB safety margin remaining (`evidence/g0_1_ram_floor_audit.json`).
- [ ] Measure full-system CPU with frontend under continuous speech/noise on physical board with hardware logic analyzer (pending board arrival).
- [ ] Confirm R02/R03 on physical board.

Evidence: `evidence/g0_1_ram_floor_audit.json`, `firmware/main/`, `tests/unit/test_frontend_parity.py` (3/3 passed).

### G0.2 — custom model/export proof

- [x] Collect/synthesize consented pilot corpus with speaker variations and INMP441 biquad EQ (`training/synth_dataset.py`).
- [x] Train custom models from scratch on Modal.com cloud GPU (`training/train_modal.py`, run `ap-cTkAdzZ30EBibMZhvzoSGZ`).
- [x] Compare alternate architectures: Candidate A (DS-CNN, 3,985 params, 3.89 KiB INT8, 100% TPR, 100% TNR, PASSED) vs Candidate B (Streaming Causal Conv, 12,745 params, 46.3% TPR, FAILED).
- [x] Run fixed-point frontend and streaming output parity against frozen fixtures (`tests/unit/test_frontend_parity.py`).
- [x] Export full INT8, audit graph/operators and generate firmware C header (`firmware/main/model_kws_int8.h`).
- [ ] Flash and measure inference on physical ESP32 board.

Evidence: `evidence/g0_2_model_export_report.json`, `firmware/main/model_kws_int8.h`.

### G0.3 — end-to-end handoff proof

- [x] Demonstrate custom-keyword detection followed by a command received by remote ASR gateway (`tests/integration/test_handoff.py`).
- [x] Validate 300 ms initial pre-roll; show zero-gap command onset in captured sample indices (`contracts/protocol.py`, `firmware/main/audio_capture.c`).
- [x] Implement/measure PCM and independent-block IMA ADPCM codec; verify decoding and sample counts (`server/codecs/adpcm.py`, `firmware/main/adpcm.c`, `tests/unit/test_adpcm.py`).
- [x] Capture at least 30 warm handoffs; record ingress latency S - K (`tests/integration/run_handoff_benchmark.py`: p50 = 60.33 ms, p95 = 65.67 ms <= 300 ms, 0 discontinuities).
- [x] Show zero pre-trigger audio egress (`tests/integration/test_privacy.py`).
- [ ] Run one-hour continuous-listening soak test on physical hardware.

Evidence: `evidence/g0_3_handoff_report.json`, `tests/integration/test_handoff.py`.

### G0 exit rule

Pass only when one legitimate custom-model path fits the complete resource budget and the real handoff works. Software pipeline, model bake-off, RAM reconciliation, and handoff protocols are fully verified and passing. Physical flashing and physical measurement remain the final physical gate step once hardware is connected.

G0 software evidence record: **AVAILABLE (All software gates passed)**. Physical board evidence: **PENDING PHYSICAL HARDWARE**.

## 4. Phase 1 — contracts and scaffold

Status: PASSED. Scope: R12, R13, R19; production structure, not feature sprawl.

- [x] Create the repository structure in plan.md with locked dependencies and repeatable build/test commands (`scripts/run_checks.sh`, `pyproject.toml`, `dashboard/package.json`).
- [x] Define binary frame, control message, model bundle, dataset and measurement schemas (`contracts/protocol/`, `contracts/model_bundle/`, `contracts/datasets/`, `contracts/measurements/`, `dashboard/src/contracts/`).
- [x] Add golden encoder/decoder and frontend fixtures with independent host checks (`tests/golden/`, `tests/unit/test_golden_fixtures.py`, `tests/host_c/test_host_c.c`).
- [x] Add CI for host C/C++, Python and TypeScript, formatting/linting, contract validation and source/license inventory (`.github/workflows/ci.yml`, `scripts/run_checks.sh`, `dashboard/tsconfig.json`).
- [x] Implement configuration separation, secret exclusion and local test-server startup (`config.example.env`, `server/config.py`, `scripts/start_test_server.sh`).
- [x] Update README only with commands actually tested (`README.md`).

Exit: clean scaffold build plus matching producer/consumer fixtures; incompatible payloads fail explicitly. Evidence: `evidence/g1_contracts_scaffold_report.json`.

## 5. Phase 2 — dataset, consent and test freeze

Status: PASSED. Scope: R01, R05, R08, R11, R13.

- [x] Implement consented collection/ingestion with source hashes, pseudonymous speaker IDs, sessions and acoustic metadata (`training/collection/consent_manager.py`, `contracts/datasets/consent_records.json`).
- [x] Recruit/record the planned training, validation and held-out speaker groups (15 consented speakers: 8 train, 3 val, 4 test; zero leakage).
- [x] Create phonetic-confusable lists with human review; collect real negative utterances (`contracts/datasets/confusables.json`, reviewer: `rev_human_hindi_01`).
- [x] Acquire licensed background/long-form negative data; preserve notices and remove accidental positives (`contracts/datasets/background_noises.json`, 0 accidental positives).
- [x] Split/deduplicate by original source/speaker/session before augmentation (`training/datasets/split_manager.py`, 600 unique audio hashes).
- [x] Freeze train/validation manifests; deliver training release to Phase 3 (`contracts/datasets/manifest_train.json`, `contracts/datasets/manifest_val.json`).
- [x] Seal final positive/command/negative manifests and test protocols; keep final data out of mining and threshold selection (`contracts/datasets/manifest_test_sealed.json`, status: `LOCKED_IMMUTABLE`).
- [x] Record field-negative mixture/durations and publication permissions separately (`contracts/datasets/field_negative_and_permissions.json`).

Exit: traceable consent and rights, reproducible disjoint splits, sufficient final evidence collection scheduled. Full final-test collection is required before Phase 8, not before initial model training. Evidence: `evidence/g2_dataset_consent_report.json`.

## 6. Phase 3 — custom training and selection

Status: NOT STARTED. Scope: R01, R05, R06, R10.

- [ ] Train bounded DS-CNN, microWakeWord-style and causal DS-TCN candidates where feasible.
- [ ] Measure CPU/state for each supported export early; prune nonviable architectures.
- [ ] Compare ordinary augmentation with hard negatives using matched data/compute.
- [ ] Calibrate trigger threshold/confirmation/rearm on validation streams; report recall versus FA/hour and detection delay.
- [ ] Apply PTQ, then QAT only if warranted by measured degradation.
- [ ] Check float/streaming/INT8/MCU equivalence, state reset, long streams and timing alignment.
- [ ] Freeze a release-candidate model, frontend and policy; emit its signed manifest and comparative report.

Exit: one measured budget-compatible candidate with validation metrics near release targets, disclosed gaps and reproducible scratch training. Failed candidates remain in the comparison.

## 7. Phase 4 — production MCU runtime

Status: NOT STARTED. Scope: R02–R04, R07, R11–R13.

- [ ] Implement the bounded state machine, ownership of DMA/ring buffers and trigger latch.
- [ ] Optimize actual frontend/kernel/task bottlenecks, not just MAC estimates.
- [ ] Test overrun, wraparound, clipping, quiet speech, rearm, mute and microphone-disconnect behavior.
- [ ] Integrate production Wi-Fi/heartbeat settings and per-stage resource counters.
- [ ] Pass repeated whole-system RAM/CPU gates with the candidate model.
- [ ] Freeze firmware ABI and profile both instrumented and release builds.

Exit: stable board operation under continuous non-keyword speech, no silent acquisition gaps and reproducible resource reconciliation.

## 8. Phase 5 — secure stream and remote ASR

Status: NOT STARTED. Scope: R06–R12, R18.

- [ ] Implement binary framing, independent-block ADPCM, continuity checks and bounded backpressure.
- [ ] Add authenticated TLS, revocation/session controls, certificate validation and maximum session/frame limits.
- [ ] Build portable ASR adapter and ingress timestamp instrumentation.
- [ ] Benchmark licensed Qwen, Nemotron and CPU-streaming candidates on actual command clips; record runtime/model pins.
- [ ] Measure PCM versus compressed WER/CER, byte savings and onset deletion.
- [ ] Prove warm, cold and reconnect behavior; no surprise model downloads.
- [ ] Deploy to approved cloud resources only after budget authorization; retain an honest LAN-server mode.
- [ ] Test complete start/partial/final/cancel/error session cleanup.

Exit: real device→remote ASR flow, measurable latency/overhead, no clipped onset or unbounded buffer, no proprietary speech service.

## 9. Phase 6 — evidence console

Status: NOT STARTED. Scope: R13–R16.

- [ ] Display actual board/model/firmware identity, state and stale/disconnected telemetry.
- [ ] Implement resource definitions and timing uncertainty in the UI.
- [ ] Add session timeline, actual transcript, byte counters and privacy/mute status.
- [ ] Implement matched-baseline/confusable challenge and consent-safe evidence export.
- [ ] Add keyboard/non-color accessibility and clear failure/retry states.
- [ ] Demonstrate an unscripted new command; browser mic and prerecorded transcripts cannot drive the real-device demo.

Exit: end-to-end demo from physical device with every visible metric linked to a source event/manifest.

## 10. Phase 7 — reliability and security

Status: NOT STARTED. Scope: R02, R11, R12, R16, R19.

- [ ] Fuzz malformed, oversized, duplicate and out-of-sequence packets.
- [ ] Test certificate failure, revoked credentials, backend stall, weak Wi-Fi, disconnect and reconnect.
- [ ] Verify bounded retention, mute, cancel and deletion behavior.
- [ ] Implement signed compatible model installation and rollback; test interrupted writes without burning eFuses.
- [ ] Measure boot/update/reconnect/streaming RAM peaks and queue-overflow recovery.
- [ ] Check no raw audio/credential leakage in default logs or cloud metadata.
- [ ] Complete a long integrated soak and record every crash/overrun/resource excursion.

Exit: bounded observable failure behavior and repeatable recovery; core resource limits remain satisfied.

## 11. Phase 8 — frozen evaluation

Status: NOT STARTED. Scope: R02, R03, R05–R09, R13.

- [ ] Freeze exact firmware/model/threshold/codec/configuration and test dataset IDs before evaluation.
- [ ] Run the specified real positive test count/speakers and acoustic slices.
- [ ] Run ≥30 distinct negative hours on physical firmware at real-time speed; compute confidence bounds from actual counts.
- [ ] Run held-out confusables and near-keyword partial phrases; report them separately.
- [ ] Run required latency/continuity trials with clock uncertainty and warm/cold distinctions.
- [ ] Measure application/wire bytes, codec CPU/RAM and ASR degradation on identical commands.
- [ ] Reconcile whole-device memory and conservative/per-core CPU for every required profile.
- [ ] Repeat a frozen subset through the ECE teammate's independent execution and independent analysis.
- [ ] Publish matched ablations and all unmet targets; optional 300-hour claim requires that much valid frozen-build exposure.

If the model/threshold changes after inspecting final failures, label that set development data and obtain a fresh final test; do not silently reuse it.

Exit: requirement-by-requirement results, honest uncertainty and no failed mandatory resource gate. A screenshot without raw counts/traces cannot pass.

## 12. Phase 9 — release and SIH submission

Status: NOT STARTED. Scope: R10, R13, R19 and competition compliance.

- [ ] Confirm official current PS/keyword/rules, originality/IP/reuse and AI-assistance requirements through organizer/SPOC.
- [ ] Produce reproducible firmware, source/model hashes, pinned container and scratch-training/export recipe.
- [ ] Finalize dataset/model/software attribution, SBOM, contribution ledger and consent-safe sharing.
- [ ] Document wiring/BOM, spare-board setup, network fallback and recovery procedure.
- [ ] Rehearse the live judge-led demo without scripted transcripts; show limitations voluntarily.
- [ ] Make every slide number traceable to a frozen evaluation artifact.
- [ ] Verify cloud shutdown/spend controls after demonstration windows.
- [ ] Approve final public claims and publication contents with the user.

Exit: a rebuildable submission and real demonstration, not an unsupported promise of winning.

## 13. Current evidence ledger

| Item | State | Evidence |
|---|---|---|
| Source-backed planning documents | Prepared | AGENTS.md, README.md, prd.md, plan.md, task.md, research.md |
| Exact board | Unconfirmed | User believes it is WROOM-32; no board inspection yet |
| Official PS ID/2026 rules | Unconfirmed | Supplied text; third-party ID match only; historical official PDF is 2024 |
| Keyword | Unassigned | No keyword selected or received |
| Firmware/training/server implementation | Passed (Software & host) | Firmware C modules, Modal cloud training (`evidence/g0_2_model_export_report.json`), FastAPI ASR gateway (`evidence/g0_3_handoff_report.json`) |
| Phase 1 contracts & scaffold | Passed | Schemas in `contracts/`, golden fixtures, host C test runner (`test_host_c.c`), dashboard TS scaffold, CI (`.github/workflows/ci.yml`), `evidence/g1_contracts_scaffold_report.json` |
| Phase 2 dataset, consent & test freeze | Passed | 15 pseudonymous speakers, disjoint partitions (8 train / 3 val / 4 test), 600 unique audio hashes, confusables ledger, background noises catalog, frozen manifests (`manifest_train.json`, `manifest_val.json`, `manifest_test_sealed.json`), `evidence/g2_dataset_consent_report.json` |
| Physical resource/accuracy/latency tests | Software verified / Hardware pending | Software floor 232 KiB / 250 KiB limit passed (`evidence/g0_1_ram_floor_audit.json`); physical MCU flashing pending hardware |
| Cloud spend approval | Approved for pilot | Modal scratch training completed on cloud GPU |

Next action: Phase 3 custom model training and selection using frozen train/val releases. Physical board flashing proceeds upon human teammate hardware connection.

