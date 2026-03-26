# BrushNet Inference Environment Diff

This note records the difference between the original `.venv` state and the adjusted state that successfully ran:

`python examples/brushnet/test_brushnet.py`

## Scope

This document only covers the environment changes made during the local inference smoke test on 2026-03-26.

Project path:

`/home/ychm/Python/BrushNet`

Python interpreter:

`/home/ychm/Python/BrushNet/.venv/bin/python`

## Result

After the changes below, `examples/brushnet/test_brushnet.py` completed successfully and generated:

`/home/ychm/Python/BrushNet/output.png`

## Version Diff

| Package | Original `.venv` | Adjusted `.venv` | Why it changed |
| --- | --- | --- | --- |
| `huggingface_hub` | `0.36.2` | `0.25.2` | The local `src/diffusers` code imports `cached_download`, which is not available in newer `huggingface_hub`. |
| `transformers` | `4.57.6` | `4.30.2` | `4.57.6` requires `huggingface_hub>=0.34.0`, which conflicts with the older `diffusers` code in this repo. |
| `setuptools` | `82.0.1` | `80.10.2` | The installed `accelerate==0.20.3` imports `pkg_resources`, which was not importable in the newer setuptools state. |

Versions observed after alignment:

- `torch==1.12.1+cu116`
- `accelerate==0.20.3`
- `transformers==4.30.2`
- `huggingface_hub==0.25.2`
- `setuptools==80.10.2`

## What Failed In The Original Environment

### 1. `diffusers` vs `huggingface_hub`

The first failure happened before inference started.

Observed error:

```text
ImportError: cannot import name 'cached_download' from 'huggingface_hub'
```

Reason:

- This repository contains a local `diffusers` fork under `src/diffusers/`.
- That code still expects the older `huggingface_hub` API.
- The original `.venv` had a much newer `huggingface_hub==0.36.2`.

## 2. `transformers` vs downgraded `huggingface_hub`

After downgrading `huggingface_hub`, the next failure was:

```text
huggingface-hub>=0.34.0,<1.0 is required ... but found huggingface-hub==0.25.2
```

Reason:

- The original `.venv` had `transformers==4.57.6`.
- That version expects a newer `huggingface_hub`.
- This conflicted with the older API requirement coming from the local `diffusers` code.

## 3. `accelerate` vs `setuptools`

After aligning `transformers`, the next failure was:

```text
ModuleNotFoundError: No module named 'pkg_resources'
```

Reason:

- `accelerate==0.20.3` imports `pkg_resources`.
- In the original effective setuptools state, `pkg_resources` was not importable.
- Downgrading `setuptools` restored compatibility.

## Commands Used

The environment was adjusted with `uv` against the existing `.venv`:

```bash
uv pip install --python .venv/bin/python 'huggingface_hub<0.26'
uv pip install --python .venv/bin/python 'transformers==4.30.2'
uv pip install --python .venv/bin/python 'setuptools<81'
```

## Practical Interpretation

The main issue was not missing inference files. The project already had the minimum local model files needed for `examples/brushnet/test_brushnet.py`.

The real blocker was that `.venv` contained a newer package mix than this repository's BrushNet code expects.

In short:

- The repo code is closer to an older `diffusers` / `transformers` ecosystem.
- The original `.venv` had newer packages that were internally inconsistent with that code.
- Once those versions were aligned, local inference ran successfully.

## Current Caution

The current adjusted environment is suitable for this BrushNet local inference test, but it may not be ideal for unrelated newer tooling in the same `.venv`.

If this environment must also support other modern scripts, it is safer to:

- keep a dedicated BrushNet inference virtual environment, or
- lock the working versions into a requirements or `uv` lock workflow.
