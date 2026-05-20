---
title: "Strimzi Test Container: Simplifying Kafka & Connect Testing"
url: "https://strimzi.io/blog/2026/01/15/strimzi-test-container/"
date: "2026-01-15T00:00:00+00:00"
author: "Maros Orsak"
feed_url: "https://strimzi.io/feed.xml"
---
Testing Kafka-based applications used to be a pain. Back in 2019, your options each came with significant trade-offs, and they still do: Full Kafka clusters are resource-intensive and hard to manage in CI Mocks don’t behave like real Kafka and become a maintenance burden Embedded Kafka can cause dependency conflicts with your application’s Kafka client libraries We built Strimzi Test Container to avoid these issues. Why We Built Strimzi Test Container We needed proper isolation from embedded Kafka’s dependency conflicts.
