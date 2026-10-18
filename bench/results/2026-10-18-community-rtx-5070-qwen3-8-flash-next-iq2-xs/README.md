# Community benchmark on NVIDIA GeForce RTX 5070

Measured on 2026-10-18 with `tools/strata_bench.py` (suite 1, 3 runs per test, medians with ranges). Speeds are the engine's own timings; first-token times are measured by the client over HTTP. Speed only: no correctness check was run.

## Hardware and software

- GPU and VRAM: NVIDIA GeForce RTX 5070, 12 GiB VRAM
- CPU: Intel(R) Core(TM) i5-14600KF (14 cores, 20 threads); installed RAM: 31.8 GiB (2 x 16 GB DDR4-3600); storage: the model files are on an aigo P2000 NVMe drive, which read at about 1.4-1.6 GB/s in an earlier measurement on this machine
- PCIe: the GPU is on PCIe 5.0 x16; Resizable BAR is off
- OS: Windows 10 Pro 22H2 (10.0.19045), Chinese locale; GPU driver: 591.74; CUDA: 13.1
- Strata: project 0.1.45, commit not a git checkout (release zip); engine 0.1.45 (official release binary)
- Background workloads and power limits: not recorded; the GPU also drives the display

## Model and configuration

- Model and quantization: qwen3.8-flash-next-iq2_xs
- Context: 32768 tokens; parallel slots: 1
- Settings: the daily configuration of this PC (low-RAM mode, `--mmap-experts`, because the 35.5 GB of experts do not fit in 31.8 GiB of RAM), with one change: the optional control-vector file that setup offers (`--control-vector-scaled`, not a speed setting) was left out so the numbers compare with other reports; thinking off for the suite; temperature 0

```text
strata.exe --pack iq2_xs --native Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00001-of-00002.gguf --ple-gguf Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00002-of-00002.gguf --expert-profile expert-profile.bin --expert-cache auto --prefill auto --spec 4 --mtp rt --max-context 32768 --kv int8 --mmap-experts --pcie-frac 0.55 --spec-min-p 0.70
```

Environment of the config: `STRATA_FETCH_THREADS=16`

## Method

Run by `python tools/strata_bench.py` against the HTTP API of a Strata server (already running); model loading is not timed. One warm-up chat and a 1K prompt are discarded first. Every prompt starts with a unique tag, so the conversation cache cannot help the fresh-prompt tests.

- short chat: the three questions of `tools/ab_engine.py`, up to 100 new tokens, 3 runs
- 4K / 16K prompts: generated word lists sized by a calibration request, 24 new tokens, 3 fresh prompts each
- agent session: system prompt and four tools, then 5 turns that each add about 1500 words of a project file and continue from the model's own earlier answers (the way agent clients resend a conversation), 64 new tokens per turn, 3 sessions; the cache numbers come from the answer's usage (`cached_tokens`) and timings (`prompt_n`)
- concurrency (when run): N requests sent at the same moment, total tokens over the wall time of the burst, latency over all requests

Two workarounds were needed to run the 0.1.45 tool on this PC; neither changes what is measured:

- The server was started separately with `serve/server.py` and the tool was pointed at it with `--url http://127.0.0.1:8080 --config <the config>`, because the tool's own server start stops with `NameError: name 'port' is not defined` (already reported in #2864, #2872 and #2908; fixes in #2919 and #2994).
- The tool was run with `PYTHONUTF8=1`, because on a Chinese-locale Windows it cannot read `CMakeLists.txt` (`UnicodeDecodeError: 'gbk' codec`, already reported in #2792; fix in #2879).

## Results

| Configuration | Actual prompt tokens | Reused tokens | Generated tokens | Runs | Prompt tok/s median and range | Decode tok/s median and range | TTFT seconds median and range |
| --- | ---: | ---: | ---: | ---: | --- | --- | --- |
| short chat | 46 (43-51) | 0 | 100 (100-100) | 9 | not measured | 20.2 (12.9-43.7) | 1.95 (0.76-2.64) |
| 4K prompt, fresh | 4019 (4019-4019) | 0 (0-0) | 24 cap | 3 | 58.3 (51.7-98.9) | not measured | 68.98 (40.70-77.75) |
| 16K prompt, fresh | 16058 (16058-16058) | 0 (0-0) | 24 cap | 3 | 127.5 (124.3-173.4) | not measured | 125.99 (92.69-129.22) |
| agent session, follow-up turns | 14573 (14566-14576) at the last turn | see below | up to 64 | 3 | see below | 6.4 (0.7-13.5) | 48.24 (33.34-86.87) |

Ranges are the lowest and highest of the runs; `Runs` counts requests that returned a token.

### Agent session (conversation cache)

Cache hit over the follow-up turns: 68.8 % (68.6-68.8); earlier-turn tokens read again per session: 4 (2-74) (earlier turns that the cache did not hold; the 13717 (13717-13717) new tokens of the files are read once); first turn (nothing cached) first token: 20.46 s (19.49-26.17).

| Turn (run 1) | Prompt tokens | Reused | Read | Prompt tok/s | First token (s) | Decode tok/s |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | 609 | 0 | 609 | 23.4 | 26.166 | 10.4 |
| 1 | 3759 | 639 | 3120 | 83.0 | 37.713 | 5.7 |
| 2 | 5292 | 3823 | 1469 | 36.9 | 39.893 | 9.2 |
| 3 | 8725 | 5356 | 3369 | 65.6 | 51.512 | 4.8 |
| 4 | 11628 | 8766 | 2862 | 60.5 | 47.355 | 3.9 |
| 5 | 14566 | 11667 | 2899 | 45.9 | 63.301 | 6.4 |

Per-request data: `results.json`. No `engine-log-excerpt.txt`: the server was started separately (see Method), so the tool did not capture its log.

## Correctness and limitations

Not measured: this is a speed suite. The conversation uses the model's own replies, so token counts differ a little between runs. Machine details come from the server, `nvidia-smi`/`rocm-smi` and the OS; background load and power limits are not recorded. Total time of the suite: 1709 s.

## Notes

- **Low RAM and a slower drive.** The RTX 5070 Ti report with 31.6 GiB of RAM (#2791, 0.1.44.1, Windows 11, WD SN770) reads a 4K prompt at 270 tok/s and a 16K prompt at 448 tok/s; this PC reads them at 58 and 128 tok/s. The GPU (16 GB against 12 GB) and the drive differ, so this run alone does not say which one matters more.
- **What earlier measurements on this PC showed** (0.1.39, with Windows performance counters during a 20K-token prompt): the drive read only 0.2-0.5 GB/s while about 300 MB of RAM stayed available, so the prompt path was limited by page faults on the mapped `experts.bin`, not by the drive's speed. Those pages also pushed the decode's cached experts out of RAM, which matches the slow decode in the agent turns here (6.4 tok/s).
- **The short-chat decode** (20.2 tok/s, range 12.9-43.7) was measured right after the warm-up; in low-RAM mode the first requests after a start read many experts from the drive, so it is lower than this PC's decode once the file cache is warm.
- Not tested here: a context above 32K (`--long`), `--clients`, a RAM budget (`--resident-budget-gib`), other models.
