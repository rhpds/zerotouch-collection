# rhpds.zerotouch

Ansible Collection with Zero Touch (ZT)-specific roles for [AgnosticD v2](https://github.com/rhpds/agnosticd-v2). Covers Phase 1 of the [Zero Touch → AgnosticD v2 migration](/home/andrew/src/proposals/zt-to-agd-v2/PROPOSAL-zerotouch-agd-v2-migration.md): the ZT-specific logic that has **no existing v2 equivalent**. See `README.md`'s "What's deliberately NOT in this collection" for what's intentionally excluded (and why) — don't build a role here for something that already has a general-purpose v2 equivalent.

## Stack

- Ansible Collection, namespace `rhpds`, name `zerotouch` (`galaxy.yml`)
- Requires ansible-core `>=2.15.0` (`meta/runtime.yml`)
- Depends on the `kubernetes.core` collection (`galaxy.yml` → `dependencies`)
- Target platform: OpenShift / CNV (kubernetes.core.k8s against OpenShift APIs — Routes, NetworkPolicies, Deployments)
- Test tooling: yamllint + ansible-lint (via `tox`), Molecule with `kind` for two of the three roles

## Commands

```bash
# Static checks (yamllint + ansible-lint) — run from the collection root
pip install tox
tox -c tests/static

# Per-role Molecule scenarios — run from the collection root (where galaxy.yml
# lives); Molecule auto-detects the collection layout from there so
# rhpds.zerotouch.* FQCNs resolve inside converge/verify playbooks
pip install -r extensions/molecule/requirements.txt
molecule test -s zt_base_config          # no live cluster needed
molecule test -s zt_security_lockdown    # needs the `kind` CLI
molecule test -s zt_containers           # needs the `kind` CLI
```

## Project Structure

```
galaxy.yml                          Collection metadata: namespace/name/version, kubernetes.core dependency
meta/runtime.yml                    requires_ansible: >=2.15.0
README.md                           User-facing docs: role table, catalog wiring, what's out of scope
roles/
  zt_base_config/                   Dynamic per-lab config loader (firewall/instances/networks from content git repo)
  zt_security_lockdown/             CNV NetworkPolicy lockdown for ZT sandbox namespaces
  zt_containers/                    Provisions the `containers:` sidecar list (Deployment + Service/Route per entry, or per pod: group)
  <role>/tasks/                     main.yml (+ workload.yml/remove_workload.yml for zt_containers)
  <role>/defaults/main.yml          Default variable values
  <role>/meta/argument_specs.yml    Full option reference — single source of truth for variables
  <role>/README.md                  Role-specific docs; links to argument_specs.yml, doesn't duplicate it
extensions/molecule/<role>/         Per-role Molecule scenario: molecule.yml, create/converge/verify/destroy.yml
tests/static/                       yamllint + ansible-lint tox environment (tox.ini, collections-requirements.yml)
.ansible-lint / .yamllint           Lint config (ansible-lint: production profile; excludes tests/testing/molecule)
.github/workflows/static-checks.yml CI: yamllint + ansible-lint on push (main, v* tags) and PRs to main
.github/workflows/molecule.yml      CI: molecule test -s <scenario> (matrix, all 3 roles) on push (main, v* tags) and PRs to main
```

## Architecture

- **Catalog wiring order matters.** `zt_base_config` must be the first `pre_infra_workloads.localhost` entry — it publishes `zero_touch_ingress_lockdown_rules`, `zero_touch_egress_lockdown_rules`, `containers`, `instances`, and `networks` as facts that `zt_containers` (pre-infra) and `zt_security_lockdown` (post-software) consume. See `README.md`'s "Catalog wiring" section for the full rationale, including why catalog items must *omit* (not empty-list) those keys.
- `zt_base_config`'s facts use `set_fact: ... cacheable: true`. On the real deploy path this isn't actually required — AgnosticD v2's `ansible/main.yml` statically imports pre-infra and infra into one `ansible-playbook` process, so a plain `set_fact` is already visible. It *is* required for this role's own Molecule scenario, where `converge.yml`/`verify.yml` run as two separate `ansible-playbook` processes relying on `molecule.yml`'s jsonfile `fact_caching`.
- Showroom deployment is deliberately **not** a role in this collection — use `agnosticd.showroom.zerotouch_showroom` (in `rhpds/showroom`), not the generic `agnosticd.showroom.ocp4_workload_showroom`. See `README.md`'s "What's deliberately NOT in this collection" for why (RBAC gap on shared CNV sandbox namespaces — Phase 3 / Key Decision #4 in the migration proposal).
- `zt_containers` and `zt_security_lockdown` call `kubernetes.core.k8s` directly rather than being upstreamed into `cloud_provider_openshift_cnv` — a deliberate choice (migration proposal Key Decision #7) to avoid touching a shared repo other teams own.

## Code Conventions

- Every role's public variables are prefixed `{role_name}_*` (e.g. `zt_security_lockdown_namespace`, `zt_base_config_content_git_repo`). Internal/computed values are prefixed `_` (e.g. `_zt_base_config_instances_file_content`).
- `meta/argument_specs.yml` is the single source of truth for a role's variables (type, default, description). Role READMEs link to it instead of duplicating a variable table — keep that pattern when adding variables.
- `zt_base_config_content_git_repo` falls back to `ocp4_workload_showroom_content_git_repo` (see `roles/zt_base_config/defaults/main.yml`) so one catalog parameter drives both showroom content and ZT config — this is intentional and still correct even though showroom deployment itself now uses `zerotouch_showroom`, not `ocp4_workload_showroom`.
- `ACTION`-gated provision/destroy convention: tasks branch on `ACTION in ['provision', 'create']` vs `ACTION in ['destroy', 'remove']`, matching the convention used by the sibling showroom roles.
- Registered results in retry loops follow `r_<role>_<thing>` naming (e.g. `r_zt_base_config_firewall`, `r_zt_security_lockdown`).

## Testing

- `tests/static/tox.ini` runs two envs against the whole repo (`changedir = ../..`): `yamllint` (extends `default`, 200-char lines, relaxed comment spacing — see `.yamllint`) and `ansible-lint` (`production` profile, `kubernetes.core` installed first via `collections-requirements.yml`; excludes `tests/`, `testing/`, `extensions/molecule/`; skips `var-naming[no-role-prefix]` and `galaxy[no-changelog]` — see `.ansible-lint`).
- Each role has its own Molecule scenario under `extensions/molecule/<role>/`. `zt_base_config`'s needs no live cluster; `zt_security_lockdown` and `zt_containers` need the `kind` CLI (the latter also applies a manual Route CRD via `files/route-crd.yaml` since `kind` doesn't have OpenShift's Route API — full Route reconciliation can only be exercised on a real OpenShift/CNV cluster).
- **CI runs both static checks and Molecule** (see below) — `kind`/Docker ship preinstalled on `ubuntu-latest` runners, so no extra setup step was needed to wire the `kind`-dependent scenarios in. You can still run `molecule test -s <role>` locally for faster iteration.
- New/changed variables need a matching update to that role's `meta/argument_specs.yml` (and Molecule fixtures if the change affects `converge.yml`/`verify.yml`).

## CI/CD

- `.github/workflows/static-checks.yml`: one job (`static-checks`, Python 3.12) running `pip install tox && tox -c tests/static` on push to `main`, push of `v*` tags, and PRs targeting `main`.
- `.github/workflows/molecule.yml`: matrix job (`zt_base_config`, `zt_containers`, `zt_security_lockdown`) running `molecule test -s <scenario>` on the same triggers.
- There is no separate build/publish job yet (no Galaxy publish automation).

## Boundaries

**Always:**
- Keep new roles scoped to genuinely ZT-specific logic with no existing v2 equivalent — check `README.md`'s "What's deliberately NOT in this collection" first.
- Document new/changed variables in the role's `meta/argument_specs.yml`; let READMEs link to it rather than duplicating.
- Run `tox -c tests/static` before committing; run the affected role's Molecule scenario before merging task-logic changes.

**Never:**
- Don't duplicate logic that already has a general-purpose AgnosticD v2 equivalent (student/lab user creation, LiteMaaS keys, showroom deployment) — reference it directly instead.
- Don't upstream OpenShift/CNV-specific logic into shared repos other teams own (e.g. `cloud_provider_openshift_cnv`) — implement it directly with `kubernetes.core` in this collection instead.
- Don't have catalog items set empty-list placeholders for `instances`/`networks`/`containers`/`zero_touch_*_lockdown_rules` — extra-vars always beat the role's `set_fact`, so an empty placeholder silently zeroes out the lab config.

**Ask first:**
- Adding a new Ansible Galaxy dependency to `galaxy.yml`.
- Renaming or removing a role's public (`{role_name}_*`) variables — existing AgnosticV catalog items depend on the current names.

## Known Gotchas

- Showroom deployment is `agnosticd.showroom.zerotouch_showroom`, **not** `agnosticd.showroom.ocp4_workload_showroom` — the latter was the original Phase 3 plan but was superseded after hitting an RBAC gap on shared CNV sandbox namespaces. Don't reintroduce `ocp4_workload_showroom` references for showroom deployment.
- `zt_base_config`'s `cacheable: true` is only load-bearing for its own Molecule scenario (two separate `ansible-playbook` processes), not the real deploy path (one process via `import_playbook`). Don't read the Molecule setup as evidence for why `cacheable: true` matters in production.
- `zt_containers`' `commands:` (one-time exec) task path was fixed and live-tested against a real cluster (GPTEINFRA-17763): `kubernetes.core.k8s_exec`'s `command` option is `type: str`, not a list — pass a single string built with the `quote` filter, not a `["/bin/sh", "-c", ...]` list, or quoted shell commands break. It still can't be exercised end-to-end in this collection's own `kind`-on-nested-Podman Molecule sandbox (`k8s_exec` fails there independent of the role — confirmed `kubectl exec` fails identically outside Ansible).
- `zt_security_lockdown`'s default egress rules have no explicit rule for the OpenShift API server (443/6443), ported verbatim from v1. Confirmed real via live testing (a raw `nc`/authenticated `curl` to the in-cluster API times out), but confirmed non-blocking in practice — `zerotouch-automation`'s `core/user_data.py` treats the in-cluster K8s API as a non-fatal fallback. Add a rule via `zero_touch_egress_lockdown_rules` if a lab actually depends on that fallback path.
- `route.openshift.io/v1` `Route` objects only exist on OpenShift, so `zt_containers`' Route creation can only be fully exercised on a real OpenShift/CNV cluster (or `kind` + a manually-applied Route CRD for structural validation only).
- `zt_base_config_ensure_bastion_group` defaults `false`, not `true` — an earlier revision defaulted `true` and silently broke real Ansible BU labs (`zt-ans-bu-eda-netbox`, `zt-ans-bu-windows-ad`) where every VM is deliberately `isolated`-only by design. When the fixup does run, it drops `isolated` entirely rather than just adding `bastions` alongside it — keeping both tags left the VM excluded from v2 core's Linux connection-setup step and SSH unresolvable. See `zt_base_config`'s README for the full live-testing writeup.
- `zt_containers`' `pod:` grouping bundles multiple `containers[]` entries into one Deployment/Pod, sharing one network namespace — containers in the same group must use distinct listening ports, since a second bind to the same port fails with a real OS-level `EADDRINUSE` that neither the Kubernetes API nor this role's validation can catch ahead of time. See `zt_containers`' README for the full write-up, including the separate (no-schema-change) option of just using Service DNS across separate Deployments for containers that only need to reach each other on an already-allowed `zt_security_lockdown` port.

## Git Workflow

- Branch from `main`; observed prefixes are `add-` (new roles/scaffolding) and `fix/` (bug fixes) — follow the same pattern for new work.
- PRs must pass the `static-checks` CI job before merging to `main`.
