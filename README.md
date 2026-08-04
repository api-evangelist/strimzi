# Strimzi (strimzi)

<!-- API-EVANGELIST-PROVENANCE:BEGIN -->
> ### About this repository
>
> **This is not our API.** This repository is an independent, third-party profile of a company's
> **publicly available** API surface, maintained by [API Evangelist](https://apievangelist.com).
> API Evangelist does not operate, host, resell, or support this company's APIs, and is not
> affiliated with or endorsed by the company unless stated on the profile.
>
> **Where the information came from.** Everything here is assembled from material a member of the
> public can reach with a browser and no credentials — the company's own website, developer portal
> and documentation, the specifications it publishes for public use (OpenAPI, AsyncAPI, JSON Schema,
> `apis.json`, `llms.txt` and similar), its public repositories, and its public status, pricing and
> changelog pages. **Nothing here is obtained by breaching a system, defeating an access control, or
> using credentials of any kind.**
>
> **The rating is an independent assessment.** The Kin Score and Agent Readiness rating are
> independently calculated scores of a company's *public* API artifacts, produced by API Evangelist
> against a published rubric. They are not certifications, endorsements, security assessments, or
> audits, and they score published artifacts — not the quality, safety, or security of the software.
>
> **Corrections, re-scores, and removal are free.** No partnership, contract, or purchase is
> required, and you do not need to justify the request.
>
> - **Something wrong?** Open an issue on this repository, or email
>   [info@apievangelist.com](mailto:info@apievangelist.com).
> - **Published something new?** Ask for a re-score and we will re-run the rating.
> - **Want the listing taken down?** Say so and we will honor it. The profile is reduced to your
>   company name, a factual description, and a link to your own site, and the company is recorded as
>   **unrated** — never scored zero for having asked.
>
> **Response times.** Acknowledgement within **one business day**; removal or restriction within
> **two business days**; corrections and re-scores within **five business days**.
>
> **On a security or compliance team?** Email
> [info@apievangelist.com](mailto:info@apievangelist.com) with *security* in the subject line and
> you will get a person, not a form. We will tell you exactly which public URLs this profile was
> built from so your team can see the same surface we did, and we will take the listing down on
> request while you work through it.
>
> Full detail: **[Where this data comes from](https://apievangelist.com/about/where-our-data-comes-from)**
<!-- API-EVANGELIST-PROVENANCE:END -->

Strimzi is a CNCF project providing a Kubernetes-native operator for running Apache Kafka on Kubernetes and OpenShift. It simplifies the deployment, management, scaling, and configuration of Kafka clusters using Kubernetes Custom Resource Definitions (CRDs). Strimzi manages the full Kafka ecosystem including brokers, ZooKeeper/KRaft, Kafka Connect, Kafka MirrorMaker 2, Kafka Bridge, and Schema Registry. The operator pattern lets teams declare desired Kafka topology via YAML manifests managed by Kubernetes.

**APIs.json:** [https://raw.githubusercontent.com/api-evangelist/strimzi/refs/heads/main/apis.yml](https://raw.githubusercontent.com/api-evangelist/strimzi/refs/heads/main/apis.yml)

## Scope

- **Type:** Index
- **Position:** Consumer
- **Access:** 3rd-Party

## Tags

- Kafka
- Kubernetes
- Messaging
- Operator
- Streaming

## Timestamps

- **Created:** 2025-01-01
- **Modified:** 2026-05-19

## APIs

### Strimzi Operator API

The Strimzi Operator API is expressed through Kubernetes Custom Resource Definitions (CRDs). Operators are controlled by creating and modifying Kafka, KafkaTopic, KafkaUser, KafkaConnect, KafkaMirrorMaker2, KafkaBridge, and related custom resources via the Kubernetes API. The Strimzi operator watches these resources and reconciles the actual cluster state to match the desired specification.

- **Human URL:** [https://strimzi.io/docs/operators/latest/configuring.html](https://strimzi.io/docs/operators/latest/configuring.html)

#### Tags

- Kafka
- Kubernetes
- Messaging
- Operator
- Streaming

#### Properties

- [Documentation](https://strimzi.io/docs/operators/latest/)
- [Git Hub](https://github.com/strimzi/strimzi-kafka-operator)
- [Kubernetes C R D](https://raw.githubusercontent.com/api-evangelist/strimzi/refs/heads/main/crd/kafka-crd.yml)
- [Kubernetes C R D](https://raw.githubusercontent.com/api-evangelist/strimzi/refs/heads/main/crd/kafkatopic-crd.yml)
- [Kubernetes C R D](https://raw.githubusercontent.com/api-evangelist/strimzi/refs/heads/main/crd/kafkauser-crd.yml)
- [JSON Schema](https://raw.githubusercontent.com/api-evangelist/strimzi/refs/heads/main/json-schema/strimzi-kafka-schema.json) — [JSON Schema](https://json-schema.org/specification)
- [Postman Collection](collections/strimzi-kafka-bridge.postman_collection.json) — [Postman Collection 2.1](https://schema.getpostman.com/json/collection/v2.1.0/collection.json)
- [Open Collection](collections/strimzi-kafka-bridge.opencollection.json) — [Open Collection 1.0](https://schema.opencollection.com/opencollection/v1.0.0.json)

### Strimzi Kafka Bridge REST API

The Strimzi Kafka Bridge provides an HTTP/REST interface to Apache Kafka, allowing HTTP clients to produce and consume messages, manage consumer groups, and query topic metadata without a native Kafka client. The Bridge implements part of the OpenAPI specification for HTTP-to-Kafka bridging.

- **Human URL:** [https://strimzi.io/docs/bridge/latest/](https://strimzi.io/docs/bridge/latest/)
- **Base URL:** `http://localhost:8080`

#### Tags

- Kafka
- Messaging
- REST API
- Streaming

#### Properties

- [Documentation](https://strimzi.io/docs/bridge/latest/)
- [OpenAPI](https://raw.githubusercontent.com/api-evangelist/strimzi/refs/heads/main/openapi/strimzi-kafka-bridge-openapi.yml) — [OpenAPI Specification](https://spec.openapis.org/oas/latest.html)
- [Git Hub](https://github.com/strimzi/strimzi-kafka-bridge)
- [Postman Collection](collections/strimzi-kafka-bridge.postman_collection.json) — [Postman Collection 2.1](https://schema.getpostman.com/json/collection/v2.1.0/collection.json)
- [Open Collection](collections/strimzi-kafka-bridge.opencollection.json) — [Open Collection 1.0](https://schema.opencollection.com/opencollection/v1.0.0.json)

## Common Properties

- [LinkedIn](https://www.linkedin.com/company/strimzi)
- [Website](https://strimzi.io)
- [Documentation](https://strimzi.io/docs/operators/latest/)
- [Git Hub](https://github.com/strimzi/strimzi-kafka-operator)
- [Blog](https://strimzi.io/blog/)
- [Helm  Chart](https://artifacthub.io/packages/helm/strimzi/strimzi-kafka-operator)
- [Slack](https://slack.cncf.io)
- [Changelog](https://github.com/strimzi/strimzi-kafka-operator/releases)
- [C N C F](https://www.cncf.io/projects/strimzi/)
- [OpenAPI](https://raw.githubusercontent.com/api-evangelist/strimzi/refs/heads/main/openapi/strimzi-kafka-bridge-openapi.yml) — [OpenAPI Specification](https://spec.openapis.org/oas/latest.html)
- [JSON Schema](https://raw.githubusercontent.com/api-evangelist/strimzi/refs/heads/main/json-schema/strimzi-kafka-schema.json) — [JSON Schema](https://json-schema.org/specification)
- [J S O N L D Context](https://raw.githubusercontent.com/api-evangelist/strimzi/refs/heads/main/json-ld/strimzi-context.jsonld)

## Maintainers

**FN:** Kin Lane
**Email:** kin@apievangelist.com
