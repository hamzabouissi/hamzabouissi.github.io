+++
title = 'GCP -> AWS Migration: Confession'
date = 2023-04-25T16:12:38+01:00
tags = ['AWS copilot',]
[cover]
  image= 'img/confession.jpg'
+++

(Knock on the door)

(door opens)(some background noise of the company like people shatter or talk)

Yeah, Come In

Hey

Hey

Hey, my Junior friend, have a seat.

Thanks

It's been 2 weeks since I assigned you the project, how it's going?

Good, I'm doing good, I just finished a few tasks and as you asked about my progress.

So, can you tell me how much progress you have made?

I think am nearly 40%, I somehow finished creating the dev environment, so I can test with developers the performance and durability of the infrastructure.

What about PostgreSQL, Redis, Configuration and Secrets Management? And I saw your email about using AWS AURORA, did you figure out the answer ?

I deployed the database as an RDS, well I didn't find a real benefit in using Aurora for the dev environment, and also it is somehow expensive, but maybe we can use it on the prod environment as it offers scalability. For Redis, AWS has a fully managed service called Elastic Cache with integrated monitoring service CloudWatch and the last one Configuration Management I read about the service (AWS Secret Manager) that I will use but didn't integrate yet.

That seems fair, have you faced some challenges learning AWS, I heard from my fellow developers that there are a few steeping curves in understanding the documentation and the services.

Somehow yes, as I don't know that much about AWS and the time isn't enough for learning and practising, I choose a tool to be abstract for me the underlying infrastructure for now, but I am willing to learn what happens in the background, so I can customize more.

What's the tool name?

AWS Copilot (the CTO typing the name, we need to hear the keyboard typing) it's an open source project ....(the CTO interrupt saying)

Hmm, I see, but wouldn't that add a new layer to learn, because most DevOps developers on the market will stick with IAC?

Yeah, I thought of that, I noticed that copilot works upon CloudFormation, which is an IAC tool for AWS and if you're asking why not use Terraform with built-in modules, I can tell with this method we will narrow in the future our range of search to DevOps with terraform knowledge as with the current situation we can teach our backend developers a few commands and that's enough to monitor the infrastructure.

Interesting, I'm the type of technical guy, can you explain more about the solution?

Ok, can I open my laptop to show you

(developer unpacking the laptop sound)

Yeah, Sure

have you thought of opening a startup (the CTO also) (CTO show some interest in the developer)

(think few seconds) No, I don't think I am ready, but maybe a small side hustle

Okay, let me log into my AWS account (while typing)(the developer said) add the MFA code and here we go.

(the sound of the CTO moving to the chair next to the developer)

So, here is the CloudFormation dashboard, those are the stacks that copilot deployed, you can also identify them by tags, generally they have tags copilotenvironment and copilot-application, at the end if somehow copilot didn't meet our expectation or we couldn't customize it further we can just modify, extend and boom. Now let me show the commands to set up and deploy the services.....

You didn't deploy the app Yet, right? (while interrupting)

Not Yet

Ok (with notification sound on the phone), Continue

To set up infrastructure first, we need to run `copilot app init` then `copilot env init` after that copilot will ask us a few questions about the public and private subnets and the availability zone of the load balancer after we confirm that we see 3 Cloudformation stack has been created which define a few resources(VPC, Subnets, RouteTables,InternetGateway, and Cluster for Containers), finally I will be writing all the steps on notion.

And the dependencies? (CTO asked)

Yeah, I did my search on deploy RDS and ElasticCache, Copilot doesn't support either of them as a built-in command, but we can extend and create the services by ourselves and this is what I have done, the extended functionality support CloudFormation's files

Got It, but, hmm(take a few seconds to rephrase ), but I'm worrying if we don't understand carefully and let copilot create the infrastructure, it may lead to creating unuseful services and you know we're short on money.

Yeah, I understand your concern but am willing to learn more about the architecture copilot created for us by that time I can see if copilot fits our needs else I remove it and advance my skills to manage by myself the rest of the infrastructure directly with cloud formation

That seems fair enough, Ok, here is what you gonna do in the next few days. After you finish deploying that config management thing, deploy the application on ECS, and handle it to the developers to try and check the performance, and what about the CI/CD

Copilot support it but didn't take a deep look into the documentation, I can deploy the application from my terminal, so let's tell the project manager whenever there is a pull request merged they notify me so I pull the code and push it for review

Noted, I will tell him

