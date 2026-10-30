# Community benchmark on NVIDIA GeForce RTX 3080

Measured on 2026-10-28 with `tools/strata_bench.py` (suite 1, 3 runs per test, medians with ranges). Speeds are the engine's own timings; first-token times are measured by the client over HTTP. Speed only: no correctness check was run.

## Hardware and software

- GPU and VRAM: NVIDIA GeForce RTX 3080, 10 GiB VRAM
- CPU: 12th Gen Intel(R) Core(TM) i7-12700K (12 cores, 20 threads); installed RAM: 62.1 GiB; storage: not measured
- OS: Linux 7.0.0-38-generic; GPU driver: 595.99.02; CUDA: 13.2
- Strata: project 0.1.44.1, commit 296b59d2; engine 0.1.44.1
- Background workloads and power limits: not recorded

## Model and configuration

- Model and quantization: qwen3.8-flash-next-iq3_s
- Context: 128144 tokens; parallel slots: 1
- Settings: the install's own configuration, unchanged; thinking off for the suite; temperature 0

```text
strata --pack iq3_s --native Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00001-of-00002.gguf --ple-gguf Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00002-of-00002.gguf --expert-profile expert-profile.bin --expert-cache auto --prefill auto --spec 4 --mtp rt --max-context 128144 --kv q4_0 --vision --vram-reserve-mib 700 --pcie-frac 0.00 --spec-min-p 0.70 --adapt-every 1 --adapt-swaps 160 --adapt-decay 0.97
```

## Method

Run by `python tools/strata_bench.py` against the HTTP API of a Strata server (started by the benchmark from the install configuration); model loading is not timed. One warm-up chat and a 1K prompt are discarded first. Every prompt starts with a unique tag, so the conversation cache cannot help the fresh-prompt tests.

- short chat: the three questions of `tools/ab_engine.py`, up to 100 new tokens, 3 runs
- 4K / 16K prompts: generated word lists sized by a calibration request, 24 new tokens, 3 fresh prompts each
- agent session: system prompt and four tools, then 5 turns that each add about 1500 words of a project file and continue from the model's own earlier answers (the way agent clients resend a conversation), 64 new tokens per turn, 3 sessions; the cache numbers come from the answer's usage (`cached_tokens`) and timings (`prompt_n`)
- concurrency (when run): N requests sent at the same moment, total tokens over the wall time of the burst, latency over all requests

## Results

| Configuration | Actual prompt tokens | Reused tokens | Generated tokens | Runs | Prompt tok/s median and range | Decode tok/s median and range | TTFT seconds median and range |
| --- | ---: | ---: | ---: | ---: | --- | --- | --- |
| short chat | 46 (43-51) | 0 | 100 (100-100) | 9 | not measured | 50.6 (48.1-60.5) | 0.72 (0.63-0.79) |
| 4K prompt, fresh | 4019 (4019-4019) | 0 (0-0) | 24 cap | 3 | 817.1 (639.5-830.1) | not measured | 4.95 (4.87-6.32) |
| 16K prompt, fresh | 16058 (16058-16058) | 0 (0-0) | 24 cap | 3 | 853.9 (851.8-854.3) | not measured | 18.84 (18.83-18.88) |
| agent session, follow-up turns | 14550 (14531-14602) at the last turn | see below | up to 64 | 3 | see below | 48.7 (36.5-55.0) | 3.93 (3.84-7.52) |

Ranges are the lowest and highest of the runs; `Runs` counts requests that returned a token.

### Agent session (conversation cache)

Cache hit over the follow-up turns: 68.6 % (68.4-68.8); earlier-turn tokens read again per session: 73 (3-144) (earlier turns that the cache did not hold; the 13729 (13728-13730) new tokens of the files are read once); first turn (nothing cached) first token: 2.28 s (2.24-3.01).

| Turn (run 1) | Prompt tokens | Reused | Read | Prompt tok/s | First token (s) | Decode tok/s |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | 609 | 0 | 609 | 207.8 | 3.009 | 38.4 |
| 1 | 3753 | 631 | 3122 | 422.7 | 7.523 | 44.3 |
| 2 | 5286 | 3746 | 1540 | 282.1 | 5.525 | 46.2 |
| 3 | 8730 | 5279 | 3451 | 523.9 | 6.653 | 49.5 |
| 4 | 11632 | 8770 | 2862 | 459.6 | 6.288 | 54.3 |
| 5 | 14550 | 11651 | 2899 | 526.3 | 5.547 | 45.1 |

Per-request data: `results.json`. Engine log lines from the start: `engine-log-excerpt.txt` (when this run started the server).

## Correctness and limitations

Not measured: this is a speed suite. The conversation uses the model's own replies, so token counts differ a little between runs. Machine details come from the server, `nvidia-smi`/`rocm-smi` and the OS; background load and power limits are not recorded. Total time of the suite: 200 s.
