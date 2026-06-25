# CONDUCTOR — QC Agent for M&E Post-Production
## Running inside NVIDIA NemoClaw + OpenShell on Dell GB10 Grace Blackwell
### Dell x NVIDIA NemoClaw Demo Contest 2026

> An AI agent that monitors incoming creative assets, diagnoses technical issues, and generates actionable QC reports — running securely inside NVIDIA OpenShell on a Dell GB10.

---

## What It Does

CONDUCTOR is a QC Agent for Media & Entertainment post-production. It runs inside NVIDIA NemoClaw + OpenShell on a Dell GB10 Grace Blackwell, scanning video files for technical issues and generating a full QC report — automatically, locally, with zero cloud dependency.

The agent:
1. Scans a folder of MP4/MOV files
2. Runs ffprobe on each file to extract technical specs
3. Diagnoses issues: wrong resolution, wrong fps, wrong audio, short clips
4. Generates a JSON + PDF report with actionable diagnosis per file
5. All inside a sandboxed OpenShell environment (Landlock + seccomp + netns)

---

## Demo Results

- 9 real M&E production files scanned
- 2 PASS, 7 FAIL diagnosed
- PDF report generated automatically
- 0 cloud API calls
- 0 humans in the loop

---

## What CONDUCTOR Diagnoses

| Check | Method | Notes |
|-------|--------|-------|
| Resolution (4K 3840x2160) | ffprobe | Exact value |
| Frame rate (24fps) | ffprobe | Flags 18fps, 29fps, 60fps outliers |
| Audio sample rate (48kHz) | ffprobe | Only flags if audio stream exists |
| Short clip under 1 second | ffprobe | Exact duration |
| Missing audio track | ffprobe | Stream count |
| SHA-256 provenance hash | sha256sum | Per asset |

---

## Stack

- Dell GB10 Grace Blackwell (128GB unified memory, NVIDIA GB10 GPU)
- NVIDIA NemoClaw v0.0.41
- NVIDIA OpenShell (Landlock + seccomp + netns isolation)
- OpenClaw TUI 2026.4.24
- Qwen3.6-35B-A3B-FP8 via vLLM (local inference)
- ffprobe / ffmpeg (installed by agent inside sandbox)
- Python fpdf2 (PDF report generation)
- Node.js jsPDF (PDF generation inside sandbox)

---

## How to Run

### Step 1 — Start vLLM (Terminal 1)

```bash
HF_HUB_OFFLINE=1 TRANSFORMERS_OFFLINE=1 \
VLLM_USE_FLASHINFER_SAMPLER=0 vllm serve Qwen/Qwen3.6-35B-A3B-FP8 \
--host 0.0.0.0 --port 9494 --dtype auto \
--max-model-len 65536 --max-num-seqs 1 \
--gpu-memory-utilization 0.35 \
--enable-auto-tool-choice --tool-call-parser qwen3_coder --enforce-eager
```

Wait for: `Application startup complete`

### Step 2 — Connect to NemoClaw sandbox (Terminal 2)

```bash
nemoclaw my-assistant connect
```

### Step 3 — Start OpenClaw TUI inside sandbox

```bash
openclaw tui
```

### Step 4 — Run the QC Agent

Copy your video files into the sandbox first:
```bash
docker cp /path/to/your/videos/. <container_id>:/sandbox/qc2/
```

Then in the OpenClaw TUI:
```
You are a QC Agent. Scan /sandbox/qc2/ for all mp4 files.
Run ffprobe on each to get resolution, fps, codec, audio sample rate, duration.
Flag FAIL if: not 4K (3840x2160), not 24fps, file HAS audio AND sample rate
is not 48kHz, or duration under 1 second.
Save report as /sandbox/qc_conductor_report.json and generate
/sandbox/qc_conductor_report.pdf. Helvetica font only, no unicode.
```

### Step 5 — Extract reports

```bash
docker cp <container_id>:/sandbox/qc_conductor_report.json ~/Downloads/
docker cp <container_id>:/sandbox/qc_conductor_report.pdf ~/Downloads/
```

---

## NemoClaw Setup

### Install NemoClaw

```bash
curl -fsSL https://www.nvidia.com/nemoclaw.sh | bash
```

### Onboard with vLLM endpoint

```bash
nemoclaw onboard --no-gpu
```

When prompted:
- Inference option: 3 (Other OpenAI-compatible endpoint)
- Base URL: http://192.168.x.x:9494/v1 (use your GB10 LAN IP, not localhost)
- API key: any string (vLLM local does not require auth)
- Model: Qwen/Qwen3.6-35B-A3B-FP8
- Sandbox name: my-assistant

Note: Use your GB10 LAN IP address (not localhost) for the vLLM endpoint.
The NemoClaw sandbox runs in a separate network namespace and cannot resolve localhost.

---

## Why NemoClaw + OpenShell

- Landlock filesystem isolation — agent cannot access host files outside /sandbox
- seccomp syscall filtering — blocks dangerous system calls
- netns network isolation — agent cannot make unauthorized outbound connections
- Agents start with zero permissions — all access is policy-enforced
- Inference stays private by default — no data leaves the GB10

---

## Hardware

Dell GB10 Grace Blackwell
- NVIDIA GB10 GPU
- 128GB unified memory
- Ubuntu Linux (native)
- Always-on, offline capable

---

Gil Balderas — M&E Filmmaker —  Mexico — shotlock.tech
