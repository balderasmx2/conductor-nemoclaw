# CONDUCTOR — QC Agent for M&E Post-Production
## Running inside NVIDIA NemoClaw + OpenShell on Dell GB10 Grace Blackwell
### Dell x NVIDIA NemoClaw Demo Contest 2026

> An AI agent that monitors incoming creative assets, diagnoses technical issues, and generates actionable QC reports — running securely inside NVIDIA OpenShell on a Dell GB10.

---

## The Problem

In M&E post-production, bad files travel silently through the pipeline until someone catches them — usually at the worst possible moment.

A single out-of-spec clip going into a batch upscale job (wrong frame rate, wrong resolution, wrong audio settings) means everything gets reprocessed. Two days lost. Downstream departments affected. Nobody knows what happened because the production manager only assigns tasks — there is no visibility across the pipeline.

CONDUCTOR catches the problem before it starts.

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

| Check | Method | Confidence |
|-------|--------|------------|
| Resolution (4K 3840x2160) | ffprobe | Exact |
| Asymmetric resolution (3840x2048) | ffprobe | Exact |
| Frame rate — flags 18fps, 29fps, 60fps | ffprobe | Exact |
| Audio sample rate (requires 48kHz) | ffprobe | Exact |
| Missing audio track | ffprobe | Exact |
| Short clip under 1 second | ffprobe | Exact |
| Black frames (>0.5s) | ffmpeg blackdetect | Confirmed |
| Freeze frames (>1s) | ffmpeg freezedetect | Confirmed |

All checks use ffprobe and ffmpeg exact values — no thresholds, no guessing, no false positives.

---

## Stack

- Dell GB10 Grace Blackwell (128GB unified memory, NVIDIA GB10 GPU)
- NVIDIA NemoClaw v0.0.41
- NVIDIA OpenShell (Landlock + seccomp + netns isolation)
- OpenClaw TUI 2026.4.24
- Qwen3.6-35B-A3B-FP8 via vLLM (local inference)
- ffprobe / ffmpeg (installed by agent inside sandbox)
- Node.js jsPDF / Python fpdf2 (PDF report generation inside sandbox)

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

### Step 4 — Copy video files into sandbox

```bash
docker cp /path/to/your/videos/. <container_id>:/sandbox/qc2/
```

### Step 5 — Run the QC Agent

In the OpenClaw TUI:

```
You are a QC Agent. Scan /sandbox/qc2/ for all mp4 files.
Run ffprobe on each to get resolution, fps, codec, audio sample rate, duration.
Flag FAIL if: not 4K (3840x2160), not 24fps, file HAS audio AND sample rate
is not 48kHz, or duration under 1 second.
Save report as /sandbox/qc_conductor_report.json and generate
/sandbox/qc_conductor_report.pdf. Helvetica font only, no unicode.
```

### Step 6 — Extract reports

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
- Inference option: `3` (Other OpenAI-compatible endpoint)
- Base URL: `http://192.168.x.x:9494/v1` — use your GB10 LAN IP, **not localhost**
- API key: any string (vLLM local does not require auth)
- Model: `Qwen/Qwen3.6-35B-A3B-FP8`
- Sandbox name: `my-assistant`

> Important: Use your GB10 LAN IP (e.g. `192.168.100.126`), not `localhost`.
> The NemoClaw sandbox runs in a separate network namespace and cannot resolve localhost.

---

## Why NemoClaw + OpenShell

- **Landlock filesystem isolation** — agent cannot access host files outside `/sandbox`
- **seccomp syscall filtering** — blocks dangerous system calls
- **netns network isolation** — agent cannot make unauthorized outbound connections
- **Zero permissions by default** — all access is policy-enforced
- **Inference stays private** — no data leaves the GB10

---

## Hardware

Dell GB10 Grace Blackwell
- NVIDIA GB10 Superchip GPU
- 128GB unified memory (CPU + GPU shared)
- Ubuntu Linux 24.04 LTS (native, not WSL)
- Always-on, offline capable after model download

---

Gil Balderas — Dell Ambassador — M&E Filmmaker — GTC GenJam Winner (Vivemos) — Mexico — shotlock.tech
