# Serverless Eventing - Using Apache Kafka as Eventing Broker

**NOTE:** This is an advanced version of the previous sample using Apache Kafka
as broker instead of a In Memory Channel.

If you deleted the `event-display` service, deploy again:

```bash
oc apply -f 04-eventing/event-display-service.yml
```

To use Apache Kafka as a persistence broker for Serverless Eventing services, we
need to deploy Red Hat AMQ Streams Operator:

```bash
oc apply -f 05-eventing-kafka/amq-streams-operator/amq-streams-subscription.yml
```

Checking the AMQ Streams Operator is installed successfully:

```bash
❯ oc get csv
NAME                             DISPLAY                             VERSION   REPLACES                         PHASE
amqstreams.v1.6.2                Red Hat Integration - AMQ Streams   1.6.2     amqstreams.v1.6.1                Succeeded
camel-k-operator.v1.3.0          Camel K Operator                    1.3.0     camel-k-operator.v1.2.1          Succeeded
knative-camel-operator.v0.18.0   Knative Apache Camel Operator       0.18.0    knative-camel-operator.v0.15.1   Succeeded
serverless-operator.v1.12.0      Red Hat OpenShift Serverless        1.12.0    serverless-operator.v1.11.0      Succeeded
```

Deploy an Apache Kafka cluster to manage the events:

```bash
oc apply -f 05-eventing-kafka/kafka/event-bus-kafka.yml
```

Deploy a Kafka topic to send the events to be processed by the `event-display` service:

```bash
oc apply -f 05-eventing-kafka/topics/events-topic.yml
```

Enable Serverless Eventing to manage Kafka cluster deploying a `KnativeKafka` definition using
the current Apache Kafka cluster deployed:

```bash
oc apply -f 05-eventing-kafka/knative-kafka.yml -n knative-eventing
```

`KafkaSource` is a new source definition to consume events. If it is not available you could
add it using the following command:

```bash
oc apply -f https://github.com/knative/eventing-contrib/releases/download/v0.18.8/kafka-source.yaml
```

Deploy Kafka Source

```bash
oc apply -f 05-eventing-kafka/kafka-source.yml
```

Now everytime a new message is sent to `events` topic, the message will be processed by a `event-display`
instance:

To send messages to events topic

```bash
❯ oc run kafka-producer -ti \
 --image=registry.redhat.io/amq7/amq-streams-kafka-25-rhel7:1.6.2 \
 --rm=true --restart=Never \
 -- bin/kafka-console-producer.sh \
 --broker-list event-bus-kafka-kafka-bootstrap.knative-demo-eventing:9092 \
 --topic events
If you don't see a command prompt, try pressing enter.
>Hello Serverless World from Apache Kafka!
```

Checking that the event was processed by the `event-display` service:

```bash
❯ oc logs -f event-display-64vkz-deployment-6dd7676664-6j7sz -c user-container
☁️  cloudevents.Event
Validation: valid
Context Attributes,
  specversion: 1.0
  type: dev.knative.kafka.event
  source: /apis/v1/namespaces/knative-demo-eventing/kafkasources/kafka-source#events
  subject: partition:0#1
  id: partition:0/offset:1
  time: 2021-02-03T14:46:48.376Z
Extensions,
  traceparent: 00-58cddc96c8c32beb14f2f1b84afdc144-61fe1778e98d1cd3-00
Data,
  Hello Serverless World from Apache Kafka!
```

## Clean up Services

To remove all the services deployed in this namespace:

```bash
kn service delete --all
oc delete kafkasource --all
```

## References

* [Using Apache Kafka with OpenShift Serverless](https://docs.openshift.com/container-platform/4.6/serverless/serverless-kafka.html)
* [Using a Kafka source](https://docs.openshift.com/container-platform/4.6/serverless/event_sources/serverless-kafka-source.html#serverless-kafka-source)
