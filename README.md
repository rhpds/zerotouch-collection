# rhpds.zerotouch

Ansible Collection with Zero Touch (ZT) specific roles for
[AgnosticD v2](https://github.com/rhpds/agnosticd-v2). This is Phase 1 of the
[Zero Touch → AgnosticD v2 migration](/home/andrew/src/proposals/zt-to-agd-v2/PROPOSAL-zerotouch-agd-v2-migration.md):
building the ZT-specific logic that has **no existing v2 equivalent**.
Everything that already has a v2 equivalent is referenced directly, not
duplicated — see "What's deliberately not in this collection" below.

- **Namespace:** `rhpds`
- **Name:** `zerotouch`
- **Scope:** RHEL BU only for now (see the proposal's Cross-Business-Unit
  Considerations for HP BU / Ansible BU follow-on work)

## Roles

| FQCN | Purpose |
|------|---------|
| `rhpds.zerotouch.zt_base_config` | Dynamic per-lab config loader — fetches firewall/instances/networks config from a lab's content git repo. See [role README](roles/zt_base_config/README.md). |
| `rhpds.zerotouch.zt_security_lockdown` | Applies the CNV NetworkPolicy that locks a ZT sandbox namespace down to only the traffic it needs. See [role README](roles/zt_security_lockdown/README.md). |
| `rhpds.zerotouch.zt_containers` | Provisions the `containers:` sidecar list (Deployment + optional Service/Route per entry) — new capability, not a v1 port. See [role README](roles/zt_containers/README.md). |

(Rendered as a table here for a quick scan — see each role's own README for
full detail, since GitHub renders Markdown tables fine even though our
internal plan-writing convention avoids them.)

## What's deliberately NOT in this collection

Zero Touch v1 has several other pieces of logic that look ZT-specific at
first glance but already have a general-purpose AgnosticD v2 equivalent.
Building new roles for these would just be duplicate maintenance:

- **Student/lab user creation + SSH key setup** (v1: `zero_touch_rhel_user`
  role + inline SSH logic in `pre_software.yml`) → use
  [`agnosticd.cloud_vm_workloads.control_user`](https://github.com/rhpds/cloud_vm_workloads)
  and `.asset_injector` directly.
- **LiteMaaS virtual keys** (v1: inline workload in the base component) →
  use [`rhpds.litellm_virtual_keys`](https://github.com/rhpds/rhpds.litellm_virtual_keys)
  directly (`ocp4_workload_litellm_virtual_keys` /
  `ocp4_workload_litellm_bastion_profile`).
- **Showroom deployment** (v1: `ocp4_workload_showroom` role bundled in
  `redhat-cop/agnosticd`) → use
  [`agnosticd.showroom.ocp4_workload_showroom`](https://github.com/rhpds/showroom)
  directly, pointed at the existing `zerotouch` Helm chart and the sandbox's
  existing namespace.

## Open design question: `zt_base_config` wiring

The migration proposal's own sample catalog item keeps the git-based config
lookups as inline catalog Jinja (identical to Zero Touch v1) rather than
invoking `zt_base_config` as a role, while its Roles Breakdown table
separately lists `zt_base_config` as a `pre_infra_workloads` role. This
collection builds `zt_base_config` as a role — usable either via
`include_role` or as a reference implementation to copy back into inline
catalog Jinja — and defers the actual choice to Phase 2, since it depends on
how AgnosticD v2's runner persists facts across step boundaries (unconfirmed
without a live test). See
[`roles/zt_base_config/README.md`](roles/zt_base_config/README.md) for the
full explanation.

## Requirements

- `kubernetes.core` collection (declared as a `galaxy.yml` dependency)
- Ansible core `>=2.15` (see `meta/runtime.yml`)

## Development

```bash
# Static checks (yamllint + ansible-lint)
pip install tox
tox -c tests/static

# Per-role Molecule scenarios
pip install -r extensions/molecule/requirements.txt
molecule test -s zt_base_config          # no live cluster needed
molecule test -s zt_security_lockdown    # needs the `kind` CLI
molecule test -s zt_containers           # needs the `kind` CLI
```

Run `molecule` commands from the collection root (where `galaxy.yml` lives)
— Molecule auto-detects the collection layout from there so
`rhpds.zerotouch.*` FQCNs resolve inside the converge/verify playbooks.

## What's out of scope for this collection (deferred to a follow-up plan)

Per the migration proposal, this collection only covers **Phase 1**. Phases
2-5 all require a live OCP/CNV test cluster to validate properly and are
tracked separately:

- **Phase 2:** confirm `cloud-vms-base` + `openshift_cnv` actually wires
  these roles in the way this collection assumes (particularly the
  `zt_base_config` open question above).
- **Phase 3:** validate `agnosticd.showroom.ocp4_workload_showroom` against
  `cloud-vms-base` + `openshift_cnv` (no existing precedent for this
  combination).
- **Phase 4:** write the new `zt-rhelbu/zt-rhel-bu-lab-developer-cnv-v2`
  AgnosticV base component that actually wires this collection's roles
  together with `cloud_vm_workloads`, `showroom`, and `rhpds.litellm_virtual_keys`.
- **Phase 5:** end-to-end testing and incremental lab migration.
