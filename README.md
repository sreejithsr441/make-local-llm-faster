# Make Your Local LLM Faster on Mac: 5 Settings, Measured

Companion code for the Ring Zero video **[Make Your Local LLM Faster on Mac: 5 Settings, Measured](VIDEO_LINK)**.

Five settings people tweak to speed up a local model, and a script to measure each one on your own Mac: one baseline, one change at a time.

| # | Setting | What it really changes | What published tests found |
|---|---|---|---|
| 1 | Quantization (8-bit → 4-bit) | bytes read for every word | **faster writing**: about 1.65× on an M4 Pro |
| 2 | MLX vs GGUF | which engine runs it | **MLX usually writes faster, GGUF reads long prompts faster** |
| 3 | Context length | memory reserved up front | **memory, not speed**, until it spills into swap |
| 4 | KV cache quantization | memory per remembered token | **memory, not speed** |
| 5 | Speculative decoding | guesses words, checks them in one go | **depends on your chip**: faster on M5 Max, slower on M1 Max |

The numbers in that table come from other people's published benchmarks (sources below). **Run the scripts to get yours.**

## Quick start

```bash
git clone https://github.com/sreejithsr441/make-local-llm-faster.git
cd ring-zero-examples/make-local-llm-faster
python3 race.py --list          # which models each server can see
./runs.sh 1                     # setting 1: quantization (pulls two Qwen3.5 9B tags)
open chart.html                 # your results as bars
```

