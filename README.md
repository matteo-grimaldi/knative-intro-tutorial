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

Provision and login into the environment

### Install OpenShift Serverless and Knative CLI

- From the administrator perspective, Operator -> OperatorHub.
- In the quicksearch input form, type "Serveless".
- Select OpenShift Serveless and click "Install".
- On the Install Operator page:
  -  The Installation Mode is All namespaces on the cluster (default), the Installed Namespace is openshift-serverless.
  - Select the stable channel as the Update Channel, The stable channel will enable installation of the latest stable release of the OpenShift Serverless Operator.
  - Select Automatic approval strategy.

Installing kn cli is also straight-forward. 
It is possible to either install it leveraging to you local package manager, for example on macOs 

```bash
brew install knative/client/kn
```

or by accessing command line tools from OpenShift Web console and download and extract the archive.

### Prepare 

Login to OpenShift as regular user

```bash
oc login -u <username> -p <password> <openshift-api-url>
```

Create a new project
```bash
oc new-project standard-deployment
oc project standard-deployment
```

Login to external Image registry with credentials
```bash
podman machine init
podman machine start

podman login -u <user> -p <password> quay.io
```

### Build a Quarkus application

Compile the application and check everything is up and running
```bash
./mvnw compile quarkus:dev
```
on a different terminal test the running Quarkus service

```bash
time curl http://localhost:8080
```

Build the application image, run and test it

```bash
./mvnw compile package
podman build -f Dockerfile.jvm -t localhost/mgrimald/rest-quarkus-jvm

#Run image with podman
podman run -p 8080:8080 localhost/mgrimald/rest-quarkus-jvm

time curl http://localhost:8080
```

Tag and push the image to the container registry of choice

```bash
podman tag localhost/mgrimald/rest-quarkus-jvm quay.io/mgrimald/rest-quarkus-jvm
podman push --tls-verify=false quay.io/mgrimald/rest-quarkus-jvm
```

Create a new service and deployment, expose the route and test the exposed service
```bash
oc new-app --image quay.io/mgrimald/rest-quarkus-jvm:latest -n standard-deployment -l app.openshift.io/runtime=quarkus

#expose the route externally
oc expose svc/rest-quarkus-jvm

time curl http://"$(oc get route rest-quarkus-jvm --template='{{ .spec.host }}')"
```

### Build a Quarkus native application 

