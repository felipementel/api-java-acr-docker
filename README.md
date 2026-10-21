# ACR Tasks bug repro — `layer does not exist` on image export

Minimal reproduction of a **deterministic ACR Tasks failure** when building a multi-stage Dockerfile. This repository contains only what is needed to reproduce the issue: a small Spring Boot app ([source/](source/)) and two Dockerfiles ([docker/](docker/)).

- [docker/Dockerfile](docker/Dockerfile) — 4-stage build with Spring Boot layered JAR extraction → **fails** on ACR Tasks
- [docker/Dockerfile.nolayers](docker/Dockerfile.nolayers) — 3-stage variant without the extraction stage → **succeeds** on ACR Tasks

Both build and push successfully with Docker BuildKit locally, so the Dockerfiles themselves are valid.

## Probable cause

The failure happens **inside ACR Tasks' classic builder export phase**, not in the Dockerfile: every run completes all 25 build steps and then fails to export the final image because a freshly-built layer blob "does not exist" in the build node's view of the registry.

The trigger pattern is the **two-level `COPY --from=<stage>`**: the final `runtime` stage copies directories from an intermediate `layers` stage that was itself produced from another stage (Spring Boot layered JAR extraction via `java -Djarmode=tools -jar app.jar extract --layers --launcher`). Removing that intermediate stage (single-level `COPY --from=build`) makes the same build succeed on the same registry, same day.

Docker BuildKit (used by `docker buildx build`) handles the same Dockerfile correctly, which points to a bug/limitation in the classic builder used by ACR Tasks when materializing intermediate-stage layers into the final image.

## Environment

- Registry: `acrmestrado001` (Brazil South, Standard SKU)
- Azure CLI: 2.91.0
- Date: 2026-10-21 (~14:05–17:38 UTC)
- Build platform: linux/amd64
- Failed run IDs: [`cq3`](logs/acr-run-cq3.log), [`cq4`](logs/acr-run-cq4.log), [`cq6`](logs/acr-run-cq6.log), [`cq7`](logs/acr-run-cq7.log), [`cq8`](logs/acr-run-cq8.log), [`cq9`](logs/acr-run-cq9.log) (with `noCache: true` via `az acr run` + task.yaml), [`cqa`](logs/acr-run-cqa-fresh-tag.log) (fresh, never-used tag)
- Succeeded run IDs: [`cqc`](logs/acr-run-cqc-SUCCESS-nolayers.log) (3-stage variant `Dockerfile.nolayers`), `cq5` (only succeeded because it was a 100% cache hit — 48s)
- Full run logs are attached in [`logs/`](logs/) — each run ID above links directly to its log file

## Command that FAILS (deterministic)

```powershell
az acr build --registry acrmestrado001 `
  --file ./docker/Dockerfile `
  --image financeiro-api-abc-java:scan-39612725954 `
  --platform linux/amd64 `
  ./source
```

Error (always at the same step — the 4th `COPY --from=layers`):

```
Step 21/25 : COPY --from=layers /build/extracted/application/           ./
failed to export image: failed to create image: failed to get layer
sha256:f4471c947b142026e813d3a7f48f0fd665fb144157a9dca9af072ab648c8b6f1: layer does not exist
Container failed during run: build. No retries remaining.
```

## Command that WORKS (same Dockerfile, same context, same machine)

```powershell
# after: az acr login --name acrmestrado001
wsl docker buildx build `
  --platform linux/amd64 `
  -f docker/Dockerfile `
  -t acrmestrado001.azurecr.io/financeiro-api-abc-java:scan-39612725954 `
  ./source --push
```

## A/B evidence — the trigger is the intermediate `layers` stage

| Dockerfile | Stages | Pattern | ACR Tasks | BuildKit (local) |
|---|---|---|---|---|
| [`docker/Dockerfile`](docker/Dockerfile) | 4 (`deps` → `build` → `layers` → `runtime`) | two-level `COPY --from` (runtime copies 4 dirs from the `layers` stage) | ❌ fails deterministically at export | ✅ works |
| [`docker/Dockerfile.nolayers`](docker/Dockerfile.nolayers) | 3 (`deps` → `build` → `runtime`) | single-level `COPY --from=build` (fat JAR) | ✅ works (run `cqc`) | ✅ works |

## What was ruled out

- **Transient failure**: 7 failed runs across ~3.5 hours, always at the same step (Step 21/25). The failing layer digest differs every run, so it is a freshly-built layer that the export phase cannot find.
- **Build cache corruption**: a run with `noCache: true` (via `az acr run` + task.yaml, run `cq9`) failed identically.
- **Tag/manifest corruption**: a brand-new, never-used tag failed identically (run `cqa`); the target repository did not even exist in the registry (no successful export ever completed).
- **Dockerfile defect**: the same Dockerfile + context builds and pushes successfully with BuildKit.

## Repro steps

1. Clone this repo.
2. Pick an ACR registry and run the failing command above (context is `./source`, Dockerfile is `./docker/Dockerfile`).
3. Observe the failure at `Step 21/25` (`COPY --from=layers ...application/`).
4. Run the same command with `--file ./docker/Dockerfile.nolayers` — it succeeds.

## Expected behavior

`az acr build` should export and push the image for the 4-stage layered Dockerfile, exactly as BuildKit does.

## Actual behavior

The build completes all 25 steps, but the export phase fails with `failed to get layer sha256:...: layer does not exist`.

## Additional notes

- The pattern that triggers the bug is the Spring Boot layered JAR extraction stage (`java -Djarmode=tools -jar app.jar extract --layers --launcher`) followed by four `COPY --from=layers` instructions in the final stage.
- Equivalent multi-stage Dockerfiles for other stacks (.NET, Node, Python) that use only one level of `COPY --from` build successfully on the same registry with `az acr build`.
