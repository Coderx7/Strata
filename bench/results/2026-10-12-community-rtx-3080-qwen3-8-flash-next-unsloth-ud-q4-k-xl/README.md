# Community benchmark on NVIDIA GeForce RTX 3080

Measured on 2026-10-12 with `tools/strata_bench.py` (suite 1, 3 runs per test, medians with ranges). Speeds are the engine's own timings; first-token times are measured by the client over HTTP. Speed only: no correctness check was run.

## Hardware and software

- GPU and VRAM: NVIDIA GeForce RTX 3080, 10 GiB VRAM
- CPU: 12th Gen Intel(R) Core(TM) i7-12700K (12 cores, 20 threads); installed RAM: 62.1 GiB; storage: not measured
- OS: Linux 7.0.0-38-generic; GPU driver: 595.99.02; CUDA: 13.2
- Strata: project 0.1.43, commit 7695859a; engine 0.1.43
- Background workloads and power limits: not recorded

## Model and configuration

- Model and quantization: qwen3.8-flash-next-unsloth-ud-q4_k_xl
- Context: 262144 tokens; parallel slots: 1
- Settings: the install's own configuration, unchanged; thinking off for the suite; temperature 0

```text
strata --pack unsloth-ud-q4_k_xl --native Qwen3.8-Flash-Next-UD-Q4_K_XL-00001-of-00004.gguf --expert-profile expert-profile.bin --expert-cache auto --prefill auto --spec 4 --mtp rt --max-context 262144 --kv q4_0 --kv-resident 32768 --resident-budget-gib 36 --vision --vram-reserve-mib 700 --pcie-frac 0.20 --spec-min-p 0.70
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
| short chat | 46 (43-51) | 0 | 100 (100-100) | 9 | not measured | 9.8 (6.3-13.0) | 4.12 (2.67-6.09) |
| 4K prompt, fresh | 4019 (4019-4019) | 0 (0-0) | 24 cap | 3 | 29.6 (29.0-29.7) | not measured | 135.85 (135.60-138.86) |
| 16K prompt, fresh | 16058 (16058-16058) | 0 (0-0) | 24 cap | 3 | 29.9 (29.9-30.0) | not measured | 536.84 (534.85-536.85) |
| agent session, follow-up turns | 14497 (14473-14504) at the last turn | see below | up to 64 | 3 | see below | 8.2 (6.2-9.1) | 125.17 (71.82-142.79) |

Ranges are the lowest and highest of the runs; `Runs` counts requests that returned a token.

### Agent session (conversation cache)

Cache hit over the follow-up turns: 68.7 % (68.5-68.8); earlier-turn tokens read again per session: 73 (4-144) (earlier turns that the cache did not hold; the 13635 (13634-13636) new tokens of the files are read once); first turn (nothing cached) first token: 54.87 s (54.42-58.02).

| Turn (run 1) | Prompt tokens | Reused | Read | Prompt tok/s | First token (s) | Decode tok/s |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | 609 | 0 | 609 | 11.2 | 54.872 | 6.8 |
| 1 | 3713 | 632 | 3081 | 21.7 | 142.335 | 8.2 |
| 2 | 5246 | 3776 | 1470 | 19.3 | 76.542 | 7.8 |
| 3 | 8690 | 5239 | 3451 | 24.6 | 140.656 | 8.5 |
| 4 | 11614 | 8754 | 2860 | 22.6 | 126.972 | 9.1 |
| 5 | 14497 | 11651 | 2846 | 23.7 | 120.185 | 6.4 |

Per-request data: `results.json`. Engine log lines from the start: `engine-log-excerpt.txt` (when this run started the server).

## Correctness and limitations

Not measured: this is a speed suite. The conversation uses the model's own replies, so token counts differ a little between runs. Machine details come from the server, `nvidia-smi`/`rocm-smi` and the OS; background load and power limits are not recorded. Total time of the suite: 4323 s.
