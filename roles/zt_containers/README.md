# zt_containers

Provisions the Zero Touch `containers:` sidecar list — genuinely new
capability, not a port (see the migration proposal's Risk #1: this doesn't
work in Zero Touch v1 either).

## What it does

For each entry in the `containers` list (typically produced by
[`zt_base_config`](../zt_base_config/README.md) from a lab's
`instances.yaml`), when `ACTION` is `provision` or `create`:

1. Creates an `apps/v1` `Deployment` (1 replica) per entry, named after the
   container, in `zt_containers_namespace` — or, for entries that share a
   `pod:` value, one Deployment per group, with all of that group's
   containers in the same Pod template. See "Multi-container Pods" below.
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

## Containers talking to each other

Two containers[] entries can already reach each other today without any
special configuration, in either of two ways — pick whichever matches what
the lab actually needs:

- **Separate Deployments, over the Service network (the default).** Every
  container lands in the same namespace regardless of grouping, and
  [`zt_security_lockdown`](../zt_security_lockdown/README.md)'s default
  `NetworkPolicy` already allows namespace-wide traffic (not just external)
  on ports `80`/`443`/`8080`/`7171`/`8081`/`9000`. So if container `a`
  declares a `services:` entry, container `b` can already reach it at
  `a.<namespace>.svc` (or just `a`) on one of those ports with zero schema
  changes. Need a different port open cluster-wide? Add it via
  `zero_touch_ingress_lockdown_rules`/`zero_touch_egress_lockdown_rules`
  (from the lab's `firewall.yaml`, consumed by `zt_base_config`) — not a
  `zt_containers` change.
- **Same Pod, via `pod:` grouping (below).** Use this when containers need
  `localhost`-speed/no-Service communication, or need to share files through
  a common `volumes:` entry — the standard Kubernetes sidecar pattern, not
  just "a port being open".

## Multi-container Pods (`pod:` grouping)

Give two or more `containers[]` entries the same `pod:` value to bundle them
into one Deployment/Pod together, instead of one Deployment each:

```yaml
containers:
  - name: app
    pod: app-with-cache          # group name - any string
    image: quay.io/example/app:latest
    ports:
      - name: http
        containerPort: 8080
        protocol: TCP
    volumeMounts:
      - name: cache
        mountPath: /var/cache/app
    services:
      - name: app
        ports:
          - port: 8080
            targetPort: 8080
            protocol: TCP
            name: http
  - name: cache-warmer
    pod: app-with-cache          # same group as "app" above
    image: quay.io/example/cache-warmer:latest
    volumeMounts:
      - name: cache
        mountPath: /var/cache/app
    volumes:                     # declared once, on either member - deduped by name
      - name: cache
        emptyDir: {}
```

Both `app` and `cache-warmer` end up as two `containers:` entries inside a
single Pod named `app-with-cache`: they share that Pod's network namespace
(so `cache-warmer` can reach `app` at `localhost:8080`, no Service needed
between them) and the `cache` `emptyDir` volume declared once above.
`services:`/`routes:`/`commands:`/`memory`/`cpu`/`environment` stay exactly
as they are today — per-entry, not per-group.

Entries that don't set `pod:` default to their own `name` as the group key,
which is a group of one — identical to today's one-container-per-Deployment
behavior. Existing `containers:` lists need no changes.

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
- Containers bundled into the same `pod:` group share one network
  namespace, so they must each listen on a distinct port — a second
  container binding the same port fails with a real OS-level `EADDRINUSE`,
  which the Kubernetes API (and this role) has no way to catch ahead of
  time. This only applies within a group; containers in different groups
  never share a network namespace, so port reuse across groups is fine.
- Setting `pod:` on an entry to another entry's bare `name:` intentionally
  joins that entry's group (there's just one Deployment either way) — this
  is not treated as a naming collision, and the role does not warn about it.

## Example

```yaml
pre_infra_workloads:
  localhost:
    - rhpds.zerotouch.zt_containers
```
