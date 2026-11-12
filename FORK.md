# About this fork

`nik-johnson-net/nomad-driver-podman` is a fork of
[hashicorp/nomad-driver-podman](https://github.com/hashicorp/nomad-driver-podman).
This file and `.github/workflows/fork-release.yml` exist only in the fork. Never send them upstream.

## Why it exists

The fork adds the `rootfs_mount` task option: it boots a container from one of the task's volume
mounts (for example a Ceph RBD CSI volume) instead of from an image
([#1](https://github.com/nik-johnson-net/nomad-driver-podman/issues/1), [#4](https://github.com/nik-johnson-net/nomad-driver-podman/pull/4)). Nomad devcontainers
need it to keep a whole root filesystem on a persistent volume. Upstream does not have it yet.

The fork **replaces** the stock driver. It registers as the `podman` plugin, and every existing
podman job runs on it unchanged. A fork release must therefore never change behavior for image-based
jobs. The release workflow runs upstream's full lint and test suite (rootful podman on
ubuntu-24.04) and publishes nothing if any check fails.

## Version scheme

`v<upstream-version>-rootfs.<n>`, for example `v0.6.6-rootfs.1`.

- `<upstream-version>` is the latest upstream release whose code is in the fork's `main`. Upstream
  tags each release on a `release/x.y.z` branch, so the tag itself is not an ancestor of `main`.
  What matters is that `main` contains that release's branch point.
- `<n>` starts at 1 and goes up by one for each fork release on the same upstream version. It
  goes back to 1 when `main` is synced to a newer upstream release.

The workflow stamps the version, without the leading `v`, into upstream's `version.Version` with
`-ldflags -X`, so the plugin reports it as its `PluginVersion`. The Nomad client logs it at startup
(`detected plugin: name=podman type=driver plugin_version=0.6.6-rootfs.1`). `version/version.go`
keeps upstream's value, which avoids merge conflicts.

## Cutting a release

1. Merge the change into `main` through a PR, with CI passing.
2. Tag the merged commit on `main` and push the tag:

   ```sh
   git fetch origin
   git tag -a v0.6.6-rootfs.1 -m "v0.6.6-rootfs.1" origin/main
   git push origin v0.6.6-rootfs.1
   ```

3. `fork-release.yml` checks that the tag matches the scheme and is on `main`, runs the tests,
   builds the binaries and creates a normal GitHub release (not a prerelease) marked latest.

To try a build without releasing, start a dry run from any branch. It tests and builds, then
uploads the binaries as a workflow artifact. A PR that changes `fork-release.yml` gets a dry run
automatically.

```sh
gh workflow run fork-release.yml --repo nik-johnson-net/nomad-driver-podman --ref <branch> -f version=0.6.6-rootfs.1
```

## Release assets

| Asset | Contents |
|---|---|
| `nomad-driver-podman_linux_amd64` | static binary (`CGO_ENABLED=0 go build -trimpath`) |
| `nomad-driver-podman_linux_arm64` | static binary |
| `SHA256SUMS` | `sha256sum` output: `<hex digest>  <asset name>` |

Asset names do not include the version, so `releases/latest/download/<asset>` works as well as
`releases/download/<tag>/<asset>`. Install with Ansible (pin the tag):

```yaml
- ansible.builtin.get_url:
    url: https://github.com/nik-johnson-net/nomad-driver-podman/releases/download/v0.6.6-rootfs.1/nomad-driver-podman_linux_amd64
    checksum: sha256:https://github.com/nik-johnson-net/nomad-driver-podman/releases/download/v0.6.6-rootfs.1/SHA256SUMS
    dest: /opt/nomad/plugins/nomad-driver-podman
    mode: "0755"
```

## Syncing with upstream

Merge upstream `main` (or the new release's branch point) into a branch, open a PR against the
fork's `main`, and merge it once CI passes. Then tag `v<new-upstream-version>-rootfs.1`.

## Retiring the fork

The goal is to get `rootfs_mount` merged upstream ([#3](https://github.com/nik-johnson-net/nomad-driver-podman/issues/3)). Once an upstream release includes it:

1. Switch ansible-homelab back to the stock HashiCorp driver at that version.
2. Stop cutting fork releases.
3. Archive this repository.
