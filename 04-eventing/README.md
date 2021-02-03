# Serverless Eventing - Deploying a Sample Application

Create a new project to deploy our Serverless applications using Eventing APIs:

```bash
oc new-project knative-demo-eventing
```

Deploy an application to display the events produced:

```bash
oc apply -f 04-eventing/event-display-service.yml
```

Deploy the broker to manage the events:

```bash
kn broker create default
```

To list the current brokers:

```bash
❯ kn broker list
NAME      URL                                                                                      AGE   CONDITIONS   READY   REASON
default   http://broker-ingress.knative-eventing.svc.cluster.local/knative-demo-eventing/default   7s    5 OK / 5     True    
```

To subscribe the application `event-display` with events managed by the broker, we need to deploy a `trigger`:

```bash
oc apply -f 04-eventing/trigger.yml
```

```bash
❯ kn trigger list
NAME             BROKER    SINK                 AGE   CONDITIONS   READY   REASON
sample-trigger   default   ksvc:event-display   4s    5 OK / 5     True   
```

The `channel` is the platform to forward and persistence the events. To deploy a simple in memory channel:

```bash
oc apply -f 04-eventing/channel.yml
```

```bash
❯ kn channel list
NAME             TYPE              URL                                                                        AGE   READY   REASON
sample-channel   InMemoryChannel   http://sample-channel-kn-channel.knative-demo-eventing.svc.cluster.local   6s    True    
```

Finally we create a `subscription` between the channel and the `event-display` application to process
the events sent to the `channel`:

```bash
oc apply -f 04-eventing/subscription.yml
```

### Camel K Components

To test the eventing we will use a Camel K integration to send events to the broker and processed by the `event-display` service.

Camel-K will enable you to craft a Kubernetes-native integration application. It also leverages serverless capabilities via Knative.

With Camel-K we can also exchange Knative Eventing events via Apache Camel endpoints; where the exchange messages
(event payloads) are automatically transformed into Cloud Events format and exchanged with other consumers.

To deploy Knative Apache Camel Operator:

```bash
oc apply -f 04-eventing/camel-k-operator/knative-apache-camel-subscription.yml
```

Check the operators installed:

```bash
❯ oc get csv
NAME                             DISPLAY                         VERSION   REPLACES                         PHASE
camel-k-operator.v1.3.0          Camel K Operator                1.3.0     camel-k-operator.v1.2.1          Succeeded
knative-camel-operator.v0.18.0   Knative Apache Camel Operator   0.18.0    knative-camel-operator.v0.15.1   Succeeded
serverless-operator.v1.12.0      Red Hat OpenShift Serverless    1.12.0    serverless-operator.v1.11.0      Succeeded
```

Use Camel-K to send events each number of seconds:

```bash
oc apply -f 04-eventing/camel-k/timer-publisher.yml
```

After some minutes the camel route will send messages to the broker as CloudEvents

```bash
❯ oc logs -f camel-source-timer-publisher-cndxv-6fc95944bc-qfr9n
...
2021-02-03 10:13:08,969 INFO  [io.quarkus] (main) Installed features: [camel-bean, camel-core, camel-endpointdsl, camel-k-core, camel-k-knative, camel-k-knative-producer, camel-k-loader-yaml, camel-k-runtime, camel-main, camel-support-common, camel-timer, cdi]
2021-02-03 10:13:09,969 INFO  [route1] (Camel (camel-1) thread #0 - timer://tick) Sending message 'HELLO EVENT IN A SERVERLESS WORLD!'
2021-02-03 10:13:14,962 INFO  [route1] (Camel (camel-1) thread #0 - timer://tick) Sending message 'HELLO EVENT IN A SERVERLESS WORLD!'
...
```

Our application will display that new events sent:

```bash
❯ oc logs -f event-display-pk9kh-deployment-54787d94df-4k8nb -c user-container
☁️  cloudevents.Event
Validation: valid
Context Attributes,
  specversion: 1.0
  type: org.apache.camel.event
  source: camel-source:knative-demo-eventing/camel-source-timer-publisher
  id: 4CEEC1ACAB2F076-0000000000000000
  time: 2021-02-03T10:13:09.963Z
  datacontenttype: text/plain
Data,
  HELLO EVENT IN A SERVERLESS WORLD!
☁️  cloudevents.Event
Validation: valid
Context Attributes,
  specversion: 1.0
  type: org.apache.camel.event
  source: camel-source:knative-demo-eventing/camel-source-timer-publisher
  id: 4CEEC1ACAB2F076-0000000000000001
  time: 2021-02-03T10:13:14.961Z
  datacontenttype: text/plain
Data,
  HELLO EVENT IN A SERVERLESS WORLD!
```

## Clean up Services

To remove all the services deployed in this namespace:

```bash
kn service delete --all
kn broker delete --all
kn trigger delete --all
kn subscription delete --all
```

## References

* [Event delivery workflows using brokers and triggers](https://docs.openshift.com/container-platform/4.6/serverless/event_workflows/serverless-using-brokers.html)
* [Knative Tutorial - Camel K Eventing](https://redhat-developer-demos.github.io/knative-tutorial/knative-tutorial/camelk/camel-k-eventing.html)
