# Engineering plan — WakeTrace

Version 0.1 · 5 September 2026. Proposed design, not benchmark results. Requirements and numerical acceptance belong to prd.md; sources and comparative evidence belong to research.md.

## 1. Decisions and rationale

| Area | Provisional choice | Reason / reversal condition |
|---|---|---|
| Board | Existing ESP32-WROOM-32, internal memory only | User's likely hardware; verify marking, flash, clock and usable pins. S3 results cannot pass this board's gates |
| Firmware | Minimal ESP-IDF C/C++, FreeRTOS, I2S DMA | Direct control of buffering, allocations and task timing; no full home-automation framework requirement |
| KWS runtime | esp-tflite-micro + actual classic-ESP32 ESP-NN kernels | Source-available inference path; freeze after operator/quantization compatibility tests |
| KWS model | Winner of DS-CNN / microWakeWord-style / narrow causal DS-TCN bake-off | Decide by false-wake/recall/latency at the whole-system budget, not model name |
| Features | Fixed-point microfrontend; start 16 kHz, 30 ms window, 10 ms hop, 40 channels | Train/firmware parity; compare log-mel and frontend normalization variants using the identical C implementation |
| Transport | One persistent authenticated TLS WebSocket; binary PCM baseline, IMA ADPCM challenger | No codec work in normal listening; small frames and bounded queues. No audio before activation |
| ASR | Portable server adapter, Qwen3-ASR-0.6B GPU candidate, Nemotron 3.5 challenger, sherpa-onnx English CPU baseline | Source/licensed weights plus measured streaming behavior; no proprietary speech API |
| Training | Scratch keyword training in locked Linux environment; GPU optional on Modal | Custom training is required. A small model does not require training a foundation model |
| UI | React, TypeScript, Vite; small SVG/Canvas timelines and charts | Show real evidence without consuming MCU RAM on graphics |
| Hosting | Portable ASGI/container first; optional Modal deployment and Cloudflare dashboard | No vendor dependency in the voice algorithm or protocol; spend/cold starts are explicit |
| Persistence | Local manifests/SQLite first; optional R2 objects and D1 metadata | Raw data/weights stay out of SQL blobs; participant audio private by default |

Start with a supported stable ESP-IDF branch compatible with the chosen esp-tflite-micro release; 5.5.x is a candidate, not an invented final pin. Phase 0 records exact SDK, compiler, component commits, sdkconfig, TensorFlow/Keras and converter versions. Training and ASR need separate environments.

The proposed use of pretrained ASR is server-only; research.md records the interpretation requiring organizer confirmation. No giant pretrained model is hidden on another local device to perform edge KWS.

## 2. System flow

```text
INMP441 → I2S/DMA → PCM16 ring → fixed-point frontend → streaming INT8 KWS
                    │                                  │
                    │                             local decision
                    │                                  │ confirmed wake only
                    └── bounded pre-roll + live audio ─┴→ encoder → TLS/WebSocket
                                                                      │
                                                         ASR ingress + decoder
                                                                      │
                                                         ASR stream → transcript
                                                                      │
                                                        evidence console/export

Training: consented recordings → frozen splits → augmentation → scratch training
          → INT8 export → host/MCU equivalence → hardware gate → signed bundle
```

The console observes this pipeline; it never secretly supplies the wake decision or microphone stream.

## 3. Hardware bring-up

ECE checklist: photograph module marking and board revision; identify flash size/partition requirements; verify supply, shared ground, mic decoupling, L/R selection, wire lengths and non-conflicting GPIOs. Use the manufacturer's electrical limits; do not wire microphone power to 5 V. Do not invent pin numbers before board inspection.

Capture I2S into DMA-backed fixed-size buffers. INMP441 supplies signed 24-bit samples; use a verified slot/channel configuration, correct sign extension and saturating conversion to PCM16. A 32-bit container does not mean 32-bit acoustic precision. Verify the one-bit I2S alignment and channel selection using known signals and captured waveforms.

Bring-up evidence: sample rate measured against duration, DC mean, clipping count, silence/noise spectrum, microphone frequency response sanity, speech audibility and channel orientation. Keep original raw captures for parity tests. Synthetic tones exercise software but cannot prove the mic's acoustic quality.

