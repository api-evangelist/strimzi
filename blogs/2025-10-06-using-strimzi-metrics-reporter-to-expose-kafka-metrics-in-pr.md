---
title: "Using Strimzi Metrics Reporter to expose Kafka metrics in Prometheus format"
url: "https://strimzi.io/blog/2025/10/06/strimzi-metrics-reporter/"
date: "2025-10-06T00:00:00+00:00"
author: "Owen Corrigan"
feed_url: "https://strimzi.io/feed.xml"
---
<h2 id="using-strimzi-metrics-reporter-in-your-kafka-cluster">Using Strimzi Metrics Reporter in your Kafka Cluster</h2>

<p>Apache Kafka generates and exposes metrics that reflect how it is operating.
These metrics are useful for monitoring, troubleshooting, tuning, and capacity planning when it comes to running your Kafka cluster.
In essence, monitoring is crucial to ensure the health and performance of your Kafka clusters.
Prometheus is a widely used monitoring solution in the cloud-native ecosystem, and Strimzi exposes metrics in this format.
The <a href="https://github.com/prometheus/jmx_exporter">JMX Exporter</a> is a solid and fully supported option for existing setups and specific use cases as a way to expose Kafka metrics, and we have provided some example YAML files in our <a href="https://github.com/strimzi/strimzi-kafka-operator/tree/main/packaging/examples/metrics">Strimzi Examples folder</a>.</p>

<p>As an alternative to JMX Exporter, <a href="https://github.com/strimzi/metrics-reporter">Strimzi Metrics Reporter</a> is a new project in Strimzi, still in early access, and was announced as part of <a href="https://strimzi.io/blog/2025/07/15/what-is-new-in-strimzi-0.47.0/">Strimzi 0.47.0</a>.
It implements the Kafka <code class="language-plaintext highlighter-rouge">MetricsReporter</code> interface to expose metrics via Prometheus.
This implementation removes the need for JMX-based exporters or additional agents.
Strimzi Metrics Reporter is designed from scratch as a Kafka plugin and allows you to configure it directly from the Kafka configuration.
This makes it more efficient than other monitoring tools that run as sidecars or JVM agents.
Strimzi Metrics Reporter uses fixed metric names, which simplifies configuration and ensures consistent naming across deployments.
Unlike tools that rely on complex mapping rules or regular expressions, it offers a straightforward way to expose metrics.
It is easier to use and provides a metrics interface that is more stable, like a well-defined API.</p>

<p>Here, we will discuss Strimzi Metrics Reporter configuration and guide you on how to use it effectively to monitor your Kafka clusters.</p>

<h3 id="key-features">Key Features</h3>
<ul>
  <li>Native Prometheus support: The reporter exposes metrics in the Prometheus format through an HTTP endpoint, without relying on JMX.</li>
  <li>Configurable Metrics Collection: Allows users to specify which metrics should be exposed using a flexible allowlist.</li>
  <li>Scalability and Performance: Strimzi Metrics Reporter scales efficiently with increasing numbers of metrics, maintaining low response times even as the metric count grows, demonstrating strong performance characteristics and minimal overhead, making it well-suited for large-scale Kafka deployments.</li>
</ul>

<h4 id="deploying-metrics-reporter">Deploying Metrics Reporter</h4>
<p>The first step is to include the Strimzi Metrics Reporter in your <code class="language-plaintext highlighter-rouge">Kafka</code> custom resource.
To do this, add the following:</p>

<div class="language-yaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="na">apiVersion</span><span class="pi">:</span> <span class="s">kafka.strimzi.io/v1beta1</span>
<span class="na">kind</span><span class="pi">:</span> <span class="s">Kafka</span>
<span class="na">metadata</span><span class="pi">:</span>
  <span class="na">name</span><span class="pi">:</span> <span class="s">my-cluster</span>
<span class="na">spec</span><span class="pi">:</span>
  <span class="na">kafka</span><span class="pi">:</span>
  <span class="c1"># ...</span>
    <span class="na">metricsConfig</span><span class="pi">:</span>
      <span class="na">type</span><span class="pi">:</span> <span class="s">strimziMetricsReporter</span>
  <span class="c1"># ...  </span>
