# Defaults & Overrides

- Feature Name: `defaults-and-overrides`
- Start Date: 2024-02-15
- RFC PR: [Kuadrant/architecture#58](https://github.com/Kuadrant/architecture/pull/58)
- Issue tracking: [Kuadrant/kuadrant-operator#431](https://github.com/Kuadrant/kuadrant-operator/issues/431)

# Summary
[summary]: #summary

This is a proposal for extending the Kuadrant Policy APIs to fully support use cases of **Defaults & Overrides (D/O)** for [Inherited Policies](https://gateway-api.sigs.k8s.io/geps/gep-713/#inherited-policy-attachment-its-all-about-the-defaults-and-overrides), including the base use cases of _full default_ and _full override_, and more specific nuances that involve merging individual policy rules (as defaults or overrides), declaring constraints and deactivating defaults.

# Motivation
[motivation]: #motivation

As of Kuadrant Operator [v0.6.0](https://github.com/Kuadrant/kuadrant-operator/releases/tag/v0.6.0), Kuadrant policy resources that have hierarchical effect across the tree of network objects (Gateway, HTTPRoute), or what is known as [Inherited Policies](https://gateway-api.sigs.k8s.io/geps/gep-713/#inherited-policy-attachment-its-all-about-the-defaults-and-overrides), provide only _limited support for setting defaults_ and _no support for overrides_ at all.

The above is notably the case of the AuthPolicy and the RateLimitPolicy v1beta2 APIs, shipped with the aforementioned version of Kuadrant. These kinds of policies can be attached to Gateways or to HTTPRoutes, with cascading effects through the hierarchy that result in one _effective policy_ per gateway-route combination. This effective policy is either the policy attached to the Gateway or, if present, the one attached to the HTTRoute, thus conforming with a strict case of _implicit defaults set at the level of the gateway_.

Enhancing the Kuadrant Inherited Policy CRDs, so the corresponding policy instances can declare `defaults` and `overrides` stanzas, is imperative:
1. to provide full support for D/O in the terms proposed at [GEP-713](https://gateway-api.sigs.k8s.io/geps/gep-713) of the Kubernetes Gateway API special group (base use cases);
2. to extend D/O support to other derivative cases, learnt to be just as important for platform engineers and app developers who require more granular policy interaction on top of the base cases;
3. to support more sophisticated hierarchies with other kinds of network objects and/or multiples policies targetting at the same level of the hierarchy (possibly, in the future.)

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Conceptialization and User story

The base use cases for Defaults & Overrides (D/O) are:
- **Defaults (D):** policies declared lower in the hierarchy supersede ones set (as "defaults") at a higher level, or _"more specific beats less specific"_
- **Overrides (O):** policies declared higher in the hierarchy (as "overrides") supersede ones set at the lower levels, or _"less specific beats more specific"_

The base cases are expanded with the following additional derivative cases and concepts:
- **Merged defaults (DR):** "higher" default policy rules that are merged into more specific "lower" policies (as opposed to an atomic less specific set of rules that is activated only when another more specific one is absent)
- **Merged overrides (OR):** "higher" override policy rules that are merged into more specific "lower" policies (as opposed to an atomic less specific set of rules that is activated fully replacing another more specific one that is present)
- **Constraints (C):** specialization of an override that, rather than declaring concrete values, specify higher level constraints for lower level values, with the semantics of a "clipping" effect over the lower level values so the latter are enforced within the boundaries/limits dictated by the constraints, in an override fashion; typically employed for constraining numeric values and regular patterns (e.g. limited sets) that are declared at the lower level (e.g., min value, max value, `in` operator)
- **Deactivation (RR):** specialization that completes a merge default use case by allowing lower level policies to disable ("deactivate") individual defaults set a higher level (as opposed to superseding those defaults with actual, more specific, policy rules with proper meaning ther than nullify the default)

Together, these concepts relate to solve the following user stories:

| User story                                                                                                                                                                                                                                                                                          | Group | Unique ID                        |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:-----:|----------------------------------|
| As a Platform Engineer, when configuring a Gateway, I want to set a default policy for all routes linked to my Gateway, that can be fully replaced with more specific ones(*).                                                                                                                      | D     | **gateway-default-policy**       |
| As a Platform Engineer, when configuring a Gateway, I want to set default policy rules (parts of a policy) for all routes linked to my Gateway, that can be individually replaced and/or expanded by more specific rules(*).                                                                        | DR    | **gateway-default-policy-rule**  |
| As a Platform Engineer, when defining a policy that configures a Gateway, I want to set constraints (e.g. minimum/maximum value, enumerated options, etc) for more specific policy rules that are declared(*) with the purpose of replacing the defaults I set for the routes linked to my Gateway. | C     | **policy-constraints**           |
| As a Platform Engineer, when configuring a Gateway, I want to set a policy for all routes linked to my Gateway, that cannot be replaced nor expanded by more specific ones(*).                                                                                                                      | O     | **gateway-override-policy**      |
| As a Platform Engineer, when configuring a Gateway, I want to set policy rules (parts of a policy) for all routes linked to my Gateway, that cannot be individually replaced by more specific ones(*), but only expanded with additional more specific rules(\*).                                   | OR    | **gateway-override-policy-rule** |
| As an Application Developer, when managing an application, I want to set a policy for my application, that fully replaces any default policy that may exist for the application at the level of the Gateway, without having to know about the existence of the default policy.                      | D     | **route-replace-policy**         |
| As an Application Developer, when managing an application, I want to expand a default set of policy rules set for my application at the level of the gateway, without having to refer to those existing rules by name.                                                                              | D/O   | **route-add-policy-rule**        |
| As an Application Developer, when managing an application, I want to deactivate an individual default rule set for my application at the level of the gateway.                                                                                                                                      | RR    | **route-disable-policy-rule**    |

<sup>(*) declared in the past or in the future, by myself or any other authorized user.</sup>

The interactive nature of setting policies at levels in the hierarchy and by different personas, make that the following additional user stories arise. These are stories here grouped under the **Observability (Ob)** aspect of D/O, but referred to as well in relation to the ["Discoverability Problem"](https://gateway-api.sigs.k8s.io/geps/gep-713/#status-and-the-discoverability-problem) described by Gateway API.

| User story                                                                                                                                                                                                     | Group | Unique ID                   |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:-----:|-----------------------------|
| As one who has read access to Kuadrant policies, I want to view the effective policy enforced at the traffic routed to an application, considering all active defaults and overrides at different policies(*). | Ob    | **view-effective-policy**   |
| As a Platform Engineer, I want to view all default policy rules that may have been replaced by more specific ones(*).                                                                                          | Ob    | **view-policy-rule-status** |
| As a Policy Manager, I want to view all gateways and/or routes whose traffic is subject to enforcement of a particular policy rule referred by name.                                                           | Ob    | **view-policy-rule-reach**  |

<sup>(*) declared in the past or in the future, by myself or any other authorized user.</sup>

## Writing D/O-enabled Kuadrant Policies

Writing a Kuadrant policy enabled for Defaults & Overrides (D/O), to be attached to a network object, involves declaring the following fields at the first level of the spec:

- `targetRef` (required): the reference to a hierarchical network object targeted by the policy, typed as a Gateway API [`PolicyTargetReference`](https://pkg.go.dev/sigs.k8s.io/gateway-api/apis/v1alpha2#PolicyTargetReference) or [`PolicyTargetReferenceWithSectionName`](https://pkg.go.dev/sigs.k8s.io/gateway-api/apis/v1alpha2#PolicyTargetReferenceWithSectionName) type
- `defaults`: a block of _default_ policy rules with further specification of a strategy (_atomic_ set of rules or individual rules to be _merged_ into lower policies), and optional conditions for applying the defaults down through the hierarchy
- `overrides`: a block of _override_ policy rules with further specification of a strategy (_atomic_ set of rules or individual rules to be _merged_ into lower policies), and optional conditions for applying the overrides down through the hierarchy
- the bare policy rules block without further qualification as a default or override set of rules – e.g. the `rules` field in a Kuadrant AuthPolicy, the `limits` field in a RateLimitPolicy.

Typically, one will specify either `defaults` and/or `overrides`, or just the bare set of policy rules block without further qualification as neither defaults nor overrides.

In case two or more of these fields are specified, they are processed in the following order:
1. `defaults`
2. bare set of policy rules without qualification (treated indistinctively as another block of defaults)
3. `overrides`

Supporting specifying the bare set of policy rules at the first level of the spec, as well and simultaneously along with `defaults` and `overrides` blocks, is a strategy that aims to provide:
1. more natural usability, especially for those who write policies attached to the lowest level of the hierarchy supported; as well as
2. backward compatibility for policies that did not support explicit D/O and later on have moved to doing so.

### Direct Policies

A policy that does not specify D/O fields (`defaults`, `overrides`) is a policy that _declares an intent_.

When a policy of such kind targets an object whose kind is the lowest kind accepted by Kuadrant in the hierarchy of network resources, such policy is treated as a [_Direct Policy_](https://gateway-api.sigs.k8s.io/geps/gep-713/#direct-policy-attachment). When, otherwise, it is used to target "higher" objects, the policy is treated as an Inherited Policy that implicitly declares atomic _defaults_.

### Inherited Policies

A policy that specifies D/O fields (`defaults`, `overrides`) is a policy explicitly declared to _modify an intent_ (or a _proper Inherited Policy_.)

There is no meaning in declaring a proper Inherited Policy that targets an object whose kind is the lowest kind accepted by Kuadrant in the hierarchy of network resources. The sets of rules specified in these policies affect indistinctively the targeted objects, whether the rules are qualified as `defaults` or as `overrides`.

### Example of D/O-enabled Kuadrant policy

**Example 1.** Atomic defaults

```yaml
kind: AuthPolicy
metadata:
  name: gw-policy
spec:
  targetRef:
    kind: Gateway
  defaults:
    rules:
      authentication:
        "a": {…}
      authorization:
        "b": {…}
    strategy: atomic
```

The above is a proper Inherited Policy that sets a **_default_ atomic set of auth rules** that will be set at lower objects in case those lower object do not have policies attached of their own at all.

The following is a sligthly different example that defines **auth rules** that will be **individually merged** into lower objects, evaluated one by one if already defined at the "lower" (more specific) level and therefore should take precedence, or if otherwise is missing at the lower level and therefore the default should be activated.

**Example 2.** Merged defaults

```yaml
kind: AuthPolicy
metadata:
  name: gw-policy
spec:
  targetRef:
    kind: Gateway
  defaults:
    rules:
      authentication:
        "a": {…}
      authorization:
        "b": {…}
    strategy: merge
```

Similarly, a set of `overrides` policy rules could be specified, instead or alongside with the `defaults` set of policy rules.

### Atomic vs. individually merged policy rules

There are 2 supported strategies for applying proper Inherited Policies down to the lower levels of the herarchy:
- **Atomic policy rules:** the bare set of policy rules in a `defaults` or `overrides` block is applied as an atomic piece; i.e., a lower object than the target of the policy, that is evaluated to be potentially affected by the policy, also has an atomic set of rules if another policy is attached to this object, therefore either the _entire_ set of rules declared by the higher (less specific) policy is taken or the _entire_ set rules declared by the lower (more specific) policy is taken (depending if it's `defaults` or `overrides`), but the two sets are never merged into one.
- **Merged policy rules:** each _individual_ policy rule within a `defaults` or `overrides` block is compared one to one against lower level policy rules and, when they clash, either one or the other (more specific or less specific) is taken (depending if it's `defaults` or `overrides`), in a way that the final effective policy is a merge between the two policies.

Each block of `defaults` and `overrides` must specify a `strategy` field whose value is set to either `atomic` or `merge`. If omitted, `atomic` is assumed.

#### Level of granularity of compared policy rules

Atomic versus merge strategies, as a specification of the `defaults` and `overrides` blocks, imply that there are only two levels of granularity for comparing policies _vis-a-vis_.

`atomic` means that the level of granularity is the entire set of policy rules within the `defaults` or `overrides` block. I.e., the policy is atomic.

For the `merge` strategy, on the other hand, the granularity is of each _named policy rule_, where the name of the policy rule is the key and the value is an atomic object that specifies that policy rule.

#### Matrix of D/O strategies and Effective Policy

When two policies are compared to compute a so-called _Effective Policy_ out of their sets of policy rules and given default or override semantics, plus specified `atomic` or `merge` strategies, the following matrix applies:

|                | Atomic (entire sets of rules)                                                                                                | Merge (individual policy rules at a given granularity)                                                                                                                      |
|----------------|------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Defaults**   | More specific _entire_ set of rules beats less specific _entire_ set of rules → takes all the rules from the _lower_ policy  | More specific _individual_ policy rules beat less specific _individual_ set of rules → compare one by one each pair of policy rules and take the _lower_ one if they clash  |
| **Overrides**  | Less specific _entire_ set of rules beats more specific _entire_ set of rules → takes all the rules from the _higher_ policy | Less specific _individual_ policy rules beat more specific _individual_ set of rules → compare one by one each pair of policy rules and take the _higher_ one if they clash |

The order of the policies, from less specific (or "higher") to more specific (or "lower), is determined according to the [Gateway API hierarchy of network resources](https://gateway-api.sigs.k8s.io/geps/gep-713/#hierarchy), based on the kind of the object targeted by the policy.

For a more detailed reference, including how to resolve conflicts in case of policies targeting objects at the same level, see GEP-713's section [Hierarchy](https://gateway-api.sigs.k8s.io/geps/gep-713/#hierarchy) and [Conflict Resolution](https://gateway-api.sigs.k8s.io/geps/gep-713/#conflict-resolution).

## Examples of D/O cases

The following sets of examples generalize D/O applications for the presented [user stories](#conceptialization-and-user-story), regardless of details about specific personas and kinds of targeted resources. They illustrate the expected behavior for different cases involving defaults, overrides, constraints and deactivations.

| Examples                                                                                                                                                | Highlighted user stories                           |
|---------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------|
| [A. Default policy entirely replaced by another at lower level](#examples-a---default-policy-entirely-replaced-by-another-at-lower-level)               | gateway-default-policy, route-replace-policy       |
| [B. Default policy rules merged into policies at lower level](#examples-b---default-policy-rules-merged-into-policies-at-lower-level)                   | gateway-default-policy-rule, route-add-policy-rule |
| [C. Override policy entirely replacing other at lower level](#examples-c---override-policy-entirely-replacing-other-at-lower-level)                     | gateway-override-policy                            |
| [D. Override policy rules merged into other at lower level](#examples-d---override-policy-rules-merged-into-other-at-lower-level)                       | gateway-override-policy-rule                       |
| [E. Override policy rules setting constraints to other at lower level](#examples-e---override-policy-rules-setting-constraints-to-other-at-lower-level) | policy-constraints                                 |
| [F. Policy rule that deactivates default from higher level](#examples-f---policy-rule-that-deactivates-default-from-higher-level)                       | route-disable-policy-rule                          |

In all the cases, a Gateway and a HTTPRoute objects are targeted by two policies, and an effective policy is presented highlighting the expected outcome. This poses no harm to generalizations involving same or different kinds of targeted resources, multiples policies targeting a same object, etc.

The leftmost YAML is always the "higher" (less specific) policy; the one in the middle, separated from the leftmost one by a "+" sign, is the "lower" (more specific) policy; and the rightmost YAML is the expected _Effective Policy_.

For a complete reference of the order of hierarchy, from least specific to most specific kinds of resources, as well as how to resolve conflicts of hierarchy in case of policies targeting objects at the same level, see Gateway API's [Hierarchy](https://gateway-api.sigs.k8s.io/geps/gep-713/#hierarchy) definition for Policy Attachment and [Conflict Resolution](https://gateway-api.sigs.k8s.io/geps/gep-713/#conflict-resolution).

### Examples A - Default policy entirely replaced by another at lower level

**Example A1.** A _default_ policy that is _replaced entirely_ if another one is set at a lower level

![A default policy that is replaced entirely if another one is set at a lower level](0009-defaults-and-overrides-assets/example-a1.png)

### Examples B - Default policy rules merged into policies at lower level

**Example B1.** A _default_ policy whose rules are _merged into other policies_ at a lower level, where individual default policy rules can be overridden or deactivated - _without conflict_

![A default policy whose rules are merged into other policies at a lower level, where individual default policy rules can be overridden or deactivated - without conflict](0009-defaults-and-overrides-assets/example-b1.png)

**Example B2.** A _default_ policy whose rules are _merged into other policies_ at a lower level, where individual default policy rules can be overridden or deactivated - _with conflict_

![A default policy whose rules are merged into other policies at a lower level, where individual default policy rules can be overridden or deactivated - with conflict](0009-defaults-and-overrides-assets/example-b2.png)

### Examples C - Override policy entirely replacing other at lower level

**Example C1.** An _override_ policy that _replaces any other_ that is set at a lower level _entirely_

![An override policy that replaces any other that is set at a lower level entirely](0009-defaults-and-overrides-assets/example-c1.png)

### Examples D - Override policy rules merged into other at lower level

**Example D1.** An _override_ policy whose rules are _merged into other policies_ at a lower level, overriding individual policy rules with same identification - _without conflict_

![An override policy whose rules are merged into other policies at a lower level, overriding individual policy rules with same identification - without conflict](0009-defaults-and-overrides-assets/example-d1.png)

**Example D2.** An _override_ policy whose rules are _merged into other policies_ at a lower level, overriding individual policy rules with same identification - _with conflict_

![An override policy whose rules are merged into other policies at a lower level, overriding individual policy rules with same identification - with conflict](0009-defaults-and-overrides-assets/example-d2.png)

### Examples E - Override policy rules setting constraints to other at lower level

The examples in this section introduce the proposal for a new `when` field for the `defaults` and `overrides` blocks. This field dictates the conditions to be found in a lower policy that would make a higher policy or policy rule to apply, according to the corresponding `defaults` or `overrides` semantics and `atomic` or `merge` strategy.

Combined with a simple case of override policy (see [Examples C](#examples-c---override-policy-entirely-replacing-other-at-lower-level)), the `when` condition field allows modeling for use cases of setting constraints for lower-level policies.

As here proposed, the value of the `when` condition field must be valid [Common-Language Expression (CEL)](https://github.com/google/cel-spec) expression.

**Example E1.** An _override_ policy whose rules _set constraints_ to field values of other policies at a lower level, overriding individual policy values of rules with same identification if those values violate the constraints - _compliant_

![An override policy whose rules set constraints to field values of other policies at a lower level, overriding individual policy values of rules with same identification if those values violate the constraints - compliant](0009-defaults-and-overrides-assets/example-e1.png)

**Example E2.** An _override_ policy whose rules _set constraints_ to field values of other policies at a lower level, overriding individual policy values of rules with same identification if those values violate the constraints - _violated_

![An override policy whose rules set constraints to field values of other policies at a lower level, overriding individual policy values of rules with same identification if those values violate the constraints - violated](0009-defaults-and-overrides-assets/example-e2.png)

**Example E3.** An _override_ policy whose rules _set constraints_ to field values of other policies at a lower level, overriding individual policy values of rules with same identification if those values violate the constraints - _merge granularity problem_

The following example illustrates the possibly unintended consequences of enforcing D/O at [strict levels of granularity](#level-of-granularity-of-compared-policy-rules), and the flip side of the `strategy` field offering a closed set of options (`atomic` vs `merge`).

On one hand, the API is simple and straightforward, and there are no deeper side effects to be concerned about, other than at the two levels provided (entire set or atomic values for each individual policy rule.) On the other hand, this design may require more offline interaction between the actors who manage conflicting policies.

![An override policy whose rules set constraints to field values of other policies at a lower level, overriding individual policy values of rules with same identification if those values violate the constraints - merge granularity problem](0009-defaults-and-overrides-assets/example-e3.png)

### Examples F - Policy rule that deactivates default from higher level

The examples in this section introduce a new field `remove: []string` at the same level as the bare set of policy rules. The value of this field, provided as a list, dictates the default policy rules declared at a higher level to be deactivated ("removed") from the effective policy, specified by name of the policy rules.

**Example F1.** A policy that _deactivates_ a _default_ policy rule set at a higher level

![A policy that deactivates a default policy rule set at a higher level](0009-defaults-and-overrides-assets/example-f1.png)

**Example F2.** A policy that tries to _deactivate_ an _override_ policy rule set a higher level

![A policy that tries to deactivate an override policy rule set a higher level](0009-defaults-and-overrides-assets/example-f2.png)

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

WIP

# Drawbacks
[drawbacks]: #drawbacks

WIP

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The following alternatives were considered for the design of the API spec to support D/O:
- [**API DESIGN 1** - `strategy` field - _RECOMMENDED_](#api-design-1---strategy-field---recommended)
- [**API DESIGN 2** - `granularity` field](#api-design-2---granularity-field)
- [**API DESIGN 3** - `when` conditions (at any level of the spec)](#api-design-3---when-conditions-at-any-level-of-the-spec)
- [**API DESIGN 4** - “path-keys”](#api-design-4---path-keys)
- [**API DESIGN 5** - JSON patch-like](#api-design-5---json-patch-like)

All the examples in the RFC are based on API DESIGN 1.

## API DESIGN 1 - `strategy` field - _RECOMMENDED_

Each block of `defaults` and `overrides` specify a `strategy`: `atomic` or `merge`, with `atomic` assumed if the field is omitted.

All the examples in the RFC are based on this design for the API spec.

Some of the implications of the design are explained in the section [Atomic vs. individually merged policy rules](#atomic-vs-individually-merged-policy-rules), with highlights to the support for specifying the level of atomicity of the rules in the policy based on only 2 granularities – entire set of policy rules (`atomic`) or to the level of each named policy rule (`merge`.)

<table>
  <thead>
    <tr>
      <th>✅ Pros</th>
      <th>❌ Cons</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>
        <ul>
          <li>Same schema as a normal policy without D/O</li>
          <li>Safe against "unmergeable objects" (e.g. two rules declaring different one-of options)</li>
          <li>Strong types</li>
          <li>Extensible (by adding more fields, e.g.: to support deactivations)</li>
          <li>Easy to learn</li>
        </ul>
      </td>
      <td>
        <ul>
          <li>2 levels of granularity only – either all (‘atomic’) or policy rule (‘merge’)</li>
          <li>1 granularity declaration per D/O block → declaring both ‘atomic’ and ‘merge’ simultaneously requires 2 separate policies targeting the same object</li>
        </ul>
      </td>
    </tr>
  </tbody>
</table>

API DESIGN 1 is the RECOMMENDED design for the implementation of Kuadrant Policies enabled for D/O. This is due to the pros above, plus the fact that this design can evolve to other, more versatile forms, such as [DESIGN 2](#api-design-2---granularity-field) and [DESIGN 3](#api-design-3---when-conditions-at-any-level-of-the-spec), in the future, while the opposite would be harder to achieve.

## API DESIGN 2 - `granularity` field

Each block of `defaults` and `overrides` would specify a `granularity` field, set to a numeric integer value that describes which level of the policy spec, from the root of the set of policy rules until that number of levels down, to treat as the key, and the rest as the atomic value.

Example:

```yaml
kind: AuthPolicy
metadata:
  name: gw-policy
spec:
  targetRef:
    kind: Gateway
  defaults:
    rules:
      authentication:
        "a": {…}
      authorization:
        "b": {…}
    granularity: 0 # the entire spec ("rules") is an atomic value
  overrides:
    rules:
      metadata:
        "c": {…}
      response:
        "d": {…}
    granularity: 2 # each policy rule ("c", "d") is an atomic value
```

<table>
  <thead>
    <tr>
      <th>✅ Pros</th>
      <th>❌ Cons</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>
        <ul>
          <li>Same as OPTION 1</li>
          <li>Unlimited levels of granularity (values can be pointed as atomic at any level)</li>
        </ul>
      </td>
      <td>
        <ul>
          <li>1 granularity declaration per D/O block → <i>N</i> levels simultaneously require <i>N</i> policies</li>
          <li>Granularity specified as a number - user needs to count the levels</li>
          <li>Setting a deep level of granularity can cause merging "unmergeable objects"</li>
        </ul>
      </td>
    </tr>
  </tbody>
</table>

## API DESIGN 3 - `when` conditions (at any level of the spec)

Inspired by the extension of the API for D/O with an additional `when` field (see [Examples E](#examples-e---override-policy-rules-setting-constraints-to-other-at-lower-level)), this design alternative would use the presence of this field to signal the granularity of the atomic operation of default or override.

Example:

```yaml
kind: AuthPolicy
metadata:
  name: gw-policy
spec:
  targetRef:
    kind: Gateway
  defaults:
    rules:
      authentication:
        "a": {…}
        when: CEL # level 1 - entire "authentication" block
      authorization:
        "b":
          "prop-1": {…}
          when: CEL # level 2 - "b" authorization policy rule
```

<table>
  <thead>
    <tr>
      <th>✅ Pros</th>
      <th>❌ Cons</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>
        <ul>
          <li>Same as OPTION 2</li>
          <li>As many granularity declarations per D/O block as complex objects in the policy</li>
          <li>Granularity specified “in-place”</li>
        </ul>
      </td>
      <td>
        <ul>
          <li>Setting a deep level of granularity can cause merging "unmergeable objects"</li>
          <li>Implementation nightmare - hard to define the API from existing types</li>
        </ul>
      </td>
    </tr>
  </tbody>
</table>

## API DESIGN 4 - “path-keys”

A more radical alternative considered consisted of defining `defaults` and `overrides` blocks whose schemas would not match the ones of a normal policy without D/O. Instead, these blocks would consist of simple key-value pairs, where the keys specify the paths in an affected policy where to apply an atomic value.

Example:

```yaml
kind: AuthPolicy
metadata:
  name: gw-policy
spec:
  targetRef:
    kind: Gateway
  defaults:
    "rules.authentication":
      "a": {G}
    "rules.authorization.b": {G}
```

<table>
  <thead>
    <tr>
      <th>✅ Pros</th>
      <th>❌ Cons</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>
        <ul>
          <li>D/O as simple key-value sets (keys: where to apply, values: what to apply)</li>
          <li>Unlimited levels of granularity (values can be pointed as atomic at any level)</li>
          <li>Unlimited merge declarations per D/O block</li>
          <li>Intuitive, easy-to-learn</li>
        </ul>
      </td>
      <td>
        <ul>
          <li>Not same schema as the normal policy (without D/O) - not very GWAPI-like</li>
          <li>Poorly typed (i.e. <code>map[string]any)</code></li>
          <li>Not extensible (e.g., cannot add other fields to the API)</code></li>
        </ul>
      </td>
    </tr>
  </tbody>
</table>

## API DESIGN 5 - JSON patch-like

Similar to [DESIGN 4](#api-design-4---path-keys) and inspired by [JSON patch](https://jsonpatch.com/)-like operations.

Example:

```yaml
kind: AuthPolicy
metadata:
  name: gw-policy
spec:
  targetRef:
    kind: Gateway
  defaults:
  - path: rules.authentication
    operation: add
    value: { "a": {G} }
  - path: rules.authorization.b
    operation: remove
  - path: |
      rules.authentication.a.
      value
    operation: le
    value: 50
```

<table>
  <thead>
    <tr>
      <th>✅ Pros</th>
      <th>❌ Cons</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>
        <ul>
          <li>Same as OPTION 4</li>
          <li>Extensible, all kinds of operations supported (add, remove, constraint)</li>
        </ul>
      </td>
      <td>
        <ul>
          <li>Not same schema as the normal policy (without D/O) - not very GWAPI-like</li>
          <li>Poorly typed (i.e. <code>value: any)</code></li>
        </ul>
      </td>
    </tr>
  </tbody>
</table>

# Prior art
[prior-art]: #prior-art

WIP

# Out of scope

## Policy requirements

A use case often described in association with D/O is the one for declaring _policy requirements_. These are high level policies that declare requirements to be fulfilled by more specific (lower level) policies without specifying concrete default or override values nor constraints. E.g.: "an authentication policy must be enforced, but none is provided by default."

A typical generic policy requirement user story is:

> _As a Platform Engineer, when configuring a Gateway, I want to set policy requirements to be fulfilled by one who manages an application/route linked to my Gateway, so all interested parties, including myself, can be aware of applications deployed to the cluster that lack a particular policy protection being enforced._

Policy requirements as here described are out of scope of this RFC.

We believe policy requirement use cases can be stated and solved as an observability problem, by defining metrics and alerts that cover for missing policies or policy rules, without necessarily having to write a policy of the same kind to express such requirement tobe fulfilled.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

WIP

# Future possibilities
[future-possibilities]: #future-possibilities

WIP
