# zt_base_config

Dynamic per-lab configuration loader for Zero Touch (ZT).

## What it does

Zero Touch labs keep their firewall rules, VM instance definitions, network
definitions, and (optionally) sidecar container definitions in a `config/`
directory inside the lab's own content git repo, alongside the Antora
documentation. This role fetches those files from the raw githubusercontent.com
mirror of that repo, parses them, and publishes the results as facts:

- `zero_touch_egress_lockdown_rules` (from `firewall.yaml`'s `egress` key)
- `zero_touch_ingress_lockdown_rules` (from `firewall.yaml`'s `ingress` key)
- `containers` (from `instances.yaml`'s `containers` key — consumed by
  [`zt_containers`](../zt_containers/README.md))
- `instances` (from `instances.yaml`'s `virtualmachines` key, with the
  `INSTANCEGUID` placeholder replaced by the real `guid`, and — only when
  `zt_base_config_ensure_bastion_group` is explicitly set `true` — the first
  VM's `AnsibleGroup` tag patched to include `bastions`; see "Bastion group
  fixup" below)
- `networks` (the full parsed contents of `networks.yaml`)

This reproduces the Jinja lookups that Zero Touch v1 duplicates as inline
catalog-item variables in every business unit's `common.yaml` (see
`zt-rhelbu-agnosticv`'s `zt-rhelbu/zt-rhel-bu-lab-developer-cnv/common.yaml`
for the v1 version of this exact logic), centralizing it in one place per
[Cross-BU concern #8](/home/andrew/src/proposals/zt-to-agd-v2/PROPOSAL-zerotouch-agd-v2-migration.md)
of the migration proposal.

## Requirements and fetch behavior

- `zt_base_config_content_git_repo` must be an `https://github.com/<org>/<repo>.git`
  URL — other git hosts, and URLs missing the `.git` suffix, are not supported
  and will fail the role early with a clear error rather than silently
  producing a broken raw-content URL.
- `instances.yaml` and `networks.yaml` are **required**: if either file can't
  be fetched (missing, or the request fails after retries), the role fails
  loudly instead of silently continuing with an empty configuration.
- `firewall.yaml` is optional. Its handling is governed by
  `zt_base_config_lookup_errors` (`warn` | `ignore` | `strict`, default
  `warn`): `warn`/`ignore` tolerate a missing file (empty ingress/egress
  rules), `strict` requires it to exist.
- All three fetches retry transient failures using
  `zt_base_config_fetch_retries` / `zt_base_config_fetch_delay` (default `5`
  retries, `3`s delay).

## Bastion group fixup

Some BUs' content repos tag their access VM `AnsibleGroup: isolated` only,
never `bastions` — confirmed against every real Ansible BU lab checked so far
([`zt-ans-bu-dev-tools`](https://github.com/rhpds/zt-ans-bu-dev-tools/blob/main/config/instances.yaml),
[`zt-ans-bu-eda-netbox`](https://github.com/rhpds/zt-ans-bu-eda-netbox/blob/main/config/instances.yaml),
[`zt-ans-bu-windows-ad`](https://github.com/rhpds/zt-ans-bu-windows-ad/blob/main/config/instances.yaml),
[`zt-ans-bu-windows90`](https://github.com/rhpds/zt-ans-bu-windows90/blob/main/config/instances.yaml)).
`cloud-vms-base`/`cloud_provider_openshift_cnv` treat `isolated` as a
**reserved exclusion group**, same tier as `windows`/`network`
(`hosts: all:!windows:!network:!isolated` in both `workloads.yml` and
`infrastructure_deployment.yml`) — a VM tagged only `isolated` gets no
SSH/wait-for-connection and no `control_user`/`asset_injector` at all.

For labs that genuinely need one of their VMs to be a real Ansible-managed
bastion (e.g. `zt-ans-bu-dev-tools`, a single-VM lab needing
`groups['bastions'][0]` to resolve for a showroom hostname), this role can
rewrite the **first** VM's tag to be `bastions` — **dropping `isolated`
specifically**, not just adding `bastions` alongside it (any other,
unrelated token in the same tag is preserved) — when set
`zt_base_config_ensure_bastion_group: true`. **This is opt-in, not the
default.**

**Why this is opt-in (`zt_base_config_ensure_bastion_group` defaults
`false`), not a default-on rewrite:** an earlier version of this role
defaulted to `true`, silently promoting the first VM whenever no VM already
tagged `bastions`. Live-tested TWICE that this actively breaks content that
was already working correctly under v1 (GPTEINFRA-17763, guids `62n8j`/
`bfvrx` for `zt-ans-bu-eda-netbox`, `d9v4h` for `zt-ans-bu-windows-ad`): both
repos tag *every* VM `isolated`-only on purpose — no VM is meant to be
Ansible-managed at all, access is purely via showroom's per-VM wetty
sidecars using each VM's own cloud-init-baked credentials. The old
true-default promoted their first VM ("control") out of `isolated` into
`bastions` regardless, pulling it into `cloud-vms-base`'s generic
cloud-user-based connection setup:

```
fatal: [control]: FAILED! => {"msg": "timed out waiting for ping module
test: Failed to connect to the host via ssh: cloud-user@ssh.ocpvdev01.rhdp.net:
Permission denied (publickey, gssapi-keyex, gssapi-with-mic, password)."}
```

— which "control" cannot support (confirmed via direct SSH into a real v1
order for the identical content, guid `bfvrx`: `cloud-user` doesn't exist on
this image in v1 either). v1 never did this kind of tag-rewriting at all. A
content repo that genuinely needs a real Ansible-managed bastion should tag
that VM `AnsibleGroup: bastions` directly in its own `config/instances.yaml`
— this role's fixup is an escape hatch for catalog authors who can't or
don't want to touch the content repo itself, not a silent default behavior.

**Why drop `isolated` entirely, not just add `bastions` alongside it, when
the fixup does run (live-tested, GPTEINFRA-17763 guid `hcshc`):** an earlier
revision of this fixup only *prepended* `bastions,` while keeping `isolated`
(e.g. `bastions,isolated`), mirroring `cloud-vms-base`'s own
`"bastions,showroom"` bastion convention. That correctly got the VM targeted
by `bastions:`-scoped workload plays (`pre_software_workloads.bastions` ran
`control_user`/`asset_injector` successfully), but SSH itself still failed
with `Could not resolve hostname vscode` — because `agnosticd-v2`'s "Step
001.3 Configure Linux hosts and wait for connection"
(`hosts: all:!windows:!network:!isolated`) is what actually sets
`ansible_ssh_extra_args: -F <generated ssh_config>`, and it excludes *any*
host still tagged `isolated`, even if also tagged `bastions`. Without that
fact set, Ansible fell back to a raw `ssh vscode` with no `-F` flag, and the
bare hostname doesn't resolve outside the generated per-guid ssh_config's
`Host` alias. Dropping `isolated` entirely (leaving just `bastions`)
resolves this, since the VM is then no longer excluded from Step 001.3
either. `ansible.builtin.add_host`'s `groups:` parameter splitting a
comma-joined string into multiple inventory groups (confirmed directly
against `ansible-core`'s `add_host` action plugin source) is still what
makes plain `bastions` alone sufficient for every other purpose (workload
targeting, `create_inventory`'s bastion detection, SSH config generation).

This is a no-op (regardless of `zt_base_config_ensure_bastion_group`) for
content that already tags its bastion-equivalent VM `bastions` (RHEL BU, HP
BU, and any Ansible BU lab that already does the right thing), and a no-op
for an empty `virtualmachines` list. The role always reports its decision —
whether it found an existing `bastions` tag, promoted a VM, or left every VM
untouched (with guidance on how to opt in) — via an unconditional
`ansible.builtin.debug` message, so this is never a silent no-op.

**Downstream consequence for catalog authors:** if you do opt in
(`zt_base_config_ensure_bastion_group: true`), the promoted VM's
`AnsibleGroup` will no longer include `isolated`, so any catalog-item Jinja
that reads `groups['isolated'][0]` (Ansible BU's v1-era convention for "the
access VM") should be changed to `groups['bastions'][0]` instead — which is
guaranteed to resolve to this same VM, and matches RHEL BU/HP BU's own
existing convention. See `tests/zt-ansiblebu-base-component/common.yaml` in
`rhpds/agnosticv` for the corrected pattern (including a
`groups['bastions'][0] if ... else groups['isolated'][0]` fallback for
content that leaves `zt_base_config_ensure_bastion_group` at its `false`
default, where no VM ever gets promoted).

## Catalog wiring

Invoke this role as the **first** `pre_infra_workloads.localhost` entry.
AgnosticD v2's `ansible/main.yml` statically imports (`import_playbook`,
not a separate ansible-runner invocation) `configs/{{ config }}/pre_infra.yml`
and then the cloud provider's `infrastructure_deployment.yml` into one
`ansible-playbook` process. Because both plays target `localhost` within
that single process, a plain `set_fact` is already visible to VM/network
creation — no fact caching is required on this path.

`cacheable: true` is used anyway, defensively, and because it is genuinely
required for this role's own Molecule scenario: `converge.yml` and
`verify.yml` run as two separate `ansible-playbook` processes, so
`molecule.yml`'s jsonfile `fact_caching` is what lets `verify.yml` see the
facts `converge.yml` published. Don't read that test setup as evidence that
`cacheable: true` is what makes the real `cloud-vms-base` deploy work — it
isn't; `import_playbook` is.

Catalog items must **omit** `instances`, `networks`, `containers`, and the
`zero_touch_*_lockdown_rules` keys. Extra-vars always beat `set_fact`, so
an empty placeholder (`instances: []`) would provision zero VMs. To
override a git file, set a real list in the catalog. To skip git config
entirely, omit this role from workloads and declare everything yourself.

Do not skip the fetch just because `instances is defined`:
`cloud-vms-base`'s CNV `default_vars` already define a generic `bastion`
via `include_vars`. Naive skip-if-defined would keep that default and never
load the lab.

The v1 inline `lookup('ansible.builtin.url', ...)` copies in each BU's
`common.yaml` are the thing this role replaces, not a supported alternate
path.

## Variables

See [`meta/argument_specs.yml`](meta/argument_specs.yml) for the full list of
`zt_base_config_*` input variables and their defaults.

## Example

```yaml
pre_infra_workloads:
  localhost:
    - rhpds.zerotouch.zt_base_config
    - rhpds.zerotouch.zt_containers
```
