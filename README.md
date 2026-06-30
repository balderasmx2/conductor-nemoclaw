# CONDUCTOR — QC Agent for M&E Post-Production
## Running inside NVIDIA NemoClaw on Dell GB10 Grace Blackwell
### Dell x NVIDIA NemoClaw Demo Contest 2026

> An AI agent that monitors incoming creative assets, diagnoses technical issues, and generates actionable QC reports — running inside NVIDIA NemoClaw on a Dell GB10.

---

## The Problem

In M&E post-production, bad files travel silently through the pipeline until someone catches them — usually at the worst possible moment.

A single out-of-spec clip going into a batch upscale job means everything gets reprocessed. Wrong frame rate, wrong resolution, wrong audio settings — Topaz processes them all anyway. Two days lost. Downstream departments affected. Nobody knows what happened.

CONDUCTOR catches the problem before it starts.

---

## What It Does

CONDUCTOR is a QC Agent for Media & Entertainment post-production. It runs inside NVIDIA NemoClaw on a Dell GB10 Grace Blackwell, scanning video files for technical issues and generating a full QC report — automatically, locally, with zero cloud dependency.

The agent:
1. Scans a folder of MP4/MOV files
2. Runs ffprobe on each file to extract exact technical specs
3. Diagnoses issues with precise failure reasons
4. Generates a JSON + PDF report — actionable, shareable, timestamped
5. All managed by NVIDIA NemoClaw with sandboxed security isolation

---

## Demo Results

- 47 real M&E production files scanned
- 5 critical issues diagnosed
- 49-page PDF report generated automatically
- 0 cloud API calls
- 0 humans in the loop

---

## What CONDUCTOR Diagnoses

All checks use ffprobe exact values — no estimation, no thresholds, no false positives.

| Check | Method | Example failure |
|-------|--------|-----------------|
| Resolution must be 4K (3840x2160) | ffprobe | 1280x720 — not 4K |
| Asymmetric resolution | ffprobe | 3840x2048 — height wrong |
| Frame rate must be 24fps | ffprobe | 18fps, 29fps, 60fps flagged |
| Audio sample rate must be 48kHz | ffprobe | 44.1kHz — sync drift risk |
| Missing audio track | ffprobe | No audio stream found |
| Short clip under 1 second | ffprobe | 0.166s — unusable for broadcast |

---

## Stack

- Dell GB10 Grace Blackwell (128GB unified memory, NVIDIA GB10 GPU)
- NVIDIA NemoClaw v0.0.41
- Hermes Agent (NousResearch) — agent runtime
- Qwen3.6-35B-A3B-FP8 via vLLM (local inference)
- ffprobe / ffmpeg — video analysis
- Python fpdf2 — PDF report generation

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

### Step 2 — Start Hermes Agent (Terminal 2)

```bash
source ~/.bashrc && hermes
```

### Step 3 — Run the QC Agent

In the Hermes TUI:

```
You are a QC Agent. Scan [your folder] for all mp4 files.
Run ffprobe -v quiet -print_format json -show_streams -show_format on each file.
Flag FAIL if: not 4K (3840x2160), not 24fps, file HAS audio AND sample rate
is not 48kHz, or duration under 1 second.
Save qc_report.json and generate qc_report.pdf. Helvetica font only, no unicode.
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

> Important: Use your GB10 LAN IP (e.g. `192.168.100.126`), not `localhost`.
> NemoClaw sandboxes run in a separate network namespace and cannot resolve localhost.

---

## Why NVIDIA NemoClaw

NVIDIA NemoClaw manages the security sandbox where the agent runs. It provides:

- **Landlock filesystem isolation** — agent cannot access files outside its designated folder
- **seccomp syscall filtering** — blocks dangerous system calls
- **Network namespace isolation** — agent cannot make unauthorized outbound connections
- **Zero permissions by default** — all access is policy-enforced
- **Inference stays private** — no data leaves the GB10

---

## Hardware

Dell GB10 Grace Blackwell
- NVIDIA GB10 Superchip GPU
- 128GB unified memory (CPU + GPU shared)
- Ubuntu Linux 24.04 LTS (native, not WSL)
- Offline capable after model download

---

Gil Balderas — Dell Ambassador — M&E Filmmaker 
