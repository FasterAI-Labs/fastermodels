# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## 0.1.1 — 2026-09-21

A card now names what its checks cover, and an INT8 artifact carries the quantization step it paid.

### Added

- A row may carry `notes`, a list of sentences rendered as paragraphs after that row's table. (#6)
- `UNMEASURED` is exported: the spellings `run_gate` accepts for a latency that was not measured. (#6)

### Changed

- The publication-checks line names what it covers — `N/N structural checks passed (they do not
  include the accuracy target, whose verdict is in each table above)` — instead of a bare count that
  read as a contradiction two lines under "not demonstrated". A failure now reads `Not passed: ...`. (#6)
- The Variants table header reads `top-1 gap (pt), worst published form`: a ladder row is the worst
  form each repository publishes, which is not the number that repository's own headline table
  carries, and a reader comparing the two needs to be told. (#6)
- A latency that was not measured is written `not measured`. `run_gate` still reads the old spelling
  `non mesurée` for one release, in conditions 6 and 9. (#6)
- Condition 3 additionally requires, of an INT8 artifact (`manifest['variant']['precision'] ==
  'int8'`), that every row carry a `quantization` record with finite `delta`, `lo`, `hi` and
  `agreement` — the step measured against the FP32 form the artifact was built from — and prints it.
  An INT8 artifact published without that record fails, saying so. (#6)
- Condition 2 prints the cross-precision parity arms alongside the same-precision and
  batch-invariance ones. They are recorded, never thresholded. (#6)

## 0.1.0 — 2026-09-11

First release carrying code. `0.0.1` was a placeholder that reserved the name on PyPI.

### Added

- `FasterModel` publishes both halves an optimized model needs: the factory and its arguments, and
  the constructor arguments of every `Conv2d`, `Linear` and `BatchNorm` leaf, read off the live
  model by `spec_from`. `from_pretrained` rebuilds the architecture from that spec and loads the
  weights strictly, so a spec that disagrees with the checkpoint raises instead of loading in part.
  `wrap` and `from_pretrained` return the instance in eval mode, and keep the recipe and provenance
  as attributes. (#1)
- `state_hash` digests every state dict entry — key, shape, dtype, bytes — which is what a reload is
  compared against. (#1)
- `load` lists what a repo publishes and returns the first of `model.safetensors` (a `FasterModel`),
  `model.torchscript.pt` (a TorchScript module) and `model.onnx` (its local path), or raises naming
  the three. `form=` forces one, and a missing forced form raises the same way. (#3)
- Evaluation harness: `predictions` and `correct_vector` keep the per-image result, so two models
  evaluated on the same images are compared as a paired sample — `wilson`, `paired_delta` (paired
  bootstrap 95 % CI, 2000 resamples, seed 0, and an exact McNemar p, without scipy) and `agreement`.
  Logits that are not finite and empty dataloaders raise rather than score a silent 0 %. (#1)
- `params`, `macs` and `peak_activation_bytes` — weights (a quantized module keeps its weight packed
  outside `parameters()`, so it is counted apart), multiply-accumulates from forward hooks on
  convolutions and linear layers at batch 1, and peak live activations from FX `ShapeProp` walked in
  graph order, where an in-place operation reuses its input's buffer. A model that cannot be traced
  raises instead of reporting 0. (#1)
- `render_card` writes the model card from measured values only — a latency that was not measured
  reads "non mesurée", never 0. Each of the four criteria appears three times: the reference, the
  artifact, and the gap between them — the paired delta in points for top-1, a signed percentage for
  size, memory and MACs, never an N× ratio. (#1, #3)
- `check_card` reads a card back and reports the phrases a reader should not have to trust, plus
  speedup claims carrying neither a device nor a runtime on their line. `FORBIDDEN` is exported. (#1)
- `run_gate` and `gate_passed` run ten publication conditions on a local artifact directory, reading
  what was produced and not what was intended: the weights are reloaded by a fresh interpreter with
  a scrubbed environment, the exported ONNX is read back from the file, and the card is read back
  from disk. A missing interval is not a pass. (#1)
- The accuracy delta is a target on the card rather than a gate refusal: condition 3 asks that the
  paired delta carry a finite interval and reports whether `delta['target']` was met without
  deciding on it, so several variants of one model can be published at different points of the
  ladder. A row carrying a target prints it under its table with the lower bound that settles it,
  and `meta['ladder']` lists the sibling variants with their top-1 gap, size, memory and MACs. (#3)
- Tutorial: load a model from the Hub and check its card. (#4)
- Documentation at <https://FasterAI-Labs.github.io/fastermodels/>.

### Changed

- nbdev 3 layout: `pyproject.toml` carries the project metadata and `[tool.nbdev]`, so `settings.ini`,
  `setup.py` and `MANIFEST.in` are gone and CI moves to the nbdev3 workflows. (#1)
- The card reports megabytes, millions and percentages; Wilson, the bootstrap and McNemar are named
  in one sentence under the criteria instead of next to each number. The recipe is optional, and
  `meta['gate']` renders the publication checks as one line. (#3)
- An interval that does not clear the target reads "not demonstrated", and the card says which of the
  two cases it is: the interval straddles the target, or it lies entirely below it. (#3)
- `license` may be `{'id', 'validated_by'}`; with an empty validator the front matter carries no
  license key and the provenance block says the license is not yet validated. A plain string renders
  as before. (#3)
- `delta['floor']` is read as the former name of `delta['target']` for one release. (#3)
- Gate condition 2 accepts batch-invariance arms; `BATCH_TOL` is 1e-3, above float32 CPU kernel
  rounding (measured here: 4.29e-06 for resnet18, 1.24e-05 for mobilenet_v3_large, 0 for their
  TorchScript INT8 forms) and below the failures it guards. Condition 6 requires size, memory and
  MACs as non-negative integers on every row, naming the criterion a row does not carry. (#1)

### Fixed

- `base_model` goes in the front matter only for a Hub `owner/name` id. A source named by the factory
  that builds it opens the provenance block as `Source model` instead, so the Hub no longer refuses
  the upload. (#2)
- Gate condition 4 had two silent passes: a manifest claiming an ONNX absent from the directory, and
  a fallback to the opset, batch and Q/DQ counts the manifest itself claimed when `onnx` could not be
  imported. Both fail now, with the evidence saying which. (#1)
- The eval harness no longer calls `.to(device).eval()` on the caller's model, and refuses a model
  left in training mode rather than scoring the statistics of the batch. Scripted modules, which
  carry no `training` attribute, are scored instead of raising. (#1)
- `check_card` requires both a device and a runtime on the line of a speedup claim, so "2.3x faster
  on CPU" is reported. (#1)
- `render_card` names the reference field that is missing instead of raising `KeyError` from the
  middle of a table. (#1)

### Removed

- `hf.py`: `PyTorchModelHubMixin` covers upload and download, and the rest of the module was unused. (#1)
