# ***Cloud Native Integration***: *A working* ***Demo*** 

This repository contains a working demonstration of how to quickly implement a Cloud Native architecture approach (with Enterprise Integration patterns as a central concern) as distilled in [Cloud Native Integration] (https://github.com/rh-ei-stp/cloud-native-integration). 

## Demo Description 

As we distill the high level system architecture as described by the *Cloud Native Integration* document ![Cloud Native Integration: The View From Space](/images/CloudNativeIntegration.00.00.001.png "Cloud Native Integration: From Space")

We notice a few distinct architectural layers: 
* **Event Mesh**
The *event mesh* intends to handle peer to peer event communication in a fashion that allows for several *Cloud Native* characteristics such as high availability, reliability, and location agnostic behaviour between peers. The event mesh acts as a rendezvous point between eventing peers (event emitters and event receivers), and provides event emitters a graph of event receivers that may span clusters or even clouds. 

* **Event Sink**
The *event sink* provides a port to the underlying event bus our integrations process events on. 

* **Event Bus**
The event bus provides a service bus so that event processors, that often need to integrate multiple different data sources to provide event aggregate level output, are able to communicate with each other in an asynchronous fashion. This decouples event emitters and receivers, as well as decoupling event aggregate behaviour that distills our series of events into meaningful business level data. As a result, integrations are bound to their domain, and are likely decomposed along their bounded context. It is along the *event bus* that features that constitute our enterprise perform the stream processing and integration work that satisfies enterprise features. Inevitably, these stream processors and integration components aggregate events into sources of truth to maintain consistency and state across the enterprise.   

* **Event Store** 
The event store represents a persistent source of truth for events and event aggregates. We define *aggregate event* as events that attempt to provide consistency boundaries for transactions, distributions and concurrency. In our view, while events may be represented in a transitive store such as Kafka, event aggregates define consistency across our data plane and represent a conceptual whole for state at a point in time for our domain entitties. As a result, our event store should be viewed as a conceptual whole that can offer capabilities as a source of truth for volatile, short term events, and a source of truth for long standing event aggregates.  

### Representing these Ports, Adapters and Layers

This demo seeks to display these architectural techniques by leveraging the Red Hat Integration platform. 

* **Event Mesh** 
To provide *event mesh* capabilities, this demo leverages *Apache Qpid Dispatch Router* to ensure a service and event communication control plane that provides capabilities for *high availability, resilience, multi and hybrid cloud* as well as policy to apply over the mesh to ensure governance is properly applied as our event emitters and event receivers communicate in a peer to peer fashion 

* **Event Sink** 
To provide an *event sink* as a port into our underlying business logic and sources of truth, we leverage *Apache Camel* and a set of complementary cloud native tooling

* **Event Bus** 
The event bus provides a normalized means of asynchronous behaviour for event consumers and emmitters. As *Cloud Native Integration* depends on and features cloud native capabilities, we leverage *Knative Eventing* to provide a cloud native service bus abstraction with *Apache Kafka* as the underlying persistence engine for our communication over our service bus channels. 

* **Event Store** 
As *Apache Kafka* is the persistence store for our volatile events and event aggregates that travel along our service bus, this demonstration relies on a traditional OLTP store as the inevitable source of truth for event aggregates. While an OLTP store is not required as a source of truth for event aggregates in our view of the world, it describes the complementary nature of *Cloud Native Integration* to traditional legacy enterprise deployments. 

## Getting Started 

*Quick Note: This demonstration uses Openshift 4.x which relies on Kubernetes 1.16 and above. As a result, despite the use of Operator Hub and Red Hat disitributed Operators, the steps outlined in this document may be followed in any Kubernetes distro that is 1.16 or higher and a matching community version of the operators being deployed* 

### Installing Operators 

From the Openshift Operator Hub, install the following operators (in our case, we'll install these operators with *cluster admin* credentials, and will allow these operators to observe all namespaces). 
* Red Hat Integration - AMQ Streams 
* Red Hat Integration - AMQ Certificate Manager 
* Red Hat Integration - AMQ Interconnect (this operator will need to be installed in multiple namespaces due to its limitation of watching a single K8s namespace)
* Openshift Serverless Operator 
* Camel K Operator 

We should now find Operators running in the OpenshiftOperators namespace: 
![OpenshiftOperators](/images/openshift-operator-ns-initial.png)

### Preparing For Deployment 

At this point, we have installed the required operators; however, our environment still isn't ready to start laying down deployments. 

#### Install Knative Serving 
To install Knative Serving which provides our serverless framework by the OpenshiftServerless Operator, there are a few more steps. 

* Create the knative-serving namespace with a user that posseses the cluster-admin role: 

```
oc create namespace knative-serving
```
* Apply the knative serving yaml to the cluster as described in the following file ![Knative Serving](/src/main/k8s/knative-serving/knative-serving-cr.yaml)

```
apiVersion: operator.knative.dev/v1alpha1
kind: KnativeServing
metadata:
    name: knative-serving
    namespace: knative-serving
``` 

Upon succesfful installation, there should be a similar result to the following: 

```
oc get pods -w -n knative-serving 
NAME                                READY     STATUS    RESTARTS   AGE
activator-7db4dc788c-spsxb          1/1       Running   0          1m
activator-7db4dc788c-zqnnd          1/1       Running   0          45s
autoscaler-659dc48d89-swtmn         1/1       Running   0          59s
autoscaler-hpa-57fdfbb45c-hsd54     1/1       Running   0          49s
autoscaler-hpa-57fdfbb45c-nqz5c     1/1       Running   0          49s
controller-856b4bd96d-29xlv         1/1       Running   0          54s
controller-856b4bd96d-4c7nn         1/1       Running   0          46s
kn-cli-downloads-7558874f44-qdr99   1/1       Running   0          1m
webhook-7d9644cb4-8xkzt             1/1       Running   0          57s
```

#### Install Knative Eventing 
Knative Eventing provides functionality around our Cloud Event abstraction and forms the operational basis of our cloud native service bus. 

To install Knative Eventing we need to perform the following steps (with a cluster admin user): 
* Create the knative-eventing namespace 

```
oc create namespace knative-eventing
```

Upon creation of the namespace, we will want to create the knative eventing operators by applying the following cr ![Knative Eventing CR](/src/main/k8s/knative-eventing/knative-eventing-cr.yaml)

In our case, as we'll have need later, we'll install the Multi Tenant Channel Based Broker as our default Knative Eventing Broker implementation: 

```
apiVersion: operator.knative.dev/v1alpha1
kind: KnativeEventing
metadata:
  name: knative-eventing
  namespace: knative-eventing
spec:
  defaultBrokerClass: MTChannelBasedBroker
```

Upon applying this yaml, something similar to this should be true: 

```
oc get pods -w -n knative-eventing 
NAME                                   READY     STATUS    RESTARTS   AGE
broker-controller-67b56668bd-sgxwg     1/1       Running   0          1m
eventing-controller-544dc9945d-pz2cl   1/1       Running   0          1m
eventing-webhook-6c774678b5-lzfkn      1/1       Running   0          1m
imc-controller-78b8566465-smpb7        1/1       Running   0          1m
imc-dispatcher-57869b44c5-s7t92        1/1       Running   0          1m
```

At this point, we have installed the knative eventing controllers, dispatchers and webhooks; however, we only have support for the in memory channel, which means we will not be able to use a persistent approach to knative eventing brokers and their respective channels in our cluster. 

##### Installing Kafka Channels
In our above *Cloud Native Integration* schematic, what lies at the heart of our event bus is *Apache Kafka* so for knative eventing to follow this architectural construct, we need our broker channels to be persisted by Kafka. While this component is not generally available, it has reached a considerable maturity level and relies largely on its surrounding ecosystem that is generally available. 

In this demonstration, we will deploy a KafkaChannel deployment that creates a set of CR's, controllers for our channels, requisite service accounts, and webhooks to instrument creation, removal and discovery of KafkaChannels that may be associated with a Knative eventing Broker. 

By issuing the following command (this assumes cluster admin permissions for the user issuing commands and that the previous knative-eventing steps have taken place): 

``` 
oc apply -f ./src/main/install/kafka-channel/kafka-channel-install.yaml
clusterrole.rbac.authorization.k8s.io/kafka-addressable-resolver created
clusterrole.rbac.authorization.k8s.io/kafka-channelable-manipulator created
clusterrole.rbac.authorization.k8s.io/kafka-ch-controller created
serviceaccount/kafka-ch-controller created
clusterrole.rbac.authorization.k8s.io/kafka-ch-dispatcher created
serviceaccount/kafka-ch-dispatcher created
clusterrole.rbac.authorization.k8s.io/kafka-webhook created
serviceaccount/kafka-webhook created
clusterrolebinding.rbac.authorization.k8s.io/kafka-ch-controller created
clusterrolebinding.rbac.authorization.k8s.io/kafka-ch-dispatcher created
clusterrolebinding.rbac.authorization.k8s.io/kafka-webhook created
customresourcedefinition.apiextensions.k8s.io/kafkachannels.messaging.knative.dev created
configmap/config-kafka created
configmap/config-leader-election-kafka created
service/kafka-webhook created
deployment.apps/kafka-ch-controller created
mutatingwebhookconfiguration.admissionregistration.k8s.io/defaulting.webhook.kafka.messaging.knative.dev created
validatingwebhookconfiguration.admissionregistration.k8s.io/validation.webhook.kafka.messaging.knative.dev created
secret/messaging-webhook-certs created
deployment.apps/kafka-webhook created

```

We should now see new contoller, dispatcher, and webhook pods in our knative-eventing namespace: 

```
NAME                                   READY     STATUS    RESTARTS   AGE
broker-controller-67b56668bd-sgxwg     1/1       Running   0          1h
eventing-controller-544dc9945d-pz2cl   1/1       Running   0          1h
eventing-webhook-6c774678b5-lzfkn      1/1       Running   0          1h
imc-controller-78b8566465-smpb7        1/1       Running   0          1h
imc-dispatcher-57869b44c5-s7t92        1/1       Running   0          1h
kafka-ch-controller-7f88b8c776-crhm4   1/1       Running   0          1m
kafka-webhook-b47dc9767-hmnm4          1/1       Running   0          1m
```

We should also now have a resource that describes resources that will inevitably describe how channels are bound to Kafka topics in our Knative Eventing framework. 

```
oc api-resources | grep messaging.knative.dev
channels                              ch               messaging.knative.dev                 true         Channel
inmemorychannels                      imc              messaging.knative.dev                 true         InMemoryChannel
kafkachannels                         kc               messaging.knative.dev                 true         KafkaChannel
subscriptions                         sub              messaging.knative.dev                 true         Subscription
```

If we have gotten this far we have most of our infrastructure assembled from an operator perspective and now its time to being installing our concrete implementation. 

## Installing the Demo 

At this point our operators are installed and we have mostly configured our environment; however, we still have a few house keeping tasks to take care of: 
* We need to create and install a trust store so that we may communicate to and between clusters in a secure fashion 
* We need to ensure proper trust and authority is distributed to the correct places in our cluster 

### Using the AMQ Certificate Manager Operator 

We will use the [cert-manager](https://cert-manager.io) operator that we've previously provisioned to issue certificates that we'll need to wire secure connections across the cluster. 

#### Creating a CA to use across the cluster 

For our puroposes we will create a Certificate Authority using OpenSSL. Initially, the following command should be issued to generate the private key for our CA

```
openssl  genrsa -des3 -out cloudEventMeshDemoCA.key 2048
```
OpenSSL will prompt for a passphrase. It is recommended to use a passphrase even in a development environment, and especially when cluster resources may be accessible from outside of the cluster. 

We'll use the key to create a root certificate that will act as our certificate authority: 

```
openssl req -x509 -new -nodes -key cloudEventMeshDemoCA.key -sha256 -days 1825 -out cloudEventMeshDemoCA.crt
```

At this point, you will be asked for a passphrase for again, and as always, it is apropos to use one and not skip this step. While creating this CA OpenSSL will ask for OU's, DN's, etc., and it may be important to use meaningful values for these as it may be required during later configuration. 

Upon completing this process, we will now have an available CA to sign with, and we'll use the AMQ distribution of the certificate manager to establish our CA as a certificate issuer across the cluster. 

For convenience sake, we have included a secret in this demo that we should apply to where our certificate manager operator lives (in our case the project *openshift-operators*)

```
oc apply -f ./src/main/k8s/CA/cloud/cloud-native-event-mesh-ca-secret.yaml
```
This could also be accomplished by creating a secret from the the CA private and public key created above. 

Now that we have created our CA keypair secret, let's create a certificate manager issuer that uses our CA: 

```
apiVersion: certmanager.k8s.io/v1alpha1
kind: ClusterIssuer
metadata:
  name: cloud-native-event-mesh-demo-cert-issuer
spec:
  ca:
    secretName: cloud-native-event-mesh-demo-ca-pair

```

Upon completing our application of the clusterissuer resource we should have a cluster issuer. 

```py
oc describe clusterissuer cloud-native-event-mesh-demo-cert-issuer 
Name:         cloud-native-event-mesh-demo-cert-issuer
Namespace:    
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"certmanager.k8s.io/v1alpha1","kind":"ClusterIssuer","metadata":{"annotations":{},"name":"cloud-native-event-mesh-demo-cert-issuer","name...
API Version:  certmanager.k8s.io/v1alpha1
Kind:         ClusterIssuer
Metadata:
  Creation Timestamp:  2020-06-29T17:11:47Z
  Generation:          2
  Resource Version:    1485153
  Self Link:           /apis/certmanager.k8s.io/v1alpha1/clusterissuers/cloud-native-event-mesh-demo-cert-issuer
  UID:                 90035a64-67c1-4440-9526-a87d1297bfa2
Spec:
  Ca:
    Secret Name:  test-key-pair
Status:
  Conditions:
    Last Transition Time:  2020-06-29T17:11:47Z
    Message:               Signing CA verified
    Reason:                KeyPairVerified
    Status:                True
    Type:                  Ready
Events:
  Type    Reason           Age              From          Message
  ----    ------           ----             ----          -------
  Normal  KeyPairVerified  8s (x2 over 8s)  cert-manager  Signing CA verified
```

We are now free to issue certificates in our cluster. This will be critical to setting up our next steps, the event mesh. 

### Installing the Event Mesh 

Our event mesh will span three different namespaces in our cluster; however, our intention is to logically represent three seperate clusters in our topology. As a result, we will need to create trust in each of these namespaces for the other routers in our event mesh, as, we would not allow insecure communication from cluster to cluster. 

#### Creating the namespaces 

At this point we will wand to create the following namespaces: 
* *cluster-1* 
* *cluster-2* 
* *edge* - this will represent an edge cluster in our topology. While some topologies may not call for an edge cluster, we still want to ensure that we use an *edge router* somewhere in our openshift cluster to ensure that we have connection concentration, a single source of policy application to incoming requests from outside of the cluster, as well as a terminal leaf node for our event mesh graph. 

##### Installing the Interconnect Router in *cluster-1*

As we have use of the operator hub in Openshift 4, we will simply install an Interconnect Operator to the namespace "cluster-1". 
![Installing the Interconnect Operator in Cluster 1](/images/interconnect-cluster-1.png "Installing the Interconnect Operator to Cluster-1")

At this point, we will be able to start creating some certificates from the cluster issuer we have already established and inevitably configure our Interconnect router for trust. 
In the namespace "cluster-1", we will provision a certificate for our Interconnect router which we will use to wire up the Interconnect router for trust across inter-router connections. 

*Please Note*: The interconnect router has self-signed a certificate using the certificate manager, and the demonstrated use of certificate management here is only applicable to inter-router conncetions

Initially, we'll lay down the certificate request custom resource: 

```py
oc apply -f ./src/main/k8s/cloud-1/router/cloud1-certificate-request.yaml
```
This certificate request: 

```yaml
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: cluster-wide-tls
spec:
  secretName: cluster-wide-tls
  commonName: openshift.com
  issuerRef:
    name: cloud-native-event-mesh-demo-cert-issuer
    kind: ClusterIssuer
  dnsNames: 
     - openshift.com
     - eipractice.com
     - opentlc.com

```

Will leverage the certificate manager to create a secret in the cluter referred to as *"cluster-wide-tls"* which holds the CA certificate, private key, and other things to ensure trust as issued by the ClusterIssuer we have created above. 

Upon issuing the certificate, it is now time to apply our Interconnect router custom resource for the cluster-1 namespace: 

```
oc apply -f ./src/main/k8s/cloud-1/router/cloud1-mesh-router.yaml 
```

This enables a few features as laid out by the Interconnect custom resource: 

```
apiVersion: interconnectedcloud.github.io/v1alpha1
kind: Interconnect
metadata:
  name: cloud1-router-mesh
spec:
   sslProfiles:
   - name: inter-router
     credentials: cluster-wide-tls
     caCert: cluster-wide-tls
   deploymentPlan: 
      role: interior 
      size: 1
      placement: AntiAffinity
   interRouterListener: 
      sslProfile: cloud1-router-tls
      expose: false
      authenticatePeer: false
      port: 55671
```

This creates an SSL Profile based on our CA certificate as issued to us by the cluster issuer as well as establishes an interRouterListener backed by this SSL Profile. The router will also configure other things by default, and with self-signed security, such as: 
* A default AMQP listener secured by a *self-signed* certificate
* Metrics and management capabilities 
* A means for Prometheus or other ***AMP*** technologies to observe the event mesh router as well as the link attachments that event peers make over the event mesh 

At this point, in cluster-1 we will have both an event mesh router and the Interconnect operator in our *"cluster-1"* namespace: 

```
[mcostell@work router]$ oc get pods -w -n cluster-1
NAME                                    READY     STATUS    RESTARTS   AGE
cloud1-router-mesh-7f698d8c65-wpnx7     1/1       Running   0          13m
interconnect-operator-56b7884d4-6j4jl   1/1       Running   0          5h

 oc get interconnects 
NAME                 AGE
cloud1-router-mesh   17m
```
We can also introspect the router via oc exec commands: 

```
[mcostell@work router]$ oc exec cloud1-router-mesh-7f698d8c65-wpnx7 -i -t -- qdstat -g
2020-06-29 23:23:49.882216 UTC
cloud1-router-mesh-7f698d8c65-wpnx7

Router Statistics
  attr                             value
  ========================================================================================
  Version                          Red Hat AMQ Interconnect 1.8.0 (qpid-dispatch 1.12.0)
  Mode                             interior
  Router Id                        cloud1-router-mesh-7f698d8c65-wpnx7
  Worker Threads                   4
  Uptime                           000:00:22:17
  VmSize                           497 MiB
  Area                             0
  Link Routes                      0
  Auto Links                       0
  Links                            2
  Nodes                            0
  Addresses                        11
  Connections                      1
  Presettled Count                 0
  Dropped Presettled Count         0
  Accepted Count                   24
  Rejected Count                   0
  Released Count                   0
  Modified Count                   0
  Deliveries Delayed > 1sec        0
  Deliveries Delayed > 10sec       0
  Deliveries Stuck > 10sec         0
  Deliveries to Fallback           0
  Links Blocked                    0
  Ingress Count                    24
  Egress Count                     23
  Transit Count                    0
  Deliveries from Route Container  0
  Deliveries to Route Container    0

```
##### Installing the Interconnect Router in *cluster-2*
Now that we have an event mesh router in *cluster-1*, we will link routers together to form an event mesh between *cluster-1* and *cluster-2*. 
*Please Note* - the intention of this logical delineation is to represent 2 seperate clusters. In practice, *interior event mesh* routers would bind remote clusters together 

The process of enabling inter-router connections between event mesh routers in namespaces *cluster-1* and *cluster-2* is similar to the provisioning that was required for the *cluster-1* event mesh router. 

Initially, we want to install the *Red Hat - Interconnect Operator* in the namespace *cluster-2*. As *cluster-2* will also issue its certificates from the *cluster-issuer* provisioned previously in the demo, upon installation of the Interconnect Operator, we will provision a CA certificate for *cluster-2* via a certificate request from the *ClusterIssuer*. 

```py 
oc apply -f src/main/k8s/cloud-2/router/cloud2-certificate-request.yaml
```

This certificate request: 

```py 
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: cluster-wide-tls
spec:
  secretName: cluster-wide-tls
  commonName: openshift.com
  issuerRef:
    name: cloud-native-event-mesh-demo-cert-issuer
    kind: ClusterIssuer
  dnsNames: 
     - openshift.com
     - eipractice.com
     - opentlc.com
```

Will again provision our CA into a secret into the cluster named "cluster-wide-tls". 

Upon the certificate manager cluster issuer issuing a CA into *cluster-2*, we can lay down the Interconnect resource that will enable our *cluster-1* and *cluster-2* event meshing. 

```py
oc apply -f src/main/k8s/cloud-2/router/cloud2-certificate-request.yaml
```
The router provisioned takes mostly default values for the Interconnect configuration; however, does create an interior router connection to the event mesh router in *cluster-1*: 

```py 
apiVersion: interconnectedcloud.github.io/v1alpha1
kind: Interconnect
metadata:
  name: cloud2-router-mesh
spec:
  sslProfiles:
  - name: inter-cluster-tls
    credentials: cluster-wide-tls
    caCert: cluster-wide-tls
  interRouterConnectors:
  - host: cloud1-router-mesh.cluster-1.svc
    port: 55671
    verifyHostname: false
    sslProfile: inter-cluster-tls
```

Upon succesful provisioning of the router in *cluster-2* we should be able to see a successfull interior router connection between the routers in *cluster-1* and *cluster-2*:

```py 
oc exec cloud2-router-mesh-d566476ff-msdrr -i -t -- qdstat -c 
2020-06-29 23:37:04.652856 UTC
cloud2-router-mesh-d566476ff-msdrr

Connections
  id  host                                    container                             role          dir  security                                authentication  tenant  last dlv      uptime
  =================================================================================================================================================================================================
  2   cloud1-router-mesh.cluster-1.svc:55671  cloud1-router-mesh-7f698d8c65-wpnx7   inter-router  out  TLSv1/SSLv3(DHE-RSA-AES256-GCM-SHA384)  x.509                   000:00:00:00  000:00:00:54
  9   127.0.0.1:33562                         ae807937-90d0-44d7-8f7b-afdc14f5c47a  normal        in   no-security                             no-auth                 000:00:00:00  000:00:00:00
```

##### Installing the Edge Interconnect Router
At this point we have a mesh of Interconnect routers in *cluster-1* and *cluster-2*; however, to properly be able to scale our router network and provide a connection concentrator for our eventing applications, we will want to establish a member of our event mesh to have the role of "edge" in our cluster. Edge routers act as connection concentrators for messaging applications. Each edge router maintains a single uplink connection to an interior router, and messaging applications connect to the edge routers to send and receive messages.

Initially, we will to create a namespace named ***"edge"***. ***Please Note***: in practice, our edge router would likely live in a seperate cluster meant to handle edge use cases, and would be seperate from *cluster-1* and *cluster-2*. 

Upon succesfull creation of the namespace, it is neccessarry to install the Interconnect Operator into this namespace. This will allow us to provision our *edge* router resource. 

Initially, as with our other logical *clusters* we will provision our edge namespace with a CA certificate: 

```py
oc apply -f src/main/k8s/edge/edge-certificate-request.yaml
```

Upon creating our *cluster-wide-tls* secret in the namespace, we'll provision our Interconnect Router which will have uplinks to both *cluster-1* and *cluster-2*: 

```py 
oc apply -f src/main/k8s/edge/edge-routers.yaml
```

This will provision our *edge* event mesh router, to perform a role as a terminal node in our event mesh. 

```py
apiVersion: interconnectedcloud.github.io/v1alpha1
kind: Interconnect
metadata:
  name: edge-routers
spec:
  sslProfiles:
   - name: edge-router-tls
     credentials: cluster-wide-tls
     caCert: cluster-wide-tls
  deploymentPlan:
    role: edge
    placement: AntiAffinity
  edgeConnectors:  
    - host: cloud1-router-mesh.cloud-1
      port: 45672
      name: cloud1-edge-connector
    - host: cloud2-router-mesh.cloud-2
      port: 45672
      name: cloud2-edge-connector
  listeners: 
    - sslProfile: cluster-wide-tls
      authenticatePeer: false
      expose: true
      http: false
      port: 5671
```

If we were to peruse the Interconnect console for one of our clusters, we would be able to see our two interior routers fronted by a single edge router: 
![Initial Event Mesh Topology](/images/event-mesh-initial-topology.png)

###Installing the Event Bus

***Cloud Native Integration*** creates an event mesh through which event emitters and receivers are able to negotiate with each other, get guarantees around delivery, and ensure proper communication flow via Interconnects wire level flow control capabilities. 

While the *event mesh* serves to provide a communications control plane for event emitters and receivers, through which event level qos can be applied to its relevant use cases, and a reliable graph of receivers for emitted events, ***Cloud Native Integration*** introduces a complementary persistent event bus, as an event sink for event stream processing and integration from the event mesh. This event bus provides a persistent source of *truth* for event stream processors, and allows event receivers to take advantage of platform capabilities such as elasticity, contractual communication, and extend those capabilities into serverless capabilities such as *scaling to zero* and other highly elastic behaviour. 

To accomplish this ***Cloud Native Integration*** relies on *Knative Eventing*, and *Knative Serving* from the *Openshift Serverless* Operator that we deployed earlier. 

As "Knative Eventing" proposes a pub-sub architecture with per namespace Brokers and subscriptions as described by:
![Knative Broker Triggers](/images/broker-trigger-overview.svg)
 
It will be neccessarry to provision a persistent source of truth for our Knative event brokers. ***Cloud Native Integration*** proposes the use of AMQ Streams as this persistent source of truth as it offers two features that uniquely assist in our pub-sub abstraction: 
* A co-ordinated distributed log that replicates across distinct physical parts of an Openshift cluster 
* A means of providing distributed log partitions to ephemeral consumer groups via the Knative channels abstraction and a Kafka Topic implementation 

As the *Integrations* that handle our events consume from our underlying event bus via Knative subscriptions to Knative event channels, AMQ Streams provides a *cloud native* means to persist these events: 
![Knative Broker Channels](/images/control-plane.png)

#### Provisioning AMQ Streams

For our use case in a single cluster, we will simply provision a single AMQ Streams cluster for both *cluster-1* and *cluster-2* event bus consumers; however, in practice, it would be apropos to at minimum extend the mult-tenant features of this demo to ensure the use of distinct/multi-tenant physical Kafka clusters. 

As we have already installed the AMQ Streams Operator cluster wide, we will simply create a namespace for our AMQ Streams cluster to live in. Let's call it "amq-streams". 

```py 
oc create namespace amq-streams
```

Upon succesful creation of the namespace, we will apply our AMQ Streams custom resource to create an AMQ Streams cluster in the namespace called *"small-event-cluster"*: 

```py
oc apply -f src/main/k8s/cloud-1/kafka/small-event-cluster.yaml
```

Upon succesfful creation of our resources, we should see our *"amq-streams"* cluster with some new pods: 

```py
oc get pods -w -n amq-streams 
NAME                                                   READY     STATUS    RESTARTS   AGE
small-event-cluster-entity-operator-7cb59948f7-r22kp   3/3       Running   0          33s
small-event-cluster-kafka-0                            2/2       Running   1          1m
small-event-cluster-zookeeper-0                        1/1       Running   0          2m
small-event-cluster-zookeeper-1                        1/1       Running   0          2m
small-event-cluster-zookeeper-2                        1/1       Running   0          2m
```

At this point we have provisioned AMQ Streams but we need to install Knative Eventing and Knative Serving so that AMQ Streams is the backbone of our enterprise service bus abstraction. 

#### Provisioning Knative Serving with the Openshift Serverless Operator 
As we have already installed the *Openshift Serverless Operator* in a previous step, we are now free to create and configure Knative Serving as a means of abstraction for our serverless enpoint calls. 

Initially, we'll create a *"Knative Serving"* namespace and then provision our simple *Knative Serving* resource: 

```py 
oc apply -f src/main/k8s/knative-serving/knative-serving-cr.yaml -n knative-serving
```

