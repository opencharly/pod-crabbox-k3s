# AGENTS.md — pod-crabbox-k3s

The **k3s control-plane lease image** for the Crabbox `containerd` provider: the
`pod-crabbox-k3s` image (`charly.yml`) and the `k3s-lease` candy
(`candy/k3s-lease/`) it composes on top of `opencharly/pod-crabbox-nested`. This
repo owns exactly the k3s delta — the binary, its config, the `k3s0` link, the
unit, and the readiness hook — plus the probes that repeat the inherited lease
contract at this image's boundary. The lease contract, systemd-as-PID-1 and the
inner containerd belong to the base repos; a change to any of them lands there,
not here.

Canonical files:

- `charly.yml` — the image: `base: ghcr.io/opencharly/pod-crabbox-nested` at a
  pinned tag, `entrypoint: [/sbin/init]`, `user: root`.
- `candy/k3s-lease/charly.yml` — the k3s packages, plan steps and `check:` steps.
- `.github/workflows/image.yml` — the image build; it triggers on
  `tag-on-merge` **completion** (`workflow_run`), never on `push: tags`
  (`opencharly/.github#277`: tag-on-merge mints tags through the REST API, which
  emits no push event).
- `.github/workflows/tag-on-merge.yml` — the org's CalVer tag + `CHANGELOG/`
  dispatcher, installed by the org bootstrap.
- `README.md` — user overview only; never agent guidance.

## Load these skills first (R0)

- `/charly-image:image` — the `candy:` image entry (base, user, entrypoint).
- `/charly-image:layer` — the candy body: `distro:` packages, plan step verbs,
  and the `check:` surface.
- `/charly-check:check` — the check verbs, the ADE requirement, and the R10 gate.
- `/charly-internals:git-workflow` — before any git/PR action.

## What is unique here

- **The image tag IS the git tag.** `.github/workflows/image.yml` passes the
  newest `v*` tag (minus its `v`) to `charly box build --tag`, which uses a
  supplied tag verbatim. `defaults.tag: auto` is only the LOCAL-build fallback;
  the CI caller always supplies the tag.
- **The base is a pinned OCI artifact, never copied.** `base:
  ghcr.io/opencharly/pod-crabbox-nested:<tag>` pulls the published nested
  image. Advancing the base is a one-line tag bump, produced first by the nested
  repo (producer-first).
- **k3s uses the inner containerd, never its own.** `container-runtime-endpoint`
  points at `/run/containerd/containerd.sock` (K3S.md §1–§2.2) and the two CNI
  keys are required (K3S.md §2.1). Do not add an embedded-containerd fallback.
- **The version is the URL's version.** `K3S_VERSION` and `K3S_VERSION_URL` are
  the same release in two spellings (the URL needs `%2B`); the gate's
  `k3s-checksum` check compares the installed binary against the release's own
  sha256 manifest. Bump both vars and the checksum/version checks together.
- **Arch package names, not Fedora spellings.** The row names
  `sqlite procps-ng kmod iputils`; the layer-k3s candy contributes
  `conntrack-tools socat ethtool fuse-overlayfs ca-certificates`. `iptables-nft`
  is provided by the base image's `iptables` package and is asserted by a check
  rather than re-declared over a package-owned file.
- **The unit starts no module.** No `ExecStartPre=modprobe`; `KillMode=process`,
  `Delegate=yes`, `Restart=always` are the row's settings and are asserted.
- **`node-ip` rides on a dummy link.** `k3s-dummy.service` creates `k3s0` and
  assigns `172.31.255.1/32`; the address and the TLS SAN must stay in step.
- **The readiness hook checks a workload, not just the node.** A clone carries
  `/var` but not `/run`, so a fork reports Ready about 8 s before it is
  workload-usable (CONTAINERD-IMAGES.md §2.1). Keep both halves.

## Build / validate / test

- `charly box validate` at the repo root — the structural check on the image and
  its candy (including the pulled base image at its pinned tag).
- `charly box generate pod-crabbox-k3s` — inspect the emitted Containerfile and
  the baked `ai.opencharly.description` plan.
- `charly check box pod-crabbox-k3s` — the build-scope `check:` steps.
- The PR image build (`.github/workflows/image.yml`, `pull_request` path) builds
  the image and publishes nothing; the merge-time `workflow_run` path publishes
  `ghcr.io/opencharly/pod-crabbox-k3s:<git-tag>`.

## Modify this repo

- Keep the base tag in step: a base-contract, systemd or inner-engine fix lands
  in its owning repo first, then this repo's `base:` tag advances.
- A package may be added only from a named source (the base candy, an existing
  `layer-*`/`pod-*` repo, the provider's docs, or the Arch package index).
  Record the source in a comment.
- Every claim in the plan needs a deterministic `check:`. A package that merely
  installs without a probe is not proof.
- `charly check box` runs only build-context `check:` steps; a claim that needs
  a running cluster belongs in a deploy-context bed, not here.

## Landing

- PR-only. Every change lands through a pull request; the org-required
  `charly/pr-validator` validates the diff and body and arms native auto-merge on
  PASS. Direct pushes to `main` are blocked.
- History lives in `CHANGELOG/` (written by `tag-on-merge` at merge time); the PR
  body IS the changelog.
- The authoritative rulebook is the umbrella `AGENTS.md` in
  `opencharly/opencharly`. Do not restate its rules here.