No speaker is required. Prefer LEDs for states. A physical mute switch can interrupt acquisition/power if electrically safe; software mute must not be advertised as a hardware privacy barrier. No beamforming, distance estimate or acoustic echo cancellation claim from a single microphone.

## 4. Resource envelope and accounting

### 4.1 RAM is a full-system gate

Interpret the ambiguous KB unit conservatively as 256,000 bytes. Count unique occupied/reserved physical RAM attributable to the firmware, including RAM-resident instructions, static data, task stacks, allocator metadata, DMA, Wi-Fi/LwIP, TLS handshake/session buffers, frontend scratch, model state/arena and transport queues. Report ROM/flash separately. External PSRAM is disabled.

Use the linker map plus instrumented heap allocation/high-water snapshots across phases. Do not sum overlapping capability pools such as DMA and 8-bit heaps twice. Conversely, total minus free heap alone omits static memory/IRAM. Publish a physical-region reconciliation that another person can reproduce. Account for memory reserved by vendor components and allocator fragmentation; report largest free block in addition to totals.

Initial **allocation hypothesis**, not a measured fit:

| Allocation owner | Envelope |
|---|---:|
| SDK/static RAM-resident code/data + RTOS/task stacks | 88 KiB |
| Wi-Fi/LwIP + TLS, including handshake peak | 64 KiB |
| KWS arena + persistent streaming state | 40 KiB |
| Frontend working memory | 12 KiB |
| I2S/DMA buffers | 6 KiB |
| Raw pre-roll ring | 12 KiB |
| Encoded/in-flight application queue | 8 KiB |
| Small metadata/counters | 2 KiB |
| Total hypothesis | 232 KiB = 237,568 bytes |
| Remaining to conservative limit | 18,432 bytes |

This may be optimistic, especially for SDK+secure-networking peaks. The first Phase 0 experiment measures the **networked platform floor before model selection**. Resize categories only from measured allocation lifetimes. A model arena must not be counted again if included in a category's traced heap.

Optimization order: remove unused Bluetooth/services; right-size stacks with guards; reduce unnecessary logging/JSON allocation; keep constant weights flash-mapped; use bounded radio/socket buffers; evaluate supported dynamic TLS buffer options. Preserve certificate and hostname verification. Test peer certificate/record behavior after every TLS buffer change. Moving frequently used code into IRAM trades RAM for speed and must be re-budgeted.

No TLS-off “production” fallback. If networking plus acquisition cannot leave a viable model envelope, report a failed WROOM path, try documented configurations and a smaller model; ask before changing hardware. The PS limit never becomes an arena-only target.

OTA/model installation is also measured. Use a maintenance state with listening explicitly unavailable; write in small flash chunks, validate then reboot. Do not hold old and new models in RAM simultaneously. No irreversible security eFuse changes in this project phase.

### 4.2 CPU and power

Measure acquisition, frontend, inference, decision logic, scheduler, interrupts, radio keepalive and telemetry. Record fixed CPU frequency and core affinity. Primary conservative CPU measure is total busy core-microseconds / elapsed microseconds, not divided by two. Publish per-core and conventional normalized utilization alongside it.

Use runtime statistics plus cycle/GPIO instrumentation with known ISR accounting. Calibrate instrumentation overhead and distinguish instrumented from release builds. Do not declare sleeping/deep-sleep CPU as compliant if continuous I2S sampling is actually stopped.

Design headroom target is ≤8% in listening windows. At a 10 ms hop this represents an average 0.8 ms of all-core work per hop for the complete listening system, not an inference-only allowance. Optimize measured hotspots first.

Run quiet, continuous non-keyword speech, noisy speech, music-like content, weak-radio and reconnect tests. Silence/VAD gating may be evaluated later, but the selected model must meet the listening budget under continuous speech without relying on skipped processing. Reconnect CPU and missed-listening time are separately visible.

CPU utilization is not energy consumption. If a USB power meter/current monitor is available, report device-level average/current peaks with radio state and LED load; battery-life claims require a power model and real measurements.

## 5. Frontend and model bake-off

### 5.1 Audio contract

Initial signal: mono PCM16 little-endian, 16,000 samples/s. Feature window 480 samples, hop 160 samples; fixed channel count and frequency limits. Freeze DC handling, scaling, mel weights, log/PCAN/noise-suppression choice, numeric rounding and quantization.

