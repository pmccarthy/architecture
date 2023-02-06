# RFC Template

- Feature Name: `policy_status_states`
- Start Date: 2023-02-03
- RFC PR: [Kuadrant/architecture#0009](https://github.com/Kuadrant/architecture/pull/9)
- Issue tracking: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/issues/0000)

# Summary
[summary]: #summary

This RFC proposes a new design for the `RateLimitPolicy` Status definition and transitions.

# Motivation
[motivation]: #motivation

At the time being, the `RateLimitPolicy` Status doesn't clearly and truthfully communicate the actual state of
reconciliation and healthiness with its Operator managed services, mainly in this case the Rate Limit service `Limitador`.

As a consequence, only misleading information is shared causing unexpected errors and flawed assumptions.

The following are some issues addressing the before mentioned drawbacks:

* https://github.com/Kuadrant/kuadrant-operator/issues/87
* https://github.com/Kuadrant/kuadrant-operator/issues/96
* https://github.com/Kuadrant/kuadrant-operator/issues/140

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The following proposal it's divided in 3 stages, where each one could be applied/develop in order and would reflect valuable
and accurate information with different degrees of acuity. The Policy CRD Status in the following diagrams, are simplified
as states, which in the [Reference-level explanation](#reference-level-explanation) will be translated to the actual Status Conditions.

## Stage 1
**The state is defined by the application and validation of the Policy CR**

This first stage is a simple version where the operator only relies on itself, not checking the healthiness of its services
and just validating the Spec.

![](0000-rlp-status-assets/rlp_status_1.png)

States rationale:

* `Created`: The initial state. It announces that the policy has successfully being created, the operator acknowledges it.
* `Applied`: This state is reached after the `Validation` event has being successfully passed.
* `Failed`: This one will be set when the `Validation` process encounters an error. This could be either condition's failed/error
state or a top-level condition.
* `Updated`: From `Failed` or `Applied`, it could be triggered a `Spec Change` event that will move it to this state.

## Stage 2
**A further reconciliation check provides a new state**

This following one, besides checking what the former stage does, it also adds the states reflecting the reconciliation
process. In the case of the RLP, it will create/update the `ConfigMap` holding the `Limitador` config file.


   ![](0000-rlp-status-assets/rlp_status_2.png)

States rationale:

* `Applied`: The __Applied__ state will not be final, and will be preceding a `Reconciliation` event.
* `Reconciled`: It communicates that the policy has successfully being reconciled, and the `ConfigMap` has been updated.
* `Failed`: This one will be reached when either of `Validation` and `Reconcilation` processes have encounter any errors.

## Stage 3
**A final health check of services refines the status**

The final stage will bring a greater degree of accuracy, thanks for a final process that will check the healthiness and 
configuration version the `Limitador` service currently enforces.

   ![](0000-rlp-status-assets/rlp_status_3.png)

States rationale:

* `Recondiled`: This state will precede the "Health check" process  graphed as `Service Probe` event.
* `Protected`: After a successfull response of the `Service Probe`, this states communicates the policy is finally enforced.
This is the final top-level condition.
* `Failed`: Now this state could also be set after encountering errors in the `Service Probe` check.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The stages before mentioned, will follow the [Kubernetes guidelines](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#typical-status-properties) 
regarding the Status object definition.


**Conditions**

| Type       | Status | Reason                | Message                                            | Top Level |
|------------|--------|-----------------------|----------------------------------------------------|-----------|
| Created    | True   | "PolicyCreated"       | "RateLimitPolicy created"                          | No        |
| Updated    | True   | "PolicyUpdated"       | "RateLimitPolicy has been updated"                 | No        |
| Applied    | True   | "PolicyApplied"       | "RateLimitPolicy has been successfully applied     | Yes       |
|            | False  | "PolicyNotApplied"    | "RateLimitPolicy is invalid"                       |           |
| Reconciled | True   | "PolicyReconciled"    | "RateLimitPolicy has been successfully reconciled" | Yes       |
|            | False  | "PolicyNotReconciled" | "RateLimitPolicy failed reconciliation"            |           |
| Protected  | True   | "PolicyEnforced"      | "RateLimitPolicy has been successfully enforced"   | Yes       |
|            | False  | "PolicyNotEnforced"   | "RateLimitPolicy has encountered an error"         |           |


A simplified version and more aligned with the Kubernetes objects implementation could be represented as the following.

**Conditions**

All conditions are top-level.

| Type        | Status | Reason                      | Message                                                    |
|-------------|--------|-----------------------------|------------------------------------------------------------|
| Progressing | True   | "PolicyCreated"             | "RateLimitPolicy created"                                  |
|             | True   | "PolicyUpdated"             | "RateLimitPolicy has been updated"                         |
|             | True   | "PolicyApplied"             | "RateLimitPolicy has been successfully applied             |
|             | True   | "PolicyReconciled"          | "RateLimitPolicy has been successfully reconciled"         |
|             | False  | "PolicyEnforced"            | "RateLimitPolicy has been successfully enforced"           |
|             | False  | "PolicyError"               | "RateLimitPolicy has encountered an error"                 |
| Protected   | True   | "PolicyEnforced"            | "RateLimitPolicy has been successfully enforced"           |
|             | False  | "PolicyNotEnforced"         | "RateLimitPolicy has encountered an error while enforcing" |
| Failed      | True   | "PolicyValidationError"     | "RateLimitPolicy has failed to validate"                   |
|             | True   | "PolicyReconciliationError" | "RateLimitPolicy has encountered a reconciliation error"   |
|             | True   | "PolicyServiceError"        | "RateLimitPolicy has encountered has failed to enforce"    |
|             | False  | "PolicyEnforced"            | "RateLimitPolicy has been successfully enforced"           |


### Notes
* The messages, the ones corresponding to the _falsey status_, might reflect the error that encountered.
* It's possible to have the _Failed_ state as a top level condition too. In this case might be useful to add a third 
"Unknown" status.

# Drawbacks
[drawbacks]: #drawbacks

- This proposal will require to change the code controllers assert the status
- Since the Status is part of the "API", won't be backwards compatible
- Documentation updating

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

-

# Prior art
[prior-art]: #prior-art

* [Kubernetes API Conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#spec-and-status)
* [Current RateLimitPolicy Status work](https://github.com/Kuadrant/kuadrant-operator/blob/main/controllers/ratelimitpolicy_status.go)


# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Should this proposal extend to the KAP as well?
- Is it worthy to implement a state machine or state machine design pattern to achieve this set of conditions?

# Future possibilities
[future-possibilities]: #future-possibilities

A future work could include the extraction of the Status into a shared module, so both policies could benefit from it.
