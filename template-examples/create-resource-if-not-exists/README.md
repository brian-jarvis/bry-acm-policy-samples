---
# Create Resource If Not Exists

Demonstrates a generic pattern for creating a resource only the first time a policy runs,
then leaving it alone — useful for seeding default configuration that a user is expected
to customize afterward, without the policy fighting those edits on every reconcile. Any
resource type can be substituted; this example seeds a `conjur-connect` ConfigMap into
every namespace opted in to CyberArk Conjur secret management, and supports an
operator-triggered one-shot reset back to the git-defined desired state.

## How It Works

`namespaceSelector` can't be combined with `object-templates-raw`, so this policy uses
`object-templates-raw` to `range` over the matching namespaces itself and only emits a
`ConfigMap` entry for a namespace when it should actually be created or reset:

```yaml
object-templates-raw: |
  {{- $mcTrigger := `{{hub dig "acm.policy.io/trigger" "" (lookup "cluster.open-cluster-management.io/v1" "ManagedCluster" "" $.ManagedClusterName).metadata.annotations hub}}` }}
  {{- range $ns := (lookup "v1" "Namespace" "" "" "cyberark-conjur-secrets-managed=true").items }}
  {{- $nsName := $ns.metadata.name }}
  {{- $existingCM := lookup "v1" "ConfigMap" $nsName "conjur-connect" }}
  {{- $lastTrigger := dig "metadata" "annotations" "acm.policy.io/last-trigger" "" $existingCM }}
  {{- $skip := and (not (empty $existingCM)) (or (empty $mcTrigger) (eq $mcTrigger $lastTrigger)) }}
  {{- if not $skip }}
  - complianceType: musthave
    objectDefinition:
      ...
  {{- end }}
  {{- end }}
```

- `$mcTrigger` reads the `acm.policy.io/trigger` annotation off this cluster's own
  `ManagedCluster` object. That object lives on the hub, so this is a hub-side lookup
  (`{{hub ... hub}}`), wrapped in a backtick string so the hub phase can text-substitute
  its resolved value in before the managed-side template parses the line — no manual
  quote-escaping needed. It's `""` if the `ManagedCluster` isn't annotated.
- `lookup "v1" "Namespace" "" "" "cyberark-conjur-secrets-managed=true"` finds every
  namespace on the cluster carrying that label, replacing what `namespaceSelector` used
  to do at the policy level.
- For each matching namespace, `$existingCM` looks up the live `conjur-connect` ConfigMap
  (empty if it doesn't exist yet), and `$lastTrigger` reads the `acm.policy.io/last-trigger`
  annotation already stamped on it (empty if absent).
- `$skip` is `true` when the ConfigMap already exists **and** either no trigger has ever
  been set on the `ManagedCluster`, or the trigger matches what was already applied last
  time. When `$skip` is `true`, the `if` block emits nothing for that namespace this
  reconcile, leaving the existing object completely untouched.
- When `$skip` is `false` — first-time creation, or a forced reset — the `ConfigMap` is
  (re)created and its `acm.policy.io/last-trigger` annotation is stamped with the current
  `$mcTrigger` value, so the very next reconcile finds the trigger already applied and
  `$skip` flips back to `true` automatically.

## Forcing a Reset

Annotate the target cluster's `ManagedCluster` object on the hub with any value:

```sh
oc annotate managedcluster <cluster-name> acm.policy.io/trigger="$(date +%s)" --overwrite
```

This forces every namespace on that cluster carrying `cyberark-conjur-secrets-managed=true`
to have its `conjur-connect` ConfigMap reset back to the git-defined desired state on the
next reconcile, and then automatically reverts to leave-alone behavior — no second,
manual flip-back step is needed. Annotating with the same value again is a no-op; use a
new value (e.g. a fresh timestamp) each time you want to trigger another reset.

## Targeting Namespaces

Namespace scoping is done inside the template itself (see "How It Works" above) rather
than via a policy-level `namespaceSelector`, since `namespaceSelector` isn't supported
alongside `object-templates-raw`. The ConfigMap is seeded into every namespace on a
targeted cluster carrying the `cyberark-conjur-secrets-managed=true` label.

## Targeting Clusters

Clusters are targeted via the `ft-create-resource-if-not-exists--enabled` placement
(clusters labeled `create-resource-if-not-exists=enabled`).
