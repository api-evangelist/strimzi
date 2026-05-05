---
title: "Monitoring of custom resources"
url: "https://strimzi.io/blog/2025/10/01/monitoring-of-custom-resources/"
date: "2025-10-01T00:00:00+00:00"
author: "Sebastian Gaiser (Hetzner)"
feed_url: "https://strimzi.io/feed.xml"
---
<p>Running Kafka on Kubernetes with Strimzi is straightforward.
But getting a complete, consistent view of everything in your Kubernetes and Kafka cluster(s) can still be hard.
Kafka topics and users are often owned by different teams and configured differently, especially when a Kafka cluster is shared across a company (for example, by multiple teams).
As usage grows, gathering information and troubleshooting gets harder.</p>

<p>This post shows a practical, low‑friction way to close monitoring gaps using <a href="https://github.com/kubernetes/kube-state-metrics">kube-state-metrics</a> so you can track every Strimzi custom resource (CR) with a simple, consistent, Kubernetes‑native approach.</p>

<h3 id="what-strimzi-exposes-today">What Strimzi exposes today</h3>

<p>All Strimzi operators (Cluster, User, and Topic Operator) expose metrics in Prometheus-compatible format.
However, only the Cluster Operator exposes a metric (<code class="language-plaintext highlighter-rouge">strimzi_resource_state</code>) that reports the state of custom resources (ready or not).</p>

<p>It provides this metric for:</p>

<ul>
  <li><code class="language-plaintext highlighter-rouge">Kafka</code></li>
  <li><code class="language-plaintext highlighter-rouge">KafkaConnect</code></li>
  <li><code class="language-plaintext highlighter-rouge">KafkaBridge</code></li>
  <li><code class="language-plaintext highlighter-rouge">KafkaMirrorMaker2</code></li>
  <li><code class="language-plaintext highlighter-rouge">KafkaRebalance</code></li>
</ul>

<p>It does not provide this metric for:</p>

<ul>
  <li><code class="language-plaintext highlighter-rouge">KafkaNodePool</code></li>
  <li><code class="language-plaintext highlighter-rouge">KafkaConnector</code></li>
  <li><code class="language-plaintext highlighter-rouge">StrimziPodSet</code></li>
</ul>

<p>The User and Topic Operators also do not provide this metric for <code class="language-plaintext highlighter-rouge">KafkaUser</code> and <code class="language-plaintext highlighter-rouge">KafkaTopic</code>.</p>

<p>During routine work, it’s easy to introduce misconfigurations, from a simple typo in a topic setting to using a property that was removed in the last release.
These small issues can trigger long debugging sessions. Often the fix is simple, but finding it is not.</p>

<h3 id="the-simple-fix-use-kube-state-metrics-for-crds">The simple fix: use kube-state-metrics for CRDs</h3>

<p>Instead of relying on the limited <code class="language-plaintext highlighter-rouge">strimzi_resource_state</code> metric, you can use kube-state-metrics to cover <em>all</em> Strimzi custom resources.</p>

<p>kube-state-metrics can read any Kubernetes object, including custom resources, and export Info-style metrics with labels you control. This approach gives you:</p>

<ul>
  <li>A complete inventory of all Strimzi CRs</li>
  <li>Consistent metric labels across teams</li>
  <li>A Kubernetes‑native approach that fits existing tooling (Prometheus)</li>
</ul>

<p>In short, you increase monitoring coverage without changing Strimzi or writing custom exporters.</p>

<h3 id="how-to-deploy">How to deploy</h3>

<p>In general, Strimzi provides plain Kubernetes manifests in the examples directory.</p>

<h4 id="option-a--apply-manifests">Option A — Apply manifests</h4>

<p>From <a href="https://github.com/strimzi/strimzi-kafka-operator/releases/tag/0.48.0">Strimzi 0.48.0</a> onwards, examples are included to deploy kube-state-metrics with the corresponding:</p>

<ul>
  <li>RBAC resources (<code class="language-plaintext highlighter-rouge">ServiceAccount</code>, <code class="language-plaintext highlighter-rouge">RoleBinding</code>, <code class="language-plaintext highlighter-rouge">ClusterRoleBinding</code>)</li>
  <li>A <code class="language-plaintext highlighter-rouge">ConfigMap</code> for CR configuration</li>
  <li>The <code class="language-plaintext highlighter-rouge">Deployment</code></li>
  <li>A <code class="language-plaintext highlighter-rouge">ServiceMonitor</code> for <a href="https://prometheus-operator.dev/">Prometheus Operator</a> to scrape itself</li>
</ul>

<p><a href="https://github.com/strimzi/strimzi-kafka-operator/tree/main/examples/metrics/kube-state-metrics">Examples directory for kube-state-metrics</a>.</p>

<h4 id="option-b--use-helm">Option B — Use Helm</h4>

<p>If you prefer deploying applications via Helm, you can use the <a href="https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-state-metrics">Prometheus Community Helm chart for kube-state-metrics</a>, which deploys the required resources in one go.
This chart is also part of the commonly used <a href="https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack">kube-prometheus-stack</a>, so you can simply use the same values directly there.</p>

<p>In the following demonstration, the kube-state-metrics Helm chart version <code class="language-plaintext highlighter-rouge">6.3.0</code> (kube-state-metrics <code class="language-plaintext highlighter-rouge">2.17.0</code>) is used. See the <a href="https://github.com/prometheus-community/helm-charts/tree/kube-state-metrics-6.3.0/charts/kube-state-metrics">chart reference</a>.
Configuration values used:</p>

<div class="language-yaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c1"># my-strimzi-kube-state-metrics-values.yaml</span>
<span class="na">prometheus</span><span class="pi">:</span>
  <span class="na">monitor</span><span class="pi">:</span>
    <span class="na">enabled</span><span class="pi">:</span> <span class="kc">true</span>
<span class="na">collectors</span><span class="pi">:</span> <span class="pi">[]</span>
<span class="na">extraArgs</span><span class="pi">:</span>
  <span class="pi">-</span> <span class="s">--custom-resource-state-only=true</span>
<span class="na">rbac</span><span class="pi">:</span>
  <span class="na">extraRules</span><span class="pi">:</span>
    <span class="pi">-</span> <span class="na">apiGroups</span><span class="pi">:</span>
        <span class="pi">-</span> <span class="s">kafka.strimzi.io</span>
      <span class="na">resources</span><span class="pi">:</span>
        <span class="pi">-</span> <span class="s">kafkatopics</span>
        <span class="pi">-</span> <span class="s">kafkausers</span>
        <span class="pi">-</span> <span class="s">kafkas</span>
        <span class="pi">-</span> <span class="s">kafkanodepools</span>
        <span class="pi">-</span> <span class="s">kafkarebalances</span>
        <span class="pi">-</span> <span class="s">kafkaconnects</span>
        <span class="pi">-</span> <span class="s">kafkaconnectors</span>
        <span class="pi">-</span> <span class="s">kafkamirrormaker2s</span>
      <span class="na">verbs</span><span class="pi">:</span> <span class="pi">[</span> <span class="s2">"</span><span class="s">list"</span><span class="pi">,</span> <span class="s2">"</span><span class="s">watch"</span> <span class="pi">]</span>
    <span class="pi">-</span> <span class="na">apiGroups</span><span class="pi">:</span>
        <span class="pi">-</span> <span class="s">core.strimzi.io</span>
      <span class="na">resources</span><span class="pi">:</span>
        <span class="pi">-</span> <span class="s">strimzipodsets</span>
      <span class="na">verbs</span><span class="pi">:</span> <span class="pi">[</span> <span class="s2">"</span><span class="s">list"</span><span class="pi">,</span> <span class="s2">"</span><span class="s">watch"</span> <span class="pi">]</span>
    <span class="pi">-</span> <span class="na">apiGroups</span><span class="pi">:</span>
        <span class="pi">-</span> <span class="s">access.strimzi.io</span>
      <span class="na">resources</span><span class="pi">:</span>
        <span class="pi">-</span> <span class="s">kafkaaccesses</span>
      <span class="na">verbs</span><span class="pi">:</span> <span class="pi">[</span> <span class="s2">"</span><span class="s">list"</span><span class="pi">,</span> <span class="s2">"</span><span class="s">watch"</span> <span class="pi">]</span>
