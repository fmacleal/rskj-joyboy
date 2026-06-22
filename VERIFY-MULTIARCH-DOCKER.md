# Verifying the multi-arch (amd64 + arm64) Docker workflows

This document explains how to verify the new multi-architecture Docker image
support **without impacting the official `rsksmart/rskj` images** on Docker Hub.

## What changed

| Workflow | Trigger | Pushes? | Change |
|----------|---------|---------|--------|
| `docker-verification.yml` | PR (touching `Dockerfile` / the workflow) | **No** | Now runs a build + `--regtest` smoke test as a matrix on amd64 (`ubuntu-24.04`) and arm64 (`ubuntu-24.04-arm`). |
| `docker-release-master-to-edge.yml` | push to `master` | **Yes** | Native matrix build per arch → push by digest → merge into one multi-arch manifest (`edge`, branch tag). |
| `docker-release-tags-to-latest.yml` | push of a git tag | **Yes** | Same matrix + merge pattern (tag / `latest` / IRIS group tags). |

The image name is now parameterized via the `IMAGE_NAME` env var in the release
workflows, which makes redirecting builds to a scratch repository trivial.

## Prerequisite: arm64 runners

All three workflows use GitHub's hosted **`ubuntu-24.04-arm`** runner label.

- Free for public repositories.
- For private repos / orgs, confirm these runners are enabled on your plan.
- If you use self-hosted arm64 runners instead, replace the `runner:` label.

If arm64 runners are unavailable, those matrix jobs will stay **queued** or fail
immediately — that is the most likely failure mode, and the run will tell you
right away.

---

## Part 1 — PR verification workflow (safe, automatic)

`docker-verification.yml` only runs `docker build` + `docker run` on the runner.
It **never pushes**, so it has zero registry impact.

1. Open a PR from this branch.
2. Because the changes touch `Dockerfile` / workflow paths, the workflow runs
   automatically.
3. You will see two checks:
   - `docker (linux/amd64)`
   - `docker (linux/arm64)`
4. Each builds the image natively for its arch and waits for `IMPORTED_BEST` in
   the container logs. `fail-fast: false` means one arch failing still lets the
   other finish so you can compare.

No further action required for this workflow.

---

## Part 2 — Release workflows (deliberate test against a scratch repo)

The release workflows only fire on `push: master` / `push: tags`, so opening the
PR **cannot** accidentally publish anything. To exercise them you must
deliberately run them against a scratch Docker Hub repository.

### Option A — Fork + personal Docker Hub repo (recommended)

1. **Fork** the repository (or use a throwaway branch on a fork you control).

2. Create a scratch Docker Hub repository, e.g. `youruser/rskj-test`.

3. In the fork's repo settings, set the secrets to **your own** account:
   - `DOCKERHUB_USERNAME`
   - `DOCKERHUB_TOKEN` (scope the token to just the scratch repo)

4. Temporarily edit the release workflow(s) in the fork so they can be triggered
   manually and point at the scratch repo. For example, in
   `docker-release-master-to-edge.yml`:

   ```yaml
   on:
     workflow_dispatch:        # adds a manual "Run workflow" button
     # push:                   # commented out for testing
     #   branches:
     #     - master

   env:
     IMAGE_NAME: youruser/rskj-test
   ```

5. From the fork's **Actions** tab, select the workflow and click
   **Run workflow**.

6. The `merge` job ends with `docker buildx imagetools inspect`, so the run log
   shows the final manifest. Confirm **both** architectures are present:

   ```bash
   docker buildx imagetools inspect youruser/rskj-test:edge
   # Expect an "image index" listing:
   #   linux/amd64
   #   linux/arm64
   ```

7. **Clean up:** delete the scratch Docker Hub repo, and **revert** the
   `on:` / `IMAGE_NAME` edits so they never land in the PR.

### Option B — Local build only (no registry, no push)

Proves the Dockerfile builds on both arches, but does **not** validate the
digest/merge logic (that is registry-only).

```bash
docker buildx build --platform linux/amd64,linux/arm64 -t rskj:multiarch-test .
```

On an Apple Silicon Mac this builds arm64 natively and amd64 via emulation. Use
it to catch Dockerfile-level architecture issues quickly.

---

## Verification checklist

- [ ] PR `docker (linux/amd64)` check passes (`IMPORTED_BEST` found).
- [ ] PR `docker (linux/arm64)` check passes (`IMPORTED_BEST` found).
- [ ] `ubuntu-24.04-arm` runners pick up jobs (not stuck queued).
- [ ] Release dry-run pushes to the **scratch** repo only.
- [ ] `imagetools inspect` shows an image index with `linux/amd64` **and**
      `linux/arm64`.
- [ ] Expected tags are present on the manifest (`edge`/branch, or tag/`latest`/
      IRIS group).
- [ ] Scratch repo deleted and temporary `on:` / `IMAGE_NAME` edits reverted
      before merging.

---

## Reminder

Never run the release-workflow dry-run against `rsksmart/rskj` or with the
production `DOCKERHUB_*` secrets. Always use a scratch repository and your own
scoped token.
