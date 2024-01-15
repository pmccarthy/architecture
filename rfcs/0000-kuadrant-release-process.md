# Kuadrant Release Process

- Feature Name: `kuadrant-release-process`
- Start Date: 2024-01-11
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000)
- Issue tracking: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/issues/0000)

# Summary
[summary]: #summary

Kuadrant is a set of components that their artifacts are built and delivered independently. This RFC aims to define every
aspect of the event of releasing a new version of the whole, in terms of versioning, cadence, communication, channels, etc.

# Motivation
[motivation]: #motivation

At the time being, there's no clear process nor guidelines to follow when releasing a new version of Kuadrant, which
leads to confusion and lack of transparency. We are currently relying on internal communication and certain people
in charge of the release process, which is not ideal.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

First, we need to define what releasing Kuadrant means, in a clear and transparent way that communicates to the community
what's happening and what to expect. This includes:

- Components included in each release
- Versioning
- Cadence
- Repositories and Hubs
- Documentation
- Communication

## Components and versioning

The set of components that are part of the _Kuadrant suite_ are the following:

- [Authorino](https://github.com/Kuadrant/authorino): Kubernetes-native authorization service for tailor-made Zero Trust API security.
- [Authorino Operator](https://github.com/Kuadrant/authorino-operator): A Kubernetes Operator to manage Authorino instances.
- [Limitador](https://github.com/Kuadrant/limitador): A generic rate-limiter written in Rust.
- [Limitador Operator](https://github.com/Kuadrant/limitador-operator/): A Kubernetes Operator to manage Limitador deployments.
- [Wasm Shim](https://github.com/Kuadrant/wasm-shim/): A Proxy-Wasm module written in Rust, acting as a shim between Envoy and Limitador.
- [Multicluster Gateway Controller](https://github.com/Kuadrant/multicluster-gateway-controller): Provides multi-cluster
connectivity and global load balancing.
- [Kuadrant Operator](https://github.com/Kuadrant/kuadrant-operator/): The Operator to install and manage the lifecycle
of the Kuadrant components deployments.

Each of them needs to be versioned independently, and the versioning scheme should follow [Semantic Versioning](https://semver.org/).
The technical details of how to release each component are out of the scope of this RFC and could be found in the
[Kuadrant components CI/CD](https://github.com/Kuadrant/architecture/pull/41) RFC.

By releasing a new version of Kuadrant, we mean releasing a new version of every component in the suite, with their
corresponding semantic versioning, and being the version of the **Kuadrant Operator** the one that defines the version of
the whole suite.

## Cadence

The frequency of the previously mentioned components, with the exception of the **Kuadrant Operator**, could vary
depending on the particular component needs. However, the **Kuadrant Operator** should be released every 2 weeks,
including the latest released version of every component in the suite.

For a more in-depth analysis of the cadence of each component, please refer to the
[Kuadrant Release Cadence RFC](https://github.com/pmccarthy/architecture/blob/release-cadence-proposal/rfcs/0007-project-release-cadence.md).

## Repositories and Hubs

Every component in Kuadrant has its own repository, and the source code is hosted in GitHub, mentioned in the previous
section. However, the images built and manifests generated are hosted in different registries, depending on the
component. The following table shows the different registries used by each component:

| Component                       | Artifacts                                      | Registry / Hub                                                                         |
|---------------------------------|------------------------------------------------|----------------------------------------------------------------------------------------|
| Authorino                       | authorino images                               | [Quay.io](https://quay.io/repository/kuadrant/authorino)                               |
| Authorino Operator              | authorino-operator images                      | [Quay.io](https://quay.io/repository/kuadrant/authorino-operator)                      |
|                                 | authorino-operator-bundle images               | [Quay.io](https://quay.io/repository/kuadrant/authorino-operator-bundle)               |
|                                 | authorino-operator-catalog images              | [Quay.io](https://quay.io/repository/kuadrant/authorino-operator-catalog)              |
|                                 | authorino-operator manifests                   | [OperatorHub.io](https://operatorhub.io/operator/authorino-operator)                   |
| Limitador                       | limitador server images                        | [Quay.io](https://quay.io/repository/kuadrant/limitador)                               |
|                                 | limitador crate                                | [Crates.io](https://crates.io/crates/limitador)                                        |
| Limitador Operator              | limitador-operator images                      | [Quay.io](https://quay.io/repository/kuadrant/limitador-operator)                      |
|                                 | limitador-operator-bundle images               | [Quay.io](https://quay.io/repository/kuadrant/limitador-operator-bundle)               |
|                                 | limitador-operator-catalog images              | [Quay.io](https://quay.io/repository/kuadrant/limitador-operator-catalog)              |
|                                 | limitador-operator manifests                   | [OperatorHub.io](https://operatorhub.io/operator/limitador-operator)                   |
| Wasm Shim                       | wasm-shim images                               | [Quay.io](https://quay.io/repository/kuadrant/wasm-shim)                               |
| Multicluster Gateway Controller | multicluster-gateway-controller images         | [Quay.io](https://quay.io/repository/kuadrant/multicluster-gateway-controller)         |
|                                 | multicluster-gateway-controller-bundle images  | [Quay.io](https://quay.io/repository/kuadrant/multicluster-gateway-controller-bundle)  |
|                                 | multicluster-gateway-controller-catalog images | [Quay.io](https://quay.io/repository/kuadrant/multicluster-gateway-controller-catalog) |
| Policy Controller               | policy-controller images                       | [Quay.io](https://quay.io/repository/kuadrant/policy-controller)                       |
| Kuadrant Operator               | kuadrant-operator images                       | [Quay.io](https://quay.io/repository/kuadrant/kuadrant-operator)                       |
|                                 | kuadrant-operator-bundle images                | [Quay.io](https://quay.io/repository/kuadrant/kuadrant-operator-bundle)                |
|                                 | kuadrant-operator-catalog images               | [Quay.io](https://quay.io/repository/kuadrant/kuadrant-operator-catalog)               |
|                                 | kuadrant-operator manifests                    | [OperatorHub.io](https://operatorhub.io/operator/kuadrant-operator)                    |

## Documentation

The documentation for the Kuadrant suite is compiled and available on the [Kuadrant website](https://kuadrant.io/). One
can find the source of the documentation within each component repository, in the `docs` directory. It's important to
note that keeping the documentation up to date is a responsibility of the component maintainers, and it needs to be
done before releasing a new version of the component. The importance of keeping a clear and up-to-date documentation
is crucial for the success of the project.

## Communication

Another important aspect of releasing a new version of the Kuadrant suite is the communication with the community and
other teams within the organization. A few examples of the communication channels that need to be updated are:

- Changelog generation
- Release notes
- Slack channel, in Red Hat and Kubernetes workspaces
- Blog post, if applicable
- Social media, if applicable

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

As mentioned in the previous section, the Kuadrant suite is composed of several components, each of them with its own
set of artifacts and versioning scheme. Defining the release process of the whole suite is a complex task, and it's
not only about the technical details of releasing the components, but also about the communication and transparency
with the community. It also involves the definition of the cadence of the releases, and the frequency of the releases
of each component. This section aims to provide a more in-depth analysis of the different aspects of the release process.

## Components and versioning

Taking into account the different components listed in the [Guide-level explanation section](#guide-level-explanation),
at the time of cutting a release for any of them, it's important to keep in mind what section of the version to bump,
given a version number MAJOR.MINOR.PATCH, increment the:

* MAJOR version when you make incompatible API changes
* MINOR version when you add functionality in a backward compatible manner
* PATCH version when you make backward compatible bug fixes

Additional labels for pre-release and build metadata are available as extensions to the MAJOR.MINOR.PATCH format.

A more detailed explanation of the versioning scheme can be found in the [Semantic Versioning](https://semver.org/) website.

## Cadence

Once the project is stable enough, and it's adoption increases, the community will be expecting a certain degree of
commitment from the maintainers, and that includes a regular release cadence. The frequency of the releases of the
different components could vary depending on the particular component needs. However, the **Kuadrant Operator** as mentioned
in the previous section, should be released every 2 weeks as it's been already proposed, including the latest released
version of every component in the suite. There are a few reasons for this:

- Delivering Unparalleled Value to Customers: Regular releases can provide customers with regular updates and improvements.
These updates can include new features and essential bug fixes, thus enhancing the overall value delivered to the customers.
- Maximizing Deployment Efficiency: By releasing software at regular intervals, teams can align their activities with
available resources and environments, ensuring optimal utilization. This leads to increased efficiency in the deployment process.
- Risk Management: Regular releases can help identify and fix issues early, reducing the risk of major failures that could
affect customers.
- Feedback Cycle: Regular releases allow for quicker feedback cycles. This means that any issues or improvements
identified by users can be addressed promptly, leading to a more refined product over time.
- Synchronization: Regular releases can help synchronize work across different teams or departments, creating a more
reliable, dependable solution development and delivery process.
- Reduced Complexity: Managing a smaller number of releases can reduce complexity. For example, having many different
releases out in the field can lead to confusion and management overhead.

By committing to a release cadence, software projects can benefit from improved efficiency, risk management, faster feedback cycles, synchronization, and reduced complexity.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- Does another project have a similar feature?
- What can be learned from it? What's good? What's less optimal?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other tentatives - successful or not, provide readers of your RFC with a fuller picture.

Note that while precedent set by other projects is some motivation, it does not on its own motivate an RFC.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would be and how it would affect the platform and project as a whole. Try to use this section as a tool to further consider all possible interactions with the project and its components in your proposal. Also consider how this all fits into the roadmap for the project and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the RFC you are writing but otherwise related.

Note that having something written down in the future-possibilities section is not a reason to accept the current or a future RFC; such notes should be in the section on motivation or rationale in this or subsequent RFCs. The section merely provides additional information.