---
title: "Server-Side Apply in Strimzi"
url: "https://strimzi.io/blog/2026/01/26/server-side-apply-in-strimzi/"
date: "2026-01-26T00:00:00+00:00"
author: "Lukas Kral"
feed_url: "https://strimzi.io/feed.xml"
---
<p>Kubernetes operators create, update, and delete resources to reflect the desired state defined by users.
When a particular resource is managed by a single operator or a single user, update conflicts are relatively rare. 
However, problems arise when multiple actors - such as different operators or automation tools - modify the same resource.</p>

<p>With client-side apply, even changes to different fields can unintentionally overwrite each other. 
This behavior is especially problematic when multiple reconciliation loops are involved, as one operator may repeatedly revert changes made by another. 
This is the case with client-side apply as used in Strimzi.</p>

<h3 id="client-side-apply-in-strimzi">Client-side apply in Strimzi</h3>

<p>When a user creates or updates a Strimzi resource, the desired state is taken by Strimzi and propagated into all needed resources.
For example (based on the configuration), when a user updates a field in the <code class="language-plaintext highlighter-rouge">Kafka</code> CR, Strimzi rebuilds the desired state for resources like <code class="language-plaintext highlighter-rouge">StrimziPodSet</code>, <code class="language-plaintext highlighter-rouge">ConfigMap</code>, <code class="language-plaintext highlighter-rouge">Service</code>, and <code class="language-plaintext highlighter-rouge">PersistentVolumeClaim</code>.
This is completely fine until another operator, running in a reconciliation loop, updates these resources with another value.
One example is Kyverno, which has a policy to add annotation <code class="language-plaintext highlighter-rouge">policies.kyverno.io/last-applied-patches:...</code> to all resources in the Kubernetes cluster using <code class="language-plaintext highlighter-rouge">MutatingWebhookConfiguration</code>.
With each update, Strimzi detects the resource change and reconciles it from the desired state, overwriting any modifications made by the other operator.
This can result in an update loop, along with warnings, errors, or other downstream issues in affected services or operators.</p>

<p>Because of these issues, we decided to add support for Server-Side Apply.</p>

<h3 id="what-is-server-side-apply">What is Server-Side Apply?</h3>

<p>Server-Side Apply (SSA) allows multiple actors to update the same Kubernetes resource while managing different fields. 
Instead of applying a full object update, each actor applies only the fields it owns, identified by a field manager. 
Kubernetes then tracks field ownership and detects conflicts when multiple actors attempt to manage the same field.</p>

<p>For operators, this provides a clear ownership model. 
The Strimzi operator can manage only the fields it’s responsible for, without overwriting changes made to other fields by users or other controllers.</p>

<p>At the same time, this model assumes that other actors modify only the fields that they are responsible for. 
If process external to Strimzi updates fields that are essential for Strimzi’s functionality, Strimzi will revert the changes back. 
However, SSA makes these ownership boundaries explicit and visible, helping surface such issues earlier and making them easier to understand and address.</p>

<h3 id="incremental-implementation-of-server-side-apply-in-strimzi">Incremental implementation of Server-Side Apply in Strimzi</h3>

<p>Originally, there was a <a href="https://github.com/strimzi/proposals/blob/main/052-k8s-server-side-apply.md">proposal</a> and a plan to implement Server-Side Apply for all resources managed by Strimzi. 
However, the scope of such a change turned out to be too large, so we decided (in <a href="https://github.com/strimzi/proposals/blob/main/105-server-side-apply-implementation-fg-timelines.md">second proposal</a>) to split the implementation into multiple phases.</p>

<h4 id="phase-1-initial-server-side-apply-support">Phase 1: Initial Server-Side Apply support</h4>

<p>Server-Side Apply support was introduced in Strimzi 0.48 behind a feature gate, and its adoption is being implemented incrementally.</p>

<p>In the first phase, we implemented Server-Side Apply for the following resources:</p>

<ul>
  <li><code class="language-plaintext highlighter-rouge">PersistentVolumeClaim</code></li>
  <li><code class="language-plaintext highlighter-rouge">ServiceAccount</code></li>
  <li><code class="language-plaintext highlighter-rouge">Service</code></li>
  <li><code class="language-plaintext highlighter-rouge">Ingress</code></li>
  <li><code class="language-plaintext highlighter-rouge">ConfigMap</code></li>
