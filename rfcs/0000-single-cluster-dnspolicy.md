# RFC Template

- Feature Name: `single-cluster-dnspolicy`
- Start Date: 2023-10-09
- RFC PR: [Kuadrant/architecture#30](https://github.com/Kuadrant/architecture/pull/30)
- Issue tracking: [Kuadrant/architecture#31](https://github.com/Kuadrant/architecture/issues/31)

# Summary
[summary]: #summary

Proposal for changes to the `DNSPolicy` API to allow it to work more effectively in a single cluster context.

# Motivation
[motivation]: #motivation

The [`DNSPolicy`](https://github.com/Kuadrant/multicluster-gateway-controller/blob/main/pkg/apis/v1alpha1/dnspolicy_types.go) API (v1alpha1), was implemented as part of our multi cluster gateway offering using OCM and as such the design and implementation were influenced heavily by how we want multi cluster dns to work.

* Decouple the API entirely from OCM and multi cluster specific concepts.
* Simplify the DNS record structure created for a gateway listeners host.
* Improve the likelihood of adoption by creating an integration path for other kubernetes dns controllers such as [external-dns](https://github.com/kubernetes-sigs/external-dns).

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The DNSPolicy can be used to target a Gateway in a single cluster context and will create dns records for each listener host in an appropriately configured external dns provider.
In this context the advanced `loadbalancing` configuration is unnecessary, and the resulting DNSRecord can be created mapping individual listener hosts to a single DNS A or CNAME record by using the `simple` strategy in the DNSPolicy.

**Example 1.** DNSPolicy using `simple` strategy

```yaml
apiVersion: kuadrant.io/v1alpha1
kind: DNSPolicy
metadata:
  name: prod-web
  namespace: my-gateways
spec:
  providerRef:
    name: my-route53-credentials
    namespace: my-gateways
    kind: Secret
  targetRef:
    name: prod-web
    group: gateway.networking.k8s.io
    kind: Gateway
  strategy: simple
```

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: prod-web
  namespace: my-gateways
spec:
  gatewayClassName: istio
  listeners:
    - allowedRoutes:
        namespaces:
          from: All
      name: api
      hostname: "myapp.mn.hcpapps.net"
      port: 80
      protocol: HTTP
status:
  addresses:
    - type: IPAddress
      value: 172.31.200.0
```

In the example the `api` listener has a hostname `myapp.mn.hcpapps.net` that matches a hosted zone being managed by the provider referenced `my-route53-credentials` in the DNSPolicy. 
As the `simple` strategy is set in the DNSPolicy a DNSRecord resource with the following contents will be created:

```yaml
apiVersion: kuadrant.io/v1alpha1
kind: DNSRecord
metadata:
  name: prod-web-api
  namespace: my-gateways
spec:
  providerRef:
    name: my-route53-credentials
    namespace: my-gateways
  endpoints:
    - dnsName: myapp.mn.hcpapps.net
      recordTTL: 60
      recordType: A
      targets:
        - 172.31.200.0
```

The `providerRef` is included in the DNSRecord to allow the MGC dns record controller to load the appropriate provider during reconciliation and create the DNS records in the dns provider service e.g. route 53.

**Example 2.** DNSPolicy using `simple` strategy with external dns provider

```yaml
apiVersion: kuadrant.io/v1alpha1
kind: DNSPolicy
metadata:
  name: prod-web
  namespace: my-gateways
spec:
  providerRef:
    name: external-dns
    kind: ExternalDNS
  targetRef:
    name: prod-web
    group: gateway.networking.k8s.io
    kind: Gateway
  strategy: simple
```

In ths example if the DNSPolicy was attached to the same gateway described in example 1, the same DNSRecord would also be created but MGC would not reconcile it. In this scenario it is expected that an external controller is being used to manage the reconciliation of the DNSRecord resources such as [external-dns](https://github.com/kubernetes-sigs/external-dns).

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## API Updates

DNSPolicy:

* - new providerRef field `spec.providerRef` 
* - new strategy field `spec.strategy` 

DNSRecord:

- `spec.managedZone` replaced with `spec.providerRef`

### DNSPolicy.spec.providerRef

The `providerRef` field is mandatory and contains a reference to a resource that shows how DNSRecords will be reconciled.
    * - `spec.providerRef.name` - name of the provider resource
    * - `spec.providerRef.namespace` - namespace of provider resource
    * - `spec.providerRef.kind` - kind of resource, can be anything but only `Secret` will be reconciled by MGC.

A kind of type `Secret` will be managed by MGC, and a secret in the given namespace with the given name must exist. The expected contents of the secrets data is comparable to the `dnsProviderSecretRef` used by ManageZones. 

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mgc-aws-credentials
  namespace: multi-cluster-gateways
type: kuadrant.io/aws
data:
  AWS_ACCESS_KEY_ID: "foo"
  AWS_SECRET_ACCESS_KEY: "bar"
  CONFIG:
    zoneIDFilter:
      - Z04114632NOABXYWH93QUl
```

The `CONFIG` section of the secrets data will be added to allow provider specific configuration to be stored alongside the providers credentials and can be used during the instantiation of the provider client, and during any provider operations.
The above for example would use the `zoneIDFilter` value to limit what hosted zones this provider is allowed to update.

A kind of any other type e.g. `ExternalDNS` informs MGC that it should not reconcile any resources referencing it. All fields of `providerRef` will be ignored by MGC and can be set to anything, however since the `providerRef` will still be copied over to any DNSRecord resources created by the policy controller the values may still be given meaning to that external DNSRecord reconciler.

### DNSPolicy.spec.strategy[simple|loadalanced]

The `strategy` field is mandatory and dictates what kind of dns record structure the policy will create. Two strategy options are allowed `simple` or `loadbalanced`.

A reconciliation of DNSPolicy processes the target gateway and creates a DNSRecord per listener that is supported by the currently configured provider(hostname matches the hosted zones accessible with the credentials and config). The strategy used will determine the contents of the DNSRecord resources Endpoints array.

#### Simple

```yaml
apiVersion: kuadrant.io/v1alpha1
kind: DNSRecord
spec:
  providerRef:
    name: my-route53-credentials
    namespace: my-gateways
    kind: Secret
  endpoints:
    - dnsName: myapp.mn.hcpapps.net
      recordTTL: 60
      recordType: A
      targets:
        - 172.31.200.0
```

#### LoadBalanced

```yaml
apiVersion: kuadrant.io/v1alpha1
kind: DNSRecord
spec:
  providerRef:
    name: my-route53-credentials
    namespace: my-gateways
    kind: Secret
  endpoints:
    - dnsName: myapp.mn.hcpapps.net
      recordTTL: 300
      recordType: CNAME
      targets:
        - lb-4ej5le.myapp.mn.hcpapps.net
    - dnsName: lb-4ej5le.myapp.mn.hcpapps.net
      providerSpecific:
        - name: geo-code
          value: '*'
      recordTTL: 300
      recordType: CNAME
      setIdentifier: default
      targets:
        - default.lb-4ej5le.myapp.mn.hcpapps.net
    - dnsName: default.lb-4ej5le.myapp.mn.hcpapps.net
      providerSpecific:
        - name: weight
          value: "120"
      recordTTL: 60
      recordType: CNAME
      setIdentifier: lrnse3.lb-4ej5le.myapp.mn.hcpapps.net
      targets:
        - lrnse3.lb-4ej5le.myapp.mn.hcpapps.net
    - dnsName: lrnse3.lb-4ej5le.myapp.mn.hcpapps.net
      recordTTL: 60
      recordType: A
      targets:
        - 172.31.200.0
```

Note: `LoadBalanced` is the current default for DNSPolicy and more details about it can be found in the original [DNSPolicy rfc](https://github.com/Kuadrant/architecture/blob/main/rfcs/0003-dns-policy.md) 

### DNSRecord.spec.providerRef

The DNSRecord API is updated to remove the `managedZone` reference in favour of directly referencing the `providerRef` credentials instead. The DNSRecord reconciliation will be unchanged except for loading the provider client from `providerRef` credentials.

The DNSPolicy reconciliation will be updated to remove the requirement for a ManagedZone resource to be created before a DNSPolicy can create dns records for it, instead it will be replaced in favour of just listing available zones directly in the currently configured dns provider. If no matching zone is found, no DNSRecord will be created.

There is a potential for a DNSRecord to be created successfully, but then a provider updated to remove access. In this case it is the responsibility of the DNSPolicy controller to remove DNSRecords it created that can no longed be reconciled into the configured provider. 


# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

```
- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
```

ToDo

# Prior art
[prior-art]: #prior-art

```
Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- Does another project have a similar feature?
- What can be learned from it? What's good? What's less optimal?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other tentatives - successful or not, provide readers of your RFC with a fuller picture.

Note that while precedent set by other projects is some motivation, it does not on its own motivate an RFC.
```

ToDo
[ExternalDNS](https://github.com/kubernetes-sigs/external-dns)


# Unresolved questions
[unresolved-questions]: #unresolved-questions

```
- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
```

ToDo

# Future possibilities
[future-possibilities]: #future-possibilities

```
Think about what the natural extension and evolution of your proposal would be and how it would affect the platform and project as a whole. Try to use this section as a tool to further consider all possible interactions with the project and its components in your proposal. Also consider how this all fits into the roadmap for the project and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the RFC you are writing but otherwise related.

Note that having something written down in the future-possibilities section is not a reason to accept the current or a future RFC; such notes should be in the section on motivation or rationale in this or subsequent RFCs. The section merely provides additional information. 
```

ToDo