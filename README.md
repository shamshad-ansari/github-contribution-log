# Contribution 1: Lint Unused Funcs

- **Student:** Shamshad Ansari
- **Issue:** [kubernetes-sigs/cluster-api#7599](https://github.com/kubernetes-sigs/cluster-api/issues/7599)
- **Status:** Phase I - IV **[Complete]** — cleanup PR [#13826](https://github.com/kubernetes-sigs/cluster-api/pull/13826) merged (June 19); gate PR opened (June 19) on branch `feature/verify-deadcode`, awaiting maintainer review

---

## Why I Chose This Issue

I chose this issue because I want to start learning Kubernetes development through a contribution that is still connected to skills I already have. Cluster API is a large Kubernetes-related project, so I wanted an issue that would help me get familiar with the codebase without needing very deep Kubernetes knowledge right away.

This issue stood out to me because it is mainly about Go linting and code quality. I have worked on a project in Go before, so I feel comfortable reading Go code and learning more about Go tooling. I also think this is a useful issue because unused functions can make a codebase harder to maintain. Through this contribution, I hope to learn more about how large open-source Go projects use linters, static analysis, and CI checks to keep their code clean.

---

## Understanding the Issue

### Problem Description

The issue is asking for a linter or linting setup that can detect unused functions in the Cluster API codebase. Some unused code was previously found manually, and the maintainers want a better way to catch this automatically in the future. The issue mentions two cases: functions that are not used anywhere, and functions that are only used by test code. The first case seems easier to detect, while the second may be harder because it could create false positives.

### Expected Behavior

The project should have a linting check that can detect functions that are not used anywhere in the codebase. This would help contributors catch unused code earlier — before it stays in the project or reaches a pull request.

### Current Behavior

Right now, unused functions may not always be caught automatically. The `unused` linter is already enabled in `.golangci.yml`, but it only flags unexported, in-package symbols — it does not catch exported functions or functions that are only referenced from test files. In the example from the issue, unused code was found and removed manually. This means the project still needs better tooling to detect this kind of problem consistently.

### Affected Components

The main affected components are the Go source code and the project's linting/static analysis setup. This involves the repository's lint configuration (`.golangci.yml`), the Makefile verify targets, and the developer/CI workflow that runs lint checks. The issue also specifically mentions unused code found in the KCP (Kubeadm Control Plane) area, so that part of the codebase is relevant too. In practice, the genuinely-dead functions I found lived in `internal/contract`.

---

## Reproduction Process

### Environment Setup

Working on branch `issue-7599` of the `kubernetes-sigs/cluster-api` fork (cleanup), and `feature/verify-deadcode` (the verify gate).

The project requires Go 1.26, which was already installed. No additional setup was necessary beyond cloning the repository, as `go run` automatically downloads the `deadcode` tool when invoked.

A key environmental detail: the repository is three separate Go modules (`.`, `test/`, `hack/tools/`) with no committed `go.work`. Cross-module usage must be made visible to whole-program analysis, otherwise functions used only by the `test/` module look dead.

### Steps to Reproduce

1. **Confirm the existing linter does not catch the issue**
   The `unused` linter is already enabled in `.golangci.yml`, but it does not flag exported functions in `internal/` packages, nor functions only referenced from test files. For example, `InjectTestManagementCluster` in `controlplane/kubeadm/internal/control_plane.go` is production code called only from test files, yet the existing linter does not report it.

2. **Run `deadcode` manually to identify unused production code**

   ```bash
   go run golang.org/x/tools/cmd/deadcode@latest \
     . ./controlplane/kubeadm/... ./bootstrap/kubeadm/... ./cmd/...
   ```

3. **Observed result**
   A naive whole-repo run reports hundreds of "unreachable" functions, but most are false positives: exported API consumed by downstream provider repositories, or functions used only by the separate `test/` module that the run could not see. Filtering these out left a small set of real candidates in `internal/contract`.

4. **Refine to a zero-false-positive subset**
   Because Go forbids importing `internal/**` from outside the module tree, anything unreachable under `internal/` is genuinely dead. Spanning all three modules with an ephemeral workspace and building the e2e test binary (`-tags=e2e`) yields a clean, reproducible result:

   ```bash
   go work init && go work use . ./test ./hack/tools   # ephemeral; go.work is gitignored
   deadcode -test -tags=e2e \
     -filter 'sigs.k8s.io/cluster-api/internal/' \
     sigs.k8s.io/cluster-api/... sigs.k8s.io/cluster-api/test/...
   ```

   This reports exactly **6 genuinely dead functions**, all in `internal/contract`:
   - `controlplane.go`: `ControlPlaneContract.FailureReason`, `FailureMessage`, `ExternalManagedControlPlane`
   - `controlplane_template.go`: `ControlPlaneTemplateMachineTemplate.NodeDrainTimeout`, `NodeVolumeDetachTimeout`, `NodeDeletionTimeout`

   These are leftover copies/variants whose sibling types (`Bootstrap` / `InfraMachine` / `InfraCluster`) have live, tested equivalents.

### Reproduction Evidence

- **Commit showing reproduction:** N/A — reproduction is a manual tool invocation, no code change required.
- **Screenshots/logs:** See observed result above (6 dead functions under `internal/` after refinement; hundreds of cross-module/downstream false positives on a naive run).
- **My findings:** The issue is confirmed. `deadcode` (`golang.org/x/tools/cmd/deadcode`) reliably detects Case 1 (functions never called by anyone) and is not currently integrated into the project's tooling. Case 2 (functions called only from test code) cannot be reliably detected for methods on instantiated types (e.g. `InjectTestManagementCluster`) because of a known limitation of `deadcode`'s Rapid Type Analysis (RTA): it conservatively treats all methods on instantiated types as reachable. Standalone test-only functions, however, are detected. A scope restricted to `internal/**` is the achievable zero-false-positive target, since `internal/` cannot be imported by downstream repos.

---

## Solution Approach

### Analysis

There are two distinct problems, and they have different root causes:

1. **The dead code itself** — six accessor methods in `internal/contract` were copied/templated from sibling contracts but never wired up or tested. They are harmless but add maintenance noise. Root cause: copy-paste during contract scaffolding, with no automated check to catch the leftovers.
2. **The missing check** — the existing `unused` linter is in-package only, so exported/test-only-used and cross-module-dead functions slip through. Root cause: no whole-program reachability analysis runs in the project's workflow.

### Proposed Solution

Split the work into two independent, low-coupling pieces:

1. **Cleanup PR** — delete the 6 confirmed dead functions. Small, obvious, low-risk.
2. **Verify gate** — add `make verify-deadcode`, a whole-program reachability check scoped to `internal/**` so it produces zero false positives, wired into `ALL_VERIFY_CHECKS` so it can run locally and in CI.

Keeping them separate matters: the cleanup is an obvious win that can merge immediately, while a hard CI gate that fails everyone's push is a maintainer policy decision (CI cost, build tags, staged code), which `CONTRIBUTING.md` says should be discussed first.

### Implementation Plan

Using the UMPIRE framework (adapted):

**Understand:** Detect functions that are never used (Case 1) and ideally those used only by tests (Case 2). The existing `unused` linter misses exported and cross-module-dead functions; a whole-program tool is required.

**Match:** The repo already has the `unused` linter (in-package only) and a well-established `verify-*` pattern in the Makefile (e.g. `verify-licenses`, `verify-modules`) backed by scripts under `hack/`, each built as a pinned tool via `GO_INSTALL`. The fix follows this exact pattern rather than inventing a new one.

**Plan:**
1. Identify the genuinely-dead functions with a refined `deadcode` recipe scoped to `internal/`.
2. Delete those 6 functions (cleanup PR) in `internal/contract/controlplane.go` and `controlplane_template.go`.
3. Add a pinned `deadcode` tool (`DEADCODE_VER := v0.45.0`) and a `$(DEADCODE)` build rule in the Makefile, mirroring the other tools.
4. Add `hack/verify-deadcode.sh` that builds an ephemeral `go.work`, runs `deadcode -test -tags=e2e` filtered to `internal/`, excludes `_test.go` and `/fake/`, and exits non-zero on findings.
5. Register `deadcode` in `ALL_VERIFY_CHECKS` and add the `verify-deadcode` target.

**Implement:**
- Cleanup: branch `issue-7599`, commit `ea7a7bcc` (🌱 Remove unused functions in internal/contract).
- Gate: branch `feature/verify-deadcode`, commit `6b81770e` (✨ Add make verify-deadcode).

**Review:** Follows the existing `verify-*` Makefile/hack convention; pinned tool version; gitmoji commit style; DCO sign-off present; no stray analysis artifacts committed; cleanup kept separate from the policy-sensitive gate.

**Evaluate:** Round-trip test — with the dead functions present, `make verify-deadcode` fails (exit 1) listing exactly the 6; with them deleted, it passes. `go build ./...`, `go vet`, and `go test ./internal/contract/...` all green after deletion.

---

## Testing Strategy

### Unit Tests

- [x] Test case 1: `go test ./internal/contract/...` passes after deleting the 6 functions (confirms nothing in-package depended on them).
- [x] Test case 2: `go build ./...` succeeds (no production caller anywhere in the main module).
- [x] Test case 3: `go vet ./...` clean.

### Integration Tests

- [x] Round-trip gate test: functions present → `make verify-deadcode` exits 1 and lists exactly the 6 dead functions; functions deleted → exits 0 ("PASS").
- [x] Cross-module visibility: the ephemeral `go.work` spans `.`, `test/`, and `hack/tools/`, and `-tags=e2e` builds the build-tagged e2e suite, so functions used only by other modules/tests are correctly counted as reachable (no false positives).

### Manual Testing

Ran the refined `deadcode` recipe by hand and verified the temp `GOWORK` file is cleaned up by the trap and never modifies the working tree (`go.work` stays gitignored). Confirmed the merged cleanup did not regress any KCP/controlplane tests.

---

## Implementation Notes

### Week 1 Progress

Reproduced the issue and learned why the existing `unused` linter is insufficient (in-package only). Discovered the three-module layout is the main source of false positives, and that `deadcode`'s RTA cannot detect test-only usage for methods on instantiated types (Case 2 limitation). Established `internal/**` as the only zero-false-positive scope.

### Week 2 Progress

Reproduction completed: built the refined recipe (`-test -tags=e2e -filter 'internal/'` under an ephemeral `go.work`), which isolated exactly 6 dead functions in `internal/contract`. Confirmed `internal/**` as a zero-false-positive scope and verified the result was reproducible across runs.

### Week 3 Progress

Moved from reproduction to PRs. Deleted the 6 dead functions in `internal/contract/controlplane.go` and `controlplane_template.go`, confirmed `go build`, `go vet`, and `go test ./internal/contract/...` all pass, and opened the cleanup PR (#13826) — merged June 19. Then built the `verify-deadcode` gate: added the pinned `deadcode` tool (`v0.45.0`) and build rule to the Makefile, wrote `hack/verify-deadcode.sh` (ephemeral `go.work`, `-test -tags=e2e`, filtered to `internal/`, excludes `_test.go` and `/fake/`), and registered `deadcode` in `ALL_VERIFY_CHECKS`. Per the maintainer-policy reasoning, kept this on a separate branch (`feature/verify-deadcode`) rather than bundling it with the cleanup, and opened the gate PR (#13831) the same day. Currently waiting on maintainer feedback on the gate PR, and using the time to look for a few more issues to pick up in the meantime.

### Code Changes

- **Files modified (cleanup, merged):** `internal/contract/controlplane.go` (−26), `internal/contract/controlplane_template.go` (−21) — 47 lines / 6 functions removed.
- **Files added/modified (gate, opened):** `hack/verify-deadcode.sh` (new, 69 lines), `Makefile` (+17: `DEADCODE_VER`/build rule/`verify-deadcode` target, `deadcode` added to `ALL_VERIFY_CHECKS`).
- **Key commits:** `ea7a7bcc` (cleanup), `6b81770e` (gate).
- **Approach decisions:**
  1. Scope the gate to `internal/**` to guarantee zero false positives.
  2. Use an ephemeral temp-`GOWORK` workspace + trap cleanup so the tree is never mutated.
  3. Split cleanup from the gate because a hard CI gate is a maintainer policy call.
  4. Pin `deadcode` at `v0.45.0` and reuse the existing `verify-*` tooling pattern.

---

## Pull Request

### Cleanup PR

**PR Link:** [#13826](https://github.com/kubernetes-sigs/cluster-api/pull/13826) (cleanup)

**PR Description (cleanup):** Removes six unused functions in `internal/contract` (`ControlPlaneContract.FailureReason`/`FailureMessage`/`ExternalManagedControlPlane` and the `ControlPlaneTemplateMachineTemplate` `Node*Timeout` accessors), identified via `deadcode` whole-program analysis scoped to `internal/`. Part of #7599. `go build`, `go vet`, and `go test ./internal/contract/...` all pass.

**Maintainer Feedback:**
- **June 18:** Opened the cleanup PR (#13826) and raised the open question on the issue — whether a follow-up `deadcode` check should be a hard CI gate or run advisory-only.
- **June 19:** Maintainer responded to the open question and merged the cleanup PR. I followed up by opening the CI gate PR the same day.

**Status:** Cleanup PR (#13826) **Merged** (June 19).

---

### Gate PR

**PR Link:** [#13831](https://github.com/kubernetes-sigs/cluster-api/pull/13831) — branch `feature/verify-deadcode`, commit `6b81770e`.

**PR Description (gate):** Adds `make verify-deadcode`, a whole-program reachability check that fails when any function under `internal/**` is unreachable from both the manager binary and every test. Adds a pinned `deadcode` tool (`v0.45.0`) and build rule to the Makefile, a new `hack/verify-deadcode.sh`, and registers `deadcode` in `ALL_VERIFY_CHECKS`. Scoped to `internal/**` because that tree cannot be imported by downstream provider repos, so "unreachable" there provably means "dead" — yielding zero false positives. Part of #7599; complements the merged cleanup (#13826).

**Maintainer Feedback:**
- **June 19:** Opened the gate PR as a follow-up after the maintainer's reply on the CI-gate question and the cleanup merge.
- *[Next date]*: *[summary of review feedback received]*

**Status:** Opened (June 19), awaiting review. Meanwhile, looking for a few more issues to pick up.

---

## Learnings & Reflections

### Technical Skills Gained

- How whole-program reachability analysis (`deadcode`) differs from in-package linters (`unused`), and where RTA breaks down (methods on instantiated types).
- Go multi-module workspaces (`go work`) and why they matter for cross-module analysis.
- How a large project structures pinned dev tools and `verify-*` checks in its Makefile/hack scripts, plus Kubernetes contribution mechanics (DCO sign-off, CLA, gitmoji commits).

### Challenges Overcome

The biggest challenge was the flood of false positives on a naive run. I traced this to the three-module layout and to downstream consumers of exported APIs, then reasoned my way to the `internal/**`-only scope — the one place where "unreachable" provably means "dead." Getting the e2e-tagged suite included (`-tags=e2e`) was the final piece that eliminated the remaining false positives.

### What I'd Do Differently Next Time

I'd reach for the module-scope reasoning sooner instead of trying to filter a noisy whole-repo run, and I'd open the discussion issue/comment about the CI gate earlier, in parallel with the cleanup, so the policy conversation could progress while the obvious win merged.

---

## Resources Used

- `golang.org/x/tools/cmd/deadcode` documentation and the RTA reachability notes.
- Cluster API `CONTRIBUTING.md` (discussion-before-infra guidance) and the existing `verify-*` targets in the Makefile as a pattern reference.
- Go workspaces reference (`go work`) for spanning the three modules.
- GitHub issue [#7599](https://github.com/kubernetes-sigs/cluster-api/issues/7599) and the linked manual-removal example.
