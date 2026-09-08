# rhpds.zerotouch

Ansible Collection with Zero Touch (ZT) specific roles for
[AgnosticD v2](https://github.com/rhpds/agnosticd-v2). This is Phase 1 of the
[Zero Touch → AgnosticD v2 migration](/home/andrew/src/proposals/zt-to-agd-v2/PROPOSAL-zerotouch-agd-v2-migration.md):
building the ZT-specific logic that has **no existing v2 equivalent**.
Everything that already has a v2 equivalent is referenced directly, not
duplicated — see "What's deliberately not in this collection" below.

- **Namespace:** `rhpds`
- **Name:** `zerotouch`
- **Scope:** all three ZT business units — RHEL BU, HP BU, Ansible BU. This
  collection's roles are BU-agnostic; per-BU rollout status (which BUs have
  a real `-v2` AgnosticV base component in progress/review vs. not started)
  is tracked in
  [`STATUS-SUMMARY.md`](/home/andrew/src/proposals/zt-to-agd-v2/STATUS-SUMMARY.md),
  not fixed here.

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
  [`agnosticd.showroom.zerotouch_showroom`](https://github.com/rhpds/showroom/blob/main/roles/zerotouch_showroom/README.adoc)
  directly, pointed at the existing `zerotouch` Helm chart and the sandbox's
  existing namespace. Phase 3 originally planned to reuse the generic
  `agnosticd.showroom.ocp4_workload_showroom` role for this, but that role
  assumes it owns the namespace it creates — CNV sandboxes hand it a shared,
  pre-existing namespace the automation SA can't manage that way. The
  documented fallback (see Key Decision #4 in the migration proposal) was a
  new sibling role, `zerotouch_showroom`, ported from v1 and added directly
  to `rhpds/showroom`. Wherever else this document or a role README says
  "showroom", read that as `zerotouch_showroom`, not `ocp4_workload_showroom`.

## Catalog wiring: `zt_base_config` in `pre_infra_workloads`

Production path: invoke `rhpds.zerotouch.zt_base_config` as the **first**
`pre_infra_workloads.localhost` entry so its facts (`instances`, `networks`,
`containers`, lockdown rules) exist before `zt_containers` and before
`cloud-vms-base` creates VMs. This works via plain `set_fact` — pre-infra
and infrastructure deployment are statically imported into one
`ansible-playbook` process by `ansible/main.yml`, not run as separate
ansible-runner invocations, so no fact-cache backend is required for the
real deploy path (see the role's own README for where `cacheable: true`
actually matters — its Molecule test, not this one). Catalog items must
**omit** those keys (do not set empty lists) so extra-vars do not shadow
the role. To override a git file, set a real list in the catalog —
extra-vars always win. To skip git config entirely, omit the role and
declare everything yourself.

See [`roles/zt_base_config/README.md`](roles/zt_base_config/README.md).

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

This collection is only Phase 1-3 of the wider migration — the roles
themselves, validated against `cloud-vms-base` + `openshift_cnv`, plus the
showroom integration. Bridging each BU's remaining gaps and actually rolling
each BU's labs onto the new `-v2` AgnosticV base components happens in other
repos and is tracked separately. Live status (which changes independently of
this README) lives in
[`STATUS-SUMMARY.md`](/home/andrew/src/proposals/zt-to-agd-v2/STATUS-SUMMARY.md) —
as of that doc's last update:

- **Phases 1-3 — Collection + `cloud-vms-base` validation + Showroom
  integration — Done.** The `cloud-vms-base` validation used the
  `rhpds/agnosticv` `tests/zt-agd-v2-demo` test catalog. The showroom outcome
  was the new `agnosticd.showroom.zerotouch_showroom` role, not
  `agnosticd.showroom.ocp4_workload_showroom` as originally planned — see
  "What's deliberately NOT in this collection" above.
- **Phase 4 — Bridge HP BU gaps — Done.**
- **Phase 5 — Bridge Ansible BU gaps — In review** (found that v2 core
  treats the `isolated` AnsibleGroup tag as a reserved exclusion group; fixed
  in this collection's `zt_base_config` bastion-group fixup, see that role's
  README).
- **Phase 6 — RHEL BU rollout — Not started.** No dependency on Phases 4/5;
  can proceed any time. Target repo `zt-rhelbu-agnosticv`.
- **Phase 7 — HP BU rollout — In review.** Real `-v2` base component +
  pilot lab in `zt-hpbu-agnosticv`.
- **Phase 8 — Ansible BU rollout — In review.** Real `-v2` base component +
  pilot lab in `zt-ansiblebu-agnosticv`.
