# Zarf Init Package for Longhorn

This custom Zarf init package bootstraps the Zarf registry and deploys Longhorn
as the cluster's storage provider. It stages Longhorn's images in the seed
registry, deploys the Longhorn Helm chart, waits for Longhorn's per-node engine
images and instance managers, then copies the images into the permanent Zarf
registry.

## Prerequisites

- Nodes that meet the [Longhorn installation requirements](https://longhorn.io/docs/latest/deploy/install/#installation-requirements),
  including `open-iscsi`.

Run Longhorn's preflight checker against the target cluster before deployment:

```bash
longhornctl check preflight --kubeconfig=<path-to-your-kubeconfig>
```

The checker requires a real kubeconfig because it creates a temporary DaemonSet,
so it runs from the workstation rather than as a package component. K3s nodes
require the additional configuration described in [Longhorn CSI on
K3s](https://longhorn.io/docs/latest/advanced-resources/os-distro-specific/csi-on-k3s/).

### Bootstrap a new K3s host

The optional `k3s` component installs K3s v1.35.8+k3s1 on a new Linux AMD64
host. It must run as `root` and must be selected explicitly. Install Longhorn's
node prerequisites before initialization:

```bash
apt-get update
apt-get install -y open-iscsi
modprobe iscsi_tcp
systemctl enable --now iscsid
```

For a new host, build the package, switch to a root shell, and select the K3s
component:

```bash
zarf package create . --confirm
sudo -i
zarf init /path/to/zarf-init-amd64-v0.86.0.tar.zst --components k3s --confirm
```

The package's default Longhorn StorageClass uses three replicas. For a
single-node demo, set `persistence.defaultClassReplicaCount: 1` in
`longhorn/values.yaml` before building. Do not select `k3s` when initializing
an existing cluster.

## Create and initialize

```bash
zarf package create . --confirm
zarf init ./zarf-init-<architecture>-<version>.tar.zst --confirm
```

After initialization, Longhorn's default storage class is available. Open its
dashboard with:

```bash
zarf connect longhorn-ui
```

## Registry configuration

`longhorn/values.yaml` sets `global.imageRegistry` to Zarf's
`###ZARF_REGISTRY###` template variable. Zarf's agent redirects image pulls,
but Longhorn also passes the manager image to `longhorn-manager` as a command
line argument. Setting this value makes that expected image match the
agent-rewritten pod image; removing it causes the manager to consider its new
pod stale during upgrades.
