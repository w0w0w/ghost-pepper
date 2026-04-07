# Development

## Cleanup Model Probe

Use the cleanup probe to run the local cleanup models directly and inspect each stage of the cleanup pipeline:

- resolved prompt
- corrected input
- raw model output
- sanitized model output
- final cleaned output

Run it through the wrapper script:

```sh
./scripts/cleanup-model-probe.sh --model fast --input "Okay, it's running now." --thinking none
```

The wrapper builds the `CleanupModelProbe` target on first use and then runs the built executable from `build/cleanup-probe-cli`.

### Common examples

Compare current app behavior against suppressed thinking:

```sh
./scripts/cleanup-model-probe.sh --model fast --input "Okay, it's running now." --thinking none
./scripts/cleanup-model-probe.sh --model fast --input "Okay, it's running now." --thinking suppressed
```

Enter interactive mode and keep the selected model loaded:

```sh
./scripts/cleanup-model-probe.sh --model fast --thinking suppressed
```

Type `:quit` or send EOF to exit interactive mode.

Pass supporting OCR text inline:

```sh
./scripts/cleanup-model-probe.sh --model full --input "ship it" --window-context "PR title: ship qwen cleanup fix"
```

Pass supporting OCR text from a file:

```sh
./scripts/cleanup-model-probe.sh --model full --input "ship it" --window-context-file /tmp/window.txt
```

### Notes

- `--model` accepts the following values:

  | Value | Alias for | Model file | Size |
  |---|---|---|---|
  | `qwen35_0_8b_q4_k_m` | — | `Qwen3.5-0.8B-Q4_K_M.gguf` | ~535 MB |
  | `fast` | `qwen35_2b_q4_k_m` | `Qwen3.5-2B-Q4_K_M.gguf` | ~1.3 GB |
  | `full` | `qwen35_4b_q4_k_m` | `Qwen3.5-4B-Q4_K_M.gguf` | ~2.8 GB |

  `qwen35_0_8b_q4_k_m` is the app's default cleanup model (the compact/very-fast option). `fast` and `full` are convenience aliases for the 2B and 4B variants respectively.

- `--thinking` accepts `none`, `suppressed`, or `enabled`
- omitting `--input` starts interactive mode
- if you want a clean rebuild, remove `build/cleanup-probe-cli` before running the wrapper again

## How the CLI loads Qwen models

Ghost Pepper uses Qwen 3.5 models in [GGUF](https://github.com/ggerganov/ggml/blob/master/docs/gguf.md) format quantized to Q4_K_M, loaded via [LLM.swift](https://github.com/eastriverlee/LLM.swift) (which in turn uses llama.cpp under the hood). The following steps describe exactly what happens when you run the probe CLI or when the app runs a cleanup pass:

### 1. Parse the `--model` flag

`CleanupModelProbeCLI.parse()` reads the `--model` argument and resolves it to a `LocalCleanupModelKind` enum value:

```
fast               → LocalCleanupModelKind.qwen35_2b_q4_k_m
full               → LocalCleanupModelKind.qwen35_4b_q4_k_m
qwen35_0_8b_q4_k_m → LocalCleanupModelKind.qwen35_0_8b_q4_k_m  (default in the app)
qwen35_2b_q4_k_m   → LocalCleanupModelKind.qwen35_2b_q4_k_m
qwen35_4b_q4_k_m   → LocalCleanupModelKind.qwen35_4b_q4_k_m
```

### 2. Resolve the model descriptor

Each `LocalCleanupModelKind` maps to a `CleanupModelDescriptor` stored in `TextCleanupManager.cleanupModels`. The descriptor holds the GGUF file name, the Hugging Face download URL, and the max token count:

| Kind | File | URL | Max tokens |
|---|---|---|---|
| `qwen35_0_8b_q4_k_m` | `Qwen3.5-0.8B-Q4_K_M.gguf` | `huggingface.co/unsloth/Qwen3.5-0.8B-GGUF` | 2048 |
| `qwen35_2b_q4_k_m` | `Qwen3.5-2B-Q4_K_M.gguf` | `huggingface.co/unsloth/Qwen3.5-2B-GGUF` | 2048 |
| `qwen35_4b_q4_k_m` | `Qwen3.5-4B-Q4_K_M.gguf` | `huggingface.co/unsloth/Qwen3.5-4B-GGUF` | 4096 |

### 3. Download on demand (if missing)

Model files are cached at:

```
~/Library/Application Support/GhostPepper/models/<filename>.gguf
```

`TextCleanupManager.loadModel(kind:)` checks whether the `.gguf` file already exists at that path. If it does not, it downloads the file from Hugging Face over HTTPS with live progress reporting before continuing. The downloaded file is moved atomically into the models directory on completion.

### 4. Load the GGUF model via LLM.swift

Once the file is on disk, model loading happens off the main thread in a detached `Task`:

```swift
let llm = LLM(from: path, maxTokenCount: descriptor.maxTokenCount)
llm.useResolvedTemplate(systemPrompt: TextCleaner.defaultPrompt)
llm.temp = 0.1
```

`LLM(from:maxTokenCount:)` is the LLM.swift entry point that calls into llama.cpp to memory-map the GGUF file and initialise the model context. The low temperature (`0.1`) produces deterministic, focused output suitable for a text cleanup task. The `useResolvedTemplate` call selects the Qwen chat template embedded in the GGUF and sets the system prompt.

### 5. Run the cleanup pipeline

After loading, `TextCleanupManager.probe()` drives the inference:

1. **Deterministic pre-corrections** — `DeterministicCorrectionEngine` applies any user-configured preferred transcriptions and misheard-word replacements to the raw transcription before it reaches the model.
2. **Format input** — `TextCleaner.formatCleanupInput()` wraps the corrected text in `<USER-INPUT>…</USER-INPUT>` tags.
3. **Build the final prompt** — `CleanupPromptBuilder` assembles the system prompt, optional OCR window context, and correction hints into the prompt that is set on the `LLM` instance.
4. **Inference** — `llm.respond(to: modelInput, thinking: thinkingMode)` runs the Qwen model. A 15-second timeout guards against a hung model.
5. **Sanitise output** — `TextCleaner.sanitizeCleanupOutput()` strips any `<think>…</think>` reasoning blocks that Qwen may emit when thinking is enabled.
6. **Deterministic post-corrections** — `DeterministicCorrectionEngine` applies post-cleanup corrections to the sanitised text.

The probe CLI prints every stage of this pipeline so you can inspect exactly what changed and why.