Needs macOS on Apple Silicon, Python 3.9+ (standard library only), [Ollama](https://ollama.com) 0.40 or later, and for setting 5, [llama.cpp](https://github.com/ggml-org/llama.cpp) (`brew install llama.cpp`).

## Files

| File | What it does |
|---|---|
| `race.py` | Times any OpenAI-compatible server: time to first token, writing speed, median of N runs, CSV out |
| `runs.sh` | One block per setting: `./runs.sh 1` … `./runs.sh 5`, or `all` |
| `chart.py` | Medians → `results/summary.csv` and a self-contained `chart.html` |
| `make_doc.py` | Rebuilds `prompts/doc.txt`, the ~3,200-token document for the reading-speed test |
| `prompts/` | `story.txt` (writing), `code.txt` (predictable text for setting 5), `doc.txt` (long prompt) |
| `TESTED.md` | Fill in your machine before you share results |

## The five settings

### 1 · Quantization
Same model, two downloads. On a Mac, writing speed is mostly limited by how many bytes the chip reads for each word, so a smaller file writes faster.

```bash
ollama run qwen3.5:9b-mxfp8 --verbose    # 8-bit, 11 GB (MLX)
ollama run qwen3.5:9b-nvfp4 --verbose    # 4-bit, 7.2 GB (MLX)
```
Watch the first-token time as well: it barely moves, because reading a prompt is limited by compute, not memory.

### 2 · MLX vs GGUF
```bash
ollama run qwen3.5:9b --verbose          # GGUF q4_K_M, 6.6 GB (llama.cpp engine)
ollama run qwen3.5:9b-nvfp4 --verbose    # MLX 4-bit, 7.2 GB
```
Run both with `prompts/doc.txt` too. Check the Ollama server log to confirm which engine each tag uses on your version.

### 3 · Context length
```bash
launchctl setenv OLLAMA_CONTEXT_LENGTH 8192   # then quit and reopen the Ollama app
ollama ps                                     # CONTEXT and PROCESSOR columns
```
Ollama's default depends on GPU memory: 4K under 24 GB, 32K for 24–48 GB, 256K above ([docs](https://docs.ollama.com/context-length)). Use Llama 3.1 8B for this test: Qwen3.5 keeps a KV cache in only one layer in four, so the effect is small.

### 4 · KV cache quantization
```bash
launchctl setenv OLLAMA_KV_CACHE_TYPE q8_0    # f16 (default) · q8_0 ≈ half · q4_0 ≈ quarter
```
Applies to all models and needs Flash Attention ([FAQ](https://docs.ollama.com/faq)). Whether it does anything on Ollama's newer MLX engine isn't documented: compare `ollama ps` SIZE with and without it.

### 5 · Speculative decoding (MTP) in llama.cpp
```bash
# apprentice off (baseline)
llama-server -hf unsloth/Qwen3.6-27B-MTP-GGUF:Q4_K_M -fa on -c 8192 --port 8080
# apprentice on
llama-server -hf unsloth/Qwen3.6-27B-MTP-GGUF:Q4_K_M -fa on -c 8192 --port 8080 \
  --spec-type draft-mtp --spec-draft-n-max 3
```
Then `./runs.sh 5`. Use the same temperature for on and off. The acceptance rate is in llama-server's log. Check `race.py --list` for the model id llama-server reports and update `SERVERS["llamacpp"]` if it differs.

## Fair-test rules (built into race.py)

- One untimed warm-up per lane, then the **median of 5 runs**
- A unique first line per run, so prompt caching can't help
- Generate **500+ tokens**: short samples flatter the speed
- One model server running at a time, plugged in, Low Power Mode off
- Change exactly one setting against the baseline

## Troubleshooting

| Symptom | Fix |
|---|---|
| Speed collapses to 1–3 tokens/sec | the model spilled out of GPU memory: `ollama ps` should say `100% GPU`; lower the context length or use 4-bit |
| An env var changes nothing | the macOS app only reads it after `launchctl setenv` **and** an app restart; then check which engine is running |
| Much slower than expected | plug in and switch off Low Power Mode |
| `no tokens came back` | the model id doesn't match: run `python3 race.py --list` |

`runs.sh` restores the default context length and KV cache type at the end of settings 3 and 4.

## Reference numbers (published, not measured here)

| Claim | Number | Source |
|---|---|---|
| M4 Pro 20-core, Llama 2 7B, writing: Q8_0 → Q4_0 | 30.7 → 50.7 tok/s | [llama.cpp #4167](https://github.com/ggml-org/llama.cpp/discussions/4167) |
| Same, prompt reading | 449.6 → 439.8 tok/s | [llama.cpp #4167](https://github.com/ggml-org/llama.cpp/discussions/4167) |
| M2 Max, Qwen3.6-35B-A3B 4-bit: MLX vs llama.cpp writing | 86.9 vs 57.8 tok/s | [RaynarDM benchmarks](https://huggingface.co/datasets/RaynarDM/apple-silicon-llm-benchmarks) |
| Same, prompt reading | 296 vs 685 tok/s | [RaynarDM benchmarks](https://huggingface.co/datasets/RaynarDM/apple-silicon-llm-benchmarks) |
| M2 Max, KV cache f16 vs q8_0 | 58.55 vs 57.25 tok/s (error bars overlap) | [RaynarDM benchmarks](https://huggingface.co/datasets/RaynarDM/apple-silicon-llm-benchmarks) |
| M5 Max, MTP off → on, dense Qwen3.6-27B | 18 → 32 tok/s | [stared/benching-local-llms](https://github.com/stared/benching-local-llms-on-apple-silicon) |
| M1 Max, MTP, Qwen3.5-9B | 25.3 baseline → 22.4 (n-max 0) → 19.3 (n-max 6) | [llama.cpp #23752](https://github.com/ggml-org/llama.cpp/issues/23752) |
| Ollama default context by GPU memory | 4K / 32K / 256K | [Ollama docs](https://docs.ollama.com/context-length) |
| Qwen3.5 9B tag sizes | mxfp8 11 GB · nvfp4 7.2 GB · q4_K_M 6.6 GB | [Ollama library](https://ollama.com/library/qwen3.5/tags) |

## Share your numbers

Post your chip, RAM, and setting 5 on vs off (both tokens/sec numbers) in the video's comments. Different chips give different answers, and that's the data nobody has yet.

---
Ring Zero · [YouTube](CHANNEL_LINK)
