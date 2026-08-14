# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

An Ansible playbook that builds an RKE2 (Rancher's Kubernetes distribution) cluster on RHEL/CentOS-family
hosts, then layers on cluster add-ons (Helm, kubectl, CSI-driver-SMB, local-path-provisioner, kube-vip,
ArgoCD, Tekton, k9s). There is no application code here — everything is YAML tasks, Jinja2 templates,
and role defaults.

## Setup and commands

Install dependencies before running anything:

```bash
pip install -r requirements.txt          # ansible, kubernetes, jmespath
ansible-galaxy collection install -r requirements.yaml   # kubernetes.core (and its dependent collections)
```

Run the playbook:

```bash
ansible-playbook -i inventory.yaml site.yaml
```

Useful flags while iterating:

```bash
ansible-playbook -i inventory.yaml site.yaml --syntax-check   # validate YAML/task syntax only
ansible-playbook -i inventory.yaml site.yaml --check          # dry run
ansible-playbook -i inventory.yaml site.yaml --tags kube-vip  # run only a tagged subset (see Tags below)
ansible-playbook -i inventory.yaml site.yaml --limit TMP-R2V1 # target a single host
```

There is no lint/test config checked in (no `.ansible-lint`, no CI). If `ansible-lint` is available it's
safe to run against `roles/` and `site.yaml`, but don't assume it's wired into any pipeline.

`inventory.yaml` is git-ignored — it holds environment-specific hosts/IPs and is expected to be created
locally per-environment, not committed.

## Live test cluster access

A working `kubectl`, pre-configured with a kubeconfig for a live test RKE2 cluster, is available on this
host — use it freely to troubleshoot, inspect current cluster state (`kubectl get/describe/logs`, etc.),
and verify that a change actually behaves as expected.

Do not treat the cluster as a place to make lasting changes by hand. Any change that should persist
(node labels, Secrets, PVs, add-on manifests, RBAC, etc.) must be expressed as Ansible task
changes/variables in this repo, not as a one-off `kubectl apply`/`edit`/`patch`/`create`. Direct
`kubectl` mutations are fine only as throwaway experiments to validate an approach before writing it into
the corresponding role, or for read-only diagnosis — the Ansible code, not live cluster state, is the
source of truth and what gets re-applied on the next run.

## Git usage

Never run `git commit`, `git push`, or any other git action that writes to the repository or a remote
(this includes staging/`git add` on the user's behalf). Only read-only git commands (`git status`,
`git diff`, `git log`, `git show`, etc.) are allowed. The user commits and pushes their own changes.

## Architecture

### Entry point and inventory

`site.yaml` is the single playbook entry point. It currently only applies the `rke2-install` role to the
`rke2` host group. The `argocd`, `tekton`, and `k9s` roles exist under `roles/` but are **not** invoked
from `site.yaml` yet — they're either applied ad hoc or are a work in progress; check before assuming
they run as part of a normal playbook execution.

`inventory.yaml` defines the `rke2` group with two children: `rke2_server` (control-plane nodes) and
(implicitly, via the `install_agent.yaml` task guard) `rke2_agent` (worker nodes) — the agent group isn't
populated in the current inventory but the role supports it. Cluster-wide vars (channel, kube-vip address,
interface, etc.) are set at the `rke2` group level in the inventory.

### `rke2-install` role — task execution order

`roles/rke2-install/tasks/main.yaml` is the orchestrator; it `include_tasks`s the following in sequence,
each independently taggable:

1. **pre-install.yaml** — OS prep: disables `nm-cloud-setup`, configures NetworkManager to ignore
   Calico/Flannel interfaces, installs required yum/pip packages, opens firewalld ports for Calico CNI.
2. **install_common.yaml** — architecture detection, resolves `rke2_version` from the release channel API
   if not pinned, downloads/extracts the RKE2 tarball, installs the matching SELinux RPM, applies CIS
   sysctl settings, creates the `etcd` system user/group.
3. **install_server.yaml** (`when: "rke2_server" in group_names`) — renders `server-config.yaml.j2`,
   starts `rke2-server.service`, opens firewalld ports 6443/9345.
4. **install_agent.yaml** (`when: "rke2_agent" in group_names`) — fetches the join token from the first
   `rke2_server` host (`delegate_to: groups['rke2_server'][0]`), renders `agent-config.yaml.j2`, starts
   `rke2-agent.service`.
5. **post-install.yaml** (tag `post-install`) — labels nodes and provisions extra k8s Secrets, both via
   `kubernetes.core.k8s` delegated to the first server node.
6. **kubectl.yaml** — installs a matching `kubectl` binary and adds it to root's `PATH`.
7. **helm.yaml** — installs Helm 3 (server node only, `run_once: yes`).
8. **csi-driver-smb.yaml** (tag `csi-driver-smb`, `when: rke2_smb_credentials is defined`) — Helm-installs
   the SMB CSI driver and provisions SMB credential Secrets.
9. **volumes.yaml** (tag `volumes`) — loops over `rke2_hostpath_volumes`, creating hostPath directories
   (with SELinux `container_file_t` context) and matching `PersistentVolume` objects.
10. **kube-vip-daemonset.yaml** / **kube-vip-static.yaml** (tags `kube-vip`, `kube-vip-daemonset` /
    `kube-vip-static`; gated by `kubevip_enable_daemonset` / `kubevip_enable_static`) — two alternative
    ways to run kube-vip for control-plane VIP/HA: a DaemonSet manifest dropped into RKE2's
    auto-deploying manifests directory, or a static pod manifest generated by running the `kube-vip`
    container image via `podman ... manifest pod` locally and copying the rendered YAML to the node.

`kubernetes.core.k8s` tasks that talk to the cluster are consistently `delegate_to:
groups['rke2_server'][0]` (and often `run_once: yes`) since only the first server node reliably has a
ready kubeconfig/API access at that point in the run.

**EL10 kernel gotcha:** AlmaLinux/RHEL 10 base images don't install `kernel-modules-extra`, which is
where the legacy `xt_*` netfilter match/target modules (plus `nft_compat`) actually live — the base
kernel package doesn't include them. Without it, anything that shells out to `iptables`/`iptables-nft`
with extensions like `-m comment`, `-m conntrack`, or `-j MARK` fails at runtime with "Extension ...
revision 0 not supported, missing kernel module?", and `ipset` calls fail with "Kernel error received:
Invalid argument". This breaks kube-proxy's service routing (`KUBE-SERVICES` chain never gets built —
ClusterIPs including the apiserver's own Service IP become unreachable, which cascades into
tigera-operator crash-looping and the node staying `NotReady`), Calico's Felix dataplane (ipset resync
failures), and the CNI `portmap` plugin (hostPort support, e.g. for `rke2-ingress-nginx-controller`).
`pre-install.yaml` installs `kernel-modules-extra` and loads `br_netfilter` (`community.general.modprobe`,
persistent) to cover this — `br_netfilter` has to be loaded *before* `install_common.yaml` applies the
CIS sysctl file, since `net.bridge.bridge-nf-call-iptables` doesn't exist as a sysctl key until that
module is loaded. Don't "fix" symptoms of this by switching kube-proxy/Calico to nftables-native modes
first — check whether `kernel-modules-extra` is installed on the target OS image before going down that
path, since it's a much smaller, root-cause fix.

