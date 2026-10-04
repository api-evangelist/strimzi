---
title: "Beyond mTLS: Configurable Security for Internal Apache Kafka Cluster Communication"
url: "https://strimzi.io/blog/2026/09/22/beyond-mtls-configurable-security-for-internal-apache-kafka-cluster-communication/"
date: "2026-09-22"
author: "Jakub Scholz"
feed_url: "https://strimzi.io/feed.xml"
---
For a long time, one thing has been common to every Strimzi-based Apache Kafka cluster. It uses TLS encryption and mTLS authentication for all internal cluster communication. Data replication between brokers, KRaft controller communication, Strimzi operators talking with Kafka … all of this always uses TLS encryption and mTLS authentication.
