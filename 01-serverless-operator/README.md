# Install OpenShift Serverless

Deploy OpenShift Serverless Operator using OLM subscription:

```bash
oc apply -f 01-serverless-operator/serverless-operator-subscription.yml
```

Check the status of the subscription:

```bash
❯ oc get csv
NAME                         DISPLAY                         VERSION   REPLACES                     PHASE
serverless-operator.v1.12.0   Red Hat OpenShift Serverless   1.12.0    serverless-operator.v1.11.0   Succeeded
```

OpenShift Serverless Operator will create the ```openshift-serverless``` namespace where
the following pod are deployed:

```bash
❯ oc get pod -n openshift-serverless
NAME                                         READY   STATUS    RESTARTS   AGE
knative-openshift-694c5fcf5-ztlnz            1/1     Running   0          176m
knative-openshift-ingress-7b9f579459-748wd   1/1     Running   0          176m
knative-operator-7985f5f465-rprnc            1/1     Running   0          176m
```

Also the following namespaces are created:

* ```knative-serving```: Namespace to manage the Serving APIs and applications
* ```knative-eventing```: Namespace to manage the Eventing APIs and applications
   
### Deploy Knative Serving APIs

To deploy the Serving APIs in the ```knative-serving``` namespace:

```bash
❯ oc apply -f 01-serverless-operator/knative-serving.yml -n knative-serving
knativeserving.operator.knative.dev/knative-serving created
```

To verify the deployment status:

```bash
oc get knativeservings knative-serving -n knative-serving --template='{{range .status.conditions}}{{printf "%s=%s\n" .type .status}}{{end}}'
```

The output should be similar to:

```text
DependenciesInstalled=True
DeploymentsAvailable=True
InstallSucceeded=True
Ready=True
VersionMigrationEligible=True
```

OpenShift Serverless Serving is ready to manage serving services.

### Deploy Knative Eventing APIs

To deploy the Eventing APIs in the ```knative-eventing``` namespace:

```bash
oc apply -f 01-serverless-operator/knative-eventing.yml -n knative-eventing
```

To verify the deployment status:

```bash
oc get knativeeventing knative-eventing -n knative-eventing --template='{{range .status.conditions}}{{printf "%s=%s\n" .type .status}}{{end}}'
```

The output should be similar to:

```
DependenciesInstalled=True
DeploymentsAvailable=True
InstallSucceeded=True
Ready=False
VersionMigrationEligible=True
```

OpenShift Serverless Eventing is ready to manage eventing services.