### Key variables (see `roles/rke2-install/defaults/main.yaml` and inventory)

- `rke2_channel` — RKE2 release channel (e.g. `v1.35`, `stable`); resolved to a concrete `rke2_version`
  at runtime via `https://update.rke2.io/v1-release/channels/<channel>` unless `rke2_version` is pinned.
- `rke2_cluster_cidr` / `rke2_service_cidr` — CNI (Calico) pod/service CIDRs.
- `rke2_hostpath_volumes` — list of `{name, path, size, access_mode, uid, gid, subdirs, reclaim_policy}`
  dicts that drive `volumes.yaml`.
- `rke2_additional_node_labels`, `rke2_additional_secrets` — post-install customization hooks.
- `rke2_smb_credentials` — presence toggles the CSI-driver-SMB role step; list of `{namespace, name,
  domain, user, password}`.
- `kubevip_api_address`, `kubevip_interface`, `kubevip_enable_static`, `kubevip_enable_daemonset` — set at
  inventory group level, control kube-vip mode/config.

### Other roles

- `roles/argocd` — installs ArgoCD, an ingress for it, repo Secrets (`argocd_repos`), and `Application`
  objects (`argocd_apps`); `tasks/app.yaml` duplicates the "Create apps" task from `main.yaml` as a
  standalone includable file (e.g. for adding a single app outside the full role run).
- `roles/tekton` — installs Tekton Pipelines and Triggers from upstream release manifests.
- `roles/k9s` — installs the `k9s` CLI via the OS's native package manager (yum or apt, RPM/DEB fetched
  from GitHub releases).

### Conventions used throughout

- Nodes are Enterprise Linux (`ansible.builtin.yum`, firewalld, SELinux `sefcontext`/`seboolean` modules
  throughout) — don't assume Debian/apt except where a role explicitly branches on `ansible_pkg_mgr`.
- "Latest version" for most third-party tools (kube-vip, k9s, csi-driver-smb, local-path-provisioner) is
  resolved at runtime from the relevant GitHub Releases API and can be pinned by defining the
  corresponding `*_version` variable to skip the lookup.
- Task names follow a `RoleOrArea | Step description` convention (e.g. `RKE2 Server | Place configation`)
  — keep new tasks consistent with this when editing.
- Handlers (`roles/rke2-install/handlers/main.yaml`) are used for restart/reload side effects
  (`Restart rke2-server`, `Reload Firewalld`, `Reload sysctl`, etc.) — trigger them via `notify:` rather
  than adding explicit restart tasks.
