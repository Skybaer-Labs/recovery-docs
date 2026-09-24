# CRC + AWX Rebuild Runbook

Reconstructed Sep 24, 2026 (the original copy was lost; it never made it into a repo or a local download). Covers the full procedure for rebuilding CRC/OpenShift and AWX from scratch, plus every gotcha hit along the way, so a future rebuild doesn't require re-deriving any of this.

## When to use this

- The CRC VM fails to boot or hangs after a host crash, and a plain `crc stop` / `crc start` doesn't fix it
- Intentionally resizing CRC's memory/CPU/disk allocation (a `crc config set ...` change only takes effect after `crc delete` + `crc start`)
- Any other situation that requires `crc delete`

## 1. Confirm the CRC VM is actually broken (crash scenario only)

- Check `/var/log/libvirt/qemu/crc.log` on `rhel10-crc` for a hung/silent boot (confirmed pattern: boot goes silent about 40 seconds in)
- Try `crc stop` + `crc start` first. If that resolves it, stop here, no rebuild needed
- If it's still hung, proceed to the full rebuild below

## 2. Rebuild CRC

```
crc stop
crc delete
# only if intentionally changing sizing:
crc config set memory <MB>
crc config set cpus <N>
crc start
```

This wipes and recreates the entire cluster: a new SSH keypair, a new kubeadmin password, and all in-cluster state gone. Sizing persists across a plain `crc start` unless it's explicitly changed via `crc config set` first.

## 3. Redeploy the AWX Operator

```
cd ~/awx-operator
git fetch --tags   # pull the latest operator release
make deploy
```

### Gotcha: kube-rbac-proxy sidecar ImagePullBackOff

`gcr.io/kubebuilder/kube-rbac-proxy:v0.15.0` no longer resolves (GCR has been migrating images over to registry.k8s.io). Fix:

```
oc set image deployment/awx-operator-controller-manager \
  kube-rbac-proxy=registry.k8s.io/kubebuilder/kube-rbac-proxy:v0.15.0 -n awx
```

### Gotcha: AWX custom resource needs an explicit Route

This operator version does **not** default to creating an OpenShift Route on its own. The AWX custom resource must explicitly set:

```
ingress_type: Route
```

`service_type: ClusterIP` is fine as the default, so no change is needed there.

## 4. Recreate AWX database objects

Everything below lived in the deleted cluster's database and must be rebuilt from scratch after every `crc delete`.

### Execution Environment

- Administration → Execution Environments → Add
- Image: `ghcr.io/jayberryman/skybaer-awx-ee-custom:<current tag>`. Check GHCR for the current tag (public image, no credential needed)
- Pull policy: **Always** (recommended while still iterating on the image)
- Organization: Skybaer Labs (or blank for globally available)
- Assign it to each Job Template that needs it (currently: anything using `password_hash()` needs `passlib`; anything using `community.windows.win_lineinfile` needs that collection)

### Credentials

- **Source Control credential** (deploy key) for the private lab automation repo
  - **Gotcha:** on the classic UI's Create Project form, the "Source Control Credential" field is a lookup/search widget. Typing the credential name directly into it, instead of clicking the magnifying glass and selecting it from the search results, sets a plain string instead of a real reference, and silently fails client-side validation with no visible error. Always select it via the picker.
- **Machine Credential** (SSH). Linux hosts authenticate as `root`; the Windows host needs `ansible_user=jayberryman` set as a host var override, since Windows has no root account

### Project

- `Ansible-Lab`, pointed at the private lab automation repo, using the Source Control credential above

### Inventory

- `RHCE-Lab` inventory, with an Inventory Source (Source Control-based) pointing at the `inventory` file already committed in that repo
- **Important:** Project sync and Inventory Source sync are two *separate* operations. Syncing the Project only refreshes the checked-out files in AWX's local copy of the repo; it does not re-parse them into Groups/Hosts. The Inventory Source itself must be synced separately (Inventories → RHCE-Lab → Sources → Sync) before new groups/hosts actually show up in AWX.

### Job Templates

Recreate each of:

- **Lab Shutdown** (has a multi-choice Survey)
- **Apache Install**
- **Password Policy Enforcement**
- **User account Creation** (Survey: `target_group` multi-choice, `target_user_ID` text, `target_comment` text, `target_temp_passwd` Password type, 14-character minimum)
- Any others currently in use (e.g. `fix_hosts_file.yml`)

### Teams / RBAC

- Recreate the Sky/Fire Team and its scoped (execute-only) permissions on the relevant Job Template(s)

## 5. Inventory variable gotcha: `ansible_shell_type`

Setting `ansible_shell_type` via `group_vars`/`host_vars` files placed alongside the SCM-sourced inventory does **not** reliably survive AWX's sync (matches the pattern described in ansible/awx GitHub issues #15438 and #12829).

**Working fix:** set it directly inside the flat inventory file itself, using INI-style `=` syntax in a `[windows:vars]`-style section, not YAML `:` syntax, and not a separate `group_vars`/`host_vars` file.

## 6. CRC node clock drift

The lab runs two layers of virtualization: the physical host, the `rhel10-crc` VM, and CRC's own inner VM. The inner VM doesn't reliably resync its clock when the physical host wakes from sleep, the way the outer VM does via VMware Tools.

- **Symptom:** AWX's UI shows stale/wrong timestamps even after a fresh login (ruling out browser cache)
- **Diagnose:** `oc debug node/<nodename>` (get the node name via `oc get nodes`), then `chroot /host`, then `date`. Do **not** use `crc ssh`; it doesn't exist in current `crc` CLI versions (confirmed absent as of 2.63.0+3a67a3). `crc console` only prints/opens the OpenShift web console URL, it doesn't provide a shell either.
- **Stopgap:** a `crc stop` / `crc start` corrects the clock immediately
- **Permanent fix (not yet implemented):** a `MachineConfig` object shipping a proper `chrony` config via Ignition to the CRC node's MachineConfigPool (likely `master`), rather than a manual/non-persistent fix on the immutable RHCOS node. Chosen deliberately for OpenShift admin practice, and because it preserves the host's power-saving/sleep settings instead of requiring sleep to be disabled.

## 7. After rebuild

- Confirm job templates run clean, starting with something low-stakes (Lab Shutdown or Apache Install) before trusting anything more complex
- Take a **fresh** VMware snapshot of `rhel10-crc` once the new build is confirmed stable. An old snapshot taken before a rebuild will restore the *previous* (pre-rebuild) config if ever reverted to, which is easy to forget and a source of confusing "why did my memory setting revert" moments later
- Note whatever CRC memory/CPU sizing was used, and check whether `rhel10-crc`'s own VMware-assigned memory needs a matching downward adjustment. Host-level RAM oversubscription is a known constraint in this lab, and CRC generally needs to run in relative isolation or alongside only light guests
