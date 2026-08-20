# zt_security_lockdown

CNV network lockdown for Zero Touch sandbox namespaces.

## What it does

Applies a single namespace-wide `NetworkPolicy` (`podSelector: {}`, so it
applies to every pod/VM in the namespace) that only allows the ingress/egress
traffic Zero Touch's bastion + showroom setup needs — DNS, SSH from the
showroom pod to the bastion VM's `virt-launcher` pod, and the small set of
HTTP(S)/showroom ports. Lab-specific extra rules (from
[`zt_base_config`](../zt_base_config/README.md)'s
`zero_touch_ingress_lockdown_rules` / `zero_touch_egress_lockdown_rules`
outputs) are appended to the fixed defaults, not replacing them.

This is a direct port of `redhat-cop/agnosticd`'s
`ansible/configs/zero-touch-base-rhel/lock_bastion_security_group_openshift_cnv.yml`
and its default rule sets in `default_vars_openshift_cnv.yaml` — the rule
sets in [`defaults/main.yml`](defaults/main.yml) are copied verbatim from v1,
not reinvented.

## Known gap — no explicit rule for the OpenShift API server

The default egress allow-list (`zt_security_lockdown_default_egress_rules`)
does not include an explicit rule allowing egress to the OpenShift API server
(typically port 443 or 6443). If a lab's in-namespace workload (bastion
scripts, in-VM tooling, etc.) needs to call back into the cluster API after
this `NetworkPolicy` is applied, that traffic will be blocked. This rule set
is ported verbatim from v1's `default_vars_openshift_cnv.yaml`, so it's
presumably been fine in production so far — but it's worth validating for any
lab that adds new in-namespace-to-API-server traffic. If you hit this, add a
rule via `zero_touch_egress_lockdown_rules` (from the lab's `firewall.yaml`)
rather than editing the fixed defaults.

## Why there's no removal task

v1 never removes this NetworkPolicy explicitly either — it's deleted for
free when the sandbox namespace itself is torn down at environment destroy
time. This role follows the same assumption.

## Variables

See [`meta/argument_specs.yml`](meta/argument_specs.yml) for the full list of
`zt_security_lockdown_*` input variables and their defaults.

## Example

```yaml
post_software_final_workloads:
  localhost:
    - agnosticd.showroom.ocp4_workload_showroom
    - rhpds.zerotouch.zt_security_lockdown
```
