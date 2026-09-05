# Product requirements — WakeTrace

Version 0.1 · 5 September 2026 · planning contract, not measured results.

## 1. Problem and authority

The supplied PS requires custom keyword spotting on a low-power device followed by efficient streaming of subsequent audio to remote ASR. The primary evaluation dimensions are RAM/flash footprint, idle-listening CPU, true-positive/false-activation performance and keyword-end-to-ASR-ingress latency. Voice software must be open source; custom keyword training is mandatory; proprietary voice-activation SDKs and heavy edge transformers are disallowed.

The pasted statement is the working authority. A matching third-party listing suggests SIH26172; official ID, category, assigned keyword and 2026 rules remain unconfirmed. The official catalogue was not retrievable during this research. Do not present a mirror or the 2024 guidelines as verified 2026 rules. See research.md §1.

Target hardware is provisionally ESP32-WROOM-32 with INMP441. No design may quietly require an S3, Raspberry Pi Linux, extra microphone or external PSRAM. A different board requires explicit agreement and a new resource report.

## 2. Product thesis

Deliver a deployable voice front end whose small resource use and wake-to-command continuity are independently testable. Target Indian-accented English and Hindi command transcription, with Hinglish tested separately rather than inferred from bilingual support.

The product is not a general assistant. It consists of the MCU listener, custom model training/export pipeline, remote ASR stream, evidence console and repeatable evaluation kit.

## 3. Requirement and acceptance register

All numeric targets below, other than the supplied RAM/CPU restrictions, are team design objectives. They are not official numerical promises or already achieved results.

| ID | Priority | Requirement | Acceptance evidence |
|---|---|---|---|
| R01 | Mandatory | Custom-keyword edge model, trained by team | Dataset rights/split manifests, training configuration/logs, scratch initialization evidence, float and INT8 artifact hashes; no prohibited keyword weights |
| R02 | Mandatory | Total edge RAM strictly below the PS limit | Conservative gate: peak attributable occupied/reserved edge RAM <256,000 bytes across boot, listening, TLS reconnect and active streaming; report DRAM, IRAM, stacks, DMA, heap peaks and allocator overhead without double-counting. No PSRAM |
| R03 | Mandatory | Idle continuous-listening CPU <10% | Whole-system measurement on physical board under quiet, noise and uninterrupted non-keyword speech; conservative sum of busy time across cores divided by wall time <10% in each predeclared 60-second steady-listening window. Also publish per-core and conventional dual-core-normalized numbers |
| R04 | Mandatory | Autonomous always-listening microphone pipeline | Real INMP441 input; no button required to activate, no laptop KWS, no browser microphone substituted; zero unnoticed audio/DMA overruns in the soak test |
| R05 | Mandatory | High recall with rare false activation | Frozen-threshold evaluation: target TPR ≥95% in clean close-field and ≥90% in specified noisy close-field conditions, with speaker-level intervals; false-activation upper bound ≤0.1/hour on the defined negative distribution |
| R06 | Mandatory | Prompt delivery of subsequent audio | Target warm WAN p95 ≤250 ms from annotated acoustic keyword end to the ASR gateway accepting a valid frame containing post-keyword samples, under the network profile below. Publish detection, transport and ingress separately |
| R07 | Mandatory | Preserve the command onset | Zero missing/duplicated samples from the annotated command start in 100 instrumented handoffs, including zero-gap keyword+command speech; no hidden clipping masked by a plausible transcript |
| R08 | Mandatory | Remote ASR of real live commands | Actual backend model ID/revision shown; transcript changes with unseen commands; measure WER/CER and first stable-token latency; never substitute canned results |
| R09 | Mandatory | Reduced data overhead | Binary audio, no base64/JSON samples. PCM baseline plus one measured compression candidate; target ≥60% application-byte reduction vs PCM with ≤2 percentage-point absolute WER degradation on the same frozen command set |
| R10 | Mandatory | Open-source voice path | Inventory every frontend/KWS/codec/ASR component and model license; code availability and custom training verified; incompatible or unknown components do not ship |
| R11 | Mandatory | Controlled privacy boundary | No audio content sent before activation, including near-match scores used as a covert audio channel; bounded post-trigger pre-roll disclosed; remote audio retention off by default; physical/software mute prevents audio transmission |
| R12 | Mandatory | Bounded failures | Disconnect, stalled socket, invalid packet, low memory and backend unavailable produce visible errors; finite buffers and session deadlines; no crash, silent overwrites or unsafe command execution |
| R13 | Mandatory | Reproducible evidence | Firmware/model/config/data split hashes, board and clocks, model arena, full resource capture, timing uncertainty and test duration/counts exported together |
| R14 | Differentiator | WakeTrace evidence console | Live board-sourced resource/timing/byte counters and state, calibrated timestamps, privacy status and model identity; stale/disconnected indication; accessible non-color labels |
| R15 | Differentiator | Confusable-phrase challenge | Human-vetted near-keyword phrases and unseen speakers; show fixed-threshold results against matched baselines, including failures |
| R16 | Differentiator | Safe signed model bundle deployment | Versioned feature/quantization/runtime contract, signature verification, last-known-good rollback and reboot/self-test; no unbudgeted double-model residency |
| R17 | Stretch | Lower-bandwidth Opus and energy accounting | Only after core gates; benchmark complete codec RAM/CPU/WER; actual power measurements, not CPU-to-battery-life extrapolation |
| R18 | Stretch | Proven multilingual/code-switch capability | Per-language and Hinglish test sets with normalization rules, not a language-count marketing claim |
| R19 | Mandatory | Rebuild and integration handoff | Clean firmware build, pinned dependencies, server container, wiring/BOM evidence, release SBOM/notices and a documented board-replacement procedure |

