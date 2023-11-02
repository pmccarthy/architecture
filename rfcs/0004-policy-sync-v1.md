# RFC - Policy Sync

- Feature Name: `policy_sync_v1`
- Start Date: 2023-10-10
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000)
- Issue tracking: [https://github.com/Kuadrant/architecture/issues/26](https://github.com/Kuadrant/architecture/issues/26)

# Summary
[summary]: #summary

The ability for the Multicluster Gateway Controller to sync policies defined in
the hub cluster downstream to the spoke clusters, therefore allowing all policies
to be defined in the same place. These policies will be reconciled by the downstream
Gateway controller.

# Nomenclature

* Policy: When refering to a Policy, this document is refering to a Gateway API
  policy as defined in the Policy Attachment Model. The Multicluster Gateway Controller
  relies on [OCM]() as a Multicluster solution, which defines its own unrelated
  set of Policies and Policy Framework. Unless explicitely mentioned, this document
  refers to Policies as Gateway API Policies.

# Motivation
[motivation]: #motivation

Currently, Kuadrant's support for the Policy Attachment Model can be divided in
two categories:
* Policies targeting the Multicluster Gateway, defined in the hub cluster and
  reconciled by the Multicluster Gateway Controller
* Policies targeting the downstream Gateway, defined in the spoke clusters and
  reconciled by the downstream Gateway controllers.

In a realistic multicluster scenario where multiple spoke clusters are present, the management of these policies can become tedious and error-prone, as policies have
to be defined in the hub cluster, as well as replicated in the multiple spoke clusters.

As Kuadrant users:
* Gateway-admin has a set of homogeneous clusters and needs to apply per cluster rate limits across the entire set.
* Platform-admin with a set of clusters with rate limits applied needs to change rate limit for one particular cluster. 

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The policy sync feature will allow a gateway-admin to configure, via GatewayClass
parameters, a set of Policy GVRs to be synced by the Multicluster Gateway Controller.

The `policiesToSync` field in the parameters defines those GVRs. For example, in
order to configure the controller to sync AuthPolicies:

```json
"policiesToSync": [
  {
    "group": "kuadrant.io",
    "version": "v1beta1",
    "resource": "authpolicies" 
  }
]
```

The support for resources that the controller can sync is limited by the following:
* The controller ServiceAccount must have permission to watch, list, and get the
  resource to be synced
* The resource must implement the Policy schema:
    * Have a `.spec.targetRef` field

When a Policy is configured to be synced in a GatewayClass, the Multicluster
Gateway Controller starts watching events on the resources, and propagates changes
by placing the policy in the spoke clusters, with the following mutations:
* The `TargetRef` of the policy is changed to reference the downstream Gateway
* The `kuadrant.io/policy-synced` annotation is set

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Process overview

#### Dynamic Policy watches

The Multicluster Gateway Controller reconciles parameters referenced by the
GatewayClass of a Gateway. A new field is added to the parameters that allows
the configuration of a set of GVRs of Policies to be synced.

The GatewayClass reconciler validates that:
* The GVRs reference existing resource definitions
* The GVRs reference resources that implement the Policy schema.

Validation failures are reported as part of the status of the GatewayClass

The Gateway reconciler sets up dynamic watches to react to events on the configured
Policies, calling the PolicySyncer component with the updated Policy as well
as the associated Gateway.

#### PolicySyncer component

The PolicySyncer component is in charge of reconciling Policy watch events to
apply the necessary changes and place the Policies in the spoke clusters.

This component is injected in the event source and called when a change is made
to a hub Policy that has been configured to be synced.

The PolicySyncer implementation uses OCM ManifestWorks to place the policies in
the spoke clusters. Through the ManifestWorks, OCM allows to:
* Place the Policy in each spoke cluster
* Report the desired status back to the hub using JSON feedback rules

### Policy Hierarchy 

In order to avoid conflict with Policies created directly in the spoke clusters,
a hierarchy must be defined to prioritise those Policies.

The controller will set the `kuadrant.io/policy-synced` annotation on the policy
when placing it in the spoke cluster.

The Kuadrant operator will be aware of the presence of this annotation, and, in case
of conflicts, override Policies that contain this annotation.

# Drawbacks
[drawbacks]: #drawbacks

## Third party Policy support

In order for a Policy to be supported for syncing, the MGC must have permissions
to watch/list/get the resource, and the implementation of the downstream Gateway
controller must be aware of the `policy-synced` annotation.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Consequences of not implementing

Gateway-admins will have no centralized system for handling spoke-level policies targeting a gateway created there from the hub.

#### OCMs Policy Framework will not be used to complete this objective:

OCMs Policy Framework is a system designed to make assertions about the state of a spoke, and potentially take actions based on that state, as such it is not a suitable replacement for manifestworks in the case of syncing resources to a spoke.

### Potential migration from ManifestWorks to ManifestWorkReplicaSets

ManifestWorkPeplicaSets may be a future improvement that the MGC could support
to simplify the placement of related resources, but beyond the scope of this RFC.

# Prior art
[prior-art]: #prior-art

No applicable prior art.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

## Status reporting

While the controller can assume common status fields among the Policies that it
syncs, there might be a scenario where certain policies use custom status fields
that are not handled by the controller. In order to support this, two alternatives
are identified:

1. Configurable rules.

    An extra field is added in the GatewayClass params that configures the policies
    to sync, to specify custom fields that the controller must propagate back from
    the spokes to the hub.

2. Hard-coded support.

    The PolicySync component can identify the Policy type and select which extra
    status fields are propagated 

# Future possibilities
[future-possibilities]: #future-possibilities

If OCMs Policy Framework is updated to enable syncing of resources status back to the hub, it could be an opportunity to refactor the MGC to use this framework in place of the current approach of creating ManifestWorks directly.

This system could mutate over time to dynamically sync more CRDs than policies to spoke clusters.