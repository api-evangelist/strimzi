---
title: "Strimzi Test Container: Simplifying Kafka &amp; Connect Testing"
url: "https://strimzi.io/blog/2026/01/15/strimzi-test-container/"
date: "2026-01-15T00:00:00+00:00"
author: "Maros Orsak"
feed_url: "https://strimzi.io/feed.xml"
---
<p>Testing Kafka-based applications used to be a pain.
Back in 2019, your options each came with significant trade-offs, and they still do:</p>
<ul>
  <li><strong>Full Kafka clusters</strong> are resource-intensive and hard to manage in CI</li>
  <li><strong>Mocks</strong> don’t behave like real Kafka and become a maintenance burden</li>
  <li><strong>Embedded Kafka</strong> can cause dependency conflicts with your application’s Kafka client libraries</li>
</ul>

<p>We built <strong>Strimzi Test Container</strong> to avoid these issues.</p>

<h3 id="why-we-built-strimzi-test-container">Why We Built Strimzi Test Container</h3>

<p>We needed proper isolation from embedded Kafka’s dependency conflicts.
Embedded Kafka runs in the same JVM as your tests, making it difficult to independently version the test cluster and your application’s Kafka client libraries.
By running Kafka in a container, we get full isolation (i.e., no classpath conflicts, and you can test against any Kafka version regardless of what your application uses).</p>

<p>But why build our own instead of using existing solutions?
The <strong>main reason</strong> is that the official <code class="language-plaintext highlighter-rouge">Testcontainers</code> Kafka module uses Confluent Platform containers rather than Apache Kafka.
This meant testing against a private fork instead of the actual Apache Kafka code we use in Strimzi.
We needed to test against the same Apache Kafka builds that run in production, and control our own release cadence to ship new Kafka versions quickly.</p>

<p>Another reason was to support multiple architectures.
Strimzi runs on various platforms: x64, aarch64, IBM Z, and IBM Power.
We needed a testing solution that works consistently across all of them.
Most existing solutions only supported x64 and aarch64 architectures.</p>

<h3 id="what-does-strimzi-test-container-provide">What Does Strimzi Test Container Provide?</h3>

<p>Strimzi Test Container provides two main components: <code class="language-plaintext highlighter-rouge">StrimziKafkaCluster</code> for Kafka broker testing and <code class="language-plaintext highlighter-rouge">StrimziConnectCluster</code> for Kafka Connect testing.</p>

<h4 id="strimzikafkacluster">StrimziKafkaCluster</h4>

<p><code class="language-plaintext highlighter-rouge">StrimziKafkaCluster</code> allows you to programmatically create and manage Docker-based KRaft Kafka clusters in your tests.
It supports both single-node and multi-node cluster configurations (i.e., you can easily create 3, 5, or more Kafka nodes for realistic testing).</p>

<blockquote>
  <p>Strimzi Test Container supports only KRaft-based clusters. ZooKeeper-based clusters are no longer supported.</p>
</blockquote>

<div class="language-java highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nc">StrimziKafkaCluster</span> <span class="n">kafkaCluster</span> <span class="o">=</span> <span class="k">new</span> <span class="nc">StrimziKafkaCluster</span><span class="o">.</span><span class="na">StrimziKafkaClusterBuilder</span><span class="o">()</span>
    <span class="o">.</span><span class="na">withNumberOfBrokers</span><span class="o">(</span><span class="mi">3</span><span class="o">)</span>
    <span class="o">.</span><span class="na">withSharedNetwork</span><span class="o">()</span>
    <span class="o">.</span><span class="na">build</span><span class="o">();</span>

<span class="n">kafkaCluster</span><span class="o">.</span><span class="na">start</span><span class="o">();</span>

<span class="c1">// Your tests here...</span>

<span class="n">kafkaCluster</span><span class="o">.</span><span class="na">stop</span><span class="o">();</span>
</code></pre></div></div>

<h4 id="strimziconnectcluster">StrimziConnectCluster</h4>

<p>Beyond Kafka brokers, we also support <strong>Kafka Connect</strong> testing with <code class="language-plaintext highlighter-rouge">StrimziConnectCluster</code>.
This allows you to test connectors, transformations, and end-to-end data pipelines in a realistic distributed environment.</p>

<div class="language-java highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c1">// assume that you have pre-defined your `kafkaCluster` before instantiating `connectCluster`. </span>

