# MediScribe Patches — `randsley/mlx-swift-lm`

This branch (`mediscribe-fixes`) is based on upstream tag `2.30.6` of
`ml-explore/mlx-swift-lm` and carries the patches listed below.

**Consumer**: [randsley/MediScribe](https://github.com/randsley/MediScribe)
**Pinned revision**: `12bc2f35ec3f2a1dca9666d4f61b7769586de39e`

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

## Upgrading from upstream

When a new upstream `ml-explore/mlx-swift-lm` release is needed:

```bash
cd /Users/nigelrandsley/GitHub/mlx-swift-lm
git checkout mediscribe-fixes
git fetch upstream   # add remote: git remote add upstream https://github.com/ml-explore/mlx-swift-lm
git merge <new-tag>
```

After merging, verify the override is still present:

```bash
grep -A2 "processorTypeOverrides" Libraries/MLXVLM/VLMModelFactory.swift
```

Expected output must include `"gemma3": "Gemma3Processor"`. If a conflict or
upstream change removed it, re-apply the patch manually.

Then push and update the pinned revision in the MediScribe project:

```bash
git push origin mediscribe-fixes

# In MediScribe repo:
# 1. Update revision in MediScribe.xcodeproj/project.pbxproj
# 2. Update revision in MediScribe.xcodeproj/project.xcworkspace/xcshareddata/swiftpm/Package.resolved
```
