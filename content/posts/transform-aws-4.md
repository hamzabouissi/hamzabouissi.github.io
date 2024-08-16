+++
title = 'Transform AWS Exam Generator Architecture to Open Source Part #4: Exam Passing'
date = 2024-08-12T14:40:10+01:00
draft = true
[cover]
    image = 'img/exam-thumb.png'
+++


## Brief Description

In this article, we create the passing part of the exam, the architecture is composed of:

- kubernetes service for Taking Exam UI.
- Knative service for taking exams.
- MongoDb for storing student answers.
- KafkaConnect to capture changed data on MongoDb and move it to a KafkaTopic.
- Knative service for calculating the scoreboard and send it into a topic.

We will follow the same approach in the previous article, we tackle dependency-free services first. the the UI app will require the knative service and mongo to work, so we start with knative service and mongo and then we move into UI and finally the Kafka connect integration with mongodb and knative.

## Knative Fn

We download the function [take_exam.py](https://raw.githubusercontent.com/aws-samples/serverless-exam-generator-from-your-lecture-amazon-bedrock/main/TakeExamFn/take_exam.py) file, create a FastAPI wrapper but this time there is no event instead the function is called directly from the UI.

If we inspect the **take_exam.py** code, we can see it’s expecting a **queryStringParameters** dict inside the event and inside it a string called: **object_name** which will be referencing the exam file.

So here is the **app.py**

```python
@app.get("/")
def intercept_event(object_name:Union[str,None]=None):
    if object_name:
        event = {
            "queryStringParameters": {
                "object_name": object_name
            }
        }
    else:
        event = {
            "queryStringParameters": {}
        }
    result = lambda_handler(event=event,context={})
    return result
```

Cool, now we build, push

```bash
docker buildx build -t gitea.enkinineveh.space/gitea_admin/exam-take-fn:v1 . --push
```

and reference the image and we don’t forget the Minio environment values.

```yaml

image:
  repository: gitea.enkinineveh.space/gitea_admin/exam-take-fn
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: "v1"

...
env:
  normal:
    BUCKET_NAME: "exams"
  secret:
    AWS_ACCESS_KEY_ID: "minio"
    AWS_SECRET_ACCESS_KEY: "minio123"
    AWS_ENDPOINT_URL_S3: "http://minions-hl.minio-tenant:9000"
```

After adding the helm chart to helmfile release and apply it, we inspect the knative service to get the ingress URL, which will be used by the frontend later.

```bash
kubectl get kservice exam-taking-fn -n exam

NAME                          URL                                                            LATESTCREATED                       LATESTREADY                         READY   REASON
exam-taking-fn   http://exam-taking-fn.exam.svc.cluster.local   exam-taking-fn-00001   exam-taking-fn-00001   True
```

## Mongodb

Next part is Mongodb, we will use the percona operator for managing the MongoDb cluster

first start by installing the chart

```yaml
repositories:
  ...
  - name: percona
    url: https://percona.github.io/percona-helm-charts/
releases:
   ...
  - name: percona-operator
    chart: percona/psmdb-operator
    namespace: mongo
```

Then we copy paste the following script to deploy a Cluster with 3 replicas across nodes “[kubernetes.io/hostname](http://kubernetes.io/hostname)” and a 1Gb of storage for now:

```yaml

apiVersion: psmdb.percona.com/v1
kind: PerconaServerMongoDB
metadata:
  name: mongo-cluster
  namespace: mongo
spec:
  image: percona/percona-server-mongodb:4.4.6-8
  replsets:
    - name: rs0
      size: 3
      volumeSpec:
        persistentVolumeClaim:
          resources:
            requests:
              storage: 1Gi

```

Rerun **helmfile apply**, wait a few minutes and here the cluster status is **ready,** and the **endpoint** for kubernetes internal access.

![percona cluster status](/img/mongo-cluster.png)

But for database access we need credentials, which will be found on a secret named **mongodb-cluster-secret**.

## Frontend UI app

As both dependencies: exam-take function and mongodb are created, we’re able to deploy the exam-take frontend now.

Having a look at the [frontend code]([http://](https://github.com/aws-samples/serverless-exam-generator-from-your-lecture-amazon-bedrock/blob/main/frontend/take-exam-fe/quiz.py)), the application needs a:

**API_GATEWAY_URL** which is in our case the knative service url we got before:

```yaml
env:
  normal:
    API_GATEWAY_URL: "http://exam-taking-fn.exam.svc.cluster.local"
```

```python
API_GATEWAY_URL = os.getenv('API_GATEWAY_URL')
```

A token to decode the authenticated user’s email, we will escape this part for now as it requires an authenticated service, we will pass a static email for every student.

```python
# headers = _get_websocket_headers()
# token = headers.get('X-Amzn-Oidc-Data')
# parts = token.split('.')
# if len(parts) > 1:
#     payload = parts[1]

#     # Decode the payload
#     decoded_bytes = base64.urlsafe_b64decode(payload + '==')  # Padding just in case
#     decoded_str = decoded_bytes.decode('utf-8')
#     decoded_payload = json.loads(decoded_str)

#     # Extract the email
#     email = decoded_payload.get('email', 'Email not found')
#     print(email)
# else:
    # print("Invalid token")
email = "tunis@gmail.com"
```

And to save the answers, a Dynamodb was used. We replaced the code with MongoClient and passed the host and table name as environment variables.

quiz.py

```python
from pymongo import MongoClient
...
def save_quiz_results(data):

    # Initialize DynamoDB table
    table_name = os.getenv('MONGO_TABLE_NAME')
    client = MongoClient(os.getenv("MONGO_URI"))
    db = client['exams']
    collection = db[table_name]
    collection.insert_one(data)
...
```

values.yaml

```yaml
env:
  normal:
    MONGO_URI: "mongodb://databaseAdmin:sHWKYbXRalmNExTMiYr@my-cluster-name-rs0.mongo.svc.cluster.local/admin?replicaSet=rs0&ssl=false"
    MONGO_TABLE_NAME: "score"
```

We add the pymongo package in the requirements.txt file, then build and push.

We do the same as the generation front-end app, reference the image, change port, enable ingress, specify websocket service and done

```bash
docker buildx build -t gitea.enkinineveh.space/gitea_admin/exam-take-frontend . --push
```

```yaml

image:
  repository: gitea.enkinineveh.space/gitea_admin/exam-take-frontend
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: "v1"

service:
  type: ClusterIP
  port: 8501

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

Add the chart into helmfile, apply it

```yaml
releases:
  ....
  - name: exam-generation-frontend
    chart: ./frontend/exam-generation-app/charts
    namespace: "exam"

  - name: exam-taking-frontend # we added this
    chart: ./frontend/exam-taking-app/charts
    namespace: "exam"

  - name: exam-generation-fn
    chart: ./knative/ExamGenFn/charts
    namespace: "exam"

  - name: exam-taking-fn # we added this
    chart: ./knative/ExamTakeFn/charts
    namespace: "exam"

```

and we can access the UI Now.
![front exam passing](/img/front-exam-passing.png)

Now we try the whole thing, remove past files and start, upload the supposed test pdf file, wait for generation, check the taking-exam app, we see it listed the exam, we click, and it generates a form, we answer and voila.

Finally let’s check the mongodb table for the answers, we list documents inside the score **table,** everything looks good.

here is a demo for more visual pleasing experience

{{< video src="/videos/exam-taking-demo.mp4" width="640" height="360" type="video/mp4" >}}


## Kafka Connect & Function

One of the reasons I choose Kafka besides its seamless integration with knative, is the ability to capture data changes (CDC) from other sources and forward it to topics, this ability is called Kafka Connect.

Kafka Connect is a tool for scalably and reliably streaming data between Apache Kafka® and other data systems. If we want to capture data the time it gets added into Mongodb, we must configure the Kafka Connect to use the Mongodb Connector which internally listens for Mongodb CDC.

Kafka Connect will be deployed as a separate Cluster but under the same kubernetes cluster of course

![kafka connect cluster](/img/kafka-connect-arch.png)

Here is the yaml **kafka-connect.yaml**  for creating the cluster:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnect
metadata:
  name: connect-cluster
  namespace: strimzi
  annotations:
    strimzi.io/use-connector-resources: "true"
spec:
  replicas: 1
  image: gitea.enkinineveh.space/gitea_admin/connect-debezium:v2
  bootstrapServers: kafka-cluster-kafka-bootstrap.strimzi.svc:9092
  config:
    group.id: connect-cluster
    offset.storage.topic: connect-cluster-offsets
    config.storage.topic: connect-cluster-configs
    status.storage.topic: connect-cluster-status
    

  logging:
    type: inline
    loggers:
      connect.root.logger.level: "INFO"
```

- **bootstrapServers** for connecting to the Kafka server.
- **group.id** unique id for defining group of workers
- And those config are internal topic for managing connector and task status, configuration, storage data
- And the most important property is **image**, the image must be from strimzi/Kafka, and it should add the mongodb connector under plugins folder to be used later by connectors, I deployed my own image , you can find the code in the repo.
- Setting **use-connector-resources** to true enables KafkaConnectors to create, remove, and reconfigure connectors

The last interesting part now, is adding the connector:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnector
metadata:
  name: mongodb-source-connector-second
  namespace: strimzi
  labels:
    strimzi.io/cluster: connect-cluster
spec:
  class: io.debezium.connector.mongodb.MongoDbConnector
  tasksMax: 2
  config:
    mongodb.connection.string: mongodb://databaseAdmin:sHWKYbXRalmNExTMiYr@my-cluster-name-rs0.mongo.svc.cluster.local/admin?replicaSet=rs0&ssl=false
    topic.prefix: "mongo-trigger-topic"
    database.include.list: "exams"
    collection.include.list: "score"

```

We specify the class for **MongoDbConnector** plugin, and the last properties are needed for mongodb, here the **topic prefix** is “mongo-trigger-topic”, but we must create a topic with following name, **mongo-trigger-topic.exam.score, why?**

Because KafkaConnector use this format: prefix.db.colletion to forward events, means if we add record in score collection the topic name should, <topic.prefix>.exam.score

So here is the code for topic creation:

```yaml

apiVersion: kafka.strimzi.io/v1beta1
kind: KafkaTopic
metadata:
  name: mongo-trigger-topic.exams.score
  namespace: strimzi
  labels:
    strimzi.io/cluster: "kafka-cluster"
spec:
  partitions: 2

```

One thing to notice is the event format that will be forwarded from kafka connect, here is an example of the format:

```json
{
   
    "payload": {
        "before": null,
        "after": "{\"_id\": {\"$oid\": \"66a279b8f98a9d0379570575\"},\"email\": \"tunisia@gmail.com\",\"score\": 0,\"result\": \"failed\",\"details\": [{\"question\": \"Which country of those countries located in balkans ?\",\"user_answer\": \"Germany\",\"correct_answer\": \"Romania\",\"is_correct\": false}]}",
        "updateDescription": null,
        "source": {
            "version": "2.7.0.Final",
            "connector": "mongodb",
            "name": "email-topic",
            "ts_ms": 1721924024000,
            "snapshot": "false",
            "db": "exams",
            "sequence": null,
            "ts_us": 1721924024000000,
            "ts_ns": 1721924024000000000,
            "collection": "score",
            "ord": 1,
            "lsid": null,
            "txnNumber": null,
            "wallTime": null
        },
        "op": "c",
        "ts_ms": 1721924024568,
        "transaction": null
    }
}
```

as we're adding a new data the `before` property is empty and the `after` should contains the persisted data.

From the architecture you may notice we need a function to consume the stored events in **mongo-trigger-topic.exam.score.** The code is using **SNS and DynamoDb** we replace them with Kafka and mongo.
We start by initialising the KafkaProducer for sending to email Topic. The `dynamodb_to_json` function will be replaced by mongodb_to_json.

```python
...
from kafka import KafkaProducer

producer = KafkaProducer(bootstrap_servers=os.environ["KAFKA_BOOTSTRAP_SERVER"])

# Utility function to convert DynamoDB item to regular JSON
def mongodb_to_json(mongo_item):
    return json.loads(mongo_item)

...
```

Then we remove the for loop as knative doesn’t use batching stream and replaceit with a code to get event data from ‘after’ property and  at the end we send messsage to Email Topic.

```python
...
def lambda_handler(event, context):
    topic_arn = os.environ['EMAIL_TOPIC_ARN']

    image = event['payload']['after']

    # Convert DynamoDB JSON to regular JSON
    item_json = mongodb_to_json(image)

    # Format the message as a score card
    message = format_score_card(item_json)
    try:
        producer.send(topic_arn, message.encode())
    except Exception as e:
        print(f"Error sending Kafka notification: {e}")
        raise

    return {
        'statusCode': 200,
        'body': json.dumps('Lambda executed successfully!')
    }
...
```

as the Kafka Connector will persist the added data into a topic, we should create a KafkaSource to consume it and forward it to the mongo function.

```yaml
apiVersion: sources.knative.dev/v1beta1
kind: KafkaSource
metadata:
  name: mongo-source
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

and finally pass environment variables to helm values.yaml

```yaml
...
env:
  normal:
    KAFKA_BOOTSTRAP_SERVER: kafka-cluster-kafka-bootstrap.strimzi.svc:9092
    SOURCE_TOPIC_ARN: mongo-trigger-topic.exams.score
    EMAIL_TOPIC_ARN: email-topic
```

Now let’s test the entire process, we first open a side terminal for watching email-topic data.
Visit the frontend app, select an exam, answer the questions, wait for second and here is the data sent to email-topic.

/demo

## Conclusion

Now, we finished both parts, the generation and the taking of the exam parts, we will move into securing the access to authenticated accounts and later send notification to educator about student scores.
