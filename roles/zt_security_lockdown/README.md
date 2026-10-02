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

## In-cluster K8s API egress (GPTEINFRA-18137)

The default egress allow-list (`zt_security_lockdown_default_egress_rules`)
includes a rule allowing egress to the in-cluster Kubernetes API, scoped to
the Service CIDR (`172.30.0.0/16`) on ports 443 and 6443: 443 is the API
Service's ClusterIP port, and 6443 is the port OVN-Kubernetes DNATs that
traffic to on the backend apiserver running on a control-plane node. A rule
that only allows port 443 to the ClusterIP — or even "port 443 to
anywhere" — does **not** cover this, since ACL evaluation happens against
the DNATed destination port.

This was previously an uncovered gap, confirmed live against a provisioned
CNV sandbox: both a raw `nc` to the internal API service IP and an
authenticated `curl` from inside the namespace (using the pod's own mounted
ServiceAccount token) timed out, blocking every pod in the namespace,
including the bastion VM's `virt-launcher` pod, not just the showroom pod.
It was previously assessed as "non-blocking in practice" because
`zerotouch-automation`'s `core/user_data.py` treats the in-cluster K8s API
as a non-fatal fallback (behind a mounted ConfigMap file) — but that
assessment was wrong: with no request timeout on that call, a blocked
connection hangs for minutes inside the app's startup handler, delaying
showroom readiness by ~10 minutes after any pod restart that happens after
lockdown is in place. See GPTEINFRA-18137 for the live confirmation.

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
    - agnosticd.showroom.zerotouch_showroom
    - rhpds.zerotouch.zt_security_lockdown
```
