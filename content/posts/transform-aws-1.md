+++
title = 'Transform AWS Exam Generator Architecture to Open Source Part #1: Introduction'
date = 2024-08-12T14:32:49+01:00
draft = false
[cover]
    image = 'img/exam-thumb.png'
+++


## Introduction

Have you thought of creating an AWS architecture but with open-source projects?

In these articles, we will challenge ourselves and transform this AWS architecture: a serverless exam generator application for educators.

![the architecture diagram](/img/cloud-exam-architecture.png)

The solution enables educators to instantly create curriculum-aligned assessments with minimal effort. Students can take personalised quizzes and get immediate feedback on their performance.

We will transform and replace each service varying from Cognito, Lambda, DynamoDb, fargate…etc with its open-source counterpart and host it, where? guessed right, on a Kubernetes cluster.

If you liked the idea, keep it up for next articles.

here is the demo the app

{{< video src="/videos/exam-demo-2.mp4" width="640" height="360" type="video/mp4" >}}