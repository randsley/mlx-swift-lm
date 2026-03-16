# MediScribe Patches — `randsley/mlx-swift-lm`

This branch (`mediscribe-fixes`) is based on upstream tag `2.30.6` of
`ml-explore/mlx-swift-lm` and carries the patches listed below.

**Consumer**: [randsley/MediScribe](https://github.com/randsley/MediScribe)
**Pinned revision**: `b3ba409d730ca933b33d17215b46e58ebdf3e0e9`

---

## Patch 1 — Gemma3 processor override in `VLMModelFactory`

**Commit**: `12bc2f3`
**File**: `Libraries/MLXVLM/VLMModelFactory.swift`

### Problem

MedGemma (Google, `mlx-community/medgemma-4b-it-4bit`) ships with the
following in its `preprocessor_config.json`:

```json
"processor_class": "PaliGemmaProcessor"
```

This is architecturally correct — MedGemma is built on PaliGemma2. However,
`PaliGemmaProcessor.prepare()` requires exactly one image:

```swift
// Paligemma.swift line 466–471
public func prepare(input: UserInput) throws -> LMInput {
    switch input.images.count {
    case 0: throw VLMError.imageRequired   // ← thrown for text-only input
    case 1: break
    default: throw VLMError.singleImageAllowed
    }
```

This caused every text-only inference call (SOAP note generation, referral
drafting) to fail immediately with:

```
Generation failed: An image is required for this operation.
```

Vision inference (imaging findings, lab extraction) was unaffected because it
always supplies exactly one image.

### Fix

Added `"gemma3": "Gemma3Processor"` to `processorTypeOverrides` inside
`VLMModelFactory._load()`. This overrides the `processor_class` field from the
config file for any model whose `config.json` declares
`"model_type": "gemma3"`, ensuring `Gemma3Processor` is always used.
`Gemma3Processor.prepare()` handles both multimodal (image + text) and
text-only input correctly.

```diff
         let processorTypeOverrides: [String: String] = [
-            "mistral3": "Mistral3Processor"
+            "mistral3": "Mistral3Processor",
+            "gemma3": "Gemma3Processor",
         ]
```

### Why not patch `preprocessor_config.json` instead?

The MediScribe deployment guide does instruct operators to set
`"processor_class": "Gemma3Processor"` in `preprocessor_config.json` when
pre-loading the model onto devices. However, relying on the config file value
alone would silently break any deployment that uses the unmodified file from
Hugging Face. The factory-level override is authoritative and independent of
the file on disk.

---

## Patch 2 — Gemma3 chat template fallback in `Gemma3Processor`

**Commit**: `b3ba409`
**File**: `Libraries/MLXVLM/Models/Gemma3.swift`

### Problem

`Gemma3Processor.prepare()` calls `tokenizer.applyChatTemplate(messages:)` unconditionally. Some `mlx-community` MedGemma quantizations ship a `tokenizer_config.json` that omits the `chat_template` field. When encountered, swift-transformers throws `TokenizerError.missingChatTemplate`:

```
Generation failed: This tokenizer does not have a chat template, and no template was passed.
```

This affected all text-only inference (SOAP notes, referral drafting). Vision inference also broke once Patch 1 switched MedGemma from `PaliGemmaProcessor` to `Gemma3Processor`.

### Fix

Catch `TokenizerError.missingChatTemplate` in `Gemma3Processor.prepare()` and retry with a hardcoded Gemma3/MedGemma Jinja fallback template passed via `.literal()`. The fallback is only used when the tokenizer configuration omits the template — if the tokenizer provides its own, it is used as before.

```swift
var promptTokens: [Int]
do {
    promptTokens = try tokenizer.applyChatTemplate(messages: messages)
} catch TokenizerError.missingChatTemplate {
    promptTokens = try tokenizer.applyChatTemplate(
        messages: messages,
        chatTemplate: .literal(Self.gemma3FallbackChatTemplate)
    )
}
```

The fallback template is the standard Gemma3 instruction-tuning template
(`<start_of_turn>user\n…<end_of_turn>\n<start_of_turn>model\n`), which is
correct for MedGemma 4B IT.

---

## Upgrading from upstream

When a new upstream `ml-explore/mlx-swift-lm` release is needed:

```bash
cd /Users/nigelrandsley/GitHub/mlx-swift-lm
git checkout mediscribe-fixes
git fetch upstream   # add remote: git remote add upstream https://github.com/ml-explore/mlx-swift-lm
git merge <new-tag>
```

After merging, verify both patches are still present:

```bash
# Patch 1
grep -A2 "processorTypeOverrides" Libraries/MLXVLM/VLMModelFactory.swift
# Must include: "gemma3": "Gemma3Processor"

# Patch 2
grep "gemma3FallbackChatTemplate\|missingChatTemplate" Libraries/MLXVLM/Models/Gemma3.swift
# Must show both the constant and the catch clause
```

If either was removed by a conflict or upstream change, re-apply the relevant patch manually.

Then push and update the pinned revision in the MediScribe project:

```bash
git push origin mediscribe-fixes

# In MediScribe repo:
# 1. Update revision in MediScribe.xcodeproj/project.pbxproj
# 2. Update revision in MediScribe.xcodeproj/project.xcworkspace/xcshareddata/swiftpm/Package.resolved
```