Repeat the steps compiling natively (native compilation steps are for macOs environment, on Linux it is going to be more straight-forward, as it's possible compile for the same arch locally)

Check that GRAALVM_HOME variable is set correctly
```bash
export GRAALVM_HOME=/Library/Java/JavaVirtualMachines/graalvm-ce-java11-22.1.0/Contents/Home
```

be sure to have a sufficient memory on podman machine by running
```
podman machine list
```
In case the machine has less than 2G, edit the file ~/.config/containers/podman/machine/qemu/podman-machine-default.json in both cmdline and memory and start / stop the machine to upgrade the machine

```bash

./mvnw clean package -X -Pnative -Dquarkus.native.remote-container-build=true -Dquarkus.native.container-runtime=podman -Dquarkus.native.native-image-xmx=8g

podman build -f Dockerfile -t localhost/mgrimald/rest-quarkus-native

podman tag localhost/mgrimald/rest-quarkus-native quay.io/mgrimald/rest-quarkus-native
podman push --tls-verify=false quay.io/mgrimald/rest-quarkus-native

oc new-app --image quay.io/mgrimald/rest-quarkus-native:latest -n standard-deployment -l app.openshift.io/runtime=quarkus

#expose the route externally
oc expose svc/rest-quarkus-native

time curl http://"$(oc get route rest-quarkus-native --template='{{ .spec.host }}')"
```

## Knative Serving

### Install Knative Serving Operator
- In the Administrator perspective, navigate to Operators → Installed Operators.
- Check that the Project dropdown at the top of the page is set to Project: knative-serving.
- Click Knative Serving in the list of Provided APIs for the OpenShift Serverless Operator to go to the Knative Serving tab.
- Click Create Knative Serving.
- In the Create Knative Serving page, you can install Knative Serving using the default settings by clicking Create.

The resource `service.serving.knative.dev`is the main one and there are many fields that could be looked up in the documentation, but the `kn` client can provide a mean to explore options quicker and directly create services or generate yaml files to apply.

Inspect servicing creation documentation
```bash
kn service create --help
```

Please note some relevant parameters:

- concurrency-limit: Hard Limit of concurrent requests to be processed by a single replica 
- concurrency-target: Soft Limit of concurrent requests to be processed by a single replica
- scale-max: maximum number of replicas
- scale-min: minimum number of replicas

### Prepare a new project for serverless deployments 
```bash
oc new-project knative-deploy
```

### Deploy Quarkus application as Knative Serving via kn CLI

Create and deploy the actual services for jvm and native modes and measure response times
```bash
#create knative services via CLI
kn service create rest-springboot-jvm-sl --image quay.io/mgrimald/rest-springboot-jvm --concurrency-target 90 --scale-max 4 --scale-min 0 --scale-window 30s 
kn service create rest-quarkus-jvm-sl --image quay.io/mgrimald/rest-quarkus-jvm --concurrency-target 90 --scale-max 4 --scale-min 0 --scale-window 30s -l app.openshift.io/runtime=quarkus
kn service create rest-quarkus-native-sl --image quay.io/mgrimald/rest-quarkus-native --concurrency-target 90 --scale-max 4 --scale-min 0 --scale-window 30s -l app.openshift.io/runtime=quarkus

time curl $(kn route list | grep rest-springboot-jvm-sl | awk '{ print $2 }')
time curl $(kn route list | grep rest-quarkus-jvm-sl | awk '{ print $2 }')
time curl $(kn route list | grep rest-quarkus-native-sl | awk '{ print $2 }')

hey -c 50 -z 10s "$(kn route list | grep rest-springboot-jvm-sl | awk '{ print $2 }')"
hey -c 50 -z 10s "$(kn route list | grep rest-quarkus-jvm-sl | awk '{ print $2 }')"
hey -c 50 -z 10s "$(kn route list | grep rest-quarkus-native-sl | awk '{ print $2 }')"
```
Alternatively Knative services can be created by locally generating yaml resources and then applying it.
For example:
```bash
kn service create rest-quarkus-native-sl --image quay.io/mgrimald/rest-quarkus-native --concurrency-target 90 --scale-max 4 --scale-min 0 --scale-window 30s -l app.openshift.io/runtime=quarkus --target=rest-quarkus-native-sl-service.yaml
```

That will generate the following YAML file

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app.openshift.io/runtime: quarkus
  name: rest-quarkus-native-sl
  namespace: mgrimald-dev
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/max-scale: "4"
        autoscaling.knative.dev/min-scale: "0"
        autoscaling.knative.dev/target: "90"
        autoscaling.knative.dev/window: 30s
        client.knative.dev/updateTimestamp: "2023-03-02T21:51:46Z"
        client.knative.dev/user-image: quay.io/mgrimald/rest-quarkus-native
      creationTimestamp: null
      labels:
        app.openshift.io/runtime: quarkus
    spec:
      containers:
      - image: quay.io/mgrimald/rest-quarkus-native
        name: ""
        resources: {}
status: {}
```     

By `kn` CLI

```
kn service create -f rest-quarkus-native-sl-service.yaml
```

or by `oc` CLI

````
oc apply -f rest-quarkus-jvm-sl.yaml
````

Inspect Kubernetes generated resources
```bash
oc get all -n knative-deploy
```

### Cleanup
```bash
kn service delete rest-springboot-jvm-sl
kn service delete rest-quarkus-jvm-sl
kn service delete rest-quarkus-native-sl
oc delete project knative-deploy
```

## Revisions and Traffic Distributions

Knative always routes traffic to the **latest** revision of the service. It is possible to split the traffic amongst the available revisions.

### Deploy a Knative service 
### Prepare the project for serverless deployments 
```bash
oc new-project knative-revision
oc adm policy add-role-to-user edit user1 -n knative-revision  

oc login -u user1 -p <password>
```

```bash
kn service create blue-green-canary \
   --image=quay.io/rhdevelopers/blue-green-canary \
   --env BLUE_GREEN_CANARY_COLOR="#6bbded" \
   --env BLUE_GREEN_CANARY_MESSAGE="Hello" \
   -l app.openshift.io/runtime=quarkus
```

### Invoke the Service

Get the service URL

```
kn service describe blue-green-canary -o url
```

Use the URL to open the service in a browser window.


## Deploy a New Revision of a Service

In line with *12-Factor* principle, Knative rolls out new deployment whenever the Service *Configuration* changes, and creates immutable version of code and configuration called *revision*. An example of configuration change could for e.g. an update of Service image, a change to environment variables or add liveness/readiness probes.

Create a new revision by updating env variables for the service

```bash
kn service update blue-green-canary \
   --image=quay.io/rhdevelopers/blue-green-canary \
   --env BLUE_GREEN_CANARY_COLOR="#5bbf45" \
   --env BLUE_GREEN_CANARY_MESSAGE="Namaste" \
   -l app.openshift.io/runtime=quarkus
```

Now invoking the service again using the service URL, will show a green color browser page with greeting *Namaste*.

### Check Revisions

Check to ensure you have two revisions of the blue-green-canary service:

```
kn revision list
```

```
kubectl get rev \
  --selector=serving.knative.dev/service=blue-green-canary \
  --sort-by="{.metadata.creationTimestamp}"
```

## Tag Revisions

List the existing revisions,

```
kn revision list -s blue-green-canary
```

When Knative rolls out a new revision, it increments the `GENERATION` by *1* and then routes *100%* of the `TRAFFIC` to it, hence we can use the `GENERATION` or `TRAFFIC` to identify the latest reivsion. 

### Tag by color name

Let us tag `blue-green-canary-00001` which shows *blue* browser page with tag name `blue` and tag `blue-green-canary-00002` which shows *green* browser page with tag name `green`.

```
kn service update blue-green-canary --tag=blue-green-canary-00001=blue
kn service update blue-green-canary --tag=blue-green-canary-00002=green
```

Let's check the result of tagging commands

```
kn revision list -s blue-green-canary  
```

## Tag Latest

Lets tag whatever revision that is latest to be tagged as *latest*.

```
kn service update blue-green-canary --tag=@latest=latest
```

```
kn revision list -s blue-green-canary
```

As *green* happend to be latest revision it has been tagged with name `lastest` in addition to `green`.


## Applying Blue-Green Deployment Pattern

Knative offers a simple way of switching 100% of the traffic from one Knative service revision (blue) to another newly rolled out revision (green). If the new revision (e.g. green) has erroneous behavior then it is easy to rollback the change.

### Rollback to blue

Route all the traffic of service `blue-green-canary` to blue revision of the service:

```
kn service update blue-green-canary --traffic blue=100,green=0,latest=0
```

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
watch "oc get pods -n knative-revision"
```

```
NAME                                                    READY   STATUS    RESTARTS   AGE
blue-green-canary-00001-deployment-54597d94b9-q8c57   2/2     Running   0          79s
blue-green-canary-00002-deployment-8564bf5b5b-gtvh4   2/2     Running   0          14s
```

## Cleanup
```
kn service delete blue-green-canary
oc delete project knative-revision
```

## Knative Eventing - Sink and Sources

### Install Knative Eventing Operator
- In the Administrator perspective, navigate to Operators → Installed Operators.
- Check that the Project dropdown at the top of the page is set to Project: knative-eventing.
- Click Knative Eventing in the list of Provided APIs for the OpenShift Serverless Operator to go to the Knative Eventing tab.
- Click Create Knative Eventing.
- Click Create.
- Click Create.

### Prepare the project for serverless deployments 
```bash
oc new-project knative-eventing-deploy
oc adm policy add-role-to-user edit user1 -n knative-eventing-deploy 

oc login -u user1 -p <password>
```

### Creating the event sink (regular knative servicing)
To test and realize a knative servicing sample architecture a sink, a regular knative servicing application, is required. 
In this case we deploy a very simple knative service
```bash
kn service create eventinghello \
  --concurrency-target=1 \
  --scale-window 30s \
  --image=quay.io/rhdevelopers/eventinghello:0.0.2 \
  -l app.openshift.io/runtime=quarkus
```

### Creating and deploying an Event Source
It is possibile to leverage the web console to simply deploy and configure a basic knative eventing resource: an event source. This knative resource simply sends data on a regular configured basis (in this case every minute).

Another way to deploy this is to leverage on the usual kn cli
```bash
kn source ping create eventinghello-ping-source \
  --schedule "*/1 * * * *" \
  --data '{"message": "Thanks for doing Knative Tutorial"}' \
  --sink ksvc:eventinghello
```

please note that "ksvc" allows to specify the reference to the event sink to connect the source to it.

or to use a YAML template
```yaml
apiVersion: sources.knative.dev/v1
kind: PingSource
metadata:
  name: eventinghello-ping-source
spec:
  schedule: "*/1 * * * *"
  data: '{"message": "Thanks for doing Knative Tutorial"}'
  sink:  
    ref:
      apiVersion: serving.knative.dev/v1 
      kind: Service
      name: eventinghello
```

````bash
stern eventinghello -c user-container -n knative-eventing-deploy
````

## Knative Eventing - Channel and subscribers
### Creating Event Channel via kn CLI
Also to create a channel, it is possible to use the Web Console, or leverage on the simple kn cli as below
```bash
kn channel create eventinghello-ch
```
or else by YAML File

```yaml
apiVersion: messaging.knative.dev/v1
kind: Channel
metadata:
  name: eventinghello-ch 
```

we add a clone service to the one already deployed

```bash
kn service create eventinghellob \
  --concurrency-target=1 \
  --scale-window 30s \
  --image=quay.io/rhdevelopers/eventinghello:0.0.2 \
  -l app.openshift.io/runtime=quarkus
```

### Creating the subscritions and verify them

We can create the subscriptions by editing the channel on the topology view, or:

```bash
kn subscription create eventinghello-sub \
  --channel eventinghello-ch \
  --sink eventinghello

kn subscription create eventinghellob-sub \
  --channel eventinghello-ch \
  --sink eventinghellob

kn subscription list
```

Now by selecting the previously created source, we can change the sink, and move it to the channel.
In this way, bot services will receive the ping.

## Knative Eventing - Triggers and brokers

### Deploy a broker
```bash
kn broker create default
```

### Deploy sink services
```bash
kn service create eventingaloha \
  --concurrency-target=1 \
  --scale-window 30s \
  --revision-name=eventingaloha-v1 \
  --image=quay.io/rhdevelopers/eventinghello:0.0.2\
  -l app.openshift.io/runtime=quarkus


kn service create eventingbonjour \
  --concurrency-target=1 \
  --scale-window 30s \
  --revision-name=eventingbonjour-v1 \
  --image=quay.io/rhdevelopers/eventinghello:0.0.2\
  -l app.openshift.io/runtime=quarkus
```

### Deploy Triggers

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

or by using kn cli

```bash
kn trigger create helloaloha \
  --broker=default \
  --sink=ksvc:eventingaloha \
  --filter=type=aloha
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

or by using kn cli
```bash
kn trigger create hellobonjour \
  --broker=default \
  --sink=ksvc:eventingbonjour \
  --filter=type=bonjour
  ```


Now let's create a pod to be able to send messages to the broker, let's do it with a simple yaml file:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: curler
  name: curler
spec:
  containers:
  - name: curler
    image: fedora:37
    tty: true
```

Let's get the url of the broker by using the cli
```bash
kubectl get broker default -o jsonpath='{.status.address.url}'
```

And the use the curler shell to send the message

````
kubectl -n knative-eventing-deploy exec -it curler -- /bin/bash
````

```bash
curl -v "xxxx" \
-X POST \
-H "Ce-Id: say-hello" \
-H "Ce-Specversion: 1.0" \
-H "Ce-Type: aloha" \
-H "Ce-Source: mycurl" \
-H "Content-Type: application/json" \
-d '{"key":"from a curl"}'
```

## Knative Functions (https://openshift-knative.github.io/docs/docs/functions/quickstart-functions.html)

As other knative resources the kn cli allows to access the feature. For example it is possible to se all the possibilities by running
```bash
kn func --help

```

By inspecting the man for function creation
```bash
kn func create --help
```
It is possible to see that there are several available combinations and the basic syntax is 

```bash
kn func create -l <language> -t <http|events> funcname
```

So, to create a quarkus rest-based function is enough to position in the destination folder where to generate source code and configurations and run

```bash
kn func create -l quarkus -t http hellofunc
```

Then it will be possible to inspect and modify resources to be able to build the function by running the build/deploy commands.

Using 
```bash
kn func build -v
kn func run 
```
it is possible to build and run the function locally. Once the functions is running, it is possible to use the CLI to test it, instead of regular curl

```bash
kn func invoke --data '{"message": "hello world!" }'
```

## Cleanup OpenShift Serverless operators
```bash
oc delete knativeservings.operator.knative.dev knative-serving -n knative-serving
oc delete namespace knative-serving

oc delete knativeeventings.operator.knative.dev knative-eventing -n knative-eventing
oc delete namespace knative-eventing
````

Remove OpenShift Serverless operator from web UI

```bash
oc get crd -oname | grep 'knative.dev' | xargs oc delete
```