Generate training features through the same C frontend or an equivalence-tested binding, not an approximately similar Python mel implementation. Add golden tests for silence, impulse, tones, clipping, gain shifts and real speech. Publish absolute/relative and quantized tolerances; do not silently rescale microphone input to match a pretrained convention.

### 5.2 Candidate ladder

- A: very small DS-CNN scratch baseline; measure repeated-window compute explicitly.
- B: microWakeWord-style streaming model trained from scratch on our corpus; use source with attribution, verify all resource-variable allocations.
- C: narrow causal depthwise-separable temporal convolution (DS-TCN) with cached state. Width and receptive field are hardware-search variables.
- D, optional only after A–C: BCResNet or linearized/TENet-like topology where actual exported kernels support the operators efficiently.

Start with a maximum of three viable models and two widths each; prune candidates with incompatible ops or failing resource floor before long training. This is a bounded experiment, not a generic neural-architecture-search platform.

Illustrative C network: 40 input features → pointwise projection width C → six causal blocks with depthwise temporal kernel 5 and dilations 1,2,4,8,16,32 → per-frame keyword logit. Each block has pointwise channel mixing and bounded activation; fold eligible normalization at export. A current-frame head avoids an unbounded history pool.

Arithmetic estimate, omitting biases, requantization, nonlinearities, indexing and runtime overhead:

```text
MACs per 10 ms step = 40*C + 6*(5*C + C*C) + C
width 16: 2,672 MAC/step ≈ 0.2672 million MAC/s
width 24: 5,160 MAC/step ≈ 0.5160 million MAC/s
width 32: 8,416 MAC/step ≈ 0.8416 million MAC/s

int8 temporal cache = C*(5-1)*(1+2+4+8+16+32)
width 16: 4,032 bytes; width 32: 8,064 bytes
receptive field ≈ 30 ms + 4*63*10 ms = 2,550 ms
```

These are architectural calculations, **not a TFLite file size, total arena, CPU measurement or trained accuracy result**. Long causal history does not impose 2.55 s of post-keyword delay, but boot/reset warmup must be declared. Dilated kernels/state operations may fall back to slow reference code; that can make C lose to B despite attractive MACs.

Prefer explicit bounded state tensors to opaque growing histories. Ring-buffer/custom kernels are allowed only after bit/tolerance-equivalence tests and measured benefit. If C needs substantial unsupported operators, use a smaller supported streaming model; do not build an entire new inference runtime.

### 5.3 Training objective and trigger policy

Train on streaming sequences with random keyword alignment and surrounding speech. Labels must reward a completed keyword, not a prefix; include word truncations and phonetic lookalikes as negatives. Mark acoustic keyword boundaries in a subset for timing supervision/evaluation. Avoid feeding future audio into a causal label target.

Start with binary cross-entropy and balanced sampling, then calibrate on realistic negative prevalence. Hard-negative mining uses training/validation pools only. Class probability is a score, not a calibrated confidence certificate.

Trigger policy sweeps threshold, short consecutive-hit/rolling-average confirmation and hysteresis. Choose the lowest-delay policy satisfying validation false-wake objectives. Record cooldown and time when detection is inhibited; do not game false-wake rates by suppressing listening for long periods.

### 5.4 Quantization and export

Train float from scratch, fold eligible layers, perform full INT8 PTQ with representative **streaming state and microphone data**, then evaluate. Reject unexpected floating-point fallback operators. Record per-tensor scales/zero-points, feature version, operator resolver and RAM requirement.

If PTQ loses >1 percentage point TPR at the same validation FA/hour operating point, try QAT; threshold recalibration alone cannot hide a severe quantization loss. Optional distillation uses a scratch-trained larger teacher on the same allowed data, with a measured ablation. No global keyword model as a hidden teacher.

Compare full-sequence float, streaming float, host INT8 and MCU INT8 output traces. Check state initialization, stride alignment, ring wrap, reset, long runs and chunk boundaries. Export the actual arena and graph audit with each model.

### 5.5 Knowledge distillation (optional, measured ablation required)

If adopted, distillation uses a scratch-trained larger teacher on the same allowed custom-keyword data. No global-keyword pretrained model serves as a hidden teacher.