<span class="nc">StrimziConnectCluster</span> <span class="n">connectCluster</span> <span class="o">=</span> <span class="k">new</span> <span class="nc">StrimziConnectCluster</span><span class="o">.</span><span class="na">StrimziConnectClusterBuilder</span><span class="o">()</span>
    <span class="o">.</span><span class="na">withKafkaCluster</span><span class="o">(</span><span class="n">kafkaCluster</span><span class="o">)</span>
    <span class="o">.</span><span class="na">withNumberOfWorkers</span><span class="o">(</span><span class="mi">2</span><span class="o">)</span>
    <span class="o">.</span><span class="na">withGroupId</span><span class="o">(</span><span class="s">"test-connect-cluster"</span><span class="o">)</span>
    <span class="o">.</span><span class="na">build</span><span class="o">();</span>

<span class="n">connectCluster</span><span class="o">.</span><span class="na">start</span><span class="o">();</span>

<span class="nc">String</span> <span class="n">restEndpoint</span> <span class="o">=</span> <span class="n">connectCluster</span><span class="o">.</span><span class="na">getRestEndpoint</span><span class="o">();</span>
<span class="c1">// Deploy and test your connectors...</span>
</code></pre></div></div>

<h3 id="how-it-works">How It Works</h3>

<h4 id="multi-architecture-support">Multi-Architecture Support</h4>

<p>Strimzi Test Container supports x64, aarch64, Z (s390x), and PPC (ppc64le) architectures.
It uses multi-arch container images published on <a href="https://quay.io/organization/strimzi-test-container">Quay.io</a>, with tags for each supported Kafka version (e.g., <code class="language-plaintext highlighter-rouge">quay.io/strimzi-test-container/test-container:latest-kafka-4.1.1</code>)
Combined with Java’s cross-platform capabilities, the same test code works regardless of where you run it.</p>

<h4 id="multi-node-setup-with-combined-or-dedicated-roles">Multi-Node Setup with Combined or Dedicated Roles</h4>

<p>One of the key features is support for different node role configurations.</p>

<p><strong>Combined roles (default configuration)</strong> - each node acts as both a KRaft controller and a broker.
This is simpler and works well for most testing scenarios:</p>

<p><img alt="Combined Roles" src="https://strimzi.io/assets/images/posts/2026-01-15-strimzi-test-container-01.svg" /></p>

<div class="language-java highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nc">StrimziKafkaCluster</span> <span class="n">kafkaCluster</span> <span class="o">=</span> <span class="k">new</span> <span class="nc">StrimziKafkaCluster</span><span class="o">.</span><span class="na">StrimziKafkaClusterBuilder</span><span class="o">()</span>
    <span class="o">.</span><span class="na">withNumberOfBrokers</span><span class="o">(</span><span class="mi">3</span><span class="o">)</span>
    <span class="o">.</span><span class="na">build</span><span class="o">();</span>
</code></pre></div></div>

<p><strong>Dedicated roles</strong> - separate controller and broker nodes are used, matching production-like deployments:</p>

<p><img alt="Dedicated Roles" src="https://strimzi.io/assets/images/posts/2026-01-15-strimzi-test-container-02.svg" /></p>

<div class="language-java highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nc">StrimziKafkaCluster</span> <span class="n">kafkaCluster</span> <span class="o">=</span> <span class="k">new</span> <span class="nc">StrimziKafkaCluster</span><span class="o">.</span><span class="na">StrimziKafkaClusterBuilder</span><span class="o">()</span>
    <span class="o">.</span><span class="na">withNumberOfBrokers</span><span class="o">(</span><span class="mi">3</span><span class="o">)</span>
    <span class="o">.</span><span class="na">withDedicatedRoles</span><span class="o">()</span>
    <span class="o">.</span><span class="na">withNumberOfControllers</span><span class="o">(</span><span class="mi">3</span><span class="o">)</span>
    <span class="o">.</span><span class="na">build</span><span class="o">();</span>

<span class="c1">// Creates 3 controller-only nodes (IDs: 0, 1, 2)</span>
<span class="c1">// and 3 broker-only nodes (IDs: 3, 4, 5)</span>
</code></pre></div></div>

<blockquote>
  <p>The diagrams above are just for illustration. 
You can easily spin up a single-node cluster or configure any number of brokers and controllers to match your testing needs.</p>
</blockquote>

<h4 id="multiple-kafka-versions">Multiple Kafka Versions</h4>

<p>Test against specific Kafka versions with a single line change:</p>

