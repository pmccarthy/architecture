# RFC Template

- Feature Name: `project_release_cadence`
- Start Date: 2023-11-21
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000)
- Issue tracking: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/issues/0000)

# Summary
[summary]: #summary

Establish a set release cadence across the Kuadrant project where all sub components of Kuadrant ship a new release of their respective codebase on a consistent basis.

# Motivation
[motivation]: #motivation

- To provide consistency and alignment across the Kuadrant project and its sub components with respect to release management
- Support consistent release testing cycles
- Act as a starting point for improving continous delivery processes to support the shipping of new releases with a view to reducing the cadence over time through process automation
- Provide end users with the ability to plan around new versions of Kuadrant by having a consistent schedule

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The below table is based on an example release cadence of 3 weeks (starting on the date of writing this proposal). On each date a new release of each component is cut from `main` which then kicks off the release testing cycle for Kuadrant as a whole. The release testing cycle will most likely involve both manual and automated testing initially, however the longer term goal would be to eventually have everything (within reason) fully automated. The reduction of the release cadence is dependent on the time it takes to validate and sign off on a new release. 

|Date   | Limitador | Authorino | MGC | Limitador Operator | Authorino Operator | Kuadrant Operator |
|---|---|---|---|---|---|---|
| Nov 21st | v1.4.0 | v0.16.0 | v0.2.0 | v0.6.0 |v0.9.0 | v0.5.0 |
| Dec 12th | v1.5.0 | v0.17.0 | v0.3.0 | v0.7.0 |v0.10.0 | v0.6.0 |
| Jan 2nd  | v1.6.0 | v0.18.0 | v0.4.0 | v0.8.0 |v0.11.0 | v0.7.0 |


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

<TBD>

# Drawbacks
[drawbacks]: #drawbacks

<TBD>

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

<TBD>

# Prior art
[prior-art]: #prior-art

<TBD>

# Unresolved questions
[unresolved-questions]: #unresolved-questions

<TBD>

# Future possibilities
[future-possibilities]: #future-possibilities

<TBD>