</ul>

<p>These resources were identified as the most problematic based on GitHub issues, community discussions, and feedback from users on the Strimzi community Slack channels. 
To minimize risk and avoid unexpected behavior, switching to Server-Side Apply is gated behind a feature gate called <code class="language-plaintext highlighter-rouge">ServerSideApplyPhase1</code>.</p>

<p>When this feature gate is enabled, the Cluster Operator uses Server-Side Apply (SSA) exclusively for these resources, applying declarative updates, such as changes to metadata, spec, and status, rather than rebuilding the entire resource from scratch.
The SSA implementation in Strimzi ensures that fields managed by Strimzi are always reconciled to the desired state, even in the presence of conflicts.</p>

<p>The reconciliation flow is as follows:</p>

<ul>
  <li>Strimzi first attempts to apply the change using Server-Side Apply without forcing ownership.</li>
  <li>If no conflict occurs, the patch is applied and reconciliation continues.</li>
  <li>If a conflict is detected, the Cluster Operator logs the error and retries the apply operation with force enabled.</li>
  <li>When force is used, the affected field is updated, overwriting any changes made by other actors, and an explicit log entry is emitted to make this behavior visible to users.</li>
</ul>

<p>This approach ensures that Strimzi can reliably configure the fields required for correct cluster functionality, while still allowing other actors to manage fields outside of Strimzi’s ownership.</p>

<h3 id="how-server-side-apply-works-in-practice">How Server-Side Apply works in practice</h3>

<p>Theory is nice, but let’s see Server-Side Apply in action.
To try out this feature, you first need to enable the <code class="language-plaintext highlighter-rouge">ServerSideApplyPhase1</code> feature gate in the <a href="https://github.com/strimzi/strimzi-kafka-operator/blob/main/install/cluster-operator/060-Deployment-strimzi-cluster-operator.yaml#L90"><code class="language-plaintext highlighter-rouge">Deployment</code> resource</a> for the Strimzi Cluster Operator:</p>

<div class="language-yaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nn">...</span>
<span class="pi">-</span> <span class="na">name</span><span class="pi">:</span> <span class="s">STRIMZI_FEATURE_GATES</span>
  <span class="na">value</span><span class="pi">:</span> <span class="s2">"</span><span class="s">+ServerSideApplyPhase1"</span>
<span class="nn">...</span>
</code></pre></div></div>

<p>With Server-Side Apply enabled in the Cluster Operator, it can be tested on one of the phase 1 SSA resources.
For this example, an ephemeral Kafka cluster is created from <a href="https://github.com/strimzi/strimzi-kafka-operator/blob/main/examples/kafka/kafka-ephemeral.yaml">the configuration examples</a> provided with Strimzi.</p>

<p>As a simple test case, a custom annotation is added to the <code class="language-plaintext highlighter-rouge">-kafka-bootstrap</code> Service. 
Before doing that, let’s inspect the current <code class="language-plaintext highlighter-rouge">.metadata</code> section of the resource.</p>

<div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="o">&gt;</span> kubectl get service
NAME                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT<span class="o">(</span>S<span class="o">)</span>                                        AGE
my-cluster-kafka-bootstrap   ClusterIP   X.X.X.X         &lt;none&gt;        9091/TCP,9092/TCP,9093/TCP                     9m17s
my-cluster-kafka-brokers     ClusterIP   None            &lt;none&gt;        9090/TCP,9091/TCP,8443/TCP,9092/TCP,9093/TCP   9m17s