<span class="na">customResourceState</span><span class="pi">:</span>
  <span class="na">enabled</span><span class="pi">:</span> <span class="kc">true</span>
  <span class="na">config</span><span class="pi">:</span>
    <span class="na">spec</span><span class="pi">:</span>
      <span class="na">resources</span><span class="pi">:</span>
        <span class="pi">-</span> <span class="na">groupVersionKind</span><span class="pi">:</span>
            <span class="na">group</span><span class="pi">:</span> <span class="s">kafka.strimzi.io</span>
            <span class="na">version</span><span class="pi">:</span> <span class="s">v1beta2</span>
            <span class="na">kind</span><span class="pi">:</span> <span class="s">KafkaTopic</span>
          <span class="na">metricNamePrefix</span><span class="pi">:</span> <span class="s">strimzi_kafka_topic</span>
          <span class="na">metrics</span><span class="pi">:</span>
            <span class="pi">-</span> <span class="na">name</span><span class="pi">:</span> <span class="s">resource_info</span>
              <span class="na">help</span><span class="pi">:</span> <span class="s2">"</span><span class="s">The</span><span class="nv"> </span><span class="s">current</span><span class="nv"> </span><span class="s">state</span><span class="nv"> </span><span class="s">of</span><span class="nv"> </span><span class="s">a</span><span class="nv"> </span><span class="s">Strimzi</span><span class="nv"> </span><span class="s">Kafka</span><span class="nv"> </span><span class="s">topic</span><span class="nv"> </span><span class="s">resource."</span>
              <span class="na">each</span><span class="pi">:</span>
                <span class="na">type</span><span class="pi">:</span> <span class="s">Info</span>
                <span class="na">info</span><span class="pi">:</span>
                  <span class="na">labelsFromPath</span><span class="pi">:</span>
                    <span class="na">name</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">metadata</span><span class="pi">,</span> <span class="nv">name</span> <span class="pi">]</span>
              <span class="na">labelsFromPath</span><span class="pi">:</span>
                <span class="na">exported_namespace</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">metadata</span><span class="pi">,</span> <span class="nv">namespace</span> <span class="pi">]</span>
                <span class="na">ready</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">conditions</span><span class="pi">,</span> <span class="s2">"</span><span class="s">[type=Ready]"</span><span class="pi">,</span> <span class="nv">status</span> <span class="pi">]</span>
                <span class="na">deprecated</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">conditions</span><span class="pi">,</span> <span class="s2">"</span><span class="s">[reason=DeprecatedFields]"</span><span class="pi">,</span> <span class="nv">type</span> <span class="pi">]</span>
                <span class="na">partitions</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">spec</span><span class="pi">,</span> <span class="nv">partitions</span> <span class="pi">]</span>
                <span class="na">replicas</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">spec</span><span class="pi">,</span> <span class="nv">replicas</span> <span class="pi">]</span>
                <span class="na">generation</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">observedGeneration</span> <span class="pi">]</span>
                <span class="na">topicId</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">topicId</span> <span class="pi">]</span>
                <span class="na">topicName</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">topicName</span> <span class="pi">]</span>
        <span class="pi">-</span> <span class="na">groupVersionKind</span><span class="pi">:</span>
            <span class="na">group</span><span class="pi">:</span> <span class="s">kafka.strimzi.io</span>
            <span class="na">version</span><span class="pi">:</span> <span class="s">v1beta2</span>
            <span class="na">kind</span><span class="pi">:</span> <span class="s">KafkaUser</span>
          <span class="na">metricNamePrefix</span><span class="pi">:</span> <span class="s">strimzi_kafka_user</span>
          <span class="na">metrics</span><span class="pi">:</span>
            <span class="pi">-</span> <span class="na">name</span><span class="pi">:</span> <span class="s">resource_info</span>
              <span class="na">help</span><span class="pi">:</span> <span class="s2">"</span><span class="s">The</span><span class="nv"> </span><span class="s">current</span><span class="nv"> </span><span class="s">state</span><span class="nv"> </span><span class="s">of</span><span class="nv"> </span><span class="s">a</span><span class="nv"> </span><span class="s">Strimzi</span><span class="nv"> </span><span class="s">Kafka</span><span class="nv"> </span><span class="s">user</span><span class="nv"> </span><span class="s">resource."</span>
              <span class="na">each</span><span class="pi">:</span>
                <span class="na">type</span><span class="pi">:</span> <span class="s">Info</span>
                <span class="na">info</span><span class="pi">:</span>
                  <span class="na">labelsFromPath</span><span class="pi">:</span>
                    <span class="na">name</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">metadata</span><span class="pi">,</span> <span class="nv">name</span> <span class="pi">]</span>
              <span class="na">labelsFromPath</span><span class="pi">:</span>
                <span class="na">exported_namespace</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">metadata</span><span class="pi">,</span> <span class="nv">namespace</span> <span class="pi">]</span>
                <span class="na">ready</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">conditions</span><span class="pi">,</span> <span class="s2">"</span><span class="s">[type=Ready]"</span><span class="pi">,</span> <span class="nv">status</span> <span class="pi">]</span>
                <span class="na">deprecated</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">conditions</span><span class="pi">,</span> <span class="s2">"</span><span class="s">[reason=DeprecatedFields]"</span><span class="pi">,</span> <span class="nv">type</span> <span class="pi">]</span>
                <span class="na">secret</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">secret</span> <span class="pi">]</span>
                <span class="na">generation</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">observedGeneration</span> <span class="pi">]</span>
                <span class="na">username</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">username</span> <span class="pi">]</span>
        <span class="pi">-</span> <span class="na">groupVersionKind</span><span class="pi">:</span>
            <span class="na">group</span><span class="pi">:</span> <span class="s">kafka.strimzi.io</span>
            <span class="na">version</span><span class="pi">:</span> <span class="s">v1beta2</span>
            <span class="na">kind</span><span class="pi">:</span> <span class="s">Kafka</span>
          <span class="na">metricNamePrefix</span><span class="pi">:</span> <span class="s">strimzi_kafka</span>
          <span class="na">metrics</span><span class="pi">:</span>
            <span class="pi">-</span> <span class="na">name</span><span class="pi">:</span> <span class="s">resource_info</span>
              <span class="na">help</span><span class="pi">:</span> <span class="s2">"</span><span class="s">The</span><span class="nv"> </span><span class="s">current</span><span class="nv"> </span><span class="s">state</span><span class="nv"> </span><span class="s">of</span><span class="nv"> </span><span class="s">a</span><span class="nv"> </span><span class="s">Strimzi</span><span class="nv"> </span><span class="s">Kafka</span><span class="nv"> </span><span class="s">resource."</span>
              <span class="na">each</span><span class="pi">:</span>
                <span class="na">type</span><span class="pi">:</span> <span class="s">Info</span>
                <span class="na">info</span><span class="pi">:</span>
                  <span class="na">labelsFromPath</span><span class="pi">:</span>
                    <span class="na">name</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">metadata</span><span class="pi">,</span> <span class="nv">name</span> <span class="pi">]</span>
              <span class="na">labelsFromPath</span><span class="pi">:</span>
                <span class="na">exported_namespace</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">metadata</span><span class="pi">,</span> <span class="nv">namespace</span> <span class="pi">]</span>
                <span class="na">ready</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">conditions</span><span class="pi">,</span> <span class="s2">"</span><span class="s">[type=Ready]"</span><span class="pi">,</span> <span class="nv">status</span> <span class="pi">]</span>
                <span class="na">deprecated</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">conditions</span><span class="pi">,</span> <span class="s2">"</span><span class="s">[reason=DeprecatedFields]"</span><span class="pi">,</span> <span class="nv">type</span> <span class="pi">]</span>
                <span class="na">kafka_version</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">kafkaVersion</span> <span class="pi">]</span>
                <span class="na">kafka_metadata_state</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">kafkaMetadataState</span> <span class="pi">]</span>
                <span class="na">kafka_metadata_version</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">kafkaMetadataVersion</span> <span class="pi">]</span>
                <span class="na">cluster_id</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">clusterId</span> <span class="pi">]</span>
                <span class="na">operator_last_successful_version</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">operatorLastSuccessfulVersion</span> <span class="pi">]</span>
                <span class="na">generation</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">observedGeneration</span> <span class="pi">]</span>
        <span class="pi">-</span> <span class="na">groupVersionKind</span><span class="pi">:</span>
            <span class="na">group</span><span class="pi">:</span> <span class="s">kafka.strimzi.io</span>
            <span class="na">version</span><span class="pi">:</span> <span class="s">v1beta2</span>
            <span class="na">kind</span><span class="pi">:</span> <span class="s">KafkaNodePool</span>
          <span class="na">metricNamePrefix</span><span class="pi">:</span> <span class="s">strimzi_kafka_node_pool</span>
          <span class="na">metrics</span><span class="pi">:</span>
            <span class="pi">-</span> <span class="na">name</span><span class="pi">:</span> <span class="s">resource_info</span>
              <span class="na">help</span><span class="pi">:</span> <span class="s2">"</span><span class="s">The</span><span class="nv"> </span><span class="s">current</span><span class="nv"> </span><span class="s">state</span><span class="nv"> </span><span class="s">of</span><span class="nv"> </span><span class="s">a</span><span class="nv"> </span><span class="s">Strimzi</span><span class="nv"> </span><span class="s">Kafka</span><span class="nv"> </span><span class="s">node</span><span class="nv"> </span><span class="s">pool</span><span class="nv"> </span><span class="s">resource."</span>
              <span class="na">each</span><span class="pi">:</span>
                <span class="na">type</span><span class="pi">:</span> <span class="s">Info</span>
                <span class="na">info</span><span class="pi">:</span>
                  <span class="na">labelsFromPath</span><span class="pi">:</span>
                    <span class="na">name</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">metadata</span><span class="pi">,</span> <span class="nv">name</span> <span class="pi">]</span>
              <span class="na">labelsFromPath</span><span class="pi">:</span>
                <span class="na">exported_namespace</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">metadata</span><span class="pi">,</span> <span class="nv">namespace</span> <span class="pi">]</span>
                <span class="c1"># KafkaNodePool is not having a ready status as this is implemented via Kafka resource</span>
                <span class="c1"># ready: [ status, conditions, "[type=Ready]", status ]</span>
                <span class="na">deprecated</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">conditions</span><span class="pi">,</span> <span class="s2">"</span><span class="s">[reason=DeprecatedFields]"</span><span class="pi">,</span> <span class="nv">type</span> <span class="pi">]</span>
                <span class="na">node_ids</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">nodeIds</span> <span class="pi">]</span>
                <span class="na">roles</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">roles</span> <span class="pi">]</span>
                <span class="na">replicas</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">replicas</span> <span class="pi">]</span>
                <span class="na">cluster_id</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">clusterId</span> <span class="pi">]</span>
                <span class="na">generation</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">observedGeneration</span> <span class="pi">]</span>
        <span class="pi">-</span> <span class="na">groupVersionKind</span><span class="pi">:</span>
            <span class="na">group</span><span class="pi">:</span> <span class="s">core.strimzi.io</span>
            <span class="na">version</span><span class="pi">:</span> <span class="s">v1beta2</span>
            <span class="na">kind</span><span class="pi">:</span> <span class="s">StrimziPodSet</span>
          <span class="na">metricNamePrefix</span><span class="pi">:</span> <span class="s">strimzi_pod_set</span>
          <span class="na">metrics</span><span class="pi">:</span>
            <span class="pi">-</span> <span class="na">name</span><span class="pi">:</span> <span class="s">resource_info</span>
              <span class="na">help</span><span class="pi">:</span> <span class="s2">"</span><span class="s">The</span><span class="nv"> </span><span class="s">current</span><span class="nv"> </span><span class="s">state</span><span class="nv"> </span><span class="s">of</span><span class="nv"> </span><span class="s">a</span><span class="nv"> </span><span class="s">Strimzi</span><span class="nv"> </span><span class="s">pod</span><span class="nv"> </span><span class="s">set</span><span class="nv"> </span><span class="s">resource."</span>
              <span class="na">each</span><span class="pi">:</span>
                <span class="na">type</span><span class="pi">:</span> <span class="s">Info</span>
                <span class="na">info</span><span class="pi">:</span>
                  <span class="na">labelsFromPath</span><span class="pi">:</span>
                    <span class="na">name</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">metadata</span><span class="pi">,</span> <span class="nv">name</span> <span class="pi">]</span>
              <span class="na">labelsFromPath</span><span class="pi">:</span>
                <span class="na">exported_namespace</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">metadata</span><span class="pi">,</span> <span class="nv">namespace</span> <span class="pi">]</span>
                <span class="na">deprecated</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">conditions</span><span class="pi">,</span> <span class="s2">"</span><span class="s">[reason=DeprecatedFields]"</span><span class="pi">,</span> <span class="nv">type</span> <span class="pi">]</span>
                <span class="na">currentPods</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">currentPods</span> <span class="pi">]</span>
                <span class="na">pods</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">pods</span> <span class="pi">]</span>
                <span class="na">readyPods</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">readyPods</span> <span class="pi">]</span>
                <span class="na">generation</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">observedGeneration</span> <span class="pi">]</span>
        <span class="pi">-</span> <span class="na">groupVersionKind</span><span class="pi">:</span>
            <span class="na">group</span><span class="pi">:</span> <span class="s">kafka.strimzi.io</span>
            <span class="na">version</span><span class="pi">:</span> <span class="s">v1beta2</span>
            <span class="na">kind</span><span class="pi">:</span> <span class="s">KafkaRebalance</span>
          <span class="na">metricNamePrefix</span><span class="pi">:</span> <span class="s">strimzi_kafka_rebalance</span>
          <span class="na">metrics</span><span class="pi">:</span>
            <span class="pi">-</span> <span class="na">name</span><span class="pi">:</span> <span class="s">resource_info</span>
              <span class="na">help</span><span class="pi">:</span> <span class="s2">"</span><span class="s">The</span><span class="nv"> </span><span class="s">current</span><span class="nv"> </span><span class="s">state</span><span class="nv"> </span><span class="s">of</span><span class="nv"> </span><span class="s">a</span><span class="nv"> </span><span class="s">Strimzi</span><span class="nv"> </span><span class="s">kafka</span><span class="nv"> </span><span class="s">rebalance</span><span class="nv"> </span><span class="s">resource."</span>
              <span class="na">each</span><span class="pi">:</span>
                <span class="na">type</span><span class="pi">:</span> <span class="s">Info</span>
                <span class="na">info</span><span class="pi">:</span>
                  <span class="na">labelsFromPath</span><span class="pi">:</span>
                    <span class="na">name</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">metadata</span><span class="pi">,</span> <span class="nv">name</span> <span class="pi">]</span>
              <span class="na">labelsFromPath</span><span class="pi">:</span>
                <span class="na">exported_namespace</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">metadata</span><span class="pi">,</span> <span class="nv">namespace</span> <span class="pi">]</span>
                <span class="na">ready</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">conditions</span><span class="pi">,</span> <span class="s2">"</span><span class="s">[type=Ready]"</span><span class="pi">,</span> <span class="nv">status</span> <span class="pi">]</span>
                <span class="na">proposal_ready</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">conditions</span><span class="pi">,</span> <span class="s2">"</span><span class="s">[type=ProposalReady]"</span><span class="pi">,</span> <span class="nv">status</span> <span class="pi">]</span>
                <span class="na">rebalancing</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">conditions</span><span class="pi">,</span> <span class="s2">"</span><span class="s">[type=Rebalancing]"</span><span class="pi">,</span> <span class="nv">status</span> <span class="pi">]</span>
                <span class="na">deprecated</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">conditions</span><span class="pi">,</span> <span class="s2">"</span><span class="s">[reason=DeprecatedFields]"</span><span class="pi">,</span> <span class="nv">type</span> <span class="pi">]</span>
                <span class="na">template</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">metadata</span><span class="pi">,</span> <span class="nv">annotations</span><span class="pi">,</span> <span class="s2">"</span><span class="s">strimzi.io/rebalance-template"</span> <span class="pi">]</span>
        <span class="pi">-</span> <span class="na">groupVersionKind</span><span class="pi">:</span>
            <span class="na">group</span><span class="pi">:</span> <span class="s">kafka.strimzi.io</span>
            <span class="na">version</span><span class="pi">:</span> <span class="s">v1beta2</span>
            <span class="na">kind</span><span class="pi">:</span> <span class="s">KafkaConnect</span>
          <span class="na">metricNamePrefix</span><span class="pi">:</span> <span class="s">strimzi_kafka_connect</span>
          <span class="na">metrics</span><span class="pi">:</span>
            <span class="pi">-</span> <span class="na">name</span><span class="pi">:</span> <span class="s">resource_info</span>
              <span class="na">help</span><span class="pi">:</span> <span class="s2">"</span><span class="s">The</span><span class="nv"> </span><span class="s">current</span><span class="nv"> </span><span class="s">state</span><span class="nv"> </span><span class="s">of</span><span class="nv"> </span><span class="s">a</span><span class="nv"> </span><span class="s">Strimzi</span><span class="nv"> </span><span class="s">Kafka</span><span class="nv"> </span><span class="s">Connect</span><span class="nv"> </span><span class="s">resource."</span>
              <span class="na">each</span><span class="pi">:</span>
                <span class="na">type</span><span class="pi">:</span> <span class="s">Info</span>
                <span class="na">info</span><span class="pi">:</span>
                  <span class="na">labelsFromPath</span><span class="pi">:</span>
                    <span class="na">name</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">metadata</span><span class="pi">,</span> <span class="nv">name</span> <span class="pi">]</span>
              <span class="na">labelsFromPath</span><span class="pi">:</span>
                <span class="na">exported_namespace</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">metadata</span><span class="pi">,</span> <span class="nv">namespace</span> <span class="pi">]</span>
                <span class="na">deprecated</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">conditions</span><span class="pi">,</span> <span class="s2">"</span><span class="s">[reason=DeprecatedFields]"</span><span class="pi">,</span> <span class="nv">type</span> <span class="pi">]</span>
                <span class="na">ready</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">conditions</span><span class="pi">,</span> <span class="s2">"</span><span class="s">[type=Ready]"</span><span class="pi">,</span> <span class="nv">status</span> <span class="pi">]</span>
                <span class="na">generation</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">observedGeneration</span> <span class="pi">]</span>
                <span class="na">connectorPluginsClass</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">connectorPlugins</span><span class="pi">,</span> <span class="nv">class</span> <span class="pi">]</span>
                <span class="na">connectorPluginsType</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">connectorPlugins</span><span class="pi">,</span> <span class="nv">type</span> <span class="pi">]</span>
                <span class="na">connectorPluginsVersion</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">connectorPlugins</span><span class="pi">,</span> <span class="nv">version</span> <span class="pi">]</span>
                <span class="na">replicas</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">replicas</span> <span class="pi">]</span>
                <span class="na">labelSelector</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">labelSelector</span> <span class="pi">]</span>
        <span class="pi">-</span> <span class="na">groupVersionKind</span><span class="pi">:</span>
            <span class="na">group</span><span class="pi">:</span> <span class="s">kafka.strimzi.io</span>
            <span class="na">version</span><span class="pi">:</span> <span class="s">v1beta2</span>
            <span class="na">kind</span><span class="pi">:</span> <span class="s">KafkaConnector</span>
          <span class="na">metricNamePrefix</span><span class="pi">:</span> <span class="s">strimzi_kafka_connector</span>
          <span class="na">metrics</span><span class="pi">:</span>
            <span class="pi">-</span> <span class="na">name</span><span class="pi">:</span> <span class="s">resource_info</span>
              <span class="na">help</span><span class="pi">:</span> <span class="s2">"</span><span class="s">The</span><span class="nv"> </span><span class="s">current</span><span class="nv"> </span><span class="s">state</span><span class="nv"> </span><span class="s">of</span><span class="nv"> </span><span class="s">a</span><span class="nv"> </span><span class="s">Strimzi</span><span class="nv"> </span><span class="s">Kafka</span><span class="nv"> </span><span class="s">Connector</span><span class="nv"> </span><span class="s">resource."</span>
              <span class="na">each</span><span class="pi">:</span>
                <span class="na">type</span><span class="pi">:</span> <span class="s">Info</span>
                <span class="na">info</span><span class="pi">:</span>
                  <span class="na">labelsFromPath</span><span class="pi">:</span>
                    <span class="na">name</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">metadata</span><span class="pi">,</span> <span class="nv">name</span> <span class="pi">]</span>
              <span class="na">labelsFromPath</span><span class="pi">:</span>
                <span class="na">exported_namespace</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">metadata</span><span class="pi">,</span> <span class="nv">namespace</span> <span class="pi">]</span>
                <span class="na">deprecated</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">conditions</span><span class="pi">,</span> <span class="s2">"</span><span class="s">[reason=DeprecatedFields]"</span><span class="pi">,</span> <span class="nv">type</span> <span class="pi">]</span>
                <span class="na">ready</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">conditions</span><span class="pi">,</span> <span class="s2">"</span><span class="s">[type=Ready]"</span><span class="pi">,</span> <span class="nv">status</span> <span class="pi">]</span>
                <span class="na">generation</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">observedGeneration</span> <span class="pi">]</span>
                <span class="na">autoRestartCount</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">autoRestart</span><span class="pi">,</span> <span class="nv">count</span> <span class="pi">]</span>
                <span class="na">autoRestartConnectorName</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">autoRestart</span><span class="pi">,</span> <span class="nv">connectorName</span> <span class="pi">]</span>
                <span class="na">tasksMax</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">tasksMax</span> <span class="pi">]</span>
                <span class="na">topics</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">topics</span> <span class="pi">]</span>
        <span class="pi">-</span> <span class="na">groupVersionKind</span><span class="pi">:</span>
            <span class="na">group</span><span class="pi">:</span> <span class="s">kafka.strimzi.io</span>
            <span class="na">version</span><span class="pi">:</span> <span class="s">v1beta2</span>
            <span class="na">kind</span><span class="pi">:</span> <span class="s">KafkaMirrorMaker2</span>
          <span class="na">metricNamePrefix</span><span class="pi">:</span> <span class="s">strimzi_kafka_mm2</span>
          <span class="na">metrics</span><span class="pi">:</span>
            <span class="pi">-</span> <span class="na">name</span><span class="pi">:</span> <span class="s">resource_info</span>
              <span class="na">help</span><span class="pi">:</span> <span class="s2">"</span><span class="s">The</span><span class="nv"> </span><span class="s">current</span><span class="nv"> </span><span class="s">state</span><span class="nv"> </span><span class="s">of</span><span class="nv"> </span><span class="s">a</span><span class="nv"> </span><span class="s">Strimzi</span><span class="nv"> </span><span class="s">Kafka</span><span class="nv"> </span><span class="s">MirrorMaker2</span><span class="nv"> </span><span class="s">resource."</span>
              <span class="na">each</span><span class="pi">:</span>
                <span class="na">type</span><span class="pi">:</span> <span class="s">Info</span>
                <span class="na">info</span><span class="pi">:</span>
                  <span class="na">labelsFromPath</span><span class="pi">:</span>
                    <span class="na">name</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">metadata</span><span class="pi">,</span> <span class="nv">name</span> <span class="pi">]</span>
              <span class="na">labelsFromPath</span><span class="pi">:</span>
                <span class="na">exported_namespace</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">metadata</span><span class="pi">,</span> <span class="nv">namespace</span> <span class="pi">]</span>
                <span class="na">deprecated</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">conditions</span><span class="pi">,</span> <span class="s2">"</span><span class="s">[reason=DeprecatedFields]"</span><span class="pi">,</span> <span class="nv">type</span> <span class="pi">]</span>
                <span class="na">ready</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">conditions</span><span class="pi">,</span> <span class="s2">"</span><span class="s">[type=Ready]"</span><span class="pi">,</span> <span class="nv">status</span> <span class="pi">]</span>
                <span class="na">generation</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">observedGeneration</span> <span class="pi">]</span>
                <span class="na">autoRestartCount</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">autoRestartStatuses</span><span class="pi">,</span> <span class="nv">count</span> <span class="pi">]</span>
                <span class="na">autoRestartConnectorName</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">autoRestartStatuses</span><span class="pi">,</span> <span class="nv">connectorName</span> <span class="pi">]</span>
                <span class="na">connectorPluginsClass</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">connectorPlugins</span><span class="pi">,</span> <span class="nv">class</span> <span class="pi">]</span>
                <span class="na">connectorPluginsType</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">connectorPlugins</span><span class="pi">,</span> <span class="nv">type</span> <span class="pi">]</span>
                <span class="na">connectorPluginsVersion</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">connectorPlugins</span><span class="pi">,</span> <span class="nv">version</span> <span class="pi">]</span>
                <span class="na">labelSelector</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">labelSelector</span> <span class="pi">]</span>
                <span class="na">replicas</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">replicas</span> <span class="pi">]</span>
        <span class="pi">-</span> <span class="na">groupVersionKind</span><span class="pi">:</span>
            <span class="na">group</span><span class="pi">:</span> <span class="s">access.strimzi.io</span>
            <span class="na">version</span><span class="pi">:</span> <span class="s">v1alpha1</span>
            <span class="na">kind</span><span class="pi">:</span> <span class="s">KafkaAccess</span>
          <span class="na">metricNamePrefix</span><span class="pi">:</span> <span class="s">strimzi_kafka_access</span>
          <span class="na">metrics</span><span class="pi">:</span>
            <span class="pi">-</span> <span class="na">name</span><span class="pi">:</span> <span class="s">resource_info</span>
              <span class="na">help</span><span class="pi">:</span> <span class="s2">"</span><span class="s">The</span><span class="nv"> </span><span class="s">current</span><span class="nv"> </span><span class="s">state</span><span class="nv"> </span><span class="s">of</span><span class="nv"> </span><span class="s">a</span><span class="nv"> </span><span class="s">Strimzi</span><span class="nv"> </span><span class="s">Kafka</span><span class="nv"> </span><span class="s">Access</span><span class="nv"> </span><span class="s">resource."</span>
              <span class="na">each</span><span class="pi">:</span>
                <span class="na">type</span><span class="pi">:</span> <span class="s">Info</span>
                <span class="na">info</span><span class="pi">:</span>
                  <span class="na">labelsFromPath</span><span class="pi">:</span>
                    <span class="na">name</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">metadata</span><span class="pi">,</span> <span class="nv">name</span> <span class="pi">]</span>
              <span class="na">labelsFromPath</span><span class="pi">:</span>
                <span class="na">exported_namespace</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">metadata</span><span class="pi">,</span> <span class="nv">namespace</span> <span class="pi">]</span>
                <span class="na">ready</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">conditions</span><span class="pi">,</span> <span class="s2">"</span><span class="s">[type=Ready]"</span><span class="pi">,</span> <span class="nv">status</span> <span class="pi">]</span>
                <span class="na">deprecated</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">conditions</span><span class="pi">,</span> <span class="s2">"</span><span class="s">[reason=DeprecatedFields]"</span><span class="pi">,</span> <span class="nv">type</span> <span class="pi">]</span>
                <span class="na">generation</span><span class="pi">:</span> <span class="pi">[</span> <span class="nv">status</span><span class="pi">,</span> <span class="nv">observedGeneration</span> <span class="pi">]</span>
<span class="c1"># Warning: Helm interpretes `{{}}`, so this needs to be escaped</span>
<span class="na">extraManifests</span><span class="pi">:</span>
  <span class="pi">-</span> <span class="na">apiVersion</span><span class="pi">:</span> <span class="s">monitoring.coreos.com/v1</span>
    <span class="na">kind</span><span class="pi">:</span> <span class="s">PrometheusRule</span>
    <span class="na">metadata</span><span class="pi">:</span>
      <span class="na">name</span><span class="pi">:</span> <span class="s">strimzi-kube-state-metrics</span>
    <span class="na">spec</span><span class="pi">:</span>
      <span class="na">groups</span><span class="pi">:</span>
        <span class="pi">-</span> <span class="na">name</span><span class="pi">:</span> <span class="s">strimzi-kube-state-metrics</span>
          <span class="na">rules</span><span class="pi">:</span>
            <span class="pi">-</span> <span class="na">alert</span><span class="pi">:</span> <span class="s">KafkaTopicNotReady</span>
              <span class="na">expr</span><span class="pi">:</span> <span class="s">strimzi_kafka_topic_resource_info{ready!="True"}</span>
              <span class="na">for</span><span class="pi">:</span> <span class="s">15m</span>
              <span class="na">labels</span><span class="pi">:</span>
                <span class="na">severity</span><span class="pi">:</span> <span class="s">warning</span>
              <span class="na">annotations</span><span class="pi">:</span>
                <span class="na">message</span><span class="pi">:</span> <span class="s2">"</span><span class="s">Strimzi</span><span class="nv"> </span><span class="s">KafkaTopic</span><span class="nv"> </span><span class="s">{{`{{</span><span class="nv"> </span><span class="s">$labels.topicName</span><span class="nv"> </span><span class="s">}}`}}</span><span class="nv"> </span><span class="s">is</span><span class="nv"> </span><span class="s">not</span><span class="nv"> </span><span class="s">ready"</span>
            <span class="pi">-</span> <span class="na">alert</span><span class="pi">:</span> <span class="s">KafkaTopicDeprecated</span>
              <span class="na">expr</span><span class="pi">:</span> <span class="s">strimzi_kafka_topic_resource_info{deprecated="Warning"}</span>
              <span class="na">for</span><span class="pi">:</span> <span class="s">15m</span>
              <span class="na">labels</span><span class="pi">:</span>
                <span class="na">severity</span><span class="pi">:</span> <span class="s">warning</span>
              <span class="na">annotations</span><span class="pi">:</span>
                <span class="na">message</span><span class="pi">:</span> <span class="s2">"</span><span class="s">Strimzi</span><span class="nv"> </span><span class="s">KafkaTopic</span><span class="nv"> </span><span class="s">{{`{{</span><span class="nv"> </span><span class="s">$labels.topicName</span><span class="nv"> </span><span class="s">}}`}}</span><span class="nv"> </span><span class="s">contains</span><span class="nv"> </span><span class="s">a</span><span class="nv"> </span><span class="s">deprecated</span><span class="nv"> </span><span class="s">configuration"</span>
            <span class="pi">-</span> <span class="na">alert</span><span class="pi">:</span> <span class="s">KafkaUserNotReady</span>
              <span class="na">expr</span><span class="pi">:</span> <span class="s">strimzi_kafka_user_resource_info{ready!="True"}</span>
              <span class="na">for</span><span class="pi">:</span> <span class="s">15m</span>
              <span class="na">labels</span><span class="pi">:</span>
                <span class="na">severity</span><span class="pi">:</span> <span class="s">warning</span>
              <span class="na">annotations</span><span class="pi">:</span>
                <span class="na">message</span><span class="pi">:</span> <span class="s2">"</span><span class="s">Strimzi</span><span class="nv"> </span><span class="s">KafkaUser</span><span class="nv"> </span><span class="s">{{`{{</span><span class="nv"> </span><span class="s">$labels.username</span><span class="nv"> </span><span class="s">}}`}}</span><span class="nv"> </span><span class="s">is</span><span class="nv"> </span><span class="s">not</span><span class="nv"> </span><span class="s">ready"</span>
            <span class="pi">-</span> <span class="na">alert</span><span class="pi">:</span> <span class="s">KafkaUserDeprecated</span>
              <span class="na">expr</span><span class="pi">:</span> <span class="s">strimzi_kafka_user_resource_info{deprecated="Warning"}</span>
              <span class="na">for</span><span class="pi">:</span> <span class="s">15m</span>
              <span class="na">labels</span><span class="pi">:</span>
                <span class="na">severity</span><span class="pi">:</span> <span class="s">warning</span>
              <span class="na">annotations</span><span class="pi">:</span>
                <span class="na">message</span><span class="pi">:</span> <span class="s2">"</span><span class="s">Strimzi</span><span class="nv"> </span><span class="s">KafkaUser</span><span class="nv"> </span><span class="s">{{`{{</span><span class="nv"> </span><span class="s">$labels.username</span><span class="nv"> </span><span class="s">}}`}}</span><span class="nv"> </span><span class="s">contains</span><span class="nv"> </span><span class="s">a</span><span class="nv"> </span><span class="s">deprecated</span><span class="nv"> </span><span class="s">configuration"</span>
            <span class="pi">-</span> <span class="na">alert</span><span class="pi">:</span> <span class="s">KafkaNotReady</span>
              <span class="na">expr</span><span class="pi">:</span> <span class="s">strimzi_kafka_resource_info{ready!="True"}</span>
              <span class="na">for</span><span class="pi">:</span> <span class="s">15m</span>
              <span class="na">labels</span><span class="pi">:</span>
                <span class="na">severity</span><span class="pi">:</span> <span class="s">warning</span>
              <span class="na">annotations</span><span class="pi">:</span>
                <span class="na">message</span><span class="pi">:</span> <span class="s2">"</span><span class="s">Strimzi</span><span class="nv"> </span><span class="s">Kafka</span><span class="nv"> </span><span class="s">{{`{{</span><span class="nv"> </span><span class="s">$labels.name</span><span class="nv"> </span><span class="s">}}`}}</span><span class="nv"> </span><span class="s">using</span><span class="nv"> </span><span class="s">{{`{{</span><span class="nv"> </span><span class="s">$labels.kafka_version</span><span class="nv"> </span><span class="s">}}`}}</span><span class="nv"> </span><span class="s">is</span><span class="nv"> </span><span class="s">not</span><span class="nv"> </span><span class="s">ready"</span>
            <span class="pi">-</span> <span class="na">alert</span><span class="pi">:</span> <span class="s">KafkaDeprecated</span>
              <span class="na">expr</span><span class="pi">:</span> <span class="s">strimzi_kafka_resource_info{deprecated="Warning"}</span>
              <span class="na">for</span><span class="pi">:</span> <span class="s">15m</span>
              <span class="na">labels</span><span class="pi">:</span>
                <span class="na">severity</span><span class="pi">:</span> <span class="s">warning</span>
              <span class="na">annotations</span><span class="pi">:</span>
                <span class="na">message</span><span class="pi">:</span> <span class="s2">"</span><span class="s">Strimzi</span><span class="nv"> </span><span class="s">Kafka</span><span class="nv"> </span><span class="s">{{`{{</span><span class="nv"> </span><span class="s">$labels.name</span><span class="nv"> </span><span class="s">}}`}}</span><span class="nv"> </span><span class="s">contains</span><span class="nv"> </span><span class="s">a</span><span class="nv"> </span><span class="s">deprecated</span><span class="nv"> </span><span class="s">configuration"</span>
            <span class="c1"># KafkaNodePool is not having a ready status as this is implemented via Kafka resource</span>
            <span class="pi">-</span> <span class="na">alert</span><span class="pi">:</span> <span class="s">KafkaNodePoolDeprecated</span>
              <span class="na">expr</span><span class="pi">:</span> <span class="s">strimzi_kafka_node_pool_resource_info{deprecated="Warning"}</span>
              <span class="na">for</span><span class="pi">:</span> <span class="s">15m</span>
              <span class="na">labels</span><span class="pi">:</span>
                <span class="na">severity</span><span class="pi">:</span> <span class="s">warning</span>
              <span class="na">annotations</span><span class="pi">:</span>
                <span class="na">message</span><span class="pi">:</span> <span class="s2">"</span><span class="s">Strimzi</span><span class="nv"> </span><span class="s">KafkaNodePool</span><span class="nv"> </span><span class="s">{{`{{</span><span class="nv"> </span><span class="s">$labels.name</span><span class="nv"> </span><span class="s">}}`}}</span><span class="nv"> </span><span class="s">contains</span><span class="nv"> </span><span class="s">a</span><span class="nv"> </span><span class="s">deprecated</span><span class="nv"> </span><span class="s">configuration"</span>
            <span class="c1"># StrimziPodSet is not having any further information as it is an internal resource and doesn't get operated by the user</span>
            <span class="pi">-</span> <span class="na">alert</span><span class="pi">:</span> <span class="s">KafkaRebalanceNotReady</span>
              <span class="na">expr</span><span class="pi">:</span> <span class="s">strimzi_kafka_rebalance_resource_info{ready!="True",template!="true"}</span>
              <span class="na">for</span><span class="pi">:</span> <span class="s">15m</span>
              <span class="na">labels</span><span class="pi">:</span>
                <span class="na">severity</span><span class="pi">:</span> <span class="s">warning</span>
              <span class="na">annotations</span><span class="pi">:</span>
                <span class="na">message</span><span class="pi">:</span> <span class="s2">"</span><span class="s">Strimzi</span><span class="nv"> </span><span class="s">KafkaRebalance</span><span class="nv"> </span><span class="s">{{`{{</span><span class="nv"> </span><span class="s">$labels.name</span><span class="nv"> </span><span class="s">}}`}}</span><span class="nv"> </span><span class="s">is</span><span class="nv"> </span><span class="s">not</span><span class="nv"> </span><span class="s">ready"</span>
            <span class="pi">-</span> <span class="na">alert</span><span class="pi">:</span> <span class="s">KafkaRebalanceProposalPending</span>
              <span class="na">expr</span><span class="pi">:</span> <span class="s">strimzi_kafka_rebalance_resource_info{ready="True",template!="true",proposal_ready="True"}</span>
              <span class="na">for</span><span class="pi">:</span> <span class="s">1h</span>
              <span class="na">labels</span><span class="pi">:</span>
                <span class="na">severity</span><span class="pi">:</span> <span class="s">warning</span>
              <span class="na">annotations</span><span class="pi">:</span>
                <span class="na">message</span><span class="pi">:</span> <span class="s2">"</span><span class="s">Strimzi</span><span class="nv"> </span><span class="s">KafkaRebalance</span><span class="nv"> </span><span class="s">{{`{{</span><span class="nv"> </span><span class="s">$labels.name</span><span class="nv"> </span><span class="s">}}`}}</span><span class="nv"> </span><span class="s">is</span><span class="nv"> </span><span class="s">in</span><span class="nv"> </span><span class="s">proposal</span><span class="nv"> </span><span class="s">pending</span><span class="nv"> </span><span class="s">state</span><span class="nv"> </span><span class="s">and</span><span class="nv"> </span><span class="s">waits</span><span class="nv"> </span><span class="s">for</span><span class="nv"> </span><span class="s">approval</span><span class="nv"> </span><span class="s">for</span><span class="nv"> </span><span class="s">more</span><span class="nv"> </span><span class="s">than</span><span class="nv"> </span><span class="s">1h."</span>
            <span class="pi">-</span> <span class="na">alert</span><span class="pi">:</span> <span class="s">KafkaRebalanceRebalancing</span>
              <span class="na">expr</span><span class="pi">:</span> <span class="s">strimzi_kafka_rebalance_resource_info{ready="True",template!="true",rebalancing="True"}</span>
              <span class="na">for</span><span class="pi">:</span> <span class="s">1h</span>
              <span class="na">labels</span><span class="pi">:</span>
                <span class="na">severity</span><span class="pi">:</span> <span class="s">info</span>
              <span class="na">annotations</span><span class="pi">:</span>
                <span class="na">message</span><span class="pi">:</span> <span class="s2">"</span><span class="s">Strimzi</span><span class="nv"> </span><span class="s">KafkaRebalance</span><span class="nv"> </span><span class="s">{{`{{</span><span class="nv"> </span><span class="s">$labels.name</span><span class="nv"> </span><span class="s">}}`}}</span><span class="nv"> </span><span class="s">is</span><span class="nv"> </span><span class="s">taking</span><span class="nv"> </span><span class="s">longer</span><span class="nv"> </span><span class="s">than</span><span class="nv"> </span><span class="s">1h."</span>
            <span class="pi">-</span> <span class="na">alert</span><span class="pi">:</span> <span class="s">KafkaRebalanceDeprecated</span>
              <span class="na">expr</span><span class="pi">:</span> <span class="s">strimzi_kafka_rebalance_resource_info{deprecated="Warning"}</span>
              <span class="na">for</span><span class="pi">:</span> <span class="s">15m</span>
              <span class="na">labels</span><span class="pi">:</span>
                <span class="na">severity</span><span class="pi">:</span> <span class="s">warning</span>
              <span class="na">annotations</span><span class="pi">:</span>
                <span class="na">message</span><span class="pi">:</span> <span class="s2">"</span><span class="s">Strimzi</span><span class="nv"> </span><span class="s">KafkaRebalance</span><span class="nv"> </span><span class="s">{{`{{</span><span class="nv"> </span><span class="s">$labels.name</span><span class="nv"> </span><span class="s">}}`}}</span><span class="nv"> </span><span class="s">contains</span><span class="nv"> </span><span class="s">a</span><span class="nv"> </span><span class="s">deprecated</span><span class="nv"> </span><span class="s">configuration"</span>
            <span class="pi">-</span> <span class="na">alert</span><span class="pi">:</span> <span class="s">KafkaConnectNotReady</span>
              <span class="na">expr</span><span class="pi">:</span> <span class="s">strimzi_kafka_connect_resource_info{ready!="True"}</span>
              <span class="na">for</span><span class="pi">:</span> <span class="s">15m</span>
              <span class="na">labels</span><span class="pi">:</span>
                <span class="na">severity</span><span class="pi">:</span> <span class="s">warning</span>
              <span class="na">annotations</span><span class="pi">:</span>
                <span class="na">message</span><span class="pi">:</span> <span class="s2">"</span><span class="s">Strimzi</span><span class="nv"> </span><span class="s">KafkaConnect</span><span class="nv"> </span><span class="s">{{`{{</span><span class="nv"> </span><span class="s">$labels.name</span><span class="nv"> </span><span class="s">}}`}}</span><span class="nv"> </span><span class="s">is</span><span class="nv"> </span><span class="s">not</span><span class="nv"> </span><span class="s">ready"</span>
            <span class="pi">-</span> <span class="na">alert</span><span class="pi">:</span> <span class="s">KafkaConnectDeprecated</span>
              <span class="na">expr</span><span class="pi">:</span> <span class="s">strimzi_kafka_connect_resource_info{deprecated="Warning"}</span>
              <span class="na">for</span><span class="pi">:</span> <span class="s">15m</span>
              <span class="na">labels</span><span class="pi">:</span>
                <span class="na">severity</span><span class="pi">:</span> <span class="s">warning</span>
              <span class="na">annotations</span><span class="pi">:</span>
                <span class="na">message</span><span class="pi">:</span> <span class="s2">"</span><span class="s">Strimzi</span><span class="nv"> </span><span class="s">KafkaConnect</span><span class="nv"> </span><span class="s">{{`{{</span><span class="nv"> </span><span class="s">$labels.name</span><span class="nv"> </span><span class="s">}}`}}</span><span class="nv"> </span><span class="s">contains</span><span class="nv"> </span><span class="s">a</span><span class="nv"> </span><span class="s">deprecated</span><span class="nv"> </span><span class="s">configuration"</span>
            <span class="pi">-</span> <span class="na">alert</span><span class="pi">:</span> <span class="s">KafkaConnectorNotReady</span>
              <span class="na">expr</span><span class="pi">:</span> <span class="s">strimzi_kafka_connector_resource_info{ready!="True"}</span>
              <span class="na">for</span><span class="pi">:</span> <span class="s">15m</span>
              <span class="na">labels</span><span class="pi">:</span>
                <span class="na">severity</span><span class="pi">:</span> <span class="s">warning</span>
              <span class="na">annotations</span><span class="pi">:</span>
                <span class="na">message</span><span class="pi">:</span> <span class="s2">"</span><span class="s">Strimzi</span><span class="nv"> </span><span class="s">KafkaConnector</span><span class="nv"> </span><span class="s">{{`{{</span><span class="nv"> </span><span class="s">$labels.name</span><span class="nv"> </span><span class="s">}}`}}</span><span class="nv"> </span><span class="s">is</span><span class="nv"> </span><span class="s">not</span><span class="nv"> </span><span class="s">ready"</span>
            <span class="pi">-</span> <span class="na">alert</span><span class="pi">:</span> <span class="s">KafkaConnectorDeprecated</span>
              <span class="na">expr</span><span class="pi">:</span> <span class="s">strimzi_kafka_connector_resource_info{deprecated="Warning"}</span>
              <span class="na">for</span><span class="pi">:</span> <span class="s">15m</span>
              <span class="na">labels</span><span class="pi">:</span>
                <span class="na">severity</span><span class="pi">:</span> <span class="s">warning</span>
              <span class="na">annotations</span><span class="pi">:</span>
                <span class="na">message</span><span class="pi">:</span> <span class="s2">"</span><span class="s">Strimzi</span><span class="nv"> </span><span class="s">KafkaConnector</span><span class="nv"> </span><span class="s">{{`{{</span><span class="nv"> </span><span class="s">$labels.name</span><span class="nv"> </span><span class="s">}}`}}</span><span class="nv"> </span><span class="s">contains</span><span class="nv"> </span><span class="s">a</span><span class="nv"> </span><span class="s">deprecated</span><span class="nv"> </span><span class="s">configuration"</span>
            <span class="pi">-</span> <span class="na">alert</span><span class="pi">:</span> <span class="s">KafkaMirrorMaker2NotReady</span>
              <span class="na">expr</span><span class="pi">:</span> <span class="s">strimzi_kafka_mm2_resource_info{ready!="True"}</span>
              <span class="na">for</span><span class="pi">:</span> <span class="s">15m</span>
              <span class="na">labels</span><span class="pi">:</span>
                <span class="na">severity</span><span class="pi">:</span> <span class="s">warning</span>
              <span class="na">annotations</span><span class="pi">:</span>
                <span class="na">message</span><span class="pi">:</span> <span class="s2">"</span><span class="s">Strimzi</span><span class="nv"> </span><span class="s">KafkaMirrorMaker2</span><span class="nv"> </span><span class="s">{{`{{</span><span class="nv"> </span><span class="s">$labels.name</span><span class="nv"> </span><span class="s">}}`}}</span><span class="nv"> </span><span class="s">is</span><span class="nv"> </span><span class="s">not</span><span class="nv"> </span><span class="s">ready"</span>
            <span class="pi">-</span> <span class="na">alert</span><span class="pi">:</span> <span class="s">KafkaMirrorMaker2Deprecated</span>
              <span class="na">expr</span><span class="pi">:</span> <span class="s">strimzi_kafka_mm2_resource_info{deprecated="Warning"}</span>
              <span class="na">for</span><span class="pi">:</span> <span class="s">15m</span>
              <span class="na">labels</span><span class="pi">:</span>
                <span class="na">severity</span><span class="pi">:</span> <span class="s">warning</span>
              <span class="na">annotations</span><span class="pi">:</span>
                <span class="na">message</span><span class="pi">:</span> <span class="s2">"</span><span class="s">Strimzi</span><span class="nv"> </span><span class="s">KafkaMirrorMaker2</span><span class="nv"> </span><span class="s">{{`{{</span><span class="nv"> </span><span class="s">$labels.name</span><span class="nv"> </span><span class="s">}}`}}</span><span class="nv"> </span><span class="s">contains</span><span class="nv"> </span><span class="s">a</span><span class="nv"> </span><span class="s">deprecated</span><span class="nv"> </span><span class="s">configuration"</span>
            <span class="pi">-</span> <span class="na">alert</span><span class="pi">:</span> <span class="s">KafkaAccessNotReady</span>
              <span class="na">expr</span><span class="pi">:</span> <span class="s">strimzi_kafka_access_resource_info{ready!="True"}</span>
              <span class="na">for</span><span class="pi">:</span> <span class="s">15m</span>
              <span class="na">labels</span><span class="pi">:</span>
                <span class="na">severity</span><span class="pi">:</span> <span class="s">warning</span>
              <span class="na">annotations</span><span class="pi">:</span>
                <span class="na">message</span><span class="pi">:</span> <span class="s2">"</span><span class="s">Strimzi</span><span class="nv"> </span><span class="s">KafkaAccess</span><span class="nv"> </span><span class="s">{{`{{</span><span class="nv"> </span><span class="s">$labels.name</span><span class="nv"> </span><span class="s">}}`}}</span><span class="nv"> </span><span class="s">is</span><span class="nv"> </span><span class="s">not</span><span class="nv"> </span><span class="s">ready"</span>
            <span class="pi">-</span> <span class="na">alert</span><span class="pi">:</span> <span class="s">KafkaAccessDeprecated</span>
              <span class="na">expr</span><span class="pi">:</span> <span class="s">strimzi_kafka_access_resource_info{deprecated="Warning"}</span>
              <span class="na">for</span><span class="pi">:</span> <span class="s">15m</span>
              <span class="na">labels</span><span class="pi">:</span>
                <span class="na">severity</span><span class="pi">:</span> <span class="s">warning</span>
              <span class="na">annotations</span><span class="pi">:</span>
                <span class="na">message</span><span class="pi">:</span> <span class="s2">"</span><span class="s">Strimzi</span><span class="nv"> </span><span class="s">KafkaAccess</span><span class="nv"> </span><span class="s">{{`{{</span><span class="nv"> </span><span class="s">$labels.name</span><span class="nv"> </span><span class="s">}}`}}</span><span class="nv"> </span><span class="s">contains</span><span class="nv"> </span><span class="s">a</span><span class="nv"> </span><span class="s">deprecated</span><span class="nv"> </span><span class="s">configuration"</span>
</code></pre></div></div>

<p>Then you can simply install kube-state-metrics by executing:</p>

<div class="language-bash highlighter-rouge"><div class="highlight"><pre class="highlight"><code>helm <span class="nb">install </span>strimzi-kube-state-metrics oci://ghcr.io/prometheus-community/charts/kube-state-metrics <span class="nt">--version</span> 6.3.0 <span class="nt">-f</span> my-strimzi-kube-state-metrics-values.yaml
</code></pre></div></div>

<h3 id="verify-it-works">Verify it works</h3>

<p>Once Prometheus is scraping kube-state-metrics, try these queries to confirm it’s working.</p>

<p>Get an overview about your <code class="language-plaintext highlighter-rouge">Kafka</code> clusters:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>sum by (namespace, name, cluster_id, ready) (strimzi_kafka_resource_info)
</code></pre></div></div>

<p>Example output:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>{cluster_id="123456789", name="my-first-kafka",  namespace="kafka-one",   ready="True"}
{cluster_id="abcdefghj", name="my-second-kafka", namespace="kafka-two",   ready="True"}
{cluster_id="zyxwvutsr", name="my-third-kafka",  namespace="kafka-three", ready="False"}
</code></pre></div></div>

<p>Get an overview of <code class="language-plaintext highlighter-rouge">KafkaTopic</code>(s) with partitions and replicas:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>sum by (name, partitions, replicas) (strimzi_kafka_topic_resource_info)
</code></pre></div></div>

