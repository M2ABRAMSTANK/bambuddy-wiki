---
title: Build Plate Check
description: Check that the build plate is empty before a print starts — with OpenCV or an AI vision backend
---

# Build Plate Check

Before a print starts, Bambuddy can check the camera to confirm the build plate is actually empty, and pause the job if it isn't. Two detection backends are available:

- **OpenCV** (default) — compares a live camera frame against reference photos of the empty plate, captured during calibration, and flags anything that differs beyond a threshold.
- **AI** (opt-in) — sends one snapshot to a vision-language model over an OpenAI-compatible API and uses the model's own judgment of whether the plate is empty. No calibration, no reference photos.

The AI backend is an *alternative*, not a replacement — OpenCV remains the default, and nothing changes for existing installs until you switch.

!!! info "When the check runs"
    Once at print start (only for printers with the check enabled), when you run a manual check from the printer card, and when you press **Test connection** in Settings. Never during a print and never on a timer — during-print monitoring is [AI Failure Detection](failure-detection.md), a separate feature.

## Setup

Settings → **Build Plate Check** tab:

| Field | What to put there |
|---|---|
| Detection backend | `OpenCV` (default) or `AI` — the global choice |
| AI backend URL | An OpenAI-compatible `/v1` endpoint — a local Ollama server, or a hosted API |
| Model name | The model to request, e.g. `qwen2.5vl:7b` for local Ollama |
| API key (optional) | Leave empty for a local server that doesn't require one |

Use the **Test connection** button to confirm the endpoint and model work before relying on it — it sends a synthetic gray test frame, not a real photo of your printer, so it's safe to use before anything else is set up.

The same tab also shows:

- **Monitored printers** — which printers run the check before a print (the per-printer enable toggle, same one as on the printer card), and which backend each printer uses: the global default, or a per-printer override.
- **Status** — the effective configuration at a glance: global backend, endpoint and model, and the resolved backend for every printer.

## Per-printer backend

Mixed fleet? Each printer can override the global backend choice — from the **Monitored printers** list, or from the printer card's plate-check dialog (**Backend for this printer**: Global default / OpenCV / AI). Printers without an override follow the global setting.

## The decision panel

After a manual check, the plate-check dialog shows a **decision panel**: which backend produced the verdict, the verdict itself, its confidence, and the evidence — the pixel-difference percentage for OpenCV, or the model's stated reasoning for AI. When the AI backend is active, the calibration references and region-of-interest editor shown below it apply to the OpenCV backend only; the AI backend evaluates the full camera frame.

## Worked example — local Ollama (privacy-preserving)

- **AI backend URL**: `http://<your-ollama-host>:11434/v1`
- **Model name**: `qwen2.5vl:7b`
- **API key**: leave empty

This keeps every snapshot on your own network — nothing leaves your LAN. Pull the model first (`ollama pull qwen2.5vl:7b`) and give it a moment to load on first use; see "Cold start" below.

## Hosted OpenAI compatibility (untested)

The AI backend sends requests using the OpenAI `chat/completions` format, so a hosted OpenAI endpoint may be configurable in principle. **Hosted OpenAI has not been tested against a live service**; no API key was available for verification. Automated tests use mocked HTTP responses to check request construction and response handling. Those tests do not establish that OpenAI, or another hosted provider, works end to end. The only live service exercised so far is local Ollama.

For experimentation, the endpoint URL would be `https://api.openai.com/v1`, with a vision-capable model name and an API key accepted by that service. Treat this as an unverified configuration; hosted compatibility is theoretical until tested against a live provider. Other services that describe themselves as OpenAI-compatible may differ in accepted request options.

**Test connection** probes request options because not every endpoint accepts the same shape. It starts with `max_tokens` and strict `json_schema` mode, then can fall back to `max_completion_tokens` and separately to plain `json_object` mode, for up to three requests. A successful probe confirms only that this endpoint accepted those options; it does not verify live provider behavior. Discovered fallback options are cached in memory for the running Bambuddy process. After a restart, run **Test connection** again if the endpoint requires a fallback shape; otherwise the default request shape is used.

## Privacy

**Snapshots only leave your local network if you configure a non-local endpoint, and only because you explicitly typed that endpoint in.** With a local Ollama URL (loopback or an RFC-1918 address), every snapshot stays on your LAN. If you configure a public URL, Bambuddy warns you in Settings before you save — but it will not stop you. There is no default hosted endpoint, and no snapshot is sent anywhere until you configure a URL and select `AI` as a backend.

## Two things worth stating plainly

1. **The per-printer "Build Plate Check" toggle still controls whether the check runs at all.** The backend choice (global, or per-printer override) selects *which* engine runs the check — it doesn't turn the check on or off. Enable the check for each printer you want covered, then choose the backend.
2. **The AI backend needs no calibration.** No reference photos, no region-of-interest, no "not yet calibrated" state — it works the moment a valid base URL and model are set. Any calibration references and ROI you've set up are used by the OpenCV backend only.

## Which printers benefit

Bambu's H2-series (H2/H2D/H2C), X2D, and P2S already run Bambu's own native foreign-object detection — this feature doesn't add anything for those models. It's most useful on printers **without** native detection: **X1, X1C, P1P, P1S, and A1**.

## Fail-open behavior

The AI backend fails **open** on every error class — it assumes the plate is empty and lets the print proceed, exactly like the OpenCV backend's existing behavior on a capture failure. It never blocks a print because of a connectivity or configuration problem.

| Situation | What you'll see | What happens |
|---|---|---|
| Base URL or model name left empty | "AI backend not configured" | Fails open, no network call attempted |
| Request times out (e.g. cold local model) | "request timed out" | Fails open |
| Can't reach the endpoint | "connection failed" | Fails open |
| Endpoint returns an HTTP error (401, 500, …) | "AI backend returned an error" | Fails open |
| Response isn't valid JSON | "invalid response from AI backend" (after one automatic retry) | Fails open |
| Camera frame is corrupt/truncated | "camera frame could not be processed" | Fails open |
| Database error reading settings | "AI backend unavailable" | Fails open |

Messages are short and generic on purpose — they never include your configured server address, even in an error. Full detail goes to the server log.

## Cold start / timeout

The request timeout is fixed at 60 seconds. Loading a vision model into memory for the first time (or after idle eviction) can take 25–30 seconds on typical local hardware; once warm, a check normally completes in one to two seconds. If the *first* check after an idle period is slow, that's expected. If checks time out consistently even warm, use **Test connection** to isolate whether the problem is the endpoint, the model name, or something else.

## Troubleshooting

- **"AI backend not configured"** — base URL or model name is empty. Fill both in.
- **"connection failed"** — the URL is wrong, or the server isn't reachable from where Bambuddy runs (firewall, VLAN isolation).
- **"AI backend returned an error" (401)** — API key missing or wrong for an endpoint that requires one.
- **"AI backend returned an error" (404 or similar)** — the model name doesn't exist on that endpoint; check spelling and that the model is pulled/deployed.
- **Test button hangs then times out** — likely a cold-start local model; try again once loaded.
