# Container Image Plan — northstar-infer

## Overview

The inference service is packaged as a multi-stage Docker image named **`northstar-infer`**. The image is built by the CI/CD pipeline on every model promotion and tagged with the model version. It runs as a non-root user on a minimal `python:3.11-slim` base with no build tooling in the final layer.

---

## Build Stages

| Stage | Base | Purpose | Reaches runtime? |
|---|---|---|---|
| `builder` | `python:3.11-slim` | Compile Python wheels from locked `requirements.txt` | No — packages copied via `COPY --from` |
| `model-fetcher` | `python:3.11-slim` | Download and verify ONNX model from S3 | No — artifact copied via `COPY --from` |
| `runtime` | `python:3.11-slim` | Final image: packages + model + app source | **Yes** |

The `builder` and `model-fetcher` stages are discarded after the build. AWS CLI, gcc, pip cache, and compilation artefacts never appear in the pushed image.

---

## Bake vs Mount Decision

**We bake the model into the image.**

| Criterion | Bake (chosen) | Mount at runtime |
|---|---|---|
| **Pod startup time** | Model available immediately on container start | Requires S3/NFS fetch on every pod start (~5–15 s for 50 MB) |
| **Availability dependency** | None at runtime — model is on local disk | Pod cannot serve until model storage is reachable |
| **Image size** | Larger (~50 MB more per version) | Smaller base image, model separate |
| **Version coupling** | Image tag = exact model version (strong guarantee) | Model version and image version can drift if config is wrong |
| **Rollback** | Rollback = router flag flip (< 5 s); old image still running | Rollback requires re-fetching old model artifact |
| **Reproducibility** | Any image tag is fully self-contained and reproducible | Reproducibility depends on artifact store retention |
| **Security** | Model artifact verified by SHA-256 at build time | Integrity check must happen at runtime; adds complexity |

The capacity plan specifies 3 replicas with rolling restarts during blue/green deploy (see `ADR-0002`). A mount-at-runtime strategy would add 15–45 seconds of unavailability per replica during restart (3 × 15 s staggered = ~45 s of reduced capacity). Baking eliminates this: the new container is ready to serve as soon as the process starts (~2 s for ONNX model load).

The re-ranker model is **~50 MB** (ONNX). This is small enough that the image size penalty is acceptable. If the model grew beyond ~500 MB, the mount strategy would be reconsidered.

---

## Base Image Rationale

**`python:3.11-slim`** (Debian Bookworm slim variant)

| Option | Pros | Cons | Decision |
|---|---|---|---|
| `python:3.11-slim` | Small, well-maintained, Debian security patches | Slightly larger than Alpine | **Chosen** |
| `python:3.11-alpine` | Smallest possible base | musl libc incompatible with many pre-built wheels (ONNX Runtime, NumPy); requires compilation | Rejected |
| `ubuntu:22.04` + Python | Full OS, familiar | ~3× larger than slim; unnecessary packages | Rejected |
| `gcr.io/distroless/python3` | Minimal attack surface | No shell; hard to debug in staging; pip not available | Rejected for now; revisit if security posture tightens |

The only runtime system dependency beyond Python is **`libgomp1`** (OpenMP), required by ONNX Runtime for CPU thread parallelism. This is the single `apt-get install` in the runtime stage.

---

## Image Size Estimate

| Layer | Estimated size |
|---|---|
| `python:3.11-slim` base | ~45 MB |
| `libgomp1` | ~0.5 MB |
| Python packages (FastAPI, ONNX Runtime, Redis client, Gunicorn, Uvicorn, Prometheus client) | ~120 MB |
| ONNX model artifact | ~50 MB |
| Application source (`src/`) | ~1 MB |
| **Total (compressed)** | **~170 MB** |
| **Total (uncompressed on disk)** | **~220 MB** |

This is within the target of < 300 MB uncompressed. ONNX Runtime is the largest single dependency (~80 MB uncompressed); it cannot be trimmed further without switching to a custom-built minimal runtime.

---

## Build Arguments

| ARG | Required | Description |
|---|---|---|
| `MODEL_S3_URI` | Yes | Full S3 URI of the ONNX model artifact (e.g. `s3://northstar-mlflow-artifacts/northstar-reranker/v2.4.1/model.onnx`) |
| `MODEL_SHA256` | Yes | SHA-256 hex digest of the model file; verified at build time |

Both are injected by the CI/CD pipeline from the MLflow registry entry. A build will fail if the downloaded artifact does not match `MODEL_SHA256`.

---

## Image Tagging Scheme

```
northstar-infer:<model_version>-<git_sha_short>
```

Examples:
- `northstar-infer:v2.4.1-a3f9c12` — production tag
- `northstar-infer:v2.4.1-a3f9c12-staging` — staging tag (same image, different deploy target)
- `northstar-infer:latest` — always points to current production champion; updated atomically with the router flag flip

The CI/CD pipeline also pushes `northstar-infer:<model_version>` as a stable reference for rollback (see `runbooks/rollback.md`).

---

## Runtime Configuration (Environment Variables)

| Variable | Default | Description |
|---|---|---|
| `MODEL_PATH` | `/app/model/model.onnx` | Path to baked-in ONNX model |
| `REDIS_URL` | — | Redis Cluster connection URL (injected by Kubernetes secret) |
| `PORT` | `8080` | HTTP listen port |
| `OMP_NUM_THREADS` | `2` | ONNX Runtime CPU thread count; matches Gunicorn worker × thread config |
| `LOG_LEVEL` | `info` | Application log level |
| `MODEL_VERSION` | — | Injected by CI/CD; emitted as `X-Model-Version` response header |
| `FEATURE_FLAG_SDK_KEY` | — | LaunchDarkly SDK key for A/B router (injected by Kubernetes secret) |

---

## Security Notes

- Runs as **non-root** (`uid=10001`, `gid=10001`). No privilege escalation path.
- No shell in the default `CMD`; shell access requires explicit `docker exec` override.
- AWS credentials used only in `model-fetcher` build stage via `--mount=type=secret`; they do not persist in any layer.
- Model artifact integrity verified by SHA-256 at build time; a tampered artifact fails the build before it can be pushed.
- `pip install` uses locked `requirements.txt` from pip-compile; no floating version ranges.
- Trivy image scan is run in CI before push (see `.github/workflows/deploy-model.yml`); HIGH/CRITICAL CVEs block promotion.
