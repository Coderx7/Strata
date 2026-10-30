# Community benchmark on NVIDIA GeForce RTX 3080

Measured on 2026-10-28 with `tools/strata_bench.py` (suite 1, 3 runs per test, medians with ranges). Speeds are the engine's own timings; first-token times are measured by the client over HTTP. Speed only: no correctness check was run.

## Hardware and software

- GPU and VRAM: NVIDIA GeForce RTX 3080, 10 GiB VRAM
- CPU: 12th Gen Intel(R) Core(TM) i7-12700K (12 cores, 20 threads); installed RAM: 62.1 GiB; storage: not measured
- OS: Linux 7.0.0-38-generic; GPU driver: 595.99.02; CUDA: 13.2
- Strata: project 0.1.44.1, commit 296b59d2; engine 0.1.44.1
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
| short chat | 46 (43-51) | 0 | 100 (100-100) | 9 | not measured | 8.2 (5.7-11.2) | 5.79 (3.66-8.08) |
| 4K prompt, fresh | 4019 (4019-4019) | 0 (0-0) | 24 cap | 3 | 32.2 (31.9-32.4) | not measured | 125.03 (124.07-126.25) |
| 16K prompt, fresh | 16058 (16058-16058) | 0 (0-0) | 24 cap | 3 | 33.2 (32.7-33.3) | not measured | 483.16 (482.91-491.40) |
| agent session, follow-up turns | 14552 (14543-14558) at the last turn | see below | up to 64 | 3 | see below | 8.8 (7.3-11.4) | 105.85 (60.62-128.03) |

Ranges are the lowest and highest of the runs; `Runs` counts requests that returned a token.

### Agent session (conversation cache)

Cache hit over the follow-up turns: 68.6 % (68.5-68.6); earlier-turn tokens read again per session: 73 (73-74) (earlier turns that the cache did not hold; the 13729 (13729-13730) new tokens of the files are read once); first turn (nothing cached) first token: 46.82 s (46.25-48.98).

| Turn (run 1) | Prompt tokens | Reused | Read | Prompt tok/s | First token (s) | Decode tok/s |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | 609 | 0 | 609 | 12.5 | 48.985 | 6.8 |
| 1 | 3744 | 621 | 3123 | 24.4 | 128.031 | 8.8 |
| 2 | 5277 | 3737 | 1540 | 23.7 | 65.268 | 8.7 |
| 3 | 8721 | 5340 | 3381 | 26.8 | 126.354 | 7.5 |
| 4 | 11602 | 8740 | 2862 | 27.0 | 105.984 | 10.2 |
| 5 | 14558 | 11660 | 2898 | 27.4 | 105.848 | 7.9 |

Per-request data: `results.json`. Engine log lines from the start: `engine-log-excerpt.txt` (when this run started the server).

## Correctness and limitations

Not measured: this is a speed suite. The conversation uses the model's own replies, so token counts differ a little between runs. Machine details come from the server, `nvidia-smi`/`rocm-smi` and the OS; background load and power limits are not recorded. Total time of the suite: 3884 s.
