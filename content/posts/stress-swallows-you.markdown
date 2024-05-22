+++
title = 'GCP -> AWS Migration: Stress Swallows You'
date = 2023-04-20T15:36:32+01:00
draft = false
tags = ['AWS copilot','ECS', 'RDS', 'lambda', 'DynamoDB']
[cover]
    image = 'img/stress.jpg'
+++


> _3 days passes, and I'm struggling on the same bug, Am I looking at the wrong side of the window, I don't know, I think the best way to understand is going back and examine every command line and line of code I wrote._

So at first, After I took the decision to use AWS copilot, I look at the task on JIRA, analyzed it carefully and saw the need to deploy application dependencies first, one of the dependencies is the database **.** We have been using PostgresSQL V14, so we just need the same version on the dev environment.

I run through the documentation, to catch any command on how to deploy a database with underlying infrastructure(VPC, Subnet, Route Table, ...). The first command I saw

    copilot init

It fulfils the need of creating the underlying infrastructure, but it requires an application ready to deploy, but this is not the case now. A few minutes after and stumbled upon another command with the description "creates a new [environment](https://aws.github.io/copilot-cli/docs/concepts/environments/) where your services will live."

    copilot env init

When I run the above command, it asked me to run `copilot app init` first. And here is the output of environment creation

<figure class="kg-card kg-image-card"><img src="/assets/img/image-5.png" class="kg-image" alt loading="lazy"></figure>

From my understanding of the output and manifest.yml file, it seems after running `copilot env deploy` it will create two public and private subnets on separate regions. So I proceeded with the command, and here is the output

<figure class="kg-card kg-image-card"><img src="/assets/img/image-6.png" class="kg-image" alt loading="lazy"></figure>

I googled a few terms I didn't understand like ECS, security groups, and DNS namespace to have basic knowledge of what happens in the background.

The following task was deploying the database and this is when things got trickier, now one of the features of copilot is a command to deploy storage services like database, file system... etc. It supports two types of databases DynamoDB, Aurora

Aurora seems a great option as it's fully compatible with PostgresSQL, so I tried to deploy a cluster using the following command

     copilot storage init -n cluster -t Aurora --lifecycle environment --engine PostgreSQL

At the same time, I opened Thunderbird and I messaged the CTO asking if it was ok deploying Aurora instead of an RDS. I went back to the command and I found

> Couldn't find any workloads associated with app noteapp, try initializing one: copilot [svc/job] init .   
> ✘ select a workload from noteapp : no workloads found in noteapp

Well, the problem is obvious, I cannot deploy a storage service unless I create a service first and by service I mean containerized application.

I walked through the documentation again, and I found a magical feature that says "Modeling Additional Environment Resources with AWS CloudFormation", this feature gave me the ability to deploy resources on an environment based.

That gave me goosebumps to understand CloudFormation, as it seems crucial in the next phases. The methodology was deploying a demo architecture to get comfortable with the services and the whole flow, I deployed one of the well-known architectures which is **lambda function & DynamoDB** and here is my recap

CloudFormation file structure consists of 3 main blocks **Parameters** (Optional), **Resources** (Required), and **Outputs** (Optional).

- **Resources** block encapsulate the services we need to deploy, Each service requires 2 properties **Type, Properties** and each service has different Properties, an example of that for `Lambda Function` there are **Handler** , **Runtime** , **Code** properties while on `DynamoDB::Table` there are different properties **AttributeDefinitions** , **KeySchema** , **ProvisionedThroughput**
<figure class="kg-card kg-image-card"><img src="/assets/img/cloudformation_example_explain.png" class="kg-image" alt loading="lazy"></figure>
- **Parameters** block is the way to pass dynamic variables to the services, a real example of that, let's suppose I need 3 lambda functions but each of them exists on a different S3 bucket, well I can create 3 functions on the same file and give each one a different bucket, but also I can add a Parameter <u><strong><strong>LambdaBucketSource </strong></strong></u>and pass it to **Function** Service
<figure class="kg-card kg-image-card"><img src="/assets/img/cloudformation_parameter.png" class="kg-image" alt loading="lazy"></figure>
- **Outputs** block the main use is showing service outputs or exporting it to another CloudFormation stack, an example may be when deploying an EC2 instance on a public, I may need the public IP address to ssh or serve the web app inside EC2 or export that output to another stack to use it
<figure class="kg-card kg-image-card"><img src="/assets/img/image-1.png" class="kg-image" alt loading="lazy" </figure>

Now, AWS CLI has a built-in command for managing CloudFormation files, to create resources the first time the command is

    aws cloudformation create-stack --stack-name resource_stack --template-body file://cloudformation.yml --capabilities CAPABILITY_NAMED_IAM

and for update

    aws cloudformation update-stack --stack-name resource_stack --template-body file://cloudformation.yml --capabilities CAPABILITY_NAMED_IAM

An additional cool feature of AWS CloudFormation is the built-in managing dashboard where you can see your stacks and their status

<figure class="kg-card kg-image-card"><img src="/assets/img/image-2.png" class="kg-image" alt loading="lazy"  ></figure>

Then I started to think about integrating CloudFormation with Copilot until I looked through the window, and it was almost dark and my back was hurting, so I took the sign and went for a little bit of social life

