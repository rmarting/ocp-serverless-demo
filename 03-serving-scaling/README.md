# Serverless Serving - Autoscaling a Sample Application

By default Knative Serving allows 100 concurrent requests into a pod. This is defined by the
`container-concurrency-target-default` setting in the configmap `config-autoscaler` in
the `knative-serving` namespace.

## Sample Prime Generator

This service will be deployed to override the number of concurrency request to only 10. The
following annotation defines that behavior:

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: prime-generator
spec:
  template:
    metadata:
      annotations:
        # Target 10 in-flight-requests per pod.
        autoscaling.knative.dev/target: "10"
```

This will cause Knative autoscaler to scale to more pods as soon as we run more than
10 requests in parallel against the revision.

To deploy the application:

```bash
oc apply -f 03-serving-scaling/prime-generator.yml -n knative-demo-serving
```

Checking that service is up and ready:

```bash
❯ kn service list
NAME              URL                                                                             LATEST                  AGE   CONDITIONS   READY   REASON
greeter-cli       http://greeter-cli-knative-demo-serving.apps.sanes.sandbox852.opentlc.com       greeter-cli-hola        18m   3 OK / 3     True    
hello             http://hello-knative-demo-serving.apps.sanes.sandbox852.opentlc.com             hello-lg498             24m   3 OK / 3     True    
prime-generator   http://prime-generator-knative-demo-serving.apps.sanes.sandbox852.opentlc.com   prime-generator-prvvx   22s   3 OK / 3     True    
```

Test the service:

```bash
❯ curl "$(oc get routes.serving.knative.dev prime-generator | awk 'END{print $2}')?sleep=3&upto=10000&memload=100"
9973
```

We confirm that the autoscaling is working sending 50 concurrent requests (`-c 50`) for the next 30s (`-z 30s`):

```bash
hey -c 50 -z 30s "$(oc get routes.serving.knative.dev prime-generator | awk 'END{print $2}')?sleep=3&upto=10000&memload=100"
```

After you’ve successfully run this small load test, you will notice the number of prime generator service pods
will have scaled up automatically.

As we defined a maximum number of pods, if we increase the load, the number of pods are increased until the maximum
number is reached.

```bash
hey -c 150 -z 30s "$(oc get routes.serving.knative.dev prime-generator | awk 'END{print $2}')?sleep=3&upto=10000&memload=100"
```

## Scaling by CPU Costs

Knative Serving allows to scale using other metrics, e.g. cpu. To do that the following annotations should
be defined in the service:

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: prime-generator
spec:
  template:
    metadata:
      annotations:
        # Standard Kubernetes CPU-based autoscaling.
        autoscaling.knative.dev/class: hpa.autoscaling.knative.dev
        autoscaling.knative.dev/metric: cpu
        autoscaling.knative.dev/target: "60"
```

That configuration defines autoscaling when the cpu cost is upper than the `target` value.

To deploy a service with CPU autoscaling:

```bash
oc apply -f 03-serving-scaling/prime-generator-cpu.yml -n knative-demo-serving
```

**NOTE**: HPA does not support scale to zero functionality.

```bash
❯ oc get hpa
NAME                        REFERENCE                                         TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
prime-generator-cpu-6gx9d   Deployment/prime-generator-cpu-6gx9d-deployment   <unknown>/60%   1         5         1          77s
```

We will try to send 60 seconds of traffic with heavy CPU usage (~1 cpu/sec/request, total 100 cpus).

```bash
hey -c 100 -z 60s -q 100 "$(oc get routes.serving.knative.dev prime-generator-cpu | awk 'END{print $2}')?upto=40000000"
```

## Clean up Services

To remove all the services deployed in this namespace:

```bash
kn service delete --all
```

## References

* [Knative Doc - Autoscaling Concepts](https://knative.dev/docs/serving/autoscaling/autoscaling-concepts/)
* [OpenShift Serverless Doc - Configuring Knative Serving autoscaling](https://docs.openshift.com/container-platform/4.6/serverless/knative_serving/configuring-knative-serving-autoscaling.html)
