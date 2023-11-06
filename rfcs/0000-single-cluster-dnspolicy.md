# RFC Template

- Feature Name: `single-cluster-dnspolicy`
- Start Date: 2023-10-09
- RFC PR: [Kuadrant/architecture#30](https://github.com/Kuadrant/architecture/pull/30)
- Issue tracking: [Kuadrant/architecture#31](https://github.com/Kuadrant/architecture/issues/31)

# Summary
[summary]: #summary

Proposal for changes to the `DNSPolicy` API to allow it to provide a simple routing strategy as an option in a single cluster context. This will remove, but not negate, the complex DNS structure we use in a multi-cluster environment and in doing so allow use of popular dns integrators such as external-dns .

# Motivation
[motivation]: #motivation

The [`DNSPolicy`](https://github.com/Kuadrant/multicluster-gateway-controller/blob/main/pkg/apis/v1alpha1/dnspolicy_types.go) API (v1alpha1), was implemented as part of our multi cluster gateway offering using OCM and as such the design and implementation were influenced heavily by how we want multi cluster dns to work.

* Decouple the API entirely from OCM and multi cluster specific concepts.
* Simplify the DNS record structure created for a gateway listeners host for single cluster use.
* Improve the likelihood of adoption by creating an integration path for other kubernetes dns controllers such as [external-dns](https://github.com/kubernetes-sigs/external-dns).

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The DNSPolicy can be used to target a Gateway in a single cluster context and will create dns records for each listener host in an appropriately configured external dns provider.
In this context the advanced `loadbalancing` configuration is unnecessary, and the resulting DNSRecord can be created mapping individual listener hosts to a single DNS A or CNAME record by using the `simple` routing strategy in the DNSPolicy.

**Example 1.** DNSPolicy using `simple` routing strategy

```yaml
apiVersion: kuadrant.io/v1alpha2
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
  routingStrategy: simple
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
As the `simple` routing strategy is set in the DNSPolicy a DNSRecord resource with the following contents will be created:

```yaml
apiVersion: kuadrant.io/v1alpha2
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

The `providerRef` is included in the DNSRecord to allow the dns record controller to load the appropriate provider configuration during reconciliation and create the DNS records in the dns provider service e.g. route 53, by default the provider `Kind` is a secret.

**Example 2.** DNSPolicy using `simple` routing strategy with external dns provider

```yaml
apiVersion: kuadrant.io/v1alpha2
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
  routingStrategy: simple
```

In ths example if the DNSPolicy was attached to the same gateway described in example 1, the same DNSRecord would also be created but the `DNSRecord` controller would not reconcile it. In this scenario it is expected that an external controller is being used to manage the reconciliation of the DNSRecord resources such as [external-dns](https://github.com/kubernetes-sigs/external-dns).

**Example 3.** DNSPolicy using `simple` routing strategy on multi cluster gateway

```yaml
apiVersion: kuadrant.io/v1alpha2
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
  routingStrategy: simple
```

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: prod-web
  namespace: my-gateways
spec:
  gatewayClassName: kuadrant-multi-cluster-gateway-instance-per-cluster
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
    - type: kuadrant.io/MultiClusterIPAddress
      value: 172.31.200.0
    - type: kuadrant.io/MultiClusterIPAddress
      value: 172.31.201.0
```

Similar to example 1, except here the Gateway is a multi cluster gateway that has had its status updated by the `Gateway` controller to include `kuadrant.io/MultiClusterIPAddress` type addresses.
As the `simple` routing strategy is set in the DNSPolicy a DNSRecord resource with the following contents will be created:

```yaml
apiVersion: kuadrant.io/v1alpha2
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
        - 172.31.201.0
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## API Updates

DNSPolicy:

- new providerRef field `spec.providerRef`
- new routingStrategy field `spec.routingStrategy`
- new api version `v1alpha2`

DNSRecord:
 
- `spec.managedZone` replaced with `spec.providerRef`
- new zoneID field `spec.zoneID` 
- new api version `v1alpha2`

ManagedZone:

- `spec.dnsProviderSecretRef` replaced with `spec.providerRef`
- new api version `v1alpha2`

### DNSPolicy.spec.providerRef

The `providerRef` field is mandatory and contains a reference to a resource that shows how DNSRecords will be reconciled.
    - `spec.providerRef.name` - name of the provider resource
    - `spec.providerRef.kind` - kind of resource, can be anything but only `Secret` or `Managedzone` will be reconciled by the `DNSPolicy` controller.

A kind of type `Secret` will be managed by the `DNSPolicy` controller, and a secret in the dns policies namespace with the given name must exist. The expected contents of the secrets data is comparable to the `dnsProviderSecretRef` used by ManageZones.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: aws-credentials
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

A kind of type `ManagedZone` will be managed by the `DNSPolicy` controller, and a ManagedZone in the dns policies namespace with the given name must exist. 

A kind of any other type e.g. `ExternalDNS` informs the `DNSPolicy` controller that it should not reconcile any resources referencing it. All fields of `providerRef` will be ignored by the `DNSPolicy` controller and can be set to anything, however since the `providerRef` will still be copied over to any DNSRecord resources created by the policy controller the values may still be given meaning to that external DNSRecord reconciler.

### DNSPolicy.spec.routingStrategy[simple|weightedGeo]

The `routingStrategy` field is mandatory and dictates what kind of dns record structure the policy will create. Two routing strategy options are allowed `simple` or `weightedGeo`.

A reconciliation of DNSPolicy processes the target gateway and creates a DNSRecord per listener that is supported by the currently configured provider(hostname matches the hosted zones accessible with the credentials and config). The routing strategy used will determine the contents of the DNSRecord resources Endpoints array.

#### simple

```yaml
apiVersion: kuadrant.io/v1alpha2
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

Simple creates a single endpoint for an A record with multiple targets. Although intended for use in a single cluster context a simple routing strategy can still be used in a multi-cluster environment (OCM hub). In this scenario each clusters address will be added to the targets array to create a multi answer section in the dns response.

#### weightedGeo

```yaml
apiVersion: kuadrant.io/v1alpha2
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

WeightedGeo creates a more complex set of endpoints which use a combination of weighted and geo routing strategies. Although intended for use in a multi-cluster environment  (OCM hub) it will still be possible to use it in a single cluster context. In this scenario the record structure described above would be created for the single cluster.

This is the current default for DNSPolicy in a multi-cluster environment (OCM hub) and more details about it can be found in the original [DNSPolicy rfc](https://github.com/Kuadrant/architecture/blob/main/rfcs/0003-dns-policy.md).

### DNSRecord.spec.providerRef

More details of `providerRef` found in [DNSPolicy.spec.providerRef](#dnspolicyspecproviderref)

The DNSRecord API is updated to remove the `managedZone` reference in favour of directly referencing the `providerRef` credentials instead. The DNSRecord reconciliation will be unchanged except for loading the provider client from `providerRef` credentials.

The DNSPolicy reconciliation will be updated to remove the requirement for a ManagedZone resource to be created before a DNSPolicy can create dns records for it, instead it will be replaced in favour of just listing available zones directly in the currently configured dns provider. If no matching zone is found, no DNSRecord will be created.

There is a potential for a DNSRecord to be created successfully, but then a provider updated to remove access. In this case it is the responsibility of the DNSPolicy controller to report appropriate status back to the policy and target resource about the failure to process the record. More details on how status will be reported can be found in [rfc-0004](https://github.com/Kuadrant/architecture/blob/main/rfcs//0004-rlp-status.md)

### DNSRecord.spec.zoneID

The `zoneID` field is mandatory and contains the provider specific id of the hosted zone that this record should be published into. 

The DNSRecord reconciliation will use this zone when creating/updating or deleting endpoints for this record set. 

The `zoneID` should not change after being selected during initial creation and as such will be marked as immutable.

### ManagedZone.spec.providerRef

More details of `providerRef` found in [DNSPolicy.spec.providerRef](#dnspolicyspecproviderref)

Replaces the existing `dnsProviderSecretRef` for consistency with other resources that require a provider reference (DNSRecord and DNSPolicy). 

In the case of a ManagedZone a providerRef kind of type `ManagedZone` will not be allowed and will be rejected during create/update.  

# Prior art
[prior-art]: #prior-art

[ExternalDNS](https://github.com/kubernetes-sigs/external-dns)

* Uses annotations on the target Gateway as opposed to a proper API.
* Requires access to the HTTP route resources.
* Supports only a single provider per external dns instance.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

When a provider is configured using a kind not supported by the `DNSPolicy` controller e.g. `ExternalDNS` we will be relying on an external controller to correctly update the status of any DNSRecord resources created by our policy. This may have a negative impact on our ability to correctly report status back to the target resource.

When using a weightedGeo routing strategy in a single cluster context it is not expected that this will offer multi cluster capabilities without the use of OCM. Currently, it is expected that if you want to create a recordset that contains the addresses of multiple clusters you must use an OCM hub.

# Future possibilities
[future-possibilities]: #future-possibilities

The ability to support other kubernetes dns controllers such as ExternalDNS would potentially allow us to contribute to some of these projects in the area of polices for dns management of Gateway resources in kubernetes.
