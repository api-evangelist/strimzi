---
title: "Gateway API and TLSRoute support in Strimzi"
url: "https://strimzi.io/blog/2026/07/09/gateway-api-tlsroute-support-in-strimzi/"
date: "2026-07-09"
author: "Paolo Patierno"
feed_url: "https://strimzi.io/feed.xml"
---
Exposing Apache Kafka to clients running outside a Kubernetes cluster has always required some creative plumbing. Over the years, Strimzi has accumulated four external listener types ( nodeport , loadbalancer , route for OpenShift only, and ingress ), and they each come with their own trade-offs. The ingress type deserves special mention here, because it was recently deprecated and the story behind that deprecation is exactly what motivates this post.