Proposed teacher: BC-ResNet-8 (Kim et al. 2021), approximately 300K parameters, approximately 1.2 MB float32. Train from scratch on the full augmented corpus with cross-entropy loss. Optional addition: Sub-Center ArcFace loss (EdgeSpot, 2026) for structured embedding geometry, applied to the teacher model only.

Distillation loss for the selected student (candidate A, B or C):

```text
L_total = α × CE(y, ŷ_student) + (1-α) × KL(softmax(z_teacher/T), softmax(z_student/T))
α = 0.3 (design target, sweep 0.1–0.5)
T = 5.0 (temperature, sweep 3–8)
```

Required ablation: (a) student accuracy with and without distillation under matched compute, (b) no accuracy regression at the selected INT8 operating point, (c) teacher trained from scratch on our data. BC-ResNet-8 accuracy on GSCv2 (paper-reported 98.7%) is the teacher's potential ceiling, not our system's expected custom-keyword accuracy.

SpecAugment (Park et al. 2019) is adopted as spectrogram-domain augmentation during training: randomly mask f consecutive mel bins and t consecutive time frames. Starting parameters f=8, t=10; tune on validation. Zero inference cost.

## 6. Dataset and experimental design

### 6.1 Keyword selection

If organizers assign a keyword, use it exactly; the system must not substitute an easier phrase. If selectable, shortlist pronounceable multi-syllable candidates and test common-speech frequency, phonetic neighbors and local-language ambiguity. Working product name WakeTrace is not the selected keyword.

A retraining recipe supports later reassignment; an arbitrary new keyword does not become reliable instantly. If assignment occurs late, collect a smaller pilot and report its narrower evidence honestly.

### 6.2 Collection plan

Available speakers: 15 consenting speakers (team + friends), 20–30 positives each, multiple phrases, actual INMP441 recordings and a small real negative pool.

Split: 8 train / 3 validation / 4 final test. With TTS augmentation (Synth4Kws pipeline, §6.3), 200 real training positives + 2,000 synthetic positives provide sufficient diversity. Validation: ~75 real positives. Final test: ~100 real positives measured exclusively on real human recordings, never TTS. The synthetic pipeline (Indic Parler-TTS, 50+ speaker prompts) covers accent, gender and age diversity that would otherwise require 50+ real speakers. Recruit additional speakers if available, but do not block on reaching a target number. Do not claim statistical representation of all India.

Keep speaker identity, session, original recording and augmentation parents grouped. Split before augmentation. Negative books/speakers/rooms/original files must also be disjoint. Deduplicate by content hashes and acoustic checks. Freeze final test IDs so a developer cannot choose favorable clips.

Command ASR set: initially 300 English and 300 Hindi consented utterances; optional 100 Hinglish utterances. Include names/numbers and unpredictable first words; define transcript normalization and CER/WER handling. Do not use the same speaker's training recording re-encoded as a “new” test.

Consent metadata states training/evaluation use, optional publication, retention and deletion contact. Store pseudonymous speaker IDs, not personal names in model artifacts. Do not publish participant audio automatically.

### 6.3 Augmentation and hard negatives

Combine gain, background noise at sampled SNR, modest speed variation, room impulse responses and bounded feature masking. Match real mic response; avoid destroying intelligibility or changing the keyword's identity. Keep final acoustic test recordings unaugmented except separately labelled stress experiments.

Build confusables through phoneme/grapheme edits, word-boundary splits and partial prefixes; native speakers vet whether a phrase actually lacks the keyword. Record real confusables.

#### Synthetic TTS data generation

Synth4Kws (Google 2024) demonstrated that mixing TTS-generated keyword samples with real recordings yields 30% EER improvement and 47% AUC improvement over real data alone. Adopt this approach with the following pipeline:

