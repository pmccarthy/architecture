# Agenda Weekly Kuadrant Technical Discussions


Meeting Link: https://meet.google.com/gmw-ddec-dvh


## Backlog 

- CI/CD for Kuadrant as a whole (at least the CI side for now). 
    - What do we do now. Are there gaps we would like to address (growing CI CD with the project is important)
    - suggestion: look at an image that contains and executes our tests (future proofing)
    - Release process for kuadrant as a whole 
- OAS flow and API product vs API 
- Observability for Kuadrant:
  - Available metrics
  - What role does Kiali have here?
  - Key areas :
    - Throughput, latency, availability (others?)
- Kuadrant and istio responsibilities
  - should kuadrant expect an installation of istio and so should we be using the istio-system namespace


## 24/11/21
- Modular installation of Kuadrant   
  - Decision : This will be done via flags to the CLI. In the future it will be handled by the Operator CRD for installation of Kuadarant
- Module APIs (part 2) 
  - Proposal to be written on how auth and rate limit will be associated with APIs and how those APIs will form APIProducts (open question on if we still need the APIProduct CR). We want to cover usecases such as:
    - Supporting unauthenticated Endpoints being specified
    - Different authentication methods per API
    - No rate limiting for a particular endpoint 


## 17/11/21

### [cbrookes/eguzki] Modular APIs for Kuadrant as a whole  what is the right level of abstraction and user experience.
- [Gui] We already know the features that should go in the API; we are discussing the format of the API
- [Gui] Format options we have to decide on:
Linking directly the low-level CRs (Authorino’s AuthConfig, Limitador’s RateLimit, etc)
APIProduct has refs to other CRs
Other CRs ref the APIProduct (in the annotations)
- Having another abstraction for the features on top, at Kuadrant level
- We may want to limit what interactions the user can have
- We may want to expand what interactions the user can have → i.e. what if auth is not handled by authorino?
- [cbrookes] option generate a bootstrap set of resources based on open api spec
#### Actions:
- [Thomas Maas] to create a github issue with the details. Start investigating generating Kuadrant resources via Kuadrantctl leveraging an OpenAPI Spec (OAS) + extension for an authentication use case not covered by the existing spec. This use case focuses on the OAS being the source of truth and the only thing that should be modified.
- [Eguski] Create a github issue with the details. Investigate using the ApiProduct CR as a reference for binding existing authconfigs and ratelimitconfig to a particular API 



