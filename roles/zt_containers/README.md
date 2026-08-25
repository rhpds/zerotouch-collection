# zt_containers

Provisions the Zero Touch `containers:` sidecar list — genuinely new
capability, not a port (see the migration proposal's Risk #1: this doesn't
work in Zero Touch v1 either).

## What it does

For each entry in the `containers` list (typically produced by
[`zt_base_config`](../zt_base_config/README.md) from a lab's
`instances.yaml`), when `ACTION` is `provision` or `create`:

1. Creates an `apps/v1` `Deployment` (1 replica) named after the container,
   in `zt_containers_namespace`.
2. Waits for the Deployment to become `Available` (unless
   `zt_containers_wait_ready` is `false`).
3. Creates a `v1` `Service` for each entry in the container's `services:`
   list, and a `route.openshift.io/v1` `Route` for each entry in its
   `routes:` list — using the exact same shape as VM `instances[].services`/
   `routes` already used by `cloud_provider_openshift_cnv`.
4. Runs any `commands:` (one-time shell commands, e.g. seeding a database or
   creating an admin user) once inside the running container via
   `kubernetes.core.k8s_exec`.

When `ACTION` is `destroy` or `remove`, it deletes the Routes, Services, and
Deployments it created (in that order), matching the `ACTION`-gated
provision/destroy convention used by `agnosticd.showroom.ocp4_workload_showroom`.

Deliberately implemented with `kubernetes.core.k8s` directly, not upstreamed
into `cloud_provider_openshift_cnv`, per the migration proposal's Key
Decision #7 (avoids touching a shared repo other teams own).

## `containers[]` entry schema

Field names intentionally match Kubernetes' own PodSpec fields directly
(`ports`, `volumeMounts`, `volumes`) so no translation layer is needed. Based
on the one confirmed-real (if "under development") usage found:
[`rhpds/zt-ans-bu-eda-netbox`](https://github.com/rhpds/zt-ans-bu-eda-netbox/blob/main/config/instances.yaml).

```yaml
containers:
  - name: gitea                # required
    image: gitea/gitea:1.16.8-rootless   # required
    ports:
      - name: gitea
        containerPort: 3000
        protocol: TCP
    environment:                # dict, converted to env: list internally
      GITEA__DEFAULT__RUN_MODE: dev
    volumeMounts:
      - name: gitea-data
        mountPath: /data/
    volumes:
      - name: gitea-data
        emptyDir: {}
    commands:                   # one-time init commands, run after the pod is Ready
      - gitea admin user create --admin --username gitea --password gitea --email dummy@dummy.com
    memory: 2Gi
    services:
      - name: gitea
        ports:
          - port: 3000
            protocol: TCP
            targetPort: 3000
            name: gitea
    routes:
      - name: gitea
        host: gitea
        service: gitea
        targetPort: 3000
        tls: true
        tls_termination: edge
```

See [`meta/argument_specs.yml`](meta/argument_specs.yml) for the full option
reference.

## Known limitations

- `route.openshift.io/v1` `Route` objects only exist on OpenShift, so this
  role can only be fully exercised against a real OpenShift/CNV cluster (or a
  `kind` cluster with the Route CRD installed for structural validation only —
  see `extensions/molecule/zt_containers`). Full Route reconciliation is a
  Phase 2/3 validation item, not covered by this collection's automated tests.
- **Live-tested and fixed** (GPTEINFRA-17763 Phase 5 validation, against a
  real `ocpvdev01` order of `zt-ans-bu-dev-tools`' `gitea` sidecar): the
  `commands:` exec step originally passed `command` as a 3-element YAML list
  (`["/bin/sh", "-c", "{{ zt_pair[1] }}"]`), but
  `kubernetes.core.k8s_exec`'s `command` option is documented and enforced as
  `type: str`, not a list — Ansible's own type coercion stringified the list
  into its Python `repr()` before the module ever saw it, and the module's
  own internal `shlex.split()` call then failed with `No closing quotation`
  on any command containing quotes (e.g. `curl -d '{"key": "value"}'`, exactly
  what `zt-ans-bu-dev-tools`'s `gitea` `commands:` use). Fixed by passing a
  single string built with Ansible's `quote` filter
  (`command: "/bin/sh -c {{ zt_pair[1] | quote }}"`), which round-trips
  correctly through the module's own `shlex.split()`.
- `kind` running on nested/rootless Podman independently fails
  `kubernetes.core.k8s_exec` calls into pod containers with
  `OCI runtime exec failed` (confirmed `kubectl exec` against the same pod
  fails identically, outside of Ansible entirely — this points at the
  container-nesting depth, not the role), so this Molecule scenario's
  `commands:` step still can't be exercised end-to-end in CI even after the
  fix above — only the real live-cluster order caught and confirmed it.

## Example

```yaml
pre_infra_workloads:
  localhost:
    - rhpds.zerotouch.zt_containers
```
