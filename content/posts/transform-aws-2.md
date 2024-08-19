+++
title = 'Transform AWS Exam Generator Architecture to Open Source Part #2: Research and Planning'
date = 2024-08-12T14:40:04+01:00
draft = false
[cover]
    image = 'img/exam-thumb.png'
tags = ['Cloud-architecture','Problem-solving']

+++


## Replacing services Phase

In this article, we will pick this AWS architecture: “a serverless exam generator application for educator,” analyse it and find an open-source alternative solution for each service AWS provides, so if you are interested keep it up if you want to know more.

To give a context of how architecture works: it starts with the educator reaching AWS Cognito to create an account either using social media account or a simple email and password. A successful account registration will subscribe the user into SNS to receive notifications.

Then he continues into accessing a front-end application created by python Streamlit hosted on ECS to upload his pdf lecture, when the lecture gets persisted, a lambda function will get invoked to process the persisted file and generate an exam with help of the AI, in this case bedrock.

The exam gets generated and persisted into another folder in the same bucket, lambda will then send a message to user SNS subscription saying, “the exam is ready.”

In other side the students will register for an account, visit the front-end for passing the exam, after they finish answering the questions, the user result will get persisted on a DynamoDB table, and with the help of DynamoDB ability of streaming data, a lambda function will get triggered to calculate the scoreboard of users and send a message to teacher SNS subscription.

Now, after we get a general overview of the architecture let us move into the fun part “replacing the AWS services with open-source ones”.

Here list of the used services and their alternatives:

- AWS Cognito, to add user signup and sign-in features, also it supports social media integration. A suitable alternative is two open-source projects developed by Ory: Kratos for handling sign-in and signup features with social media integration, while Oathkeeper for authenticating and authorizing incoming requests.
- ECS, Elastic container service it offers easy mechanism for deploying container to the cloud, a replacement for that is Kubernetes services because we are hosting our stack on k8s
- S3, for storing PDF files, a well-known alternative is Minio, an object storage system and guess what? It has S3 compatibility, so we don’t have to change the code that interact with S3
- Lambda, a serverless function that will be spawned in case of an event or a direct communication with API Gateway, A known project for the serverless community is Knative, integrated with Istio and Kafka will get the job for us.
- API Gateway for the direct communication with lambda, there are many tools out there that can serve Knative traffic like Kong, ambassador but as we installed “Istio”, the Istio gateway will be enough to get the job.
- DynamoDB a fully managed, serverless, key-value NoSQL database, the main case of its existence is storing data and using CDC(change data capture) to invoke a lambda function for processing new insert data, as of this requirements any NoSQL database can get the job done, so we will use mongo managed with Percona operator.
- Bedrock, fully managed service that offers a choice of high-performing foundation models (FMs), for this one, am going to ignore it and use a static data.
- SNS fully managed Pub/Sub service for A2A and A2P messaging, its core functionality in this architecture is sending emails, so I replaced it with Knative function and Mailgun and then used a MongoDB table(subscribers) for storing educator’s email after signing up.

here is the final version of the architecture

![open_source](/img/exam-open-source.png)

As we finish the task of replacing the services, we will move into the practical side of the implementation.

The next articles will be divided into 3 parts: generation of the exam, passing the exam, authentication & notification.