<p>Example output:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>{name="my-first-kafka-topic",  partitions="1",  replicas="1"}
{name="my-second-kafka-topic", partitions="12", replicas="3"}
{name="my-third-kafka-topic",  partitions="60", replicas="3"}
</code></pre></div></div>

<p>Get an overview of <code class="language-plaintext highlighter-rouge">KafkaUser</code>(s) with username:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>sum by (name, namespace, username) (strimzi_kafka_user_resource_info)
</code></pre></div></div>

<p>Example output:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>{name="my-first-kafkauser",  username="CN=my-first-kafkauser"}
{name="my-second-kafkauser", username="CN=my-second-kafkauser"}
{name="my-third-kafkauser",  username="CN=my-third-kafkauser"}
</code></pre></div></div>

<h3 id="deprecation">Deprecation</h3>

<p>With the new kube-state-metrics–based metrics, the built‑in Cluster Operator metrics are deprecated as of <a href="https://github.com/strimzi/strimzi-kafka-operator/releases/tag/0.48.0">Strimzi 0.48.0</a> and will be removed in Strimzi 0.51.0.
If you still rely on strimzi_resource_state, plan migration now.
Progress on the removal is tracked here: <a href="https://github.com/strimzi/strimzi-kafka-operator/issues/11696">https://github.com/strimzi/strimzi-kafka-operator/issues/11696</a>.</p>

<h3 id="wrapup">Wrap‑up</h3>

<p>By using kube-state-metrics, you can monitor all Strimzi custom resources.
This makes it easier to spot misconfigurations and stay informed about deprecations.
These metrics can also power custom Grafana dashboards for a high‑level overview of deployed infrastructure without needing direct Kubernetes access.</p>