1. **Keyword generation:** Use Indic Parler-TTS (AI4Bharat, Apache-2.0) on Modal A10G to generate the custom keyword across diverse Indian accents, genders, ages and speaking rates via text-prompt control (e.g. "Male, Hindi, clear audio, normal pace" / "Female, Tamil-accented English, fast pace"). Target approximately 2,000 synthetic positive samples across 50+ distinct speaker prompts.
2. **Hard-negative generation:** Generate phonetically confusable variants (GraphemeAug edits) using the same TTS engine. Target approximately 500 hard-negative samples.
3. **Speaker diversity via cloning:** Optionally use CosyVoice 2/3 (Apache-2.0) with openly licensed IndicVoices speaker references for additional voice diversity. Do not clone identifiable individuals without consent.
4. **INMP441 domain adaptation:** Apply a digital filter approximating the INMP441 frequency response (high-pass roll-off below ~100 Hz, relatively flat 100 Hz–10 kHz, roll-off above ~15 kHz) to all TTS-generated audio before training. This reduces the acoustic domain gap between clean TTS output and real microphone recordings. A biquad filter chain suffices.
5. **Noise layering:** Apply MUSAN backgrounds at random SNR (5–20 dB) and OpenSLR 28 room impulse responses to all synthetic samples. Apply speed variation (0.8–1.2×) and volume perturbation (±6 dB).

Synthetic data bootstraps training before real recordings are available and provides accent diversity that would require 50+ consenting speakers to collect manually. Real INMP441 recordings (minimum ~50–100 clips) remain essential for closing the remaining domain gap. Final test accuracy must be measured exclusively on real human recordings.

Log provenance and augmentation seeds. Keep ordinary-augmentation and hard-negative arms compute-matched. Classify false activations by source/phrase rather than only reporting a single rate.

## 7. Firmware state machine

BOOT_SELFTEST → LISTENING_ONLINE or LISTENING_OFFLINE → CANDIDATE → STREAMING → FINALIZING → COOLDOWN → LISTENING. MUTED and FAULT are explicit transitions from relevant states.

- BOOT_SELFTEST: verify feature/model contract, initialize buffers, measure memory floor, validate mic input.
- LISTENING: increment sample counter, compute features/state continuously, overwrite only the bounded pre-trigger history. Keep audio-free control connection if policy permits.
- CANDIDATE: local score history only. No cloud confirmation or audio transfer.
- STREAMING: latch trigger sample index, snapshot ring ownership without blocking DMA, transmit chronological retained samples then new samples. KWS can pause during this explicit command state; do not count that as idle listening.
- FINALIZING: flush final audio/count; receive final transcript or deadline error. A simple energy-based end detector is optional, trained/tested for quiet speech; hard maximum session duration remains.
- COOLDOWN: short bounded rearm policy with a reset/low-score condition. Report inhibited duration.
- OFFLINE/FAULT: continue local detection if safe; mark remote ASR unavailable and discard private audio after bounded local retention. Do not replay old user commands unexpectedly on reconnect.
- MUTED: no capture/transmit payload; clear queued audio and retained ring safely.

Initial limits: 10 s maximum command, 600 ms endpoint hangover candidate, 2 s silence/no-command deadline candidate and a bounded connection retry backoff with jitter. Tune from validation; first-syllable preservation takes priority over an aggressive endpoint. An LED-only demo avoids self-wake from speaker playback.

## 8. Sample-preserving streaming protocol

### 8.1 Buffer and codec policy

Start with a 300 ms PCM pre-roll (9,600 bytes) inside the 12 KiB raw envelope. Choose final length from measured detection-delay distribution and R07 tests. Once triggered, this may include part of the keyword; disclose that fact. “Nothing is sent before activation” is different from “all transmitted samples were captured after activation.”

Retained history must cover the first command sample when a user speaks without a pause. If observed detector delay exceeds the retained interval, it is a failed continuity test—not a transcript-only pass.

Use 20 ms frames initially (320 PCM samples). Do not buffer an entire utterance. PCM16 mono is 32,000 bytes/s before framing. Plain 4-bit IMA ADPCM has roughly one quarter of the raw sample payload plus per-block initialization/header cost. Small independent blocks bound decoder-state loss and permit deterministic fixtures.

The ADPCM-XQ source is a reference; use a measured low-complexity setting or a compatible minimal implementation, with notices. Opus fixed-point/low-complexity is an optional branch only if complete encoder state and scratch fit; lower bitrate alone is not enough.

### 8.2 Version 1 wire contract

An authenticated connection establishes session identity, firmware/model/feature hashes, codec and sample rate once through bounded control messages (≤2 KiB). Each session uses a fresh unpredictable ID. No credential in a URL/log.

Proposed 24-byte binary frame header:

| Field | Bytes | Meaning |
|---|---:|---|
| version, codec | 1 + 1 | Reject unsupported versions/codecs |
| flags | 2 | Start/end/discontinuity flags; reserved bits must be zero |
| sequence | 4 | Monotonic within session |
| first_sample_index | 8 | Device sample-clock position |
| sample_count | 2 | Decoded PCM sample count |
| payload_bytes | 2 | Bounded encoded length |
| payload_crc32 | 4 | Diagnostic corruption check, not authentication |

Specify little-endian fields, signed PCM, ADPCM nibble ordering, initial predictor/index and valid padded-nibble behavior in a machine-readable contract plus golden frames. TLS provides transport security. Sequence/sample continuity checks detect logic errors that TLS cannot.

Gateway validates length, count, codec/version, session state, sequence and sample offsets before decoding. It converts PCM into whatever float tensor the backend needs. Do not transmit float32 because an ASR example does.

A 20 ms PCM frame is approximately 640 + 24 application bytes; an illustrative independent ADPCM block is approximately 160 + 4 + 24. Report actual application, WebSocket, TLS and network bytes separately. The theoretical reduction is not a measured wire-saving claim.

### 8.3 Backpressure and security

Separate real-time acquisition from socket writes through a fixed-capacity queue. Network code may not block DMA/frontend processing. If a bounded send deadline or queue capacity is exceeded, abort visibly with a sample-gap/error record; never silently replace old command samples. Test slow receiver, Wi-Fi loss and partial writes.

One active session per device; bounded per-account concurrency and maximum frame/session sizes. TLS hostname/certificate validation remains enabled; authenticate devices with scoped revocable credentials provisioned locally. Public cloud credentials never reside on the MCU. Reject replayed session identifiers and unauthorized configuration/model changes.

Mic audio is discarded server-side after inference by default. Opt-in evaluation capture uses encrypted/private storage with retention/deletion controls. Do not send secrets, raw audio or transcripts to generic error logs.

## 9. Timing instrumentation

Let K be annotated keyword-end sample/time, D local detection, F first outbound frame containing genuine post-keyword audio and S the ASR ingress acceptance of that frame. Primary latency is S minus K, not D minus K and not receipt of a JSON “start” message.

Capture acquisition timestamps and monotonically increasing sample indices. The gateway timestamps validated-frame ingress **before ASR batching**, then separately records decoder acceptance, first partial, stable partial and final output.

Never subtract an ESP32 clock and server clock without calibration. Phase 0 uses a controlled timing fixture: annotated audio playback with a recorded reference, device GPIO/serial markers, host ingress trace and measured clock-offset bounds. A LAN timing host or external logic trace establishes the local stages. WAN one-way timing requires explicit offset uncertainty; a return acknowledgement gives a conservative bound. RTT/2 is only an assumption, not exact clock synchronization.

Publish intervals when synchronization uncertainty is material. Validate a subset with independent hardware timing. Report p50/p95/p99 and failures, warm versus cold, individual stages and whole trials. Do not add separate stage percentiles into an end-to-end percentile.

Privacy packet capture: a TLS capture can show packet sizes, not prove encrypted contents are non-audio. Correlate a controlled decryptable laboratory capture or instrumented server payload logs with firmware review and normal encrypted production behavior.

## 10. Server and cloud design

Portable Python ASGI gateway owns authentication, binary validation/decoding, timestamps, session deadlines and backpressure. An adapter interface owns start_stream, push_audio, poll_partial, finalize and cancel. Model work runs outside the gateway event loop. Bound memory and concurrent decoder states.

Run Qwen3-ASR-0.6B and Nemotron 3.5 as GPU candidates; sherpa-onnx with the specifically licensed English streaming Zipformer is the CPU baseline. Compare backend chunk size and first stable token separately from immediate ingress. Hindi/English/Hinglish quality must be measured on our set. Do not adopt an opaque hosted speech endpoint.

A container can run on a LAN workstation, a cloud VM or Modal. LAN operation is a valid remote-server continuity demonstration but not a fake cloud result. Download/cache versioned weights before demos; a missing model never causes a surprise multi-GB fetch during a request.