### Metric profiles

- Clean close-field: 0.5–1 m, stated microphone orientation, unclipped normal speech in a documented quiet room.
- Noisy close-field: 1 m, measured +10 dB speech-to-noise ratio using a declared measurement procedure, with fan/cafeteria/competing speech represented. +5 dB, 0 dB, reverberant rooms and 3 m are stress slices, not silently included in a favorable average.
- Use at least 20 held-out speakers and 1,000 real positive trials across slices for the final report; allocate counts before testing. Test duration and count are separate: overlapping positives do not create independent speakers.
- Negative test: ≥30 distinct hours of conversational speech, confusables, music/TV-like consented/licensed content, room noise and silence. Report each class and the mixture. Synthetic concatenation alone is not a deployment guarantee.
- With zero events, a one-sided 95% Poisson upper bound is approximately 2.996/T hours. Thus 30 hours supports about 0.1/hour, not zero; a 0.01/hour stretch requires roughly 300 zero-event hours under the same assumptions. If events occur, compute the exact count-based bound and report it. Correlated/deployment-shift limitations remain.
- CPU: fixed recorded frequency, production radio/heartbeat policy, actual frontend and scheduler. Listening to continuous non-keyword speech must not be excluded as “active mode.” Boot, reconnect and command-stream CPU are additionally reported, not hidden inside the idle average.
- Warm WAN: associated Wi-Fi, valid established TLS/WebSocket, ready ASR instance; controlled median RTT ≤50 ms, p95 RTT ≤80 ms, packet loss <0.5%, uplink ≥1 Mbps, with actual measurements. Also test cold session, radio reconnect and impaired network separately. This is a test profile, not a promise about venue Internet.
- Latency trials: ≥100 warm trials plus ≥30 cold/reconnect trials, raw events and distribution. Do not add individual p95 values to claim an end-to-end p95.
- ASR design targets on the consented command set: WER ≤15% per selected language in clean conditions; first stable partial target p95 ≤1 s from command onset when enough speech exists. Label backend chunking limitations; core PS latency is audio ingress, not transcript availability.
- Flash: no supplied hard limit; target model file ≤64 KiB and firmware fitting the verified board partition layout. Publish actual model/firmware sizes. Do not count flash size as RAM.

## 4. User experience

Listener states: boot/self-test, listening-online, listening-offline, keyword-candidate, streaming, finalizing, cooldown, muted and fault. Candidate state never transmits audio. Rearm after a bounded documented interval and score reset, not minutes of suppression that artificially lowers false activations.

Display recording/transmission clearly. Physical mute is real hardware if fitted; otherwise label software mute accurately. No speaker/echo cancellation or beamforming claim for one microphone.

The console has four views: live session, resource budget, accuracy/negative-test results, and reproducibility manifest. Show unavailable metrics as unavailable; a stale cached value cannot look live.

## 5. Three-minute non-hardcoded demo

1. A judge chooses a confusable phrase: device keeps listening; console shows actual local state and no audio uplink.
2. An unseen speaker says the custom keyword immediately followed by an arbitrary English/Hindi command. The LED/state changes, real transcript appears, and the handoff trace identifies preserved command onset.
3. Toggle PCM versus the validated compressed mode between sessions; show measured bytes and output quality.
4. Interrupt the network: local wake still works, cloud transcription visibly becomes unavailable. Reconnect and perform a fresh live command. Optional LAN ASR fallback must be labelled LAN, not cloud.
5. Export the evidence record and open its matching firmware/model/configuration hashes and test results.
6. (If validated) Show the noise challenge: play a declared noise source, report zero or low false activations over a timed window, and display actual threshold/score behavior. Label the comparison baseline accurately as non-ML sound detection.
7. (If validated) Show the latency waterfall with per-stage timing and stated measurement uncertainty.
8. (If validated) Demonstrate OTA model swap: deploy a pre-trained alternative model to the device in ~30 seconds via signed bundle update, with automatic rollback on contract mismatch. Live retraining from scratch takes ~1 hour and is a non-goal (§6); show the deployment mechanism, not instant learning.

A telemetry privacy proof alone does not prove all firmware behavior: support it with packet capture, code review and instrumented server logs.

## 6. Non-goals

No general-purpose chatbot, cloud wake verification, biometric authentication, anti-replay guarantee, medical/safety-critical actuation, on-device model training, real-time arbitrary wake-word learning from three utterances, or heavy edge transformer. An optional LED demonstration may consume recognized commands; no unattended mains switching.

No universal “zero false activations,” “works in every accent,” “world first” or guaranteed SIH victory.

## 7. Open decisions and owners

The user is product/integration owner; their ECE teammate owns physical wiring and measurement execution. Names beyond those roles are not invented. AI prepares implementation/tests and analyzes evidence.

Confirm official PS ID, keyword assignment timing, RAM accounting interpretation, CPU denominator and official evaluation board/clock. Continue engineering under the stricter definitions above while requesting clarification. Questions do not authorize weakening a failed gate. Confirm consent, cloud budget and official 2026 originality/IP requirements before public submission.