</code></pre></div></div>

<p>By adding <code class="language-plaintext highlighter-rouge">type: strimziMetricsReporter</code> to the <code class="language-plaintext highlighter-rouge">metricsConfig</code> section of your <code class="language-plaintext highlighter-rouge">Kafka</code> custom resource, Strimzi will export a sensible set of default metrics.
However, you can add your own custom values by filtering the metrics by name. 
This is achieved by adding another field called <code class="language-plaintext highlighter-rouge">values</code>, and within that field, adding <code class="language-plaintext highlighter-rouge">allowList</code>.
Then you can specify which metrics you want to collect, where each entry is used to filter allowed metrics.
You can put individual metric names or use a regex to include a group of metrics and avoid having a long list.
For example:</p>

<div class="language-yaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="na">apiVersion</span><span class="pi">:</span> <span class="s">kafka.strimzi.io/v1beta1</span>
<span class="na">kind</span><span class="pi">:</span> <span class="s">Kafka</span>
<span class="na">metadata</span><span class="pi">:</span>
  <span class="na">name</span><span class="pi">:</span> <span class="s">my-cluster</span>
<span class="na">spec</span><span class="pi">:</span>
  <span class="na">kafka</span><span class="pi">:</span>
  <span class="c1"># ...</span>
    <span class="na">metricsConfig</span><span class="pi">:</span>
      <span class="na">type</span><span class="pi">:</span> <span class="s">strimziMetricsReporter</span>
      <span class="na">values</span><span class="pi">:</span>
        <span class="na">allowList</span><span class="pi">:</span>
          <span class="pi">-</span> <span class="s2">"</span><span class="s">kafka_log.*"</span>
          <span class="pi">-</span> <span class="s2">"</span><span class="s">kafka_network.*"</span>
  <span class="c1"># ...  </span>
</code></pre></div></div>

<p>If you add Strimzi Metrics Reporter to an existing cluster, all Kafka broker and Controller pods roll to apply the new configuration without any downtime to your cluster.
If you would like to deploy your Kafka cluster with the Strimzi Metrics Reporter enabled from the start, you can use the examples in our <a href="https://github.com/strimzi/strimzi-kafka-operator/tree/0.48.0/examples/metrics/strimzi-metrics-reporter">Strimzi Metrics Reporter Examples folder</a> by running the following command:</p>

<div class="language-bash highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span>kubectl apply <span class="nt">-f</span> https://strimzi.io/examples/0.48.0/metrics/strimzi-metrics-reporter/kafka-metrics.yaml <span class="nt">-n</span> myproject
</code></pre></div></div>

<p>We also provide <a href="https://github.com/strimzi/strimzi-kafka-operator/tree/0.48.0/examples/metrics/strimzi-metrics-reporter/grafana-dashboards">Grafana Dashboards</a> to help you visualize Kafka metrics in Prometheus format.
Strimzi Metrics Reporter can also be used with Strimzi Kafka Bridge, Kafka Connect, and Kafka MirrorMaker 2, and we provide <a href="https://github.com/strimzi/strimzi-kafka-operator/tree/0.48.0/examples/metrics/strimzi-metrics-reporter">examples</a> for these too.</p>

<h3 id="conclusion">Conclusion</h3>
<p>We’ve demonstrated how to deploy the Strimzi Metrics Reporter in your Kafka cluster.
As the feature is currently in early access, we invite users to experiment with it and provide feedback based on their experience, so please <a href="https://strimzi.io/community/">reach out to us</a>.
The Strimzi Metrics Reporter was recently <a href="https://www.youtube.com/watch?v=evKGEziQj54">featured as part of StrimziCon 2025</a> which you may also find worth watching.
We also provide you with a practical demo video on our <a href="https://www.youtube.com/watch?v=Za04jVp8f5c">YouTube</a> channel in which we go through the steps outlined above, and hopefully this will encourage you to try out Strimzi Metrics Reporter for yourself.</p>

<p>Thanks for reading and keep an eye on our blog posts for future updates.</p>