On Modal, benchmark the current Server primitive versus the simpler ASGI Web Function path for this use case. Pin the selected API and test WebSocket duration, authentication, cold-start/503 behavior, rolling updates and disconnect cleanup. Maintain a warm model only during approved test/demo windows. Persistent device control connections may keep compute occupied; measure their billing effect instead of assuming “idle” means free.

Set an approved budget, container/concurrency caps and shutdown procedure before paid jobs. Record GPU model, region, latency and cost per experiment. Cloudflare Workers serve dashboard/config/metadata; D1 holds device/model/evaluation pointers and R2 holds private datasets or signed release artifacts. No live audio relay through Worker/D1 by default. Core local SQLite/filesystem adapters preserve portability.

Do not implement fleet orchestration, queues or multi-tenant analytics before a single-device end-to-end proof.

### Cloudflare deployment (dashboard and metadata only)

Cloudflare Workers serve the evidence console static assets and API endpoints. Durable Objects with the Hibernation API can maintain device session state with minimal billing during idle periods. D1 (SQLite) stores session metadata, device/model pointers and evaluation results. R2 stores consent-safe evidence artifacts and signed model bundles.

No live audio relays through Cloudflare Workers. Audio streams directly from the ESP32 to the ASR gateway (on Modal or LAN). The Worker/D1 layer handles metadata and presentation only. Free-tier Cloudflare resources cover expected demo traffic; verify Worker CPU limits (10 ms on free, 30 ms on paid) are sufficient for session management.

### Modal warm-container guidance

For GPU ASR, Modal's `keep_warm=1` with `container_idle_timeout=300` keeps one GPU container hot during demo windows. Before demo day, measure: (a) actual cold-start latency for the selected ASR model, (b) WebSocket support via the Server or Web Function primitive, (c) billing behavior of idle warm containers, (d) container limits and concurrency under the approved budget. Do not assume `keep_warm` is free or that cold starts are sub-second. Maintain the LAN-server fallback regardless of cloud availability.

## 11. Evidence console and wow features

Priority 1: live timeline with keyword boundary annotation when available, detection, first valid ASR audio and transcript milestones. Show measurement uncertainty rather than a falsely precise number.

Priority 2: resource bar with static/peak memory and CPU definitions, real model identity, byte counter and online/muted/streaming state. Experimental telemetry can use USB serial to avoid distorting production Wi-Fi cost; label the transport and measure instrumented versus release builds.

Priority 3: matched baseline view and confusable challenge, fixed frozen thresholds, actual mistakes retained. A judge chooses a new command; no predefined transcript lookup.

Priority 3a (presentation of measured results, dependent on validated data):

- Live spectrogram: display scrolling mel-spectrogram from actual device telemetry (USB serial or instrumented WebSocket). Label the transport method. This is visualization of real data, not a simulation.
- Latency waterfall: break detected latency into measured stages (detection confirmation, WebSocket establishment, edge-to-gateway transit, ASR processing). Show measurement uncertainty. Do not present illustrative mockup numbers as measurements.
- Noise challenge: play a declared noise source near the microphone and show the false-activation count over a timed interval. Report the specific noise, SPL if measurable, and mic distance. An energy-threshold detector is a useful didactic baseline but does not represent existing KWS systems—label it as "non-ML sound detection" rather than implying it represents competing products.
- Confusable phrase challenge: a judge selects phrases from the human-vetted confusable list. Show the device correctly rejecting near-miss words. Display the threshold and score, not just a pass/fail indicator.
- Retraining demonstration is a **non-goal** (prd.md §6). Live retraining of a keyword model from scratch takes ~1 hour (TTS generation + training + quantization + export). This cannot be performed in front of judges. Instead, demonstrate the **OTA model-swap mechanism**: show a pre-trained alternative model being deployed to the device in ~30 seconds via signed model bundle update, with automatic rollback on contract mismatch. This demonstrates engineering maturity (versioned deployment) without masquerading as instant arbitrary-keyword learning.

All console features display board-sourced data with source labels and staleness indicators. No browser-side microphone, cached results, or pre-rendered visualizations substituted for live device output.

Priority 4: versioned model bundle update/rollback with a feature-contract rejection demo. Model updates cannot masquerade as instant arbitrary-keyword learning.

Stretch only: codec A/B with WER trade-off, measured energy, dual Hindi/English demonstrations, opt-in hard-negative capture tooling. Defer speaker verification, beamforming, chatbot, emotion inference and on-device continual learning.

