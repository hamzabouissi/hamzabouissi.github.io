+++
title = 'Transform AWS Exam Generator Architecture to Open Source Part #3: Exam Generation'
date = 2024-08-12T14:40:08+01:00
draft = false
[cover]
    image = 'img/exam-thumb.png'
tags = ['Kafka','Kafka-Connect','Minio','Knative','Kubernetes']

+++

## Brief description

In this part we should talk about:

1. Kafka topics and difference of deploying between zookeeper and kraft.
2. Create Minio cluster and k8s jobs for adding event notification
3. Knative installation and the invocation with Kafka topics.
4. Hosting of generate-exam frontend in k8s services with ingress, subdomain and reference the WebSocket service.

## Current Stack

I have a K8s cluster composed of three nodes (1 master, 2 control plane) with `Talos` as the running OS, `MetalLB` deployed as a load balancer combined with `Nginx` (nginx.io) as an ingress controller.

Ingress hosts that we will see are mapped to my `Cloudflare` domain(enkinineveh.space) secured with TLS certs generated using `cert-manager` and `letsencrypt`. One side note is that am using [reflector](https://github.com/emberstack/kubernetes-reflector) to share TLS secret into other namespaces.

`Helm` is my favorite tool for deploying resources, `Helmfile` in case of deploying multiple charts, and `k9s` for navigating resources.

`Gitea` is my baby GitHub and `CNPG` is the default database operator.

here is the repo of current project: ![Github](https://github.com/hamzabouissi/exam-architecture)

## Introduction

The processing phase is divided into 2 parts: the generation and the passing of the exam.

The generation part is composed of 5 key elements: frontend UI, Minio, Kafka topics, Knative, Bedrock.

When you create an architecture, you must tackle the dependency-free component first, here as an example, for Minio to send notifications it needs a destination in our case it means a topic, and for Knative to get triggered it also needs a source, a topic also, so that leads to Kafka being the first element to create followed by Minio and Knative.

For installing packages in Kubernetes, Helm is my love language, but to deploy multiple charts together helmfile is the get-go, because it assembles charts and deploys them as a single stack.

## Kafka

Now to deploy a Kafka cluster, we need an operator to handle all the heavy and tedious tasks of managing the cluster, which will lead us into Strimzi.

Strimzi offers a way to run an Apache Kafka cluster on Kubernetes in various deployment configurations, before deploying we must understand the different architecture Kafka comes with.

Kafka have two deployment methods: kraft and zookeeper, the zookeeper method was creating a separate controller for managing metadata that can leads into burden or latency during heavy load environment, in the other side kraft managed to introduce a new method of handling the metadata within Kafka itself.

![kafka architecture](https://images.ctfassets.net/gt6dp23g0g38/7gQZn9CnRAT60NeyYBYflL/b144fee6dad28ce97c3e91e6d09d1167/20230616-Diagram-KRaft.jpg)

From my experience, as we do not have that much of workload, deploying Kafka with zookeeper is sufficient, but for more up to date approach we will deploy the cluster with kraft mode

Internally Kafka is composed of multiple components, which are: producer, consumer, broker, topics and partitions.

To understand the components better, let’s take the generation phase as an example.

Producers are the event emitter, like Minio bucket notification.

Consumers are the event receiver, Knative function is the case, as it consumes the events the moment they reach the topic.

Broker is the connection point between the producer and the consumer, topics on the other side are subcomponents of the broker. As best practice each topic should handle a single goal.

Now Let’s start with installing the operator first, I found the documentation highly informative to starts with.

Installing the operator with kraft mode, will require other feature gates like: **KafkaNodePool**, **UseKraft** and **UnidirectionalTopicOperator**, we add those values into featureGate property in **strimzi-values.yaml** file

```yaml
featureGates: +UseKRaft,+KafkaNodePools,+UnidirectionalTopicOperator
```

we create a a repository and a release inside the helmfile.yaml

```yaml
repositories:
  - name: kafka-operator
    url: https://strimzi.io/charts

releases:
  - name: kafka-operator
    chart: kafka-opertor/strimzi-kafka-operator
    values:
      - ./kafka-values.yaml
```

and then run

```bash
helmfile apply 
```

Now our operator is ready, we continue into deploying **KafkaNodePool** and **Kraft Cluster**

KafkaNodePool is an essential part for the kraft mode cluster, because it defines a set of node pools to install kafka cluster on and has multiple configurations like number of replicas, roles of nodes, storage configuration, resource requirements, etc…

Here is the yaml file:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: broker_controller
  labels:
    strimzi.io/cluster: kafka-cluster
spec:
  replicas: 3
  roles:
    - broker
    - controller
  storage:
    type: jbod
    volumes:
      - id: 0
        type: persistent-claim
        size: 10Gi
        deleteClaim: false


```

We defined 3 nodes, a storage of 10Gi for the whole pool and  each node have 2 roles **broker** and **controller**.

Good, let’s initialise the kraft cluster on those nodes, here is the yaml:

```yaml

apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: kafka-cluster
  namespace: strimzi
  annotations:
    strimzi.io/node-pools: enabled
    strimzi.io/kraft: enabled
spec:
  kafka:
    version: 3.7.1
    # The replicas field is required by the Kafka CRD schema while the KafkaNodePools feature gate is in alpha phase.
    # But it will be ignored when Kafka Node Pools are used
    replicas: 3
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
    # The storage field is required by the Kafka CRD schema while the KafkaNodePools feature gate is in alpha phase.
    # But it will be ignored when Kafka Node Pools are used
    storage:
      type: jbod
      volumes:
      - id: 0
        type: persistent-claim
        size: 10Gi
        deleteClaim: false
  # The ZooKeeper section is required by the Kafka CRD schema while the UseKRaft feature gate is in alpha phase.
  # But it will be ignored when running in KRaft mode
  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 10Gi
      deleteClaim: false
  entityOperator:
    userOperator: {}
    topicOperator: {}

```

**topicOperator and userOperator** are for managing topics and user, **Zookeeper** definition is needed but it will be ignored later, and cluster is exposing two ports for TLS/Non-TLS Connection.

As we talked before the Minio Cluster will require a destination topic for sending bucket notifications(events), let’s create a one:

```yaml

apiVersion: kafka.strimzi.io/v1beta1
kind: KafkaTopic
metadata:
  name: exam-generator-topic
  labels:
    strimzi.io/cluster: "kafka-cluster"
spec:
  partitions: 3
  replicas: 1

```

I found the 1 replicas with 3 partition is a good combination for latency and throughput(parallelism)

## Minio & Bucket Notification

The Minio Cluster Deployment process is the same as Kafka, We install operator first:

```yaml
repositories:
  ...
  - name: minio-operator
    url: https://operator.min.io
releases:
  ...
  - name: minio-operator
    chart: minio-operator/operator
    namespace: minio-operator

```

Then we create a Minio Tenant (same as cluster), start by downloading the default **values.yaml** and changing few properties to match our needs:

- First, add a bucket named **exams** to get created when the tenant initialised:

```yaml
  buckets:
    - name: exams
      objectLock: false
```

- Change **pools.server=1** because a standalone server is enough
- As the Minio will be accessed internally in Kubernetes, we **disable ingress** and if we need to access the console we port-forward the service.
- Secrets: we use simple **access key** and **secret key** in my case: **minio** and **minio123**

then add the chart and values to helmfile:

```yaml
  - name: minio-tenant
    chart: minio-operator/tenant
    namespace: minio-tenant
    values:
      - ./minio-tenant-values.yaml
    needs:
      - minio-operator

```

We check the pods, and here is the tenant deployed.

![minio-tenant](/img/k9s-minio-tenant.png)

To test things out Let’s forward the console port and access it.

```bash
 kpf -n Minio-operator services/console 9090:9090
```

![minio-console](/img/minio-console.png)

It asked for the JWT, which will be retrieved with:

```bash
kubectl get secret/console-sa-secret -n minio-operator -o JSON| jq -r '.data.token' | base64 -d
```

Cool, the tenant is working, and the bucket was created successfully

![minio-bucket](/img/minio-bucket.png)

But we’re missing the notification side when uploading an object. To add Kafka endpoint as the notification destination, we will use the configuration command: **mc**

Here is the configuration script:

``` bash
# First Part
mc config host add dc-minio http://minions-hl.minio-tenant:9000 minio minio123;

mc admin config get dc-minio notify_kafka;

mc admin config set dc-minio notify_kafka:primary \\

    brokers="kafka-cluster-kafka-bootstrap.strimzi.svc:9092" \\

    topic="exam-generator-topic" \\

    tls_skip_verify=true \\

    enabled=on;

mc admin service restart dc-minio;

echo "finish restart"; sleep 10s;

# Second Part

mc event add --event "put" --prefix exams dc-minio/exams arn:minio:sqs::primary:kafka ;

echo "setup event bridging";

```

The first part is adding the **Minio host**, **Kafka bootstrap server** and **destination topic**.

The second part is adding the event hook for the **“put”** command targeted for **exams folders inside exams buckets**.

We wrap the above code into a Kubernetes job and run it.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: configure-bucket-notification
spec:
  template:
    spec:
      containers:
        - name: minio-client
          image: minio/mc
          command: ["/bin/sh", "-c"]
          args:
            - |
              mc config host add dc-minio http://minions-hl.minio-tenant:9000 minio minio123;
              mc admin config get dc-minio notify_kafka;
              mc admin config set dc-minio notify_kafka:primary \
                brokers="kafka-cluster-kafka-bootstrap.strimzi.svc:9092" \
                topic="exam-generator-topic" \
                tls_skip_verify=true \
                enabled=on;
              mc admin service restart dc-minio;
              echo "finish restart"; sleep 10s;
              mc event add --event "put" --prefix exams dc-minio/exams arn:minio:sqs::primary:kafka ;
              echo "setup event bridging";
      restartPolicy: OnFailure
```

Let’s test things out, we start a Kafka consumer for the **exam-generator-topic** topic, then we try uploading a sample file, we wait and an **ObjectCreated** event will appear.

{{< video src="/videos/minio-event-kafka.mp4" width="640" height="360" type="video/mp4" >}}

you may notice it’s S3 compatible, that will help us for not changing the code of Knative function to adapt Minio events.

## Knative & KafkaSource

The Knative architecture holds multiple components, so we will give a general overview of the architecture without going into much details. Knative is divided into two major components: serving and eventing.

The serving part handles managing serverless workload inside the cluster, here is the request flow of HTTP requests to an application which is running on Knative Serving.

![knative-request-flow](https://knative.dev/docs/serving/images/request-flow.png)

The eventing part is a collection of APIs that enable you to use an [event-driven architecture](https://en.wikipedia.org/wiki/Event-driven_architecture) with your applications. This part handles event listening on brokers and delivering it to SINKS (knative services)

![knative-eventing](https://knative.dev/docs/eventing/images/mesh.png)

As we finished explaining the core architecture, let’s move into installing Knative. The problem is that Knative doesn’t have a helm chart, so we will create one.

First to install the Serving part, two yaml files are needed: **serving-crds, serving-core**

Let’s create a helm chart with:

```bash
helm create knative/core
```

Delete all the files in templates folders except **_helpers.tpl** then download the two yaml files and place them inside templates folder.

```bash
wget https://github.com/knative/serving/releases/download/knative-v1.15.2/serving-crds.yaml -o serving-crds.yaml
```

```bash
wget https://github.com/knative/serving/releases/download/knative-v1.15.2/serving-core.yaml -o serving-core.yaml
```

Need another file to integrate knative with istio

```bash
wget https://github.com/knative/net-istio/releases/download/knative-v1.15.1/net-istio.yaml -o net-istio.yaml
```

Good, let’s add the chart to helmfile and install it.

```yaml
 - name: knative-core
    chart: ./knative/core
```

After installation, the above components will be created, and an ingress-gateway for serving Knative services(functions).

![knative-pods-k9s](/img/knative-pods.png)

Each Knative service must have a DNS to map to, am using Cloudflare to manage my domain.
We retrieve the Istio ingress gateway IP address

```bash
kubectl --namespace istio-system get service istio-ingressgateway
```

and create an A record referencing “kn-function” subdomain

![cloudflare image](/img/cloudflare.png)

Then we tell knative to serve services with the new domain

```bash
kubectl patch configmap/config-domain \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"kn-functions.enkinineveh.space":""}}'
```

Good, the serving component is ready, let's move into the event part.

The event part is the same as the serving with few added steps,
We first install the 2 yaml files: **CRDs** and **Core**, then we move into **Knative source for Apache Kafka**.

The KafkaSource reads messages stored in an existing [Apache Kafka](https://kafka.apache.org/) topics, and sends those messages as CloudEvents through HTTP to its configured sink(function)

KafkaSource composed of **controller** and **data plane.**

```bash
wget https://github.com/knative-extensions/eventing-Kafka-broker/releases/download/knative-v1.15.0/eventing-Kafka-controller.yaml -o eventing-kafka-broker.yaml
```

```bash
wget https://github.com/knative-extensions/eventing-kafka-broker/releases/download/knative-v1.15.1/eventing-kafka-source.yaml -o eventing-kafka-source.yaml
```

For testing purpose now, we can deploy the below code to create an event display service for displaying received events and KafkaSource to link the topic with Knative service.

service.yaml

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: event-display
  namespace: exam
spec:
  template:
    spec:
      containers:
        - image: gcr.io/knative-releases/knative.dev/eventing/cmd/event_display
```

kafka-source.yaml

```yaml
apiVersion: sources.knative.dev/v1beta1
kind: KafkaSource
metadata:
  name: kafka-source
  namespace: exam
spec:
  consumerGroup: kafka-group
  bootstrapServers:
    - kafka-cluster-kafka-bootstrap.strimzi.svc:909
  topics:
    - exam-generator-topic
  consumers: 2
  sink:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: event-display
      namespace: exam
```

The source needs the Kafka server connection URL, topic and the service name to send events to.


## Knative Service Building

But instead of an event display service, we need our exam generated service, luckily the AWS project has the code of the service.

We download the file, and we can see it’s using SNS and bedrock; two packages aren’t essential. Because SNS will be replaced by Kafka, and bedrock will be removed as we will use a static question for simplicity

```python
...
# sns = boto3.client("sns")
producer = KafkaProducer(bootstrap_servers=os.environ["KAFKA_BOOTSTRAP_SERVER"])
# bedrock = HelperBedrock()
...
```

and because we removed bedrock, we will pass a bunch of static questions

```python
...
    # response = bedrock.get_response(file, template_instruction)
    # response = bedrock.get_response(response, template_formatted)
    # json_exam = file_helper.convert_to_json_in_memory(response)
    output = io.StringIO()

    # Write the JSON data to the in-memory stream
    questions = [
        {
            "question": "Which country of those countries located in balkans ?",
            "options": ["Romania", "Germany", "Croatia", "Czech"],
            "correct_answer": "Romania",
        },
        {
            "question": "Where Tunisia is located?",
            "options": ["South America", "Asia", "Africa", "Oceania"],
            "correct_answer": "Africa",
        },
    ]
    json.dump(questions, output)

    # To ensure the content is in the buffer, seek the pointer back to the start of the stream
    output.seek(0)

...

```

One requirement for the code to work is wrapping it inside an API, FastAPI will get the job done.

Define a single endpoint that receives the event, converts it into JSON and passes it to the main function.

```python

from typing import Any, Union
import logging
from main import main
LOG = logging.getLogger(__name__)
LOG.info("API is starting up")

from fastapi import FastAPI,Request

app = FastAPI()


@app.get("/health")
def health():
    
    return {"Hello": "World"}

@app.post("/")
async def intercept_event(request:Request):
    event = await request.json()
    return main(event,None)

    

```

The service will need a container image to run, so we create a Dockerfile and requirements.txt for referencing package dependencies like FastAPI,Kafka..etc.

Dockerfile

```dockerfile
# 
FROM python:3.10

# 
WORKDIR /code

# 
COPY ./requirements.txt /code/requirements.txt

# 
RUN pip install --no-cache-dir -r /code/requirements.txt

# 
COPY ./ /code/app

# 
CMD ["fastapi", "run", "app/app.py", "--port", "80"]

```

requiremtents.txt

```text
boto3
fastapi
httptools
kafka-python
langchain
langchain-community
pdfminer.six
pip-chill
python-dotenv
uvloop
watchfiles
websockets
```

Now, we build the image, push into the registry and reference it inside the Knative service yaml.

```bash
 docker buildx build  -t gitea.enkinineveh.space/gitea_admin/exam-gen-fn:v1 . --push
```

We add the environment variables and secrets inside the value.yaml and create a helper to pass values as key,value into service.yaml and other values will be referenced directly.

The service will be accessed internally by the front, so there is no need for exposing it publicly, we can disable access outside of cluster by adding the following label:

```yaml
metadata:
  ...
  ...
  labels:
    networking.knative.dev/visibility: cluster-local

```

and the last missing resource `KafkaSource` will invoke the knative service when a new event reach the topic, let's create one

```yaml
apiVersion: sources.knative.dev/v1beta1
kind: KafkaSource
metadata:
  name: kafka-source
spec:
  consumerGroup: kafka-group
  bootstrapServers:
    - {{ .Values.env.normal.KAFKA_BOOTSTRAP_SERVER }}
  topics:
    - {{ .Values.env.normal.SOURCE_TOPIC_ARN }}
  consumers: 2
  sink:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: {{ include "charts.fullname" . }}

```

Cool, now redeploy the service

```bash
helmfile apply
```

Go to Console, upload a file, we notice a pod has been created from the knative service, wait for a few seconds and a new folder “question_bank” will be created with a JSON file holding the questions.

## Generation Frontend

Here comes the UI part, we copy the 3 files code from the repo and paste them over.

We will leave the code the same, so let’s build the image and push it to the registry

```bash
docker buildx build –t gitea.enkinineveh.space/gitea_admin/exam_generator_front_end . --push
```

We initialise helm chart and update the values to adjust service needs:

Reference the image inside the values.yaml

```yaml
image:
  repository: gitea.enkinineveh.space/gitea_admin/exam-take-frontend
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: "v1"
```

the service should listen on port 8501

```yaml
service:
    port: 8501
```

to access UI outside of kubernetes, we enable ingress, pass the host we want. One thing I noticed is that streamlit uses websocket for communication, so by adding this label, we tell nginx the websocket service it should use in case of websocket requests.

```yaml
ingress:
  enabled: true
  className: "nginx"
  annotations: 
    nginx.org/proxy-connect-timeout: "3600s"
    nginx.org/proxy-read-timeout: "3600s"
    nginx.org/client-max-body-size: "4m"
    nginx.org/proxy-buffering: "false"
    nginx.org/websocket-services: exam-taking-frontend-charts

  hosts:
    - host: exam-taking-frontend.enkinineveh.space
      paths:
        - path: /
          pathType: ImplementationSpecific
          backend:
            service:
              name: exam-taking-frontend-charts
              port:
                number: 8501
  tls: 
   - secretName: enkinineveh.space-tls-prod
     hosts:
       - exam-taking-frontend.enkinineveh.space
```

And finally create the environment and secret variable, we don’t forget the helper

values.yaml

```yaml
env:
  normal:
    API_GATEWAY_URL: "http://exam-taking-fn-exam-take-fn.default.svc.cluster.local"
    MONGO_URI: "mongodb://databaseAdmin:sHWKYbXRalmNExTMiYr@my-cluster-name-rs0.mongo.svc.cluster.local/admin?replicaSet=rs0&ssl=false"
    MONGO_TABLE_NAME: "score"
```

_helper.tpl

```tpl
{{- define "helpers.list-env-variables"}}
{{- range $key, $val := .Values.env.normal }}
- name: {{ $key }}
  value: {{ $val }}
{{- end}}
{{- end }}
```

Pass it in helmfile, apply the chart

```yaml
releases:
    ...
    - name: exam-generation-frontend
      chart: ./frontend/exam-generation-app/charts
      namespace: "exam"
```

```bash
helmfile apply
```

we visit the host, and here is the Exam Generation UI

![exam generation part](/img/exam-generation-screen.png)

We Upload another test file and get the green mark for successful upload.

![exam generation part](/img/exam-generation-green.png)

## Summary

As we finished the generation part of the architecture, the next step will focus on adding "taking-exam" part.

