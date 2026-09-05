# WakeTrace — custom wake word to remote ASR

Edge keyword spotting with bounded memory on ESP32-WROOM-32, streaming audio to an open-source ASR gateway over TLS WebSockets.

## Repository Layout

```text
firmware/       main/, components/, sdkconfig.defaults, partitions.csv
training/       collection/, datasets/, models/, export/, evaluation/
server/         gateway/, codecs/, asr_adapters/
dashboard/      src/
contracts/      protocol/, model_bundle/, measurements/, datasets/
tests/          golden/, unit/, integration/, hardware/, host_c/
infra/          containers/, modal/, cloudflare/
artifacts/      local ignored runs; published consent-safe evidence only
```

## Quick Start (Tested Commands)

### 1. Environment Setup

```bash
# create python 3.11 virtual environment and install test/runtime dependencies
uv venv --python 3.11 .venv
uv pip install pytest numpy scipy websockets fastapi uvicorn ruff jsonschema starlette httpx modal --python .venv/bin/python

# install dashboard typescript dependencies
cd dashboard && npm install && cd ..
```

### 2. Run All Automated Verification Checks

Runs secret exclusion, license checks, host C test suite, Python linting, TypeScript typecheck, schema validation, pytest unit/integration tests, and RAM budget audit:

```bash
bash scripts/run_checks.sh
```

### 3. Run Individual Test Suites

```bash
# run standalone C host test runner (protocol layout, ADPCM codec, frontend filterbank)
clang -Wall -Wextra -Werror tests/host_c/test_host_c.c firmware/main/adpcm.c firmware/main/frontend.c -lm -I firmware/main -I contracts -o /tmp/wt_host_c_test && /tmp/wt_host_c_test

# run python unit and integration tests (19 test cases)
PYTHONPATH=. .venv/bin/pytest tests/ -v

# run typescript contract typecheck
cd dashboard && npm run check && cd ..

# audit memory budget against the 256 kB internal SRAM ceiling
.venv/bin/python firmware/scripts/ram_budget_audit.py
```

### 4. Local Test Server

```bash
# start local ASR gateway server (reads config.example.env or .env)
bash scripts/start_test_server.sh

# in another terminal, verify health endpoint
curl -s http://127.0.0.1:8000/health
```

### 5. Remote GPU Training on Modal

```bash
# train candidate models (DS-CNN vs Streaming Conv) from scratch on cloud GPU
modal run training/train_modal.py
```

## Core Contracts and Constraints

- **Hardware Target**: ESP32-WROOM-32 (ESP32-D0WDQ6-V3), 240 MHz, 520 kB SRAM (328 kB DRAM). Strictly zero PSRAM.
- **Microphone**: INMP441 I2S MEMS microphone, 16 kHz 16-bit mono.
- **Memory Ceiling**: 256,000 bytes hard maximum allocation across all tasks.
- **Wire Protocol**: 24-byte packed binary frame header with CRC32 payload verification.
- **Codecs**: Uncompressed PCM16LE and independent-block IMA ADPCM (4:1 compression, ~164 bytes per 20 ms frame).
- **Licensing**: Strict open-source only (Apache 2.0 / MIT / BSD). Zero proprietary wake SDKs. Zero pretrained global wake weights.
