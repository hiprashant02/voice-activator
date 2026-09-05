# Research and decision record

Research cutoff: 5 September 2026. This is a source-backed design review, not an exhaustive proof that no better unpublished system exists. No project-specific model has been trained or benchmarked on the board yet.

Evidence labels: **paper** = reported author result; **implementation** = repository/documentation inspected, not executed here; **proposal** = our design inference; **gate** = unresolved until measured or clarified. Paper accuracy, MAC counts and model-file bytes do not establish whole-firmware CPU or RAM compliance.

## 1. Problem authority and originality rules

The user-supplied statement is the working requirement source. Its text matches [this third-party SIH26172 listing](https://sih2026.vuce.in/en/ps/SIH26172), titled “Low Latency and Efficient Voice Activator for Edge Devices.” The mirror's organization/category/date fields are not independently verified. The [official 2026 catalogue](https://sih.gov.in/sih2026PS) could not be retrieved in this research. Obtain the official entry/export from the college SPOC before submission.

The official [College SPOC guidelines PDF](https://www.sih.gov.in/letters/Guidelines-College-SPOC.pdf) available at research time is explicitly **SIH 2024**, despite generic URL/search metadata. Pages 19 and 23 discuss novelty/feasibility and previous-event/IP conditions. It is historical context, **not evidence of the current 2026 rules**. Confirm the applicable 2026 originality, reuse, AI assistance, publication and IP terms with the organizer. Do not promise that attribution automatically satisfies them.

Working interpretation: the resource and heavy-transformer restrictions concern the edge device; remote ASR can use an openly licensed pretrained model. Confirm this interpretation if the organizer's complete text is ambiguous.

## 2. Executive technical verdict

The defensible choice is a **hardware-selected streaming INT8 custom KWS model**, not “the newest model” by name. Benchmark three real candidates: a small DS-CNN reference, a custom-trained microWakeWord-style streaming model, and a narrow causal depthwise temporal network with cached state. Select the best recall/false-wake/latency point that passes the complete WROOM-32 resource envelope.

The highest-value recent ideas are confusable-phrase data generation and whole-pipeline optimization. The highest-risk engineering issue is the combined microphone/frontend/KWS/RTOS/Wi-Fi/TLS footprint. Use a warm audio-free control connection when feasible; send only post-activation audio in bounded binary frames. A GPU cloud ASR upgrade cannot repair poor edge recall or clipped command onset.

Our strongest prospective contribution is the **measured combination** of custom-keyword robustness, strict resource compliance, preserved handoff and evidence tooling. None of those components is individually new.

## 3. KWS paper shortlist and applicability

| Source | Evidence and relevant contribution | Decision and boundary |
|---|---|---|
| [Hello Edge: Keyword Spotting on Microcontrollers, Zhang et al., 2017](https://arxiv.org/abs/1711.07128) · [author code](https://github.com/ARM-software/ML-KWS-for-MCU) | Foundational DS-CNN resource/accuracy baseline; author implementation is available | Required small baseline trained on our keyword. Legacy training code is not a current pinned stack; reported classification results do not establish false activations/hour |
| [SpecAugment, Park et al., Interspeech 2019](https://arxiv.org/abs/1904.08779) | Frequency and time masking in the spectrogram domain; zero inference cost. Reported 2–5% accuracy gain on small datasets | Adopt for training augmentation. No on-device cost. Starting parameters (f=8 of 40 bins, t=10 of 49 frames) are not proven optima for our keyword |
| [Temporal Convolution for Real-time KWS, Choi et al., 2019](https://arxiv.org/abs/1904.03814) | TC-ResNet replaces expensive spatial processing with temporal convolution | Useful topology reference. Mobile-device timings are not MCU timings |
| [Streaming Keyword Spotting on Mobile Devices, Rybakov et al., 2020](https://arxiv.org/abs/2005.06720) · [Google implementation](https://github.com/google-research/google-research/blob/master/kws_streaming/README.md) | Conversion to streaming execution with state; multiple architecture references | Adopt state-equivalence tests and incremental execution. Prove exported operators on Xtensa; desktop/mobile support is insufficient |
| [Small-Footprint KWS with Multi-Scale Temporal Convolution, Li et al., 2020](https://arxiv.org/abs/2010.09960) | TENet uses multi-scale training structure that can simplify for inference | Optional reparameterization experiment after core streaming path. Do not assume arbitrary branches can be folded without matching linearity/padding |
| [Broadcasted Residual Learning for Efficient KWS, Kim et al., 2021](https://arxiv.org/abs/2106.04140) · [BCResNet code](https://github.com/Qualcomm-AI-research/bcresnet) | Strong compact Speech Commands classification results | Accuracy challenger, not default. Broadcasting, normalization and streaming conversion may cost more than parameter count suggests |
| [LiCo-Net, Yang et al., 2022](https://arxiv.org/abs/2211.04635) | Hardware-friendly linearized operations; author-reported cycle improvements on HiFi DSP | Promising research challenger. No verified WROOM-32 implementation/result in this review; no schedule dependency on recreating it |
| [Sparse Binarization for Fast KWS, Svirsky et al., Interspeech 2024](https://www.isca-archive.org/interspeech_2024/svirsky24_interspeech.html) | Sparse representations and a linear classifier offer a different compute trade-off | Watchlist. Reported speedups do not demonstrate benefit with this MCU's kernels and acoustic target |
| [Synth4Kws, Google, 2024](https://arxiv.org/abs/2407.16840) | Proves TTS-only training is viable for KWS. Synthetic data alone improves monotonically with diversity; mixing TTS + small real data yields 30% EER improvement and 47% AUC improvement over real data alone | Adopt the synthetic data generation pipeline. TTS provides speaker diversity; real INMP441 recordings close the domain gap. Final test must use real recordings only |
| [GraphemeAug, Zhang et al., Interspeech 2025](https://arxiv.org/abs/2505.14814) | Systematic insertion/deletion/substitution creates difficult negative phrases; paper reports improvement on synthetic hard negatives | Adopt human-vetted confusable generation plus real recordings. The reported 61% AUC improvement is not 61% fewer field false wakes |
| [LLM-Synth4KWS, Zhu et al., Interspeech 2025](https://arxiv.org/abs/2505.22995) | Synthetic confusable groups and a confusable-focused evaluation metric | Adopt the data/evaluation principle. LLM/TTS is optional offline tooling, not an edge dependency; the paper's custom-enrollment architecture is not automatically MCU-sized |
| [MFA-KWS, 2025](https://arxiv.org/abs/2505.19577) | Frame-asynchronous decoding for custom KWS | Do not use as the Phase 0 edge path: decoding speedups do not establish the 256 KB limit |
| [AdaKWS, Interspeech 2025](https://www.isca-archive.org/interspeech_2025/xiao25b_interspeech.pdf) | Test-time adaptation to acoustic shift | Defer: changing a deployed detector complicates false-wake guarantees and consumes resources. Prefer fixed, validated profiles and offline retraining |
| [End-to-End Efficiency in KWS, Bartoli et al., 2025](https://arxiv.org/abs/2509.07051) · [full text](https://arxiv.org/html/2509.07051v1) | Compares lightweight families and includes feature extraction/runtime on STM32 targets | Adopt whole-pipeline measurement. Neither its TKWS results nor STM32 accelerators prove WROOM-32 performance |
| [Synaspot, 2025](https://arxiv.org/abs/2512.15124) | Streaming open-vocabulary/multimodal direction | Research context only; different scope from one custom MCU keyword |
| [EdgeSpot, 2026](https://arxiv.org/abs/2601.16316) | BC-ResNet backbone paired with Sub-Center ArcFace loss for few-shot KWS. Largest variant: 128K params, 29.4M MACs, 82% ten-shot accuracy at 1% FAR | Evaluate ArcFace loss for our scratch-trained teacher only (not edge model). Does not establish MCU resource compliance or near-zero false wakes. The teacher/distillation framing is the valid use |
| [MLPerf Tiny v1.4, MLCommons, July 2026](https://mlcommons.org/2026/07/mlperf-tiny-v1-4-results/) | Industry-standard benchmark for TinyML inference including keyword spotting. Defines measurement protocols for latency, throughput and energy | Adopt measurement methodology for internal comparison. Distinguish from an official MLPerf submission. Streaming task specification is more relevant than isolated-word classification |
| [Efficient Polish-Language KWS on Microcontrollers, Sobczyk and Fonał, 6 August 2026](https://doi.org/10.3390/app16157844) | Recent non-English compact-model/quantization study with Pico 2 device validation | Reinforces testing on the actual acquisition hardware and language. Different MCU/microphone, not a transferable compliance certificate |

### Why not select a transformer, spiking network or binary network immediately?

Attention, spiking and extreme quantization papers can be valuable, but their speed/energy often depends on specialized hardware, custom kernels or simulated energy accounting. Standard int8 depthwise/pointwise/linear operations are a lower-risk baseline on a classic ESP32. More exotic candidates must outperform that baseline on the **same** microphone data, threshold protocol, clock, full firmware and license criteria.

Structured width/channel reduction has a clearer MCU payoff than unstructured sparsity unless the runtime actually skips sparse work. QAT is a conditional recovery tool after measured PTQ degradation, not an automatic prerequisite.

## 4. Actual implementations and compliance traps

| Component/source | What was verified | Use |
|---|---|---|
| [microWakeWord](https://github.com/OHF-Voice/micro-wake-word) | Apache-2.0 training framework; streaming, quantized, microcontroller-oriented | Strongest practical reference and matched custom-trained baseline. Do not ship bundled assistant keyword weights |
| [ESPHome microWakeWord integration](https://esphome.io/components/micro_wake_word/) · [runtime source](https://raw.githubusercontent.com/esphome/esphome/dev/esphome/components/micro_wake_word/streaming_model.cpp) | Separate variable and tensor allocations, quantized feature contract and runtime-dependent allocation behavior are visible | Count all state, not only one arena. Bare ESP-IDF core is the proposed production implementation; full ESPHome stack is not assumed budget-compliant |
| [esp-tflite-micro](https://github.com/espressif/esp-tflite-micro) | Open Apache-2.0 component with examples and optimized-kernel integration | Primary runtime; freeze compatible IDF/component commits after build and operator tests |
| [ESP-NN](https://github.com/espressif/esp-nn) | Target-specific optimizations, including S3 vector assembly | Use only kernels actually selected for classic ESP32. CMSIS-NN is not a drop-in Xtensa runtime |
| [Google kws_streaming](https://github.com/google-research/google-research/blob/master/kws_streaming/README.md) | Multiple streaming architectures; input/stride/state constraints | Architecture/export reference; isolate legacy TensorFlow/Keras compatibility |
| [BCResNet license](https://raw.githubusercontent.com/Qualcomm-AI-research/bcresnet/main/LICENSE) | Redistribution conditions and an explicit absence of a patent grant | Retain notices; legal compatibility review if adopting. Clean-room code is not a patent-clearance guarantee |
| [openWakeWord](https://github.com/dscripka/openWakeWord) | Code Apache-2.0; included pretrained models CC-BY-NC-SA-4.0; embedding-based pipeline | Prior art, not the default MCU implementation. Code license does not make every included model permissively licensed |
| [ESP-SR/WakeNet](https://github.com/espressif/esp-sr) · [license](https://raw.githubusercontent.com/espressif/esp-sr/master/LICENSE) | Repository includes prebuilt libraries; license grants use on Espressif products. Recent WakeNet releases exist | Not adopted: public headers/GitHub hosting do not establish an entirely source-rebuildable, compliant custom-training voice pipeline |
| [Wyoming](https://github.com/OHF-Voice/wyoming) | Existing voice-system interoperability project | Prior-art reference; a custom minimal MCU protocol still needs clear justification through byte/latency measurements |
| [IMA ADPCM implementation](https://github.com/dbry/adpcm-xq) · [license](https://raw.githubusercontent.com/dbry/adpcm-xq/master/license.txt) | Source encoder/decoder and permissive notice-preserving terms | Prototype plain low-complexity independent-block encoding; do not assume XQ lookahead settings are inexpensive |
| [Opus encoder API](https://opus-codec.org/docs/opus_api-1.5/group__opus__encoder.html) | Configurable encoder, state-size API and frame controls | Optional codec challenger; include all state/scratch allocations and actual encode time, not bitrate alone |

Vendor SDK Wi-Fi/radio binaries are a separate issue from prohibited voice SDKs. Document their presence; do not describe the entire ESP32 hardware/software chain as universally source-rebuildable. Ask the organizer if its open-source clause extends beyond the voice pipeline.

## 5. Remote ASR shortlist — not edge models

| Model/runtime | Evidence | Decision |
|---|---|---|
| [Qwen3-ASR-0.6B model card](https://huggingface.co/Qwen/Qwen3-ASR-0.6B) · [2026 report](https://arxiv.org/abs/2601.21337) | Apache-2.0; Hindi and English included in multilingual coverage | Primary permissively licensed GPU candidate, selected provisionally rather than on assumed latency |
| [Qwen3-ASR implementation](https://github.com/QwenLM/Qwen3-ASR) · [streaming example](https://github.com/QwenLM/Qwen3-ASR/blob/main/examples/example_qwen3_asr_vllm_streaming.py) | Streaming example exists; current documented streaming path uses vLLM and lacks batch/timestamp support | Benchmark chunk size, state growth, first stable token, GPU memory and concurrency. A file-fed example is not an already built MCU WebSocket service |
| [Nemotron 3.5 ASR streaming 0.6B](https://huggingface.co/nvidia/nemotron-3.5-asr-streaming-0.6b) | Current 2026 cache-aware streaming model; Hindi and English in transcription-ready tier; OpenMDW-1.1 | Serious GPU challenger. Freeze exact weights/runtime, test local accents and license suitability; do not confuse earlier English-only models with this release |
| [OpenMDW-1.1 actual license](https://openmdw.ai/license/1-1/) | Permissive model-material grant with notice and other conditions | Do not falsely call it an NC-only license. It is distinct from Apache; record exact terms and obtain rule clarification if needed |
| [sherpa-onnx](https://github.com/k2-fsa/sherpa-onnx) · [English streaming Zipformer artifact](https://huggingface.co/csukuangfj/sherpa-onnx-streaming-zipformer-en-2023-06-26) · [WebSocket guide](https://k2-fsa.github.io/sherpa/onnx/python/streaming-websocket-server.html) | Apache runtime and an explicitly Apache-tagged candidate model; streaming server tooling | CPU-server baseline and English fallback. Different models have different licenses/languages; converting PCM to runtime floats happens on server |
| [faster-whisper](https://github.com/SYSTRAN/faster-whisper) | Open CTranslate2-based Whisper inference implementation | Multilingual fallback; chunked/re-decoded transcription must be labelled, not presented as native low-delay streaming |
| [Parakeet TDT 0.6B v3](https://huggingface.co/nvidia/parakeet-tdt-0.6b-v3) | Multilingual checkpoint focused on European languages | Not the default Hindi path; no reason to adopt merely because of benchmark popularity |

Selection is by measured WER/CER, command-onset deletion, transcript delay, real-time factor, warm/cold behavior, license and operating cost. Public WER scores are not comparable across different corpora, punctuation rules, languages and latency settings. Never infer Hinglish quality solely from separate Hindi/English support.

Implementation note: Qwen3-ASR's documented streaming path uses vLLM and currently lacks batch/timestamp support. Before adopting, verify: (a) WebSocket integration with our binary gateway, (b) chunk size and state growth under continuous streaming, (c) actual first-stable-token latency versus file-fed example results, (d) Hindi command accuracy on our consented test set. faster-whisper is a proven batch-mode fallback but its chunked pseudo-streaming must be labelled honestly, not presented as native token-by-token output.

The large-cloud-model restriction question must be resolved before submission. No ASR model from this table runs on the MCU. If pretrained remote models are disallowed by an organizer clarification, re-scope with them explicitly; do not hide the dependency.

## 6. Data plan and rights

| Source | Verified role and rights starting point | Limit |
|---|---|---|
| Team-recorded custom keyword/commands | Primary positives, accents, microphone/channel adaptation and final live tests; consented collection | Essential; generic datasets cannot replace the assigned keyword |
| [Speech Commands paper](https://arxiv.org/abs/1804.03209) · [TFDS catalogue](https://www.tensorflow.org/datasets/catalog/speech_commands) · [Google release announcement](https://www.research.google/blog/launching-the-speech-commands-dataset/) | Raw labeled words for baseline and negative diversity; released under CC-BY-4.0 | Data, not prohibited pretrained keyword weights; classification benchmark is not ambient false-wake evidence |
| [MUSAN / OpenSLR 17](https://www.openslr.org/17/) | Speech/music/noise, CC-BY-4.0 listing | Keep attribution and original-source grouping; remove accidental keyword positives from negatives |
| [RIRS_NOISES / OpenSLR 28](https://www.openslr.org/28/) | Room/noise augmentation, Apache-2.0 listing | Preserve archive notices and audit component files; simulated rooms do not replace physical mic tests |
| [LibriSpeech / OpenSLR 12](https://www.openslr.org/12/) | Long-form speech negative candidate | Verify selected archive license and speaker IDs at download; book speech differs from Indian conversation |
| [Mozilla Data Collective](https://mozilladatacollective.com/datasets?q=common+voice) | Optional language/accent diversity via selected Common Voice releases | Dataset/version-specific access and license audit required; not a mandatory registration dependency |
| Consented field negatives | Campus conversation, rooms, fan and nearby playback with known source rights | Record only consented participants; real private conversations are not free training data |
| [Indic Parler-TTS](https://huggingface.co/ai4bharat/indic-parler-tts) (AI4Bharat) | Apache-2.0 TTS supporting 23 Indian languages including Hindi. Text-prompt control over accent, gender, tone, pace. Trained on IndicVoices (23K hours, 51K speakers) | Primary TTS engine for synthetic keyword/negative generation. Provides Indian accent diversity via text prompts without voice cloning consent issues |
| [CosyVoice 2/3](https://github.com/FunAudioLLM/CosyVoice) (Alibaba) | Apache-2.0 multilingual TTS with zero-shot voice cloning from 3-10s reference. Used in SynTTS-Commands research for KWS data generation | Secondary TTS for speaker cloning. Requires reference audio — use only openly licensed speaker references, not identifiable individuals |
| [IndicVoices / IndicVoices-R](https://huggingface.co/ai4bharat) (AI4Bharat) | 23K hours natural/spontaneous speech, 51K speakers, 22 Indian languages. IndicVoices-R subset: 1,704 hours high-quality multi-speaker for TTS research | Speaker reference pool for CosyVoice cloning and accent diversity validation. Check per-file consent terms before use as training negatives |

Suggested collection scale and splits live in plan.md. Data licenses are not a substitute for speaker consent, privacy or publication permission. Synthetic TTS is a validated augmentation strategy (Synth4Kws, Google 2024 showed 30% EER improvement when mixed with real data). Choose an openly licensed engine **and voice weights** (Indic Parler-TTS Apache-2.0, CosyVoice Apache-2.0), preserve generation parameters, do not clone an identifiable person's voice without permission. Do not assume a cloud TTS API satisfies this PS. Final test accuracy must be measured on real human recordings, not TTS output.

## 7. Hardware and transport evidence

- [ESP32 datasheet](https://documentation.espressif.com/esp32_datasheet_en.html): classic Xtensa platform and SRAM architecture. Silicon SRAM capacity does not authorize exceeding the PS's application budget.
- [INMP441 manufacturer datasheet](https://invensense.tdk.com/wp-content/uploads/2015/02/INMP441.pdf): I2S, signed 24-bit acquisition. PDF fetching was intermittent; indexed manufacturer content supports the format. The ECE teammate must verify exact wiring/electrical details from the datasheet before power-up.
- [ESP-IDF RAM guide](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/performance/ram-usage.html): static and dynamic footprint must be considered.
- [ESP-IDF timing guide](https://docs.espressif.com/projects/esp-idf/en/v5.2/esp32/api-guides/performance/speed.html): per-core cycle counters and task profiling facilities. Use the selected IDF version's equivalent APIs; account for ISR attribution rather than blindly trusting a formatted CPU percentage.
- [ESP-IDF Mbed TLS guide](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/protocols/mbedtls.html): published examples show TLS configuration materially affects heap use; these are examples, not our peak memory result.
- [Wi-Fi buffer/power guide](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-guides/wifi-driver/wifi-performance-and-power-save.html): buffering and power-save choices have performance costs. Do not minimize buffers without testing stalls/loss.
- [Modal Web Functions](https://modal.com/docs/guide/webhooks), [Servers](https://modal.com/docs/guide/servers), [cold starts](https://modal.com/docs/guide/cold-start): WebSocket support and warm-container controls exist. New Server lifecycle/authentication/scale-from-zero behavior differs from Functions; pin and test the chosen deployment primitive.
- [Cloudflare Static Assets](https://developers.cloudflare.com/workers/static-assets/), [D1](https://developers.cloudflare.com/d1/), [R2](https://developers.cloudflare.com/r2/): suitable optional dashboard, metadata and artifact services. Our proposal keeps live audio direct to the ASR gateway and provides self-hosted alternatives.

## 8. Prior art, originality and ablations

Existing work already covers local wake-to-cloud transcription, streaming tiny models, synthetic confusables, pre-roll, compressed audio and dashboards. The rejection risk is high if the entry is only an ESPHome configuration with a new name.

Scoped claim to earn: “We designed and evaluated a custom-keyword voice front end on the specified MCU budget, combining phonetic-confusion-focused training with bounded, sample-preserving ASR handoff and reproducible end-to-end evidence.”

This is an engineering contribution claim, not a patent/world-first claim. Required evidence:

| Proposed contribution | Fair comparison |
|---|---|
| Cached-state causal KWS | Same training split/frontend versus small sliding-window DS-CNN and custom-trained microWakeWord reference |
| Confusion-focused dataset | Matched model/compute, ordinary augmentation versus added human-vetted hard negatives |
| INT8 deployment quality | Float versus PTQ versus QAT if used; complete streaming equivalence and MCU metrics |
| Low-loss handoff | Identical utterances with/without bounded pre-roll; count actual missing samples and first-word deletions |
| Efficient transport | PCM versus IMA ADPCM; optional Opus only if compliant; full bytes, CPU, RAM and WER |
| Warm control channel | Warm/cold connection distributions, idle cost and network conditions |
| Reproducible evidence | Another team member rebuilds and reruns a frozen subset without developer coaching |

Keep an attribution inventory and a contribution ledger identifying upstream code, modifications, team-written components, dataset creation and model training. Preserve dates/commits and record any previous-event participation. Document AI assistance according to current competition rules.

## 9. Remaining uncertainty

No inspected paper proves **this board + this microphone + this keyword + secure networking** simultaneously meets both limits. That is the strict Phase 0 question.

Unknowns: exact board/flash, official PS metadata and evaluation interpretation, final keyword, dataset recruitment, artifact licenses at frozen revisions, converter/operator compatibility, total TLS peak, whole-system CPU, WAN region/latency, ASR chunk behavior and final competition rules.

The research supports a strong experimental path, not guaranteed feasibility or victory. Pass hardware gates first, then make comparative claims from actual results.

