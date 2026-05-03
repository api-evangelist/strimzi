# Strimzi
Kubernetes-native operator for running Apache Kafka on Kubernetes and OpenShift, providing simplified deployment, management, and configuration of Kafka clusters via Custom Resource Definitions and the Kafka Bridge REST API.

**URL:** [https://strimzi.io/](https://strimzi.io/)

## Tags

- Kafka, Kubernetes, Operator, Messaging, Streaming, CRD, Event Streaming

## Timestamps

- **Created:** 2025
- **Modified:** 2026-05-02

## Artifacts

### APIs Index
| File | Description |
|------|-------------|
| [apis.yml](apis.yml) | Index of all Strimzi APIs and artifacts |

### Custom Resource Definitions (CRDs)
| File | Description |
|------|-------------|
| [crd/kafka-crd.yml](crd/kafka-crd.yml) | Kafka cluster CRD (kafkas.kafka.strimzi.io/v1beta2) |
| [crd/kafkatopic-crd.yml](crd/kafkatopic-crd.yml) | KafkaTopic CRD for declarative topic management |
| [crd/kafkauser-crd.yml](crd/kafkauser-crd.yml) | KafkaUser CRD for authentication and ACL management |

### OpenAPI Specifications
| File | Description |
|------|-------------|
| [openapi/strimzi-kafka-bridge-openapi.yml](openapi/strimzi-kafka-bridge-openapi.yml) | Strimzi Kafka Bridge REST API (OpenAPI 3.1.0) |

### JSON Schema
| File | Description |
|------|-------------|
| [json-schema/strimzi-kafka-schema.json](json-schema/strimzi-kafka-schema.json) | JSON Schema for the Kafka CRD resource |

### JSON Structure
| File | Description |
|------|-------------|
| [json-structure/strimzi-kafka-structure.json](json-structure/strimzi-kafka-structure.json) | Field structure and relationships for Kafka cluster resources |

### JSON-LD
| File | Description |
|------|-------------|
| [json-ld/strimzi-context.jsonld](json-ld/strimzi-context.jsonld) | JSON-LD context mapping Strimzi resources to Kafka and Kubernetes ontologies |

### Examples
| File | Description |
|------|-------------|
| [examples/strimzi-kafka-cluster-example.json](examples/strimzi-kafka-cluster-example.json) | Production Kafka cluster manifest (3 brokers, TLS, persistent storage) |
| [examples/strimzi-bridge-produce-example.json](examples/strimzi-bridge-produce-example.json) | Producing messages via the Kafka Bridge REST API |

### Spectral Rules
| File | Description |
|------|-------------|
| [rules/strimzi-rules.yml](rules/strimzi-rules.yml) | Spectral linting rules for Strimzi Kafka Bridge API conventions |

### Naftiko Capabilities
| File | Description |
|------|-------------|
| [capabilities/shared/kafka-bridge-api.yaml](capabilities/shared/kafka-bridge-api.yaml) | Shared Kafka Bridge API capability definition |
| [capabilities/kafka-messaging.yaml](capabilities/kafka-messaging.yaml) | Unified Kafka messaging workflow capability (REST + MCP) |

### Vocabulary
| File | Description |
|------|-------------|
| [vocabulary/strimzi-vocabulary.yml](vocabulary/strimzi-vocabulary.yml) | Domain vocabulary for Strimzi, Kafka, and Kubernetes operator concepts |
