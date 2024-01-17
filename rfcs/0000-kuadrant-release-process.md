# Kuadrant Release Process

- Feature Name: `kuadrant-release-process`
- Start Date: 2024-01-11
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000)
- Issue tracking: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/issues/0000)

# Summary
[summary]: #summary

Kuadrant is a set of components that their artifacts are built and delivered independently. This RFC aims to define every
aspect of the event of releasing a new version of the whole, in terms of versioning, cadence, communication, channels,
handover to other teams, etc.

# Motivation
[motivation]: #motivation

At the time being, there's no clear process nor guidelines to follow when releasing a new version of Kuadrant, which
leads to confusion and lack of transparency. We are currently relying on internal communication and certain people
in charge of the release process, which is not ideal.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

First, we need to define what releasing Kuadrant means, in a clear and transparent way that communicates to the community
what's happening and what to expect. The Kuadrant suite is composed of several components, each of them with its own
set of artifacts and versioning scheme. Defining the release process of the whole suite is a complex task, and it's
not only about the technical details of releasing the components, but also about the communication and transparency
with the community, the definition of the frequency of the releases, and when it's ready to be handover to other teams like
QA. This section aims to provide guidelines for the different aspects of the release process.

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

Each of them needs to be versioned independently, and the versioning scheme should follow [Semantic Versioning]. At
the time of cutting a release for any of them, it's important to keep in mind what section of the version to bump,
given a version number MAJOR.MINOR.PATCH, increment the:

* MAJOR version when you make incompatible API changes
* MINOR version when you add functionality in a backward compatible manner
* PATCH version when you make backward compatible bug fixes

Additional labels for pre-release and build metadata are available as extensions to the MAJOR.MINOR.PATCH format.

A more detailed explanation of the versioning scheme can be found in the [Semantic Versioning](https://semver.org/) website.

By releasing a new version of Kuadrant, we mean releasing the set of components with their corresponding semantic versioning,
some of them maybe freshly released, or others still using versioning from the previous one, and being the version of the
**Kuadrant Operator** the one that defines the version of the whole suite.

The technical details of how to release each component are out of the scope of this RFC and could be found in the
[Kuadrant components CI/CD](https://github.com/Kuadrant/architecture/pull/41) RFC.

## Handover to QA

Probably the most important and currently missing step in the release process is the handover to the Quality Assurance
(QA) team. The QA team is responsible for testing the different components of the Kuadrant suite, and they need to
be aware of the new version of the suite that is going to be released, what are the changes that are included, bug fixes
and new features in order they can plan their testing processes accordingly. The handover to QA should happen once the
release candidate is ready, and it's been tested by the Engineering team. The QA team should be notified in advance
of the release, and they should be able to test the release candidate before the actual release happens.

There is an ideal time to hand over to the QA team for testing, especially since we are using GitHub for orchestration,
we could briefly define it in the following steps:

1. Complete Development Work: The engineering team completes their work included in the milestone.
2. Create Release Candidate: The engineering team creates Release Candidate builds and manifests for all components required
for the release
3. Notify QA Team: At this time, the engineering team can notify the QA team that the deliverables for the release candidate
are ready for testing. This notification can be done through the GitHub Kuadrant board, or simply a public message in the
Kuadrant Slack channel.
4. Testing: The QA team tests the release candidate, checking for any bugs or issues. Then QA reports all the bugs as
GitHub issues and communicates testing status back publicly on Slack and/or email.
5. Iterate: Based on the feedback from the QA team, the development team makes any necessary adjustments and repeats the
process until the release candidate is deemed ready for production.
6. Publish Release: Once QA communicates that the testing has been successfully finished, the engineering team will publish
the release both on Github and in the corresponding registries, updates documentation for the new release, and communicates
it to all channels specified in Communication section.

## Cadence

Once the project is stable enough, and it's adoption increases, the community will be expecting a certain degree of
commitment from the maintainers, and that includes a regular release cadence. The frequency of the releases of the
different components could vary depending on the particular component needs. However, the **Kuadrant Operator**
it's been discussed in the past that it should be released every 3-4 weeks initially, including the latest released version
of every component in the suite. There's another RFC that focuses on the actual frequency of each component, one could
refer to the [Kuadrant Release Cadence RFC](https://github.com/pmccarthy/architecture/blob/release-cadence-proposal/rfcs/0007-project-release-cadence.md).

There are a few reasons for this:

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

By committing to a release cadence, software projects can benefit from improved efficiency, risk management, faster feedback
cycles, synchronization, and reduced complexity.

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

It's important to  note that keeping the documentation up to date is a responsibility of the component maintainers, and
it needs to be done before releasing a new version of the component. The importance of keeping a clear and up-to-date
documentation is crucial for the success of the project.

The documentation for the Kuadrant suite is compiled and available on the [Kuadrant website](https://kuadrant.io/). One
can find the source of the documentation within each component repository, in the `docs` directory. However, making this
information available on the website is a manual process, and should be done by the maintainers of the project. The
process of updating the documentation is simple and consists of the following steps:

1. Update the documentation in the corresponding component repository.
2. In [https://github.com/Kuadrant/docs.kuadrant.io/](https://github.com/Kuadrant/docs.kuadrant.io/), update the value of
the `import_url` within the `multirepo` in the `plugins` section of the [mkdocs.yml](https://github.com/Kuadrant/docs.kuadrant.io/blob/main/mkdocs.yml)
file, to point to the tag or branch of the component repository that contains the updated documentation.
3. Once the changes are merged to main, the workflow that updates the website will be triggered, and the documentation
will be updated.
4. If for some reason it's needed to trigger the workflow manually, one can do it from the GitHub Actions tab in the
[docs.kuadrant.io](https://github.com/Kuadrant/docs.kuadrant.io/actions/workflows/build.yaml) (`Actions > ci > Run Workflow`).

## Communication

Another important aspect of releasing a new version of the Kuadrant suite is the communication with the community and
other teams within the organization. A few examples of the communication channels that need to be updated are:

- Changelog generation
- Release notes
- Github Release publication
- Slack channel in Kubernetes workspace
- Blog post, if applicable
- Social media, if applicable


# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The alternative to the proposal is to keep the current process, which is not ideal and leads to confusion and lack of
transparency.

# Prior art
[prior-art]: #prior-art

There's been an organically grown process for releasing new versions of the Kuadrant suite, which is not documented and
it's been changing over time. However, there are some documentation for some of the components, worth mentioning:

* [Authorino release process](https://docs.google.com/document/d/1tLveyv8Zwe0wKyfUTWOlEnFeMB5aVGqIVDUjVYWax0U)
* [Limitador release process](https://github.com/Kuadrant/limitador/blob/main/RELEASE.md)

# Unresolved questions
[unresolved-questions]: #unresolved-questions

* What would be Kuadrant support policy?
* How many version are we going to back-port security and bug fixes to?
* What other teams need to be involved in the release process?

# Future possibilities
[future-possibilities]: #future-possibilities

Once the release process is accepted and battle-tested, we could aim to automate the process as much as possible.