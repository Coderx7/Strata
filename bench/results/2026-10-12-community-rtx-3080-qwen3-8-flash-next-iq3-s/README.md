# Community benchmark on NVIDIA GeForce RTX 3080

Measured on 2026-10-12 with `tools/strata_bench.py` (suite 1, 3 runs per test, medians with ranges). Speeds are the engine's own timings; first-token times are measured by the client over HTTP. Speed only: no correctness check was run.

## Hardware and software

- GPU and VRAM: NVIDIA GeForce RTX 3080, 10 GiB VRAM
- CPU: 12th Gen Intel(R) Core(TM) i7-12700K (12 cores, 20 threads); installed RAM: 62.1 GiB; storage: not measured
- OS: Linux 7.0.0-38-generic; GPU driver: 595.99.02; CUDA: 13.2
- Strata: project 0.1.43, commit 7695859a; engine 0.1.43
- Background workloads and power limits: not recorded

## Model and configuration

- Model and quantization: qwen3.8-flash-next-iq3_s
- Context: 131072 tokens; parallel slots: 1
- Settings: the install's own configuration, unchanged; thinking off for the suite; temperature 0

```text
strata --pack iq3_s --native Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00001-of-00002.gguf --ple-gguf Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00002-of-00002.gguf --expert-profile expert-profile.bin --expert-cache auto --prefill auto --spec 4 --mtp rt --max-context 131072 --kv q4_0 --vision --vram-reserve-mib 700 --pcie-frac 0.00 --spec-min-p 0.70 --adapt-every 1 --adapt-swaps 80 --adapt-decay 0.97
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
| short chat | 46 (43-51) | 0 | 100 (100-100) | 9 | not measured | 46.6 (34.8-57.9) | 0.78 (0.69-0.88) |
| 4K prompt, fresh | 4019 (4019-4019) | 0 (0-0) | 24 cap | 3 | 321.5 (282.9-322.3) | not measured | 12.53 (12.50-14.24) |
| 16K prompt, fresh | 16058 (16058-16058) | 0 (0-0) | 24 cap | 3 | 333.7 (332.2-333.9) | not measured | 48.15 (48.13-48.37) |
| agent session, follow-up turns | 14486 (14462-14508) at the last turn | see below | up to 64 | 3 | see below | 39.2 (23.8-50.5) | 10.24 (5.27-15.38) |

Ranges are the lowest and highest of the runs; `Runs` counts requests that returned a token.

### Agent session (conversation cache)

Cache hit over the follow-up turns: 68.8 % (68.6-68.9); earlier-turn tokens read again per session: 4 (3-73) (earlier turns that the cache did not hold; the 13634 (13633-13635) new tokens of the files are read once); first turn (nothing cached) first token: 3.55 s (2.49-4.29).

| Turn (run 1) | Prompt tokens | Reused | Read | Prompt tok/s | First token (s) | Decode tok/s |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | 609 | 0 | 609 | 145.5 | 4.286 | 33.3 |
| 1 | 3717 | 635 | 3082 | 204.2 | 15.308 | 43.2 |
| 2 | 5250 | 3780 | 1470 | 168.8 | 8.796 | 35.3 |
| 3 | 8694 | 5314 | 3380 | 220.9 | 15.379 | 39.2 |
| 4 | 11615 | 8755 | 2860 | 202.9 | 14.174 | 38.2 |
| 5 | 14508 | 11662 | 2846 | 229.8 | 12.448 | 32.0 |

Per-request data: `results.json`. Engine log lines from the start: `engine-log-excerpt.txt` (when this run started the server).

## Correctness and limitations

Not measured: this is a speed suite. The conversation uses the model's own replies, so token counts differ a little between runs. Machine details come from the server, `nvidia-smi`/`rocm-smi` and the OS; background load and power limits are not recorded. Total time of the suite: 421 s.
