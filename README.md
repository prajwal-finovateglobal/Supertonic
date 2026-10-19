# Supertonic Bench

Local playground for [Supertonic 3](https://huggingface.co/Supertone/supertonic-3). The browser is a Next.js app. Speech is synthesized by a FastAPI service using the official `supertonic` Python package and ONNX Runtime. Nothing is sent to a cloud TTS API.

## Run

```bash
docker compose up --build
```

Open http://localhost:3000. The API is also on http://localhost:8000/docs.

The first image build downloads the open weights (about 400 MB) from Hugging Face into the backend image. Later starts reuse that image. Custom voice JSON files and batch audio live in the `bench-data` volume.

## What the open model actually exposes

`TTS.synthesize` in supertonic 1.3.1 accepts:

| Control | Default | Range | Notes |
| --- | --- | --- | --- |
| Voice | M1 | M1–M5, F1–F5 | The only free presets in the Hugging Face repo. One voice covers every language. |
| Language | en in this bench | 31 ISO codes + `na` | `na` is the unknown-language fallback. |
| Steps | 8 | 1–100, slider 1–20 | Recommended 5–12. Published speed tables used 2 and 5. |
| Speed | 1.05 | 0.7–2.0 | Higher is faster. |
| Chunk length | auto | 10–2000 | Auto is 300 characters, or 120 for Korean. |
| Silence | 0.3 s | 0–3 s in the bench | Inserted only between chunks. |
| Verbose | off | flag | Writes chunk progress to the API log. |

There is no pitch, emotion, or seed parameter. Expression tags are inline text: `<laugh>`, `<breath>`, and `<sigh>` are the ones Supertone documents. Seven more (`throatclear`, `cough`, `sneeze`, `sniff`, `groan`, `yawn`, `thinking`) show up in community notes and are unreliable. Tags are most likely to land in English, Korean, and Japanese, especially when repeated at the start or end of a sentence.

Supertone Play’s larger voice library and Voice Builder’s reference-audio cloning are hosted products. This bench can import a Voice Builder JSON (`style_ttl` and `style_dp`). It cannot clone a WAV, because the public ONNX bundle has no speaker encoder.

Weights are OpenRAIL-M (no harm, no impersonation without consent). The Python SDK is MIT.

## CSV map

Upload a headered CSV, write spoken lines with placeholders like `Hello [name], your EMI is [EMI]`, then map each placeholder to a CSV column. The placeholder name does not have to match the header.

- Combine fillers with rows by cycling, using only the first, picking at random, or expanding every filler for every row.
- Pick a filename identity column (usually a name). Files look like `001_ava-chen_M1_en.wav`, or `001_ava-chen_line02_M1_en.wav` when multiple fillers are used.
- Optional voice and language columns override the shared settings per row.

A five-row EMI sample is built in on the CSV map tab. Jobs keep running on the server after you close the browser tab (as long as Docker stays up). Reopen the CSV map tab to see progress.

CSV map has two modes:
- **Chunk compose** (default): you build an ordered list of static and generative segments. Static phrases are synthesized once and reused; generative segments pull from CSV columns and cache identical values across rows; clips are stitched into one WAV per row.
- **Full sentence**: classic fillers with `[placeholders]` mapped to columns, regenerating the whole line per row.

Choose concurrent workers (up to the loaded pool) to synthesize several lines at once. Each worker loads its own ONNX session, so more workers use more RAM.

Jobs support **Pause** / **Resume** / **Cancel**. On a job, set **Audios per batch** (e.g. 100); the UI auto-calculates how many exportable ZIPs you get. Tick which batches to export (one ZIP, selected bundle, or all + full). CSV map accepts up to 1000 rows.

**Output Hz** (Studio + CSV): choose 8 / 16 / 22.05 / 44.1 kHz. The model always synthesizes at 44.1 kHz; lower rates are downsampled for telephony (8 kHz). That makes call-ready smaller WAVs — it does not speed up ONNX inference.

**Speed** comes from **Steps** (`total_steps`), not Hz. Official published RTF tables used **2 steps** (Fast). The SDK default is now 8 (Quality) and is much slower. Use Fast / Balanced (5) / Quality chips in Studio and CSV.

Local Docker also matters: each synth worker loads a full ONNX session (~400MB+). **8 workers** on a ~5GB Docker Desktop VM pegged RAM (~3GB RSS, 200+ threads) and turned a ~0.5s synth into **6–8s**. Default pool is now **2** workers (`SYNTH_WORKERS`). Raise it only if you give Docker more memory.

Pool size is set with `SYNTH_WORKERS` / `SYNTH_WORKERS_MAX` in Docker Compose (current defaults: 8 / 10).

## Local development

```bash
cd backend && python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000

cd frontend && npm install && npm run dev
```

The dev server proxies `/api` to `http://127.0.0.1:8000`. Override that with `API_INTERNAL_URL`.
# Supertonic
# Supertonic
