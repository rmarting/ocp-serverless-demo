# Serverless Serving - Deploying a Sample Application

Create a new project to deploy our Serverless applications using Serving APIs:

```bash
oc new-project knative-demo-serving
```

## Sample Hello Serverless

Sample Hello World application:

```bash
oc apply -f 02-serving/hello-service.yml -n knative-demo-serving
```

To test the application:

```bash
❯ curl $(oc get routes.serving.knative.dev hello | awk 'END{print $2}')
Hello Serverless!
```

This output shows how the pod are created and after a while, when there is no traffic,
the pods are destroyed (scaled down to zero).

```bash
❯ oc get pod -w
NAME                                      READY   STATUS    RESTARTS   AGE
hello-67rc9-deployment-5dcf679984-w4vhl   0/2     Pending   0          0s
hello-67rc9-deployment-5dcf679984-w4vhl   0/2     Pending   0          0s
hello-67rc9-deployment-5dcf679984-w4vhl   0/2     ContainerCreating   0          0s
hello-67rc9-deployment-5dcf679984-w4vhl   0/2     ContainerCreating   0          2s
hello-67rc9-deployment-5dcf679984-w4vhl   1/2     Running             0          3s
hello-67rc9-deployment-5dcf679984-w4vhl   2/2     Running             0          9s
hello-67rc9-deployment-5dcf679984-w4vhl   2/2     Terminating         0          69s
hello-67rc9-deployment-5dcf679984-w4vhl   1/2     Terminating         0          89s
hello-67rc9-deployment-5dcf679984-w4vhl   0/2     Terminating         0          115s
hello-67rc9-deployment-5dcf679984-w4vhl   0/2     Terminating         0          116s
hello-67rc9-deployment-5dcf679984-w4vhl   0/2     Terminating         0          116s
```

## Deploying Services using `kn` client

To get the latest version of the `kn` cli, please, download from [here](https://mirror.openshift.com/pub/openshift-v4/clients/serverless/latest)

### Listing services

The new application could be checked using ```kn``` CLI as:

```bash
❯ kn service list
NAME      URL                                                                           LATEST        AGE     CONDITIONS   READY   REASON
hello     http://hello-knative-serving-sample-app.apps.sanes.sandbox852.opentlc.com     hello-67rc9   8m16s   3 OK / 3     True   
```

### Creating a new service

To create a new service:

```bash
kn service create greeter-cli \
  --image quay.io/rhdevelopers/knative-tutorial-greeter:quarkus \
  --env "MESSAGE_PREFIX=hello" \
  --revision-name greeter-cli-hello \
  -n knative-demo-serving
```

A sample output of that command should be similar:

```text
Creating service 'greeter-cli' in namespace 'knative-serving-sample-app':

  0.055s The Configuration is still working to reflect the latest desired specification.
  0.149s The Route is still working to reflect the latest desired specification.
  0.199s Configuration "greeter-cli" is waiting for a Revision to become ready.
  5.710s ...
  5.815s Ingress has not yet been reconciled.
  5.929s Waiting for load balancer to be ready
  6.674s Ready to serve.

Service 'greeter-cli' created to latest revision 'greeter-cli-hello' is available at URL:
http://greeter-cli-knative-demo-serving.apps.sanes.sandbox852.opentlc.com
```

To test the application:

```bash
❯ curl $(oc get routes.serving.knative.dev greeter-cli | awk 'END{print $2}')
hello  greeter => '9861675f8845' : 1
```

### Updating the service

To update the service to define a new revision to use other environment values:

```bash
kn service update greeter-cli --env "MESSAGE_PREFIX=hola" --revision-name greeter-cli-hola
```

We have a new revision of the application

```bash
❯ kn service list
NAME          URL                                                                         LATEST             AGE     CONDITIONS   READY   REASON
greeter-cli   http://greeter-cli-knative-demo-serving.apps.sanes.sandbox852.opentlc.com   greeter-cli-hola   80s     3 OK / 3     True    
hello         http://hello-knative-demo-serving.apps.sanes.sandbox852.opentlc.com         hello-lg498        6m47s   3 OK / 3     True    
```

If we test again the application, the output will be:

```bash
❯ curl $(oc get routes.serving.knative.dev greeter-cli | awk 'END{print $2}')
hola  greeter => '9861675f8845' : 1
```

### Splitting traffic between revisions

At the moment we have two revisions of our application, where the latest revision getting the full
traffic:

```bash
❯ kn revision list
NAME                SERVICE       TRAFFIC   TAGS   GENERATION   AGE     CONDITIONS   READY   REASON
greeter-cli-hola    greeter-cli   100%             2            79s     4 OK / 4     True    
greeter-cli-hello   greeter-cli                    1            2m22s   3 OK / 4     True    
hello-lg498         hello         100%             1            7m49s   3 OK / 4     True    
```

As we have a set of different revisions of our application, we could split the traffic between them using
the following command:

```bash
kn service update greeter-cli --tag greeter-cli-hello=prev --tag greeter-cli-hola=current --traffic prev=50,current=50,@latest=0
```

With this command basically:

* Tag the previous revision (`prev`) as ```greeter-cli-hello```
* Tag the current revision (`current`) as ```greater-cli-hola```
* Load balance in 50% between the previous and current revision

We could check the traffic balancing with:

```bash
❯ kn revision list
NAME                SERVICE       TRAFFIC   TAGS      GENERATION   AGE     CONDITIONS   READY   REASON
greeter-cli-hola    greeter-cli   50%       current   2            5m54s   3 OK / 4     True    
greeter-cli-hello   greeter-cli   50%       prev      1            6m57s   3 OK / 4     True    
hello-lg498         hello         100%                1            12m     3 OK / 4     True    
```

If we invoke the service some times we could check the responses are balanced between revisions:

```bash
❯ curl $(oc get routes.serving.knative.dev greeter-cli | awk 'END{print $2}')
hola  greeter => '9861675f8845' : 1
❯ curl $(oc get routes.serving.knative.dev greeter-cli | awk 'END{print $2}')
hello  greeter => '9861675f8845' : 1
```

Other command to split traffic using the latest version:

```bash
kn service update greeter-cli --traffic prev=25,@latest=75
```

## Preventing scaling down to zero

If we want to prevent scaling down to zero the application, execute the following command:

```bash
kn service update greeter-cli --env "MESSAGE_PREFIX=always on" --annotation autoscaling.knative.dev/minScale="1"
```

## Clean up Services

To remove all the services deployed in this namespace:

```bash
kn service delete --all
```

## References

* [OpenShift Serverless Doc - Creating and managing serverless applications](https://docs.openshift.com/container-platform/4.6/serverless/serving-creating-managing-apps.html)