<span class="o">&gt;</span> kubectl get service my-cluster-kafka-bootstrap <span class="nt">-o</span> <span class="nv">jsonpath</span><span class="o">=</span><span class="s1">'{.metadata}'</span> | jq
<span class="o">{</span>
  <span class="s2">"annotations"</span>: <span class="o">{</span>
    <span class="s2">"strimzi.io/discovery"</span>: <span class="s2">"[ {</span><span class="se">\n</span><span class="s2">  </span><span class="se">\"</span><span class="s2">port</span><span class="se">\"</span><span class="s2"> : 9092,</span><span class="se">\n</span><span class="s2">  </span><span class="se">\"</span><span class="s2">tls</span><span class="se">\"</span><span class="s2"> : false,</span><span class="se">\n</span><span class="s2">  </span><span class="se">\"</span><span class="s2">protocol</span><span class="se">\"</span><span class="s2"> : </span><span class="se">\"</span><span class="s2">kafka</span><span class="se">\"</span><span class="s2">,</span><span class="se">\n</span><span class="s2">  </span><span class="se">\"</span><span class="s2">auth</span><span class="se">\"</span><span class="s2"> : </span><span class="se">\"</span><span class="s2">none</span><span class="se">\"\n</span><span class="s2">}, {</span><span class="se">\n</span><span class="s2">  </span><span class="se">\"</span><span class="s2">port</span><span class="se">\"</span><span class="s2"> : 9093,</span><span class="se">\n</span><span class="s2">  </span><span class="se">\"</span><span class="s2">tls</span><span class="se">\"</span><span class="s2"> : true,</span><span class="se">\n</span><span class="s2">  </span><span class="se">\"</span><span class="s2">protocol</span><span class="se">\"</span><span class="s2"> : </span><span class="se">\"</span><span class="s2">kafka</span><span class="se">\"</span><span class="s2">,</span><span class="se">\n</span><span class="s2">  </span><span class="se">\"</span><span class="s2">auth</span><span class="se">\"</span><span class="s2"> : </span><span class="se">\"</span><span class="s2">none</span><span class="se">\"\n</span><span class="s2">} ]"</span>
  <span class="o">}</span>,
  ...
  <span class="s2">"managedFields"</span>: <span class="o">[</span>
    <span class="o">{</span>
      <span class="s2">"apiVersion"</span>: <span class="s2">"v1"</span>,
      <span class="s2">"fieldsType"</span>: <span class="s2">"FieldsV1"</span>,
      <span class="s2">"fieldsV1"</span>: <span class="o">{</span>
        <span class="s2">"f:metadata"</span>: <span class="o">{</span>
          <span class="s2">"f:annotations"</span>: <span class="o">{</span>
            <span class="s2">"f:strimzi.io/discovery"</span>: <span class="o">{}</span>
          <span class="o">}</span>,
          ...
      <span class="s2">"manager"</span>: <span class="s2">"strimzi-kafka-operator"</span>,
      <span class="s2">"operation"</span>: <span class="s2">"Apply"</span>
    <span class="o">}</span>,
    ...
  <span class="o">]</span>
<span class="o">}</span>
</code></pre></div></div>

<p>At this point, the Service contains a single annotation, <code class="language-plaintext highlighter-rouge">strimzi.io/discovery</code>.
The <code class="language-plaintext highlighter-rouge">managedFields</code> section shows that this annotation is owned by the <code class="language-plaintext highlighter-rouge">strimzi-kafka-operator</code> field manager and was applied using Server-Side Apply.</p>

<p>Now let’s simulate another actor updating the same resource by adding a custom annotation using SSA - in our case it will be <code class="language-plaintext highlighter-rouge">my.annotation/some: value</code>.</p>

<div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="o">&gt;</span> kubectl apply <span class="nt">--server-side</span> <span class="nt">--field-manager</span><span class="o">=</span>different-agent <span class="nt">-f</span> - <span class="o">&lt;&lt;</span><span class="no">EOF</span><span class="sh">
apiVersion: v1
kind: Service
metadata:
  annotations:
    my.annotation/some: value # add new annotation
    strimzi.io/discovery: |-
      [ {
        "port" : 9092,
        "tls" : false,
        "protocol" : "kafka",
        "auth" : "none"
      }, {
        "port" : 9093,
        "tls" : true,
        "protocol" : "kafka",
        "auth" : "none"
      } ]
  labels:
    app.kubernetes.io/instance: my-cluster
    app.kubernetes.io/managed-by: strimzi-cluster-operator
    app.kubernetes.io/name: kafka
    app.kubernetes.io/part-of: strimzi-my-cluster
    strimzi.io/cluster: my-cluster
    strimzi.io/component-type: kafka
    strimzi.io/discovery: "true"
    strimzi.io/kind: Kafka
    strimzi.io/name: my-cluster-kafka
  name: my-cluster-kafka-bootstrap
  namespace: test
spec:
  clusterIP: 10.97.174.54
  clusterIPs:
  - 10.97.174.54
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - name: tcp-replication
    port: 9091
    protocol: TCP
    targetPort: tcp-replication
  - name: tcp-clients
    port: 9092
    protocol: TCP
    targetPort: tcp-clients
  - name: tcp-clientstls
    port: 9093
    protocol: TCP
    targetPort: tcp-clientstls
  selector:
    strimzi.io/broker-role: "true"
    strimzi.io/cluster: my-cluster
    strimzi.io/kind: Kafka
    strimzi.io/name: my-cluster-kafka
  sessionAffinity: None
  type: ClusterIP
</span><span class="no">EOF
</span></code></pre></div></div>

<p>The <code class="language-plaintext highlighter-rouge">--field-manager</code> flag identifies the actor performing the change.
If we inspect the metadata again, we can see that the annotation was added and is now owned by a different field manager.</p>

<div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="o">&gt;</span> kubectl get service my-cluster-kafka-bootstrap <span class="nt">-o</span> <span class="nv">jsonpath</span><span class="o">=</span><span class="s1">'{.metadata}'</span> | jq
<span class="o">{</span>
  <span class="s2">"annotations"</span>: <span class="o">{</span>
    <span class="s2">"my.annotation/some"</span>: <span class="s2">"value"</span>,
    <span class="s2">"strimzi.io/discovery"</span>: <span class="s2">"[ {</span><span class="se">\n</span><span class="s2">  </span><span class="se">\"</span><span class="s2">port</span><span class="se">\"</span><span class="s2"> : 9092,</span><span class="se">\n</span><span class="s2">  </span><span class="se">\"</span><span class="s2">tls</span><span class="se">\"</span><span class="s2"> : false,</span><span class="se">\n</span><span class="s2">  </span><span class="se">\"</span><span class="s2">protocol</span><span class="se">\"</span><span class="s2"> : </span><span class="se">\"</span><span class="s2">kafka</span><span class="se">\"</span><span class="s2">,</span><span class="se">\n</span><span class="s2">  </span><span class="se">\"</span><span class="s2">auth</span><span class="se">\"</span><span class="s2"> : </span><span class="se">\"</span><span class="s2">none</span><span class="se">\"\n</span><span class="s2">}, {</span><span class="se">\n</span><span class="s2">  </span><span class="se">\"</span><span class="s2">port</span><span class="se">\"</span><span class="s2"> : 9093,</span><span class="se">\n</span><span class="s2">  </span><span class="se">\"</span><span class="s2">tls</span><span class="se">\"</span><span class="s2"> : true,</span><span class="se">\n</span><span class="s2">  </span><span class="se">\"</span><span class="s2">protocol</span><span class="se">\"</span><span class="s2"> : </span><span class="se">\"</span><span class="s2">kafka</span><span class="se">\"</span><span class="s2">,</span><span class="se">\n</span><span class="s2">  </span><span class="se">\"</span><span class="s2">auth</span><span class="se">\"</span><span class="s2"> : </span><span class="se">\"</span><span class="s2">none</span><span class="se">\"\n</span><span class="s2">} ]"</span>
  <span class="o">}</span>,
  ...
  <span class="s2">"managedFields"</span>: <span class="o">[</span>
    ...
    <span class="o">{</span>
      <span class="s2">"apiVersion"</span>: <span class="s2">"v1"</span>,
      <span class="s2">"fieldsType"</span>: <span class="s2">"FieldsV1"</span>,
      <span class="s2">"fieldsV1"</span>: <span class="o">{</span>
        <span class="s2">"f:metadata"</span>: <span class="o">{</span>
          <span class="s2">"f:annotations"</span>: <span class="o">{</span>
            <span class="s2">"f:my.annotation/some"</span>: <span class="o">{}</span>
          <span class="o">}</span>
        <span class="o">}</span>
      <span class="o">}</span>,
      <span class="s2">"manager"</span>: <span class="s2">"different-agent"</span>,
      <span class="s2">"operation"</span>: <span class="s2">"Update"</span>,
      <span class="s2">"time"</span>: <span class="s2">"2026-01-14T00:54:46Z"</span>
    <span class="o">}</span>
</code></pre></div></div>

<p>Without Server-Side Apply, Strimzi would not track ownership of individual fields, and this custom annotation would likely be removed during the next reconciliation.</p>

<h4 id="handling-conflicts">Handling conflicts</h4>

<p>Now let’s see what happens when another actor attempts to modify a field owned by Strimzi.
In this case, you will need to use <code class="language-plaintext highlighter-rouge">--force-conflicts</code>, as the field we are trying to update is managed by Strimzi.</p>

<div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="o">&gt;</span> kubectl apply <span class="nt">--server-side</span> <span class="nt">--field-manager</span><span class="o">=</span>different-agent <span class="nt">--force-conflicts</span> <span class="nt">-f</span> - <span class="o">&lt;&lt;</span><span class="no">EOF</span><span class="sh">
apiVersion: v1
kind: Service
metadata:
  annotations:
    my.annotation/some: value # keep the annotation
    strimzi.io/discovery: "this-is-wrong" # change the annotation managed by Strimzi
  labels:
    app.kubernetes.io/instance: my-cluster
    app.kubernetes.io/managed-by: strimzi-cluster-operator
    app.kubernetes.io/name: kafka
    app.kubernetes.io/part-of: strimzi-my-cluster
    strimzi.io/cluster: my-cluster
    strimzi.io/component-type: kafka
    strimzi.io/discovery: "true"
    strimzi.io/kind: Kafka
    strimzi.io/name: my-cluster-kafka
  name: my-cluster-kafka-bootstrap
  namespace: test
spec:
  clusterIP: 10.97.174.54
  clusterIPs:
  - 10.97.174.54
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - name: tcp-replication
    port: 9091
    protocol: TCP
    targetPort: tcp-replication
  - name: tcp-clients
    port: 9092
    protocol: TCP
    targetPort: tcp-clients
  - name: tcp-clientstls
    port: 9093
    protocol: TCP
    targetPort: tcp-clientstls
  selector:
    strimzi.io/broker-role: "true"
    strimzi.io/cluster: my-cluster
    strimzi.io/kind: Kafka
    strimzi.io/name: my-cluster-kafka
  sessionAffinity: None
  type: ClusterIP
</span><span class="no">EOF
</span></code></pre></div></div>

<p>At this point, the annotation is updated (as we used force apply):</p>

<div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="o">&gt;</span> kubectl get service my-cluster-kafka-bootstrap <span class="nt">-o</span> <span class="nv">jsonpath</span><span class="o">=</span><span class="s1">'{.metadata}'</span> | jq
<span class="o">{</span>
  <span class="s2">"annotations"</span>: <span class="o">{</span>
    <span class="s2">"my.annotation/some"</span>: <span class="s2">"value"</span>,
    <span class="s2">"strimzi.io/discovery"</span>: <span class="s2">"this-is-wrong"</span>
  <span class="o">}</span>,
  ...
<span class="o">}</span>
</code></pre></div></div>

<p>During the next reconciliation, Strimzi detects a conflict on the <code class="language-plaintext highlighter-rouge">strimzi.io/discovery</code> annotation. 
Since this field is owned by Strimzi, the operator logs a warning and retries the apply operation with <code class="language-plaintext highlighter-rouge">force</code> enabled:</p>

<div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code>2026-01-14 01:01:09 DEBUG AbstractNamespacedResourceOperator:280 - Reconciliation <span class="c">#68(timer) Kafka(test-suite-namespace/my-cluster): Service test-suite-namespace/my-cluster-kafka-bootstrap is being patched using Server Side Apply</span>
2026-01-14 01:01:09 WARN  AbstractNamespacedResourceOperator:286 - Reconciliation <span class="c">#68(timer) Kafka(test-suite-namespace/my-cluster): Service test-suite-namespace/my-cluster-kafka-bootstrap failed to patch because of conflict: Failure executing: PATCH at: https://X.X.X.X:443/api/v1/namespaces/test-suite-namespace/services/my-cluster-kafka-bootstrap?fieldManager=strimzi-kafka-operator&amp;force=false. Message: Apply failed with 1 conflict: conflict with "different-agent" using v1: .metadata.annotations.strimzi.io/discovery. Received status: Status(apiVersion=v1, code=409, details=StatusDetails(causes=[StatusCause(field=.metadata.annotations.strimzi.io/discovery, message=conflict with "different-agent" using v1, reason=FieldManagerConflict, additionalProperties={})], group=null, kind=null, name=null, retryAfterSeconds=null, uid=null, additionalProperties={}), kind=Status, message=Apply failed with 1 conflict: conflict with "different-agent" using v1: .metadata.annotations.strimzi.io/discovery, metadata=ListMeta(_continue=null, remainingItemCount=null, resourceVersion=null, selfLink=null, additionalProperties={}), reason=Conflict, status=Failure, additionalProperties={})., applying force</span>
</code></pre></div></div>

<p>After the forced apply, Strimzi restores the correct value of its managed annotation, while the custom annotation remains untouched:</p>

<div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="o">&gt;</span> kubectl get service my-cluster-kafka-bootstrap <span class="nt">-o</span> <span class="nv">jsonpath</span><span class="o">=</span><span class="s1">'{.metadata}'</span> | jq
<span class="o">{</span>
  <span class="s2">"annotations"</span>: <span class="o">{</span>
    <span class="s2">"my.annotation/some"</span>: <span class="s2">"value"</span>,
    <span class="s2">"strimzi.io/discovery"</span>: <span class="s2">"[ {</span><span class="se">\n</span><span class="s2">  </span><span class="se">\"</span><span class="s2">port</span><span class="se">\"</span><span class="s2"> : 9092,</span><span class="se">\n</span><span class="s2">  </span><span class="se">\"</span><span class="s2">tls</span><span class="se">\"</span><span class="s2"> : false,</span><span class="se">\n</span><span class="s2">  </span><span class="se">\"</span><span class="s2">protocol</span><span class="se">\"</span><span class="s2"> : </span><span class="se">\"</span><span class="s2">kafka</span><span class="se">\"</span><span class="s2">,</span><span class="se">\n</span><span class="s2">  </span><span class="se">\"</span><span class="s2">auth</span><span class="se">\"</span><span class="s2"> : </span><span class="se">\"</span><span class="s2">none</span><span class="se">\"\n</span><span class="s2">}, {</span><span class="se">\n</span><span class="s2">  </span><span class="se">\"</span><span class="s2">port</span><span class="se">\"</span><span class="s2"> : 9093,</span><span class="se">\n</span><span class="s2">  </span><span class="se">\"</span><span class="s2">tls</span><span class="se">\"</span><span class="s2"> : true,</span><span class="se">\n</span><span class="s2">  </span><span class="se">\"</span><span class="s2">protocol</span><span class="se">\"</span><span class="s2"> : </span><span class="se">\"</span><span class="s2">kafka</span><span class="se">\"</span><span class="s2">,</span><span class="se">\n</span><span class="s2">  </span><span class="se">\"</span><span class="s2">auth</span><span class="se">\"</span><span class="s2"> : </span><span class="se">\"</span><span class="s2">none</span><span class="se">\"\n</span><span class="s2">} ]"</span>
  <span class="o">}</span>,
  ...
<span class="o">}</span>
</code></pre></div></div>

<p>This example demonstrates how Server-Side Apply allows Strimzi to reliably enforce the fields it owns, while safely coexisting with other actors managing the same resource.</p>

<h4 id="removal-of-fields">Removal of fields</h4>

<p>In Server-Side Apply, every actor have a possibility to remove the fields - but only those they manage.
That means, in case that Strimzi owns the <code class="language-plaintext highlighter-rouge">strimzi.io/discovery</code> annotation and we want to remove it with our <code class="language-plaintext highlighter-rouge">different-agent</code> field manager, the field will not be deleted after the update.
Only the <code class="language-plaintext highlighter-rouge">my.annotation/some</code> will be removed, as it is owned by <code class="language-plaintext highlighter-rouge">different-agent</code> field manager:</p>

<div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="o">&gt;</span> kubectl apply <span class="nt">--server-side</span> <span class="nt">--field-manager</span><span class="o">=</span>different-agent <span class="nt">-f</span> - <span class="o">&lt;&lt;</span><span class="no">EOF</span><span class="sh">
apiVersion: v1
kind: Service
metadata:
  annotations: {} # set annotations to null
  labels:
    app.kubernetes.io/instance: my-cluster
    app.kubernetes.io/managed-by: strimzi-cluster-operator
    app.kubernetes.io/name: kafka
    app.kubernetes.io/part-of: strimzi-my-cluster
    strimzi.io/cluster: my-cluster
    strimzi.io/component-type: kafka
    strimzi.io/discovery: "true"
    strimzi.io/kind: Kafka
    strimzi.io/name: my-cluster-kafka
  name: my-cluster-kafka-bootstrap
  namespace: test
spec:
  clusterIP: 10.97.174.54
  clusterIPs:
  - 10.97.174.54
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - name: tcp-replication
    port: 9091
    protocol: TCP
    targetPort: tcp-replication
  - name: tcp-clients
    port: 9092
    protocol: TCP
    targetPort: tcp-clients
  - name: tcp-clientstls
    port: 9093
    protocol: TCP
    targetPort: tcp-clientstls
  selector:
    strimzi.io/broker-role: "true"
    strimzi.io/cluster: my-cluster
    strimzi.io/kind: Kafka
    strimzi.io/name: my-cluster-kafka
  sessionAffinity: None
  type: ClusterIP
</span><span class="no">EOF
</span></code></pre></div></div>

<p>The <code class="language-plaintext highlighter-rouge">annotations</code> field in the <code class="language-plaintext highlighter-rouge">Service</code> after the apply looks like this:</p>
<div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="o">&gt;</span> kubectl get service my-cluster-kafka-bootstrap <span class="nt">-o</span> <span class="nv">jsonpath</span><span class="o">=</span><span class="s1">'{.metadata}'</span> | jq
<span class="o">{</span>
  <span class="s2">"annotations"</span>: <span class="o">{</span>
    <span class="s2">"strimzi.io/discovery"</span>: <span class="s2">"[ {</span><span class="se">\n</span><span class="s2">  </span><span class="se">\"</span><span class="s2">port</span><span class="se">\"</span><span class="s2"> : 9092,</span><span class="se">\n</span><span class="s2">  </span><span class="se">\"</span><span class="s2">tls</span><span class="se">\"</span><span class="s2"> : false,</span><span class="se">\n</span><span class="s2">  </span><span class="se">\"</span><span class="s2">protocol</span><span class="se">\"</span><span class="s2"> : </span><span class="se">\"</span><span class="s2">kafka</span><span class="se">\"</span><span class="s2">,</span><span class="se">\n</span><span class="s2">  </span><span class="se">\"</span><span class="s2">auth</span><span class="se">\"</span><span class="s2"> : </span><span class="se">\"</span><span class="s2">none</span><span class="se">\"\n</span><span class="s2">}, {</span><span class="se">\n</span><span class="s2">  </span><span class="se">\"</span><span class="s2">port</span><span class="se">\"</span><span class="s2"> : 9093,</span><span class="se">\n</span><span class="s2">  </span><span class="se">\"</span><span class="s2">tls</span><span class="se">\"</span><span class="s2"> : true,</span><span class="se">\n</span><span class="s2">  </span><span class="se">\"</span><span class="s2">protocol</span><span class="se">\"</span><span class="s2"> : </span><span class="se">\"</span><span class="s2">kafka</span><span class="se">\"</span><span class="s2">,</span><span class="se">\n</span><span class="s2">  </span><span class="se">\"</span><span class="s2">auth</span><span class="se">\"</span><span class="s2"> : </span><span class="se">\"</span><span class="s2">none</span><span class="se">\"\n</span><span class="s2">} ]"</span>
  <span class="o">}</span>,
  ...
<span class="o">}</span>
</code></pre></div></div>

<h3 id="conclusion">Conclusion</h3>

<p>In this blog post, we described Server-Side Apply, how Strimzi uses it, how to enable it, and how it can simplify working with Strimzi — especially in environments where multiple operators modify the same Kubernetes resources.
Although Server-Side Apply has been available in Strimzi since version 0.48.0, it is still in the alpha stage and ready for broader testing.
We plan to move it to beta (enabled by default) in next version - Strimzi 0.51.0.
In case that you will find any issue with the implementation, or have suggestions related to Server-Side Apply in Strimzi, you can share your feedback with us on <a href="https://slack.cncf.io/">Slack</a>, or by opening <a href="https://github.com/orgs/strimzi/discussions">a discussion</a> or <a href="https://github.com/strimzi/strimzi-kafka-operator/issues">an issue</a> on GitHub.</p>
