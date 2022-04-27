# Knative Tutorial - Introduction to Knative

 ![Knative Tutorial](https://github.com/redhat-developer-demos/knative-tutorial/workflows/Knative%20Tutorial/badge.svg) [![Knative Serving v1.1.0](https://img.shields.io/badge/Knative%20Serving-v1.1.0-blue)](https://knative.dev/docs/serving/)
 [![Knative Eventing v1.1.0](https://img.shields.io/badge/Knative%20Eventing-v1.1.0-blue)](https://knative.dev/docs/eventing/)
 [![Strimzi Kafka](https://img.shields.io/badge/Strimzi%20Kafka-v0.26.1-blue)](https://strimzi.io)
 [![Apache Camel K](https://img.shields.io/badge/Apache%20Camel--K-v1.8.0-blue)](https://camel.apache.org/camel-k/latest/)
  [![OpenShift Serverless](https://img.shields.io/badge/OpenShift%20Serverless-v1.19.0-blue)](https://www.openshift.com/learn/topics/serverless)

## Documentation

 Start your serverless journey today with <https://redhat-developer-demos.github.io/knative-tutorial>

## What is Serverless

 Serverless epitomize the very benefits of what cloud platforms promise: offload the management of infrastructure while taking advantage of a consumption model for the actual utilization of services. While there are a number of server frameworks out there, [Knative](https://knative.dev) is the first serverless platform specifically designed for Kubernetes and OpenShift.

 This tutorial will act as step-by-step guide in helping you to understand Knative starting with setup, understanding fundamentals concepts such as service, configuration, revision etc., and finally deploying some use cases which could help deploying serverless applications.

 The following content is based from the [Red Hat Developer Program](http://developers.redhat.com) - Register today!

## Preparation

### Provision and login into the environment

developer-sandbox

Install OpenShift Serverless from OperatorHub

- From the administrator perspective, Operator -> OperatorHub.
- In the quicksearch input form, type "Serveless".
- Select OpenShift Serveless and click "Install".
- On the Install Operator page:
  -  The Installation Mode is All namespaces on the cluster (default), the Installed Namespace is openshift-serverless.
  - Select the stable channel as the Update Channel, The stable channel will enable installation of the latest stable release of the OpenShift Serverless Operator.
  - Select Automatic approval strategy.


Login on OpenShift 

```bash
oc login -u <username> -p <password>
```

Create a new project
```bash
oc new-project knative-intro-demo
oc project knative-intro-demo
```

Login to OpenShift registry
```bash
podman login -u "$(oc whoami)" -p "$(oc whoami -t)" "$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')"
```


### Build a Quarkus application

Compile the application and check everything is up and running
```bash
./mvnw compile quarkus:dev

time curl http://localhost:8080
```

Build the application image and test it

```bash
podman build -f Dockerfile.jvm -t localhost/mgrimald/rest-quarkus-jvm .

#Run image with podman
podman run -p 8080:8080 localhost/mgrimald/rest-quarkus-jvm

time curl http://localhost:8080
```

Tag and push the image to the container registry of choice

```bash
podman tag mgrimald/rest-quarkus-jvm:latest "$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')/standard-deploy/rest-quarkus-jvm:latest"
podman push --tls-verify=false "$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')/standard-deploy/rest-quarkus-jvm:latest" 
```

Create a new service and deployment, expose the route and test the exposed service
```bash
oc new-app rest-quarkus-jvm:latest -n standard-deploy

oc expose svc/rest-quarkus-jvm

time curl http://"$(oc get route rest-quarkus-jvm --template='{{ .spec.host }}')"
```

### Build a Quarkus native application 

Repeate the steps compiling a native service

```bash
./mvnw clean package -Pnative
./target/greeter-runner

time curl http://localhost:8080

podman build -f Dockerfile -t localhost/mgrimald/rest-quarkus-native .

time curl http://localhost:8080

podman tag mgrimald/rest-quarkus-native:latest "$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')/standard-deploy/rest-quarkus-native:latest"
podman push --tls-verify=false "$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')/standard-deploy/rest-quarkus-native:latest" 

oc new-app rest-quarkus-native:latest -n standard-deploy

oc expose svc/rest-quarkus-native

time curl http://"$(oc get route rest-quarkus-native --template='{{ .spec.host }}')"
```

### Prepare the project
```bash
oc new-project knative-deploy

#permission to pull from standard-deploy registry
oc policy add-role-to-user \
    system:image-puller system:serviceaccount:knative-deploy:default \
    --namespace=standard-deploy
```

## Knative Serving

### Install Knative Serving Operator
- In the Administrator perspective, navigate to Operators → Installed Operators.
- Check that the Project dropdown at the top of the page is set to Project: knative-serving.
- Click Knative Serving in the list of Provided APIs for the OpenShift Serverless Operator to go to the Knative Serving tab.
- Click Create Knative Serving.
- In the Create Knative Serving page, you can install Knative Serving using the default settings by clicking Create.



### Deploy Quarkus application as Knative Serving via kn CLI
```bash
#create knative service via CLI
kn service create rest-quarkus-jvm-sl --image image-registry.openshift-image-registry.svc:5000/standard-deploy/rest-quarkus-jvm --concurrency-limit 2 --concurrency-target 90 --max-scale 4 --min-scale 0

kn service create rest-quarkus-native-sl --image image-registry.openshift-image-registry.svc:5000/standard-deploy/rest-quarkus-native --concurrency-limit 2 --concurrency-target 90 --max-scale 4 --min-scale 0


time curl $(kn route list | grep rest-sb-sl | awk '{ print $2 }')
time curl $(kn route list | grep rest-quarkus-jvm-sl | awk '{ print $2 }')
time curl $(kn route list | grep rest-quarkus-native-sl | awk '{ print $2 }')


hey -c 50 -z 10s "$(kn route list | grep rest-quarkus-native-sl | awk '{ print $2 }')"
```

Inspect Kubernetes generated resources
```bash
```


### Deploy Quarkus application as Knative Serving applying YAML
```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: greeter
spec:
  template:
    spec:
      containers:
      - image: quay.io/rhdevelopers/knative-tutorial-greeter:quarkus
        livenessProbe:
          httpGet:
            path: /healthz
        readinessProbe:
          httpGet:
            path: /healthz
```            

## Revisions and Traffic Distributions

Knative always routes traffic to the **latest** revision of the service. It is possible to split the traffic amongst the available revisions.


```bash
kn service create blue-green-canary \
   --image=quay.io/rhdevelopers/blue-green-canary \
   --env BLUE_GREEN_CANARY_COLOR="#6bbded" \
   --env BLUE_GREEN_CANARY_MESSAGE="Hello"
```

### Invoke the Service

Get the service URL,

```
kn service describe blue-green-canary -o url
```

Use the URL to open the service in a browser window.If the service was deployed correctly you should see blue background browser page, with greeting as *Hello*.


## Deploy a New Revision of a Service

In line with *12-Factor* principle, Knative rolls out new deployment whenever the Service *Configuration* changes, and creates immutable version of code and configuration called *revision*. An example of configuration change could for e.g. an update of Service image, a change to environment variables or add liveness/readiness probes.

Let us now change the configuration of the service by updating the service environment variable *BLUE_GREEN_CANARY_COLOR* to make the browser display *green* color with greeting text as *Namaste*. 

```bash
kn service update blue-green-canary \
   --image=quay.io/rhdevelopers/blue-green-canary \
   --env BLUE_GREEN_CANARY_COLOR="#5bbf45" \
   --env BLUE_GREEN_CANARY_MESSAGE="Namaste"
```

Now invoking the service again using the service URL, will show a green color browser page with greeting *Namaste*.

### revisions

Check to ensure you have two revisions of the blue-green-canary service:

```
kn revision list
```

```
NAME                        SERVICE             TRAFFIC   TAGS   GENERATION   AGE     CONDITIONS   READY   REASON
blue-green-canary-00002   blue-green-canary   100%             2            2m33s   3 OK / 4     True
blue-green-canary-00001   blue-green-canary                    1            30m     3 OK / 4     True
```

```
kubectl get rev \
  --selector=serving.knative.dev/service=blue-green-canary \
  --sort-by="{.metadata.creationTimestamp}"
```

```
NAME                        CONFIG NAME         K8S SERVICE NAME            GENERATION   READY   REASON
blue-green-canary-00001   blue-green-canary   blue-green-canary-00001   1            True
blue-green-canary-00002   blue-green-canary   blue-green-canary-00002   2            True
```

## Tag Revisions

As you had observed that the Knative service `blue-green-canary` now has two revisions namely *blue-green-canary-00001* and *blue-green-canary-00002*. As the Revision names are autogenerated it is hard to comprehend to which code/configuration set it corresponds to. To overcome this problem Knative provides *tagging* of revision names that allows one to tag a revision name to a logical human understanable names called *tags*.

As our colors service shows different colors on the browser let us tag the revisions with color,

List the existing revisions,

```
kn revision list -s blue-green-canary
```

```
NAME                        SERVICE             TRAFFIC   TAGS   GENERATION   AGE   CONDITIONS   READY   REASON
blue-green-canary-00002   blue-green-canary   100%             2            23m   3 OK / 4     True
blue-green-canary-00001   blue-green-canary                    1            51m   3 OK / 4     True
```

When Knative rolls out a new revision, it increments the `GENERATION` by *1* and then routes *100%* of the `TRAFFIC` to it, hence we can use the `GENERATION` or `TRAFFIC` to identify the latest reivsion. 

### Tag Blue

Let us tag `blue-green-canary-00001` which shows *blue* browser page with tag name `blue`.

```
kn service update blue-green-canary --tag=#blue-green-canary-00001#=blue
```

```
Updating Service 'blue-green-canary' in namespace 'knativetutorial':

  0.037s The Route is still working to reflect the latest desired specification.
  0.126s Ingress has not yet been reconciled.
  0.162s Waiting for load balancer to be ready
  0.303s Ready to serve.

Service 'blue-green-canary' with latest revision 'blue-green-canary-00002' (unchanged) is available at URL:
http://blue-green-canary.knativetutorial.192.168.64.13.nip.io
```

### Tag Green

Let us tag `blue-green-canary-00002` which shows *green* browser page with tag name `green`.

```
kn service update blue-green-canary --tag=#blue-green-canary-00002#=green
```

```
Updating Service 'blue-green-canary' in namespace 'knativetutorial':

  0.037s The Route is still working to reflect the latest desired specification.
  0.126s Ingress has not yet been reconciled.
  0.162s Waiting for load balancer to be ready
  0.303s Ready to serve.

Service 'blue-green-canary' with latest revision 'blue-green-canary-00002' (unchanged) is available at URL:
http://blue-green-canary.knativetutorial.192.168.64.13.nip.io
```

## Tag Latest

Lets tag whatever revision that is latest to be tagged as *latest*.

```
kn service update blue-green-canary --tag=@latest=latest
```

```
Updating Service 'blue-green-canary' in namespace 'knativetutorial':

  0.037s The Route is still working to reflect the latest desired specification.
  0.126s Ingress has not yet been reconciled.
  0.162s Waiting for load balancer to be ready
  0.303s Ready to serve.

Service 'blue-green-canary' with latest revision 'blue-green-canary-00002' (unchanged) is available at URL:
http://blue-green-canary.knativetutorial.192.168.64.13.nip.io
```

Let us query the Service Revisions again,

```
kn revision list -s blue-green-canary
```

```
NAME                        SERVICE             TRAFFIC   TAGS    GENERATION   AGE   CONDITIONS   READY   REASON
blue-green-canary-00002   blue-green-canary   100%      #latest,green#   2            29m   3 OK / 4     True
blue-green-canary-00001   blue-green-canary             #blue#    1            57m   3 OK / 4     True
```

As *green* happend to be latest revision it has been tagged with name `lastest` in addition to `green`.

Let us use the tag names for easier identification of the revision and perform traffic distribution amongst them.

## Applying Blue-Green Deployment Pattern

Knative offers a simple way of switching 100% of the traffic from one Knative service revision (blue) to another newly rolled out revision (green). If the new revision (e.g. green) has erroneous behavior then it is easy to rollback the change.

### Rollback to blue

Route all the traffic of service `blue-green-canary` to blue revision of the service:

```
kn service update blue-green-canary --traffic blue=100,green=0,latest=0
```

NOTE: We use the tag names to identify the revisions

Let us list all revisions with tags:

```
kn revision list
```

Based on the revision tags that we created earlier, the output should be like:

```
NAME                        SERVICE             TRAFFIC   TAGS           GENERATION   AGE   CONDITIONS   READY   REASON
blue-green-canary-00002   blue-green-canary             latest,green   2            56m   4 OK / 4     True
blue-green-canary-00001   blue-green-canary   #100%#      #blue#           1            83m   4 OK / 4     True
```

Let us list the available sub-routes:

```
{kubernetes-cli} -n {tutorial-namespace} get ksvc blue-green-canary -oyaml \
 | yq r - 'status.traffic[*].url'
```

The above command should return you three sub-routes for the main `greeter` route:

```
http://latest-blue-green-canary.knativetutorial.192.168.64.13.nip.io #<.>
http://blue-blue-green-canary.knativetutorial.192.168.64.13.nip.io #<.>
http://green-blue-green-canary.knativetutorial.192.168.64.13.nip.io  #<.>
```

<1> the sub route for the traffic tag `latest`
<2> the sub route for the traffic tag `blue`
<3> the sub route for the traffic tag `green`

You will notice that the command does not create any new configuration/revision/deployment as there was no application update (e.g. image tag, env var, etc), but when you call the service, Knative scales up the `blue` that shows *blue* browser page with greeting *Hello*.

.Blue Green Canary::Blue
image::blue-green-canary-blue.png[]

```
{kubernetes-cli} get pods
```

```
NAME                                     READY   STATUS        RESTARTS   AGE
#blue-green-canary-00001-deployment-54597d94b9-25x4r#   2/2     Running       0          12s
blue-green-canary-00002-deployment-6cb545df65-ktqc2   0/2     Terminating   0          2m
```

Since `blue` is the active revision now, the existing `green` pod is getting terminated as no future requests will be served by it.


## Applying Canary Release Pattern

Knative allows you to split the traffic between revisions in increments as small as 1%.
```
kn service update blue-green-canary \
  --traffic="blue=80" \
  --traffic="green=20" 
```

As in the previous section on <<blue-green>> deployments, the command will not create any new configuration/revision/deployment. To observe the traffic distribution, open the Service Route URL in your browser window. 

You will notice the browser alternating betwee green and blue color, with majority of the time staying with blue.


You should also notice that two pods are running representing both `blue` and `green`:

```
watch "{kubernetes-cli} get pods -n {tutorial-namespace}"
```

```
NAME                                                    READY   STATUS    RESTARTS   AGE
blue-green-canary-00001-deployment-54597d94b9-q8c57   2/2     Running   0          79s
blue-green-canary-00002-deployment-8564bf5b5b-gtvh4   2/2     Running   0          14s
```

## Cleanup
```
kn service delete blue-green-canary
```


## Knative Eventing

### Install Knative Eventing Operator
- In the Administrator perspective, navigate to Operators → Installed Operators.
- Check that the Project dropdown at the top of the page is set to Project: knative-eventing.
- Click Knative Eventing in the list of Provided APIs for the OpenShift Serverless Operator to go to the Knative Eventing tab.
- Click Create Knative Eventing.
- Click Create.
- Click Create.


### Creating and deploying an Event Source via kn CLI

```bash
kn source ping create eventinghello-ping-source \
  --schedule "*/2 * * * *" \
  --data '{"message": "Thanks for doing Knative Tutorial"}' \
  --sink ksvc:eventinghello
```

ksvc stands for the reference to the knative event sink (basically a regular knative servicing service)

### Creating and deploying an Event Source applying YAML

```yaml
apiVersion: sources.knative.dev/v1
kind: PingSource
metadata:
  name: eventinghello-ping-source
spec:
  schedule: "*/2 * * * *"
  data: '{"message": "Thanks for doing Knative Tutorial"}'
  sink:  
    ref:
      apiVersion: serving.knative.dev/v1 
      kind: Service
      name: eventinghello
```

### Creating the event sink (regular knative servicing)
```bash
kn service create eventinghello \
  --concurrency-target=1 \
  --image=quay.io/rhdevelopers/eventinghello:0.0.2
```

### channel and subscribers
### Creating Event Channel via kn CLI

```bash
kn channel create eventinghello-ch
```

### Creating Event Channel applying YAML
```yaml
apiVersion: messaging.knative.dev/v1
kind: Channel
metadata:
  name: eventinghello-ch 
```

### Creating the event sink (regular knative servicing)
```bash
kn service create eventinghellob \
  --concurrency-target=1 \
  --image=quay.io/rhdevelopers/eventinghello:0.0.2
```

### Creating the subscritions and verify them
```bash
kn subscription create eventinghello-sub \
  --channel eventinghello-ch \
  --sink eventinghello

kn subscription create eventinghellob-sub \
  --channel eventinghello-ch \
  --sink eventinghellob

kn subscription list
```

### triggers and brokers
```bash
kn broker create default
```

### deploy sink services
```bash
kn service create eventingaloha \
  --concurrency-target=1 \
  --revision-name=eventingaloha-v1 \
  --image=quay.io/rhdevelopers/eventinghello:0.0.2


kn service create eventingbonjour \
  --concurrency-target=1 \
  --revision-name=eventingbonjour-v1 \
  --image=quay.io/rhdevelopers/eventinghello:0.0.2
```


```yaml
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: helloaloha
spec:
  broker: default
  filter:
    attributes:
      type: aloha 
  subscriber:
    ref:
     apiVersion: serving.knative.dev/v1
     kind: Service
     name: eventingaloha
```

```yaml
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: hellobonjour
spec:
  broker: default
  filter:
    attributes:
      type: bonjour
  subscriber:
    ref:
     apiVersion: serving.knative.dev/v1
     kind: Service
     name: eventingbonjour
```

```bash
stern eventing -c user-container
```