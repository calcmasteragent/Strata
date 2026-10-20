# Community benchmark on Quadro RTX 8000

Measured on 2026-10-20 by calcmasteragent with `tools/strata_bench.py` (suite 1, 3 runs per test, medians with ranges). Speeds are the engine's own timings; first-token times are measured by the client over HTTP. Speed only: no correctness check was run.

## Hardware and software

- GPU and VRAM: Quadro RTX 8000, 45 GiB VRAM
- CPU: Intel(R) Xeon(R) CPU E5-2680 v3 @ 2.50GHz (12 cores, 24 threads); installed RAM: 125.3 GiB; storage: not measured
- OS: Linux 4.18.0-553.34.1.el8_10.x86_64; GPU driver: 560.35.05; CUDA: 12.6
- Strata: engine 0.1.44.1 (the release installed on this machine); the benchmark scripts come from the project repository at commit 296b59d2; source build of the engine not verified
- Background workloads and power limits: not recorded

## Model and configuration

- Model and quantization: qwen3.8-flash-next-iq3_s
- Context: 524288 tokens; parallel slots: 3
- Settings: the install's own configuration, unchanged; thinking off for the suite; temperature 0

```text
<host> --pack iq3_s --native Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00001-of-00002.gguf --ple-gguf Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00002-of-00002.gguf --expert-profile expert-profile.bin --expert-cache auto --prefill auto --spec 4 --mtp rt --max-context 524288 --rope-scaling yarn --rope-scale 2 --kv k8v4 --mmap-experts --kv-resident 32768 --vision --vram-reserve-mib 700 --spec-min-p 0.5 --pool-workers 6 --adapt-every 1 --adapt-swaps 160 --adapt-decay 0.97
```

## Method

Run by `python tools/strata_bench.py` against the HTTP API of a Strata server (already running); model loading is not timed. One warm-up chat and a 1K prompt are discarded first. Every prompt starts with a unique tag, so the conversation cache cannot help the fresh-prompt tests.

- short chat: the three questions of `tools/ab_engine.py`, up to 100 new tokens, 3 runs
- 4K / 16K prompts: generated word lists sized by a calibration request, 24 new tokens, 3 fresh prompts each
- agent session: system prompt and four tools, then 5 turns that each add about 1500 words of a project file and continue from the model's own earlier answers (the way agent clients resend a conversation), 64 new tokens per turn, 3 sessions; the cache numbers come from the answer's usage (`cached_tokens`) and timings (`prompt_n`)
- concurrency (when run): N requests sent at the same moment, total tokens over the wall time of the burst, latency over all requests

## Results

| Configuration | Actual prompt tokens | Reused tokens | Generated tokens | Runs | Prompt tok/s median and range | Decode tok/s median and range | TTFT seconds median and range |
| --- | ---: | ---: | ---: | ---: | --- | --- | --- |
| short chat | 46 (43-51) | 0 | 100 (100-100) | 9 | not measured | 17.8 (10.3-51.7) | 0.89 (0.44-2.04) |
| 4K prompt, fresh | 4019 (4019-4019) | 0 (0-0) | 24 cap | 3 | 690.7 (564.1-705.1) | not measured | 5.86 (5.74-7.16) |
| 16K prompt, fresh | 16058 (16058-16058) | 0 (0-0) | 24 cap | 3 | 846.6 (780.0-883.1) | not measured | 19.02 (18.24-20.64) |
| agent session, follow-up turns | 9096 (9073-9098) at the last turn | see below | up to 64 | 3 | see below | 36.4 (5.4-92.5) | 5.64 (4.93-60.53) |

Ranges are the lowest and highest of the runs; `Runs` counts requests that returned a token.

### Agent session (conversation cache)

Cache hit over the follow-up turns: 71.1 % (70.4-71.1); earlier-turn tokens read again per session: 4 (3-213) (earlier turns that the cache did not hold; the 8208 (8207-8208) new tokens of the files are read once); first turn (nothing cached) first token: 3.82 s (3.58-4.50).

| Turn (run 1) | Prompt tokens | Reused | Read | Prompt tok/s | First token (s) | Decode tok/s |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | 609 | 0 | 609 | 160.6 | 3.823 | 16.0 |
| 1 | 2274 | 634 | 1640 | 239.0 | 6.896 | 23.9 |
| 2 | 3980 | 2267 | 1713 | 259.8 | 6.63 | 40.3 |
| 3 | 5686 | 3973 | 1713 | 242.5 | 7.104 | 26.0 |
| 4 | 7392 | 5679 | 1713 | 293.1 | 5.887 | 59.6 |
| 5 | 9098 | 7456 | 1642 | 293.1 | 5.645 | 38.3 |

Per-request data: `results.json`. Engine log lines from the start: `engine-log-excerpt.txt` (when this run started the server).

## Correctness and limitations

Not measured: this is a speed suite. The conversation uses the model's own replies, so token counts differ a little between runs. Machine details come from the server, `nvidia-smi`/`rocm-smi` and the OS; background load and power limits are not recorded. Total time of the suite: 348 s. One earlier attempt on the same day produced no numbers: the bench was run without the server's API key and every request returned HTTP 401; this report is the re-run with the key supplied. The agent-session TTFT range includes one 60.5 s outlier (run 3, turn 3) not seen in the other runs; decode tok/s is wide for the same reason (the card also serves other traffic on this machine).