## 12. Verification stack

Host tests: C/C++ unit tests for arithmetic/buffers/state, Python pytest for data/training/evaluation, schema/decoder fuzz/property tests, TypeScript UI/contract tests. Firmware warnings and sanitizers in host builds; bounded allocation checks on device. Exact tools/versions locked at scaffolding.

Golden fixtures: PCM16 conversion, frontend output, INT8 logits, state across chunk boundaries, codec round-trip/sample count, malformed frames and truncated sessions. Do not overwrite goldens just to make a changed model pass.

Hardware-in-loop runner: real-time playback, board serial/GPIO logs, gateway capture, fixed artifact hashes and analysis command. Fast offline corpus scoring helps iteration but does not certify real-time MCU scheduling. Replay tests and live-speaker trials are labelled separately.

Regression matrix: model choice, quantization, microphone/SNR/distance, network, codec, warm/cold, threshold and firmware build. Publish counts for every reported slice. Use speaker-clustered resampling for uncertainty on recall; use count-based false-wake intervals with stated assumptions.

Security/fault tests: bad certificate, expired/revoked token, forged frame, oversized payload, replay, stalled ASR, flash update interruption, model-feature mismatch, brownout recovery, heap fragmentation and long listening soak. No destructive eFuse testing.

## 13. Repository and contracts

Create only when implementation begins:

```text
firmware/       main/, components/, sdkconfig.defaults, partitions.csv
training/       collection/, datasets/, models/, export/, evaluation/
server/         gateway/, codecs/, asr_adapters/
dashboard/      src/
contracts/      protocol/, model_bundle/, measurements/
tests/          golden/, unit/, integration/, hardware/
infra/          containers/, modal/, cloudflare/
artifacts/      local ignored runs; published consent-safe evidence only
```

Model bundle fields: keyword/version, training origin, source/weights license, data split hash, architecture, frontend version, sample rate/window/hop, tensor shapes/types/scales, operator requirements, arena/state estimate and measured allocation, threshold policy, cooldown, checksums/signature, firmware ABI and target board.

Measurement manifest: run ID, UTC recording time and monotonic domains, board/chip/flash/clock, source/firmware/model/config hashes, dataset subset IDs, acoustic/network profile, timestamps and clock bounds, all resource definitions, raw count/duration, errors and analysis version.

Keep semantic contracts independent of host programming language. Producers and consumers need matching fixtures before an incompatible version change.

## 14. Work allocation and risk responses

User/IT: product, integration, server/dashboard, dataset organization. ECE teammate: electrical checks, acquisition, device profiling and physical test execution. AI: code, literature checks, fixtures, training automation, log analysis and clean-context technical review. Everyone can help collect consented recordings; no extra external agent is a prerequisite.

Risks and actions:

- Platform floor too high: simplify verified SDK/network configuration first; reduce model width/state; do not hide memory categories.
- CPU too high: profile frontend and fallback operators; reuse state and reduce hop work; only change hop with retraining/latency evidence.
- False wakes: improve real confusables and source diversity; adjust validation threshold jointly with recall/latency.
- Missed initial word: measure detection and queue delay; tune bounded pre-roll within RAM, not a giant hidden buffer.
- Single mic fails far-field: improve physical placement/enclosure, collect matched data and scope claimed range; do not promise software-only beamforming.
- ASR streaming too slow: choose a genuinely lower-delay backend/chunk; keep ingress and transcript claims separate.
- Cloud outage: honest unavailable state and labelled LAN fallback; never a canned transcript.
- Rules/keyword uncertainty: obtain official clarification, preserve configurable contracts; no retroactive invented approval.

## 15. Release and claims

Release artifact set: source, reproducible model training/export recipe, pinned firmware/container, rights inventory, SBOM/notices, physical wiring/BOM, signed model bundle, evaluation manifests, failure log and consent-safe demo procedure. Team publication license and participant dataset release require explicit approval; do not assume all recordings are publishable.

Adopt a claim only after matched evidence proves it. “Uses a recent paper” is not originality. “Under 256 KB” requires the complete measured definition, and “under 10% CPU” requires the stated continuous-listening profile. The research and Cloudflare skills informed source verification and the separation of metadata hosting from the real-time voice path.

