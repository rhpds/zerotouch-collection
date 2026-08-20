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
  `INSTANCEGUID` placeholder replaced by the real `guid`)
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

## Open question — deferred to Phase 2

The migration proposal's own sample AgnosticV catalog item keeps these
lookups as **inline catalog Jinja** (identical to v1), and never actually
invokes this role. Its Roles Breakdown table, on the other hand, separately
lists `zt_base_config` as a role meant to run in `pre_infra_workloads`.

This role is written so either path works:

- **As inline Jinja** (proven, zero risk): copy the lookup expressions out of
  `tasks/main.yml` directly into the catalog item's top-level variables, the
  same way v1 does it.
- **As a role** (cleaner, single-sourced, needs validation): include
  `rhpds.zerotouch.zt_base_config` from `pre_infra_workloads.localhost`. This
  role publishes its outputs via `set_fact ... cacheable: true` specifically
  so they have the best chance of surviving into later deployment steps
  (instance/network creation) if those steps run as separate `ansible-runner`
  invocations sharing a fact cache backend. **Whether AgnosticD v2's runner
  architecture actually requires — or even supports — cacheable facts across
  step boundaries is unconfirmed and is explicitly a Phase 2 validation item,
  not resolved by this role.**

## Variables

See [`meta/argument_specs.yml`](meta/argument_specs.yml) for the full list of
`zt_base_config_*` input variables and their defaults.

## Example

```yaml
pre_infra_workloads:
  localhost:
    - rhpds.zerotouch.zt_base_config
```
