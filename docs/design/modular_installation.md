# Kuadrant Proposal - Modular Installation

Kuadrant is developing a set of loosely coupled functionalities built directly on top of Kubernetes.
Kuadrant aims to allow customers to just install, use and understand those functionalities they need.

## Problem Statement

Currently, the installation tool of kuadrant, the [kuadrantctl CLI](https://github.com/Kuadrant/kuadrantctl),
installs all or nothing. Installing more than the customer needs adds unneeded complexity and operational effort.
For example, if a customer is looking for rate limiting and not interested in authentication functionality,
then the customer should be able to just install and run that part of Kuadrant.

## High Level Goals

* Install only required components. Operate only required components.

Reduce system complexity and operational effort to the minimum required.
Components in this context make reference to deployments and running instances.

* Expose only the activated functionalities

A user of a partial Kuadrant install should not be confronted with data in custom resources that
has no meaning or is not accessible in their partial Kuadrant install. The design of the kuadrant
API should have this goal into account.

## Proposed Solution

The kuadrant installation mechanism should offer modular installation to enable/disable loosely coupled pieces of kuadrant.
Modular installation options should be feature oriented rather than deployment component oriented.
Then, it is up to the installation tool to decide what components need to be deployed and how to
configure it.

Each feature, or part of it, is eligible to be included or excluded when installing kuadrant.

Some profiles can be defined to group set of commonly required features. Naming the profiles
allows the customer to easily express wanted installation configuration. Furthermore, profiles
not only can be used to group a set of features, profiles can be used to define deployment options.

| Name | Description |
| ---  | --- |
| **Minimal** | Minimal installation required to run an API without any protection, analytics or API management. Default deployment option |
| **AuthZ** | Authentication and authorization mechanisms activated  |
| **RateLimit** | Basic rate limit (only pre-auth rate limit) features |
| **Full** | Full featured kuadrant installation |

A kuadrant operator, together with a design of a kuadrant CRD is desired.
Not only for kuadrant installation, but also for lifecycle management.
Additionally, the [kuadrantctl](https://github.com/Kuadrant/kuadrantctl) CLI tool can also
be useful to either deploy kuadrant components and manifests or just deploy the kuadrant operator.

The kuadrant control plane should be aware of the installed profile via env vars or command line params
in the control plane running components. With that information, the control plane can decide to
enable or disable CRD watching, label and annotation monitoring and ultimately reject any configuration
object that relies on disabled functionality. The least a customer can expect from kuadrant is to be
consistent and reject any functionality request that cannot provide.
