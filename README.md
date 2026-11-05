# pod-crabbox-k3s

The **k3s control-plane CachyOS lease image** for the Crabbox `containerd`
provider — the [`pod-crabbox-nested`](https://github.com/opencharly/pod-crabbox-nested)
lease contract (systemd as PID 1, an inner containerd and krun) plus k3s.

## What it provides

Every lease contract property of the base image is inherited unchanged:

| Property | Value |
|---|---|
| Base | `ghcr.io/opencharly/pod-crabbox-nested:2026.309.0812` (CachyOS) |
| SSH | `sshd` on **2222**, public-key only, user `crabbox` |
| Sudo | `crabbox` has passwordless sudo |
| Inputs | read-only `/crabbox`: `authorized_keys`, `lease.env`, `bootstrap.sh` |
| Readiness | `/usr/local/libexec/crabbox/ready` — polled over SSH until exit 0 |
| PID 1 | `/sbin/init` → `/usr/lib/systemd/systemd` |
| Inner engine | containerd (`nerdctl-full`) + `crun`/`krun`, bridge `10.99.0.1/24` |

This variant adds k3s:

| Property | Value |
|---|---|
| k3s | pinned `v1.37.0+k3s1`, sha256-verified by a deterministic check |
| Install | `/usr/local/bin/k3s` + `kubectl`/`crictl`/`ctr` symlinks |
| Runtime | the **inner** containerd (`/run/containerd/containerd.sock`, K3S.md §2.2) |
| CNI | the two k3s directory keys in `/etc/containerd/conf.d/50-k3s-cni.toml` (K3S.md §2.1) |
| kubeconfig | `/etc/rancher/k3s/k3s.yaml`, mode `0644` |
| Node address | `172.31.255.1`, carried by the `k3s0` dummy link |
| TLS SAN | `172.31.255.1` |
| Rootless gate | `KubeletInUserNamespace=true` kubelet feature gate |
| Unit | `k3s.service`: `KillMode=process`, `Delegate=yes`, `Restart=always`, no `modprobe` |
| Readiness | a Ready node **and** a workload exec through apiserver → kubelet |

The k3s binary is fetched from the release URL pinned by the `K3S_VERSION` /
`K3S_VERSION_URL` vars in `candy/k3s-lease/charly.yml`; the release's own sha256
manifest ships beside it and a deterministic `check:` step compares the installed
binary against it.

## How to use it

The image is run by the Crabbox `containerd` provider. It runs as a pod
(nested) and as a krun VM. Under crun/runc it needs the runtime's systemd mode;
under krun, the guest's PID-1 handoff:

```sh
# crun / runc
--systemd=always --device /dev/kmsg
# krun
-e KRUN_INIT_PID1=1 --ulimit nofile=524288:524288
```

`--device /dev/kmsg` is required by the `pod-crabbox-k3s` row: k3s's kubelet
reads the kernel message ring for its OOM/eviction accounting, and a lease
without the device logs the failure but runs.

The lease inputs are the same read-only `/crabbox` directory the base image
uses. The kubeconfig is world-readable so the consumer can copy it over SSH:

```sh
mkdir -p ~/.kube
scp -P 2222 crabbox@<lease>:/etc/rancher/k3s/k3s.yaml ~/.kube/config
```

## Requirements and limits

- **The lease needs egress for workloads.** k3s uses the shared inner containerd,
  which ships no images; the kubelet's pause image and any workload image are
  pulled on demand. The readiness hook's workload probe pulls `alpine:3.21`, so
  it fails honestly (and the consumer reports the elapsed time) when the lease
  has no egress.
- **The lease needs the host's `dummy` module.** `k3s-dummy.service` runs
  `ip link add k3s0 type dummy` inside the lease, and a container cannot
  `modprobe`: the host must already have the `dummy` module loaded. A rootless
  lease runs as namespace-root in its own user namespace, but that does **not**
  by itself carry `CAP_NET_ADMIN` into the container — see the next point.
- **The lease needs `NET_ADMIN` and `SYS_ADMIN`.** podman's default capability
  set includes neither. Without `NET_ADMIN` the `k3s0` link cannot be created
  (`RTNETLINK answers: Operation not permitted`), and `k3s.service` then fails
  with result `dependency` — which reads like a k3s defect and is not one.
  With `NET_ADMIN` but without `SYS_ADMIN`, kubelet dies at
  `error configuring root directory: failed to bind-mount /var/lib/kubelet:
  operation not permitted`. A consumer therefore declares:

  ```yaml
  security:
      cap_add:
          - NET_ADMIN
          - SYS_ADMIN
  ```

- **The host must delegate the `cpuset` cgroup v2 controller into the user
  slice subtree.** Without it k3s exits with
  `level=fatal msg="Error: failed to find cpuset cgroup (v2)"`. On this host
  `user.slice` has `cpuset` available but `cgroup.subtree_control` carries only
  `cpu memory pids`; a `Delegate=` drop-in does not add it.
- **No `ExecStartPre=modprobe`.** The unit deliberately starts no kernel module;
  the modules k3s needs (`br_netfilter`, `overlay`, `nf_conntrack`) are the
  host's responsibility (CONTAINERD-IMAGES.md §2).

## Layout

- `charly.yml` — the `pod-crabbox-k3s` image (pinned base image + systemd entrypoint).
- `candy/k3s-lease/` — the k3s binary, its configuration, the `k3s0` link, the
  unit, the readiness hook, and the `check:` steps that prove the row.
- `.github/workflows/tag-on-merge.yml` — CalVer tag + `CHANGELOG/` on merge.
- `.github/workflows/image.yml` — builds on `pull_request` and publishes at the
  merge tag, so the image tag equals the git tag.
- `CHANGELOG/` — per-CalVer history.

## Related

- [`opencharly/pod-crabbox-nested`](https://github.com/opencharly/pod-crabbox-nested) —
  the nested-engine lease image this one imports.
- [`opencharly/layer-k3s`](https://github.com/opencharly/layer-k3s) — the k3s
  binary candy this image's pinned version and package list come from.
- [`CONTAINERD-IMAGES.md`](https://github.com/opencharly/opencharly/blob/main/CONTAINERD-IMAGES.md)
  §2 — the lease-image contract and this image's row.
- [`K3S.md`](https://github.com/opencharly/opencharly/blob/main/K3S.md) §1–§2 —
  the shared-containerd design, the endpoint, and the k3s CNI keys.
