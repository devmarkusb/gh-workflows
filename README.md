# gh-workflows

[![CI](https://github.com/devmarkusb/gh-workflows/actions/workflows/ci.yml/badge.svg)](https://github.com/devmarkusb/gh-workflows/actions/workflows/ci.yml)
[![License: BSL-1.0](https://img.shields.io/badge/License-BSL--1.0-blue.svg)](./LICENSE)

Reusable [GitHub Actions](https://docs.github.com/en/actions/using-workflows/reusing-workflow-configuration) workflows for C++ CMake projects that follow a **beman**-similar layout (`devenv/cmake/…`, FetchContent lockfile, optional `namespace::component` install tests).
Cf. <https://github.com/devmarkusb/devenv>.

## How to use

Reference workflows from another repository with a **full ref** (tag, SHA, or branch):

```yaml
jobs:
  build:
    uses: your-org/gh-workflows/.github/workflows/build-and-test.yml@main
    secrets: inherit
    with:
      matrix_config: ${{ vars.MATRIX_JSON }}
```

Pin the callee ref (e.g. `@v1.0.0` or commit SHA) so upstream changes cannot break your CI unexpectedly.

---

## CI in this repository

On push to `main` and on pull requests, [.github/workflows/ci.yml](.github/workflows/ci.yml) runs [actionlint](https://github.com/rhysd/actionlint) and [zizmor](https://github.com/zizmorcore/zizmor) (with [.github/zizmor.yml](.github/zizmor.yml)) on these workflow files. For the same checks locally, install [pre-commit](https://pre-commit.com) and run `pre-commit run --all-files` (see [.pre-commit-config.yaml](.pre-commit-config.yaml)).

---

## Workflows

### `build-and-test.yml`

**Name:** `build-and-test matrix`  
**Trigger:** `workflow_call`  
**Inputs:**

| Input           | Required | Description |
|-----------------|----------|-------------|
| `matrix_config` | yes      | JSON string: compiler-keyed matrix specification (see below). |

**Assumptions in the consumer repo:**

- CMake project with `devenv/cmake/toolchains/*.cmake` and `devenv/cmake/fetch-content-from-lockfile.cmake`.
- `CMAKE_GENERATOR: Ninja Multi-Config` for configure/build steps.
- Docker compiler images (gcc/clang) use `ghcr.io/bemanproject/infra-containers-<compiler>:<version>`.
- Install prefix in CI uses `/opt/namespace.component` (adjust in the workflow copy if your project differs).

**Matrix JSON shape (conceptual):** top-level keys are compiler names (`gcc`, `clang…`, `appleclang`, `msvc`). Each maps to a list of objects with `versions` and nested `tests` / `cxxversions` / `stdlibs` / leaf `test` labels. The configure job expands this to a flat list of matrix rows; each row includes `image` as either a Docker image string or JSON `null` (host jobs: macOS/Windows).

**Coverage:** Rows whose test type resolves to `Coverage` run `gcovr` and upload via Coveralls when the event is not `schedule`.

---

### `preset-test.yml`

**Name:** `preset test matrix`  
**Trigger:** `workflow_call`  
**Inputs:**

| Input           | Required | Description |
|-----------------|----------|-------------|
| `matrix_config` | yes      | JSON **array** of `{ "preset": "...", "runner": "..." }` and/or `{ "preset": "...", "image": "..." }`. |

**Windows / MSVC:** Set `"runner": "windows-latest"` (or another `windows-*` hosted runner). Relying on `image` alone while `runner` defaults to Linux will skip MSVC setup and misconfigure the job.

**CMake preset:** Uses `cmake --workflow --preset <name>`; the consumer repo must define matching presets (e.g. in `CMakePresets.json`).

---

### `install-test.yml`

**Name:** `test cmake install`  
**Trigger:** `workflow_call`  
**Inputs:**

| Input             | Required | Description |
|-------------------|----------|-------------|
| `image`           | yes      | Container image for building the library. |
| `cxx_standard`    | yes      | `CMAKE_CXX_STANDARD` value (e.g. `20`). |
| `namespace`       | yes      | CMake namespace / package prefix (e.g. `mycompany`). |
| `include_header`  | no       | Full include path for the consumer smoke test (`#include "..."`). |
| `main_header`     | no       | Header file name defaulting to `<library>.hpp` when include path is inferred. |

Configures the project with the FetchContent lockfile helper, resolves the top-level project name from the CMake file API **codemodel** reply (newest `codemodel-v2-*.json` if several exist), installs to `dist`, then builds a tiny `find_package` consumer.

---

### `pre-commit.yml`

**Name:** `pre-commit check`  
**Trigger:** `workflow_call` (invoked with the **caller's** `on` events, e.g. `push` / `pull_request_target`).

- **Push:** runs `pre-commit` on the default checkout.
- **`pull_request_target`:** checks out the PR head with `gh pr checkout`, runs hooks, and on failure runs `reviewdog/action-suggester` so fixes can be suggested on the PR.

**Security:** `pull_request_target` runs with **base**-repo permissions and secrets. The workflow deliberately checks out the PR branch to lint it; that still means **untrusted code from forks runs on the runner** with the token/scopes you grant. Keep third-party actions pinned, minimize permissions, and follow GitHub’s guidance on `pull_request_target`. Only use this pattern if you accept that tradeoff for forked PRs.

---

### `update-pre-commit.yml`

**Name:** `pre-commit auto-update`  
**Trigger:** `workflow_call`  
**Secrets:**

| Secret        | Required |
|---------------|----------|
| `APP_ID`      | GitHub App ID for creating the PR. |
| `PRIVATE_KEY` | GitHub App private key. |

Runs `pre-commit autoupdate`, runs `pre-commit run --all-files` (non-blocking for PR creation if hooks fail), opens or updates a PR via `peter-evans/create-pull-request`, adds a **warning** to the PR body and job summary when `--all-files` fails, then **fails the job** so the run is red until follow-up fixes land.

---

## Pins and supply chain

Third-party actions in these workflows use **version comments** next to commit SHAs where recommended (`lukka/get-cmake`, `coverallsapp/github-action`). Re-pin periodically after review. Other actions use tag pins (`@v4`); you may tighten those to full SHAs in your fork.

---

## License

See [LICENSE](./LICENSE) (Boost Software License 1.0).
