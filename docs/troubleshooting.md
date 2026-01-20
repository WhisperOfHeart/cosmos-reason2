# Troubleshooting

<!--TOC-->

______________________________________________________________________

**Table of Contents**

- [Resources](#resources)
- [FAQ](#faq)
  - [Where is requirements.txt](#where-is-requirementstxt)
- [Errors](#errors)
  - [Docker Build Lockfile Mismatch](#docker-build-lockfile-mismatch)
  - [Docker Build Dependency Download Failures](#docker-build-dependency-download-failures)
  - [OpenAI API Connection Error](#openai-api-connection-error)
  - [PTXAS Error](#ptxas-error)

______________________________________________________________________

<!--TOC-->

## Resources

* [vLLM Troubleshooting](https://docs.vllm.ai/en/latest/usage/troubleshooting/#hangs-loading-a-model-from-disk)

## FAQ

### Where is requirements.txt

For most use cases, you should not need `requirements.txt`. `pip` can install directly from `pyproject.toml`. See the [nightly Dockerfile](../docker/nightly.Dockerfile) for an example installing into the NVIDIA vLLM container.

You can generate a `requirements.txt` file with [`uv export`](https://docs.astral.sh/uv/concepts/projects/export/).

```shell
uv export --format requirements.txt --output-file requirements.txt
```

## Errors

### Docker Build Lockfile Mismatch

Error message: `The lockfile at uv.lock needs to be updated, but --locked was provided.`

Fix: regenerate `uv.lock` with the same `uv` and Python versions used in the Docker build (uv 0.8.12 + Python 3.12.11), then commit the updated lockfile.

```shell
python -m pip install --user uv==0.8.12
~/.local/bin/uv python install 3.12.11
UV_PYTHON=3.12.11 ~/.local/bin/uv lock --project .
```

If you build the CUDA 12.8 image, verify the lock resolves the `cu128` extra:

```shell
~/.local/bin/uv sync --locked --extra cu128 --no-install-project --no-editable
```

### Docker Build Dependency Download Failures

Error message: `Failed to extract archive: nvidia_cusolver_cu12-...` or DNS errors fetching `nvidia-cosmos.github.io` / `wheels.vllm.ai`.

Fixes:

1. Ensure the build runner has enough free disk and retry the build (the CUDA wheels are very large and extraction can fail under disk/IO pressure).
2. Make sure the build environment can reach the custom indexes used by CUDA extras:
   * `https://download.pytorch.org/whl/cu128`
   * `https://nvidia-cosmos.github.io/cosmos-dependencies/...`
   * `https://wheels.vllm.ai/...`
3. If your environment blocks these domains, configure an internal mirror and point `uv` to it during lock generation.

### OpenAI API Connection Error

Error message: `openai.APIConnectionError: Connection error.`

Check the server log. Common issues:

1. Server is not fully started. Wait until you see `Application startup complete.`.
1. Server died due to Out of Memory (OOM).
    1. Verify your GPU satisfies the [minimum requirements](../README.md#inference).
    1. Reduce `--max-model-len`. Recommended range: 8192 - 16384.

### PTXAS Error

Error message: `(EngineCore_DP0 pid=1477831) ptxas fatal   : Value 'sm_121a' is not defined for option 'gpu-name'`

Fix: Use CUDA 13.0 [Docker container](../README.md#setup)

Alternatively, to use the virtual environment, set `TRITON_PTXAS_PATH` to your system `PTXAS`:

```shell
export TRITON_PTXAS_PATH="/usr/local/cuda/bin/ptxas"
```

Your system CUDA version must match the torch CUDA version.