<div class="language-java highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nc">StrimziKafkaCluster</span> <span class="n">kafkaCluster</span> <span class="o">=</span> <span class="k">new</span> <span class="nc">StrimziKafkaCluster</span><span class="o">.</span><span class="na">StrimziKafkaClusterBuilder</span><span class="o">()</span>
    <span class="o">.</span><span class="na">withNumberOfBrokers</span><span class="o">(</span><span class="mi">3</span><span class="o">)</span>
    <span class="o">.</span><span class="na">withKafkaVersion</span><span class="o">(</span><span class="s">"4.0.1"</span><span class="o">)</span>
    <span class="o">.</span><span class="na">build</span><span class="o">();</span>
</code></pre></div></div>

<p>You can also use your own custom images if you have special requirements:</p>

<div class="language-java highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nc">StrimziKafkaCluster</span> <span class="n">kafkaCluster</span> <span class="o">=</span> <span class="k">new</span> <span class="nc">StrimziKafkaCluster</span><span class="o">.</span><span class="na">StrimziKafkaClusterBuilder</span><span class="o">()</span>
    <span class="o">.</span><span class="na">withNumberOfBrokers</span><span class="o">(</span><span class="mi">3</span><span class="o">)</span>
    <span class="o">.</span><span class="na">withImage</span><span class="o">(</span><span class="s">"my-registry.io/my-kafka:custom"</span><span class="o">)</span>
    <span class="o">.</span><span class="na">build</span><span class="o">();</span>
</code></pre></div></div>

<h4 id="rich-configuration-options">Rich Configuration Options</h4>

<p>Pass any Kafka broker configuration you need:</p>

<div class="language-java highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nc">StrimziKafkaCluster</span> <span class="n">kafkaCluster</span> <span class="o">=</span> <span class="k">new</span> <span class="nc">StrimziKafkaCluster</span><span class="o">.</span><span class="na">StrimziKafkaClusterBuilder</span><span class="o">()</span>
    <span class="o">.</span><span class="na">withNumberOfBrokers</span><span class="o">(</span><span class="mi">3</span><span class="o">)</span>
    <span class="o">.</span><span class="na">withAdditionalKafkaConfiguration</span><span class="o">(</span><span class="nc">Map</span><span class="o">.</span><span class="na">of</span><span class="o">(</span>
        <span class="s">"log.retention.hours"</span><span class="o">,</span> <span class="s">"1"</span><span class="o">,</span>
        <span class="s">"min.insync.replicas"</span><span class="o">,</span> <span class="s">"2"</span><span class="o">,</span>
        <span class="s">"compression.type"</span><span class="o">,</span> <span class="s">"snappy"</span>
    <span class="o">))</span>
    <span class="o">.</span><span class="na">build</span><span class="o">();</span>
</code></pre></div></div>

<h4 id="oauth-support">OAuth Support</h4>

<p>Strimzi Test Container supports OAuth authentication with both <code class="language-plaintext highlighter-rouge">OAUTHBEARER</code> and <code class="language-plaintext highlighter-rouge">OAUTH_OVER_PLAIN</code> mechanisms.
This lets you test secure Kafka deployments by configuring the Kafka cluster to use OAuth.
Note that you need to supply your own identity provider (e.g., Keycloak, or any OAuth-compliant provider) for authentication.</p>

<h3 id="from-strimzi-to-the-broader-ecosystem">From Strimzi to the Broader Ecosystem</h3>

<p>Strimzi Test Container started as an “internal” library for Strimzi projects like Strimzi Kafka Operator and Strimzi Kafka Bridge.
Over time, it expanded beyond Strimzi and is now used by projects like Quarkus and Debezium.</p>

<p><img alt="Strimzi Test Container Ecosystem" src="https://strimzi.io/assets/images/posts/2026-01-15-strimzi-test-container-03.svg" /></p>

<p>Within the Strimzi ecosystem, we currently use it across multiple subprojects:</p>

<ul>
  <li><a href="https://github.com/strimzi/strimzi-kafka-operator">strimzi-kafka-operator</a></li>
  <li><a href="https://github.com/strimzi/strimzi-kafka-bridge">strimzi-kafka-bridge</a></li>
  <li><a href="https://github.com/strimzi/strimzi-mqtt-bridge">strimzi-mqtt-bridge</a></li>
  <li><a href="https://github.com/strimzi/metrics-reporter">metrics-reporter</a></li>
  <li><a href="https://github.com/strimzi/test-clients">test-clients</a></li>
</ul>

<h3 id="conclusion">Conclusion</h3>

<p>We built Strimzi Test Container because we needed it ourselves.
Turns out, others needed it too.</p>

<p>If you are tired of fighting with embedded Kafka or maintaining Docker Compose files for your tests, give it a spin.
Real Kafka, real behavior, minimal setup.</p>
