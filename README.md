# Strimzi (strimzi)

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
