# etcd Spread Testing — Project Notes

## Overview

This directory contains a [spread](https://github.com/canonical/spread) testing
suite for the etcd source tree at `etcd/`. Spread is Canonical's system testing
distribution tool — it allocates VMs/containers, deploys the project, and runs
tasks across a matrix of backends × systems.

---

## Repository Layout

```
etcd-tests/
├── spread.yaml                        # Spread configuration (backends, systems, suites)
├── CLAUDE.md                          # This file
└── tests/
    └── spread/
        ├── unit/run/task.yaml         # Phase 1: unit tests
        ├── integration/run/task.yaml  # Phase 2: integration tests
        ├── e2e/run/task.yaml          # Phase 3: e2e tests
        └── robustness/run/task.yaml   # Phase 4: robustness tests
```

---

## etcd Test Infrastructure

### Go version

etcd requires **Go 1.26** (declared in `etcd/go.mod`). Ubuntu 22.04/24.04 do
not ship this in their default apt repos — the project-level `prepare` script
downloads it directly from `go.dev/dl/`.

### Supported architecture

Linux amd64 only (per `etcd/CONTRIBUTING.md`).

### How tests are run

All tests go through `etcd/scripts/test.sh`, controlled by the `PASSES`
environment variable. The Makefile targets are thin wrappers around this:

| Test type    | Direct command                           | Makefile target         | Timeout |
|--------------|------------------------------------------|-------------------------|---------|
| Unit         | `PASSES=unit ./scripts/test.sh`          | `make test-unit`        | 3m      |
| Integration  | `PASSES=integration ./scripts/test.sh`   | `make test-integration` | 15m     |
| E2E          | build + `PASSES=e2e ./scripts/test.sh`   | `make test-e2e`         | 30m     |
| Robustness   | gofail + build + `PASSES=robustness ...` | `make test-robustness`  | 30m     |

In spread tasks, call `./scripts/test.sh` directly — do not use `make test-*`
because `etcd/Makefile` line 1 runs `git rev-parse --show-toplevel` and will
fail if the git repo state is unusual.

### Go workspace + `-mod=readonly`

`test.sh` sets `GOFLAGS=-mod=readonly`, which forbids downloading new
dependencies at test time. The project-level spread `prepare` script runs
`go work sync && go mod download` to pre-populate the module cache.

etcd uses a Go workspace (`go.work`) linking 9 sub-modules: api, cache,
client/pkg, client/v3, etcdctl, etcdutl, pkg, server, tests.

### E2E and robustness test requirements

- **E2E**: requires a pre-built `./bin/etcd` binary — build with
  `GO_BUILD_FLAGS="-v -mod=readonly" ./scripts/build.sh` before running
- **Robustness**: additionally requires gofail failpoints (`make gofail-enable` /
  `make gofail-disable`) and the lazyfs FUSE fault injector (`make install-lazyfs`)

---

## Spread Configuration

### Running spread

```bash
# Install spread (note: Go module path is snapcore, not canonical)
go install github.com/snapcore/spread/cmd/spread@latest

# List all jobs without running
spread -list qemu:

# Run unit tests on Ubuntu 24.04
spread qemu:ubuntu-24.04:tests/spread/unit/

# Run on both systems
spread qemu:tests/spread/unit/

# Debug mode — interactive shell on failure
spread -debug qemu:ubuntu-24.04:tests/spread/unit/

# Reuse allocated VMs between runs (faster iteration)
spread -reuse qemu:ubuntu-24.04:tests/spread/unit/
```

### Monitoring task output in real time

Spread does not stream task stdout to the terminal — it captures output and only
shows it on failure. There is no spread.yaml option to change this.

To watch a running task live, tail the log file inside the container from a
second terminal. The container name is printed in the spread output
(e.g. `spread-9-ubuntu-24-04`):

```bash
lxc exec spread-9-ubuntu-24-04 -- tail -f /tmp/integration-tests.log
# or for unit tests:
lxc exec spread-9-ubuntu-24-04 -- tail -f /tmp/unit-tests.log
```

### Backends

**Current default: QEMU** — used on Arch Linux where snap LXD has AppArmor issues.

**Future: LXD** — uncomment the `lxd:` block in `spread.yaml` when running on
Ubuntu, where the snap LXD ↔ AppArmor integration works out of the box.

### QEMU image setup

Spread looks for images at `~/.spread/qemu/<system-name>.img`:

```bash
mkdir -p ~/.spread/qemu

# Ubuntu 24.04
curl -O https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img
qemu-img convert -O qcow2 noble-server-cloudimg-amd64.img ~/.spread/qemu/ubuntu-24.04-64.img
qemu-img resize ~/.spread/qemu/ubuntu-24.04-64.img +20G
ln -s ubuntu-24.04-64.img ~/.spread/qemu/ubuntu-24.04.img

# Ubuntu 22.04
curl -O https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img
qemu-img convert -O qcow2 jammy-server-cloudimg-amd64.img ~/.spread/qemu/ubuntu-22.04-64.img
qemu-img resize ~/.spread/qemu/ubuntu-22.04-64.img +20G
ln -s ubuntu-22.04-64.img ~/.spread/qemu/ubuntu-22.04.img
```

Note: spread expects the filename `ubuntu-24.04.img` (without `-64`), so the
symlinks are required even though the actual files are named with `-64`.

---

## Known Issues

### AppArmor on Arch Linux with snap LXD

The LXD snap bundles its own `apparmor_parser`
(`/snap/lxd/current/bin/apparmor_parser`). On Arch, the snapd-generated
AppArmor profiles for the LXD snap (`snap.lxd.lxd` etc.) are not loaded at
boot, so LXD fails when trying to load container AppArmor profiles.

Workaround: `sudo systemctl restart snapd` causes the profiles to load.
Permanent fix: `sudo systemctl enable snapd.apparmor`.

Long-term fix: use native LXD from AUR (`yay -S lxd`) or run on Ubuntu.

---

## Implementation Phases

| Phase | Test type    | Status         | Notes                                              |
|-------|--------------|----------------|----------------------------------------------------|
| 1     | Unit         | Done           | `tests/spread/unit/run/task.yaml`                  |
| 2     | Integration  | Done           | `tests/spread/integration/run/task.yaml`           |
| 3     | E2E          | Done           | `tests/spread/e2e/run/task.yaml`                   |
| 4     | Robustness   | Done           | `tests/spread/robustness/run/task.yaml`            |
