+++
title = 'GCP -> AWS Migration: The Search'
date = 2023-04-12T15:28:36+01:00
draft = false
[cover]
    image = 'img/the search.jpg'
tags = ['AWS CDK']
+++


## **Key points**

- wake up with an email
- deciding not to go to the workspace
- read the task description and start searching
- lay on the bed again, thinking of ways to approach the problem
- get up and write his thoughts in his journal
- Search again and find copilot as a solution
- his alarm bumps for taking a walk

It's 8:35 AM, and the phone started vibrating with notifications, he used to receive the "Good Morning Babe" message from his girlfriend with a few newsletters emails, but this time, it was an email on his Zoho mail with the Subject: **[JIRA] Yassine assigned DEV-550 to you**

He opened his JIRA account from his phone, he was assigned a single task with the title

<figure class="kg-card kg-image-card"><img src="/assets/img/image.png" class="kg-image" alt loading="lazy" ></figure>

He didn't take a deep look at the description, but he felt he wasn't in the mood to go to the workspace, so he sent a message to the company Slack channel stating he is going to work from home today.

Took a quick bath, eat breakfast and returned to the desk, he opened again the JIRA task and now taking a better look at the description by catching some keywords: DEV Environment, VPC, Subnet, Route Table, LoadBalancer, Database, Redis, Containers, IAC

He wasn't familiar with the majority of keywords, and this was an advantage to stimulate him for learning. &nbsp;50 min passes and he should take a break for 20 min as he follows The 50-Minute Focus Technique, but it doesn't seem he cares, his eyes are filled with passion for finding the solution, he knew there is no direct tutorial or article to his problem.

His browser was filled with tabs of random articles with a side note app to summarise what he understood from each article

He lay down on the bed, taking a break and connecting the dots. The task seemed more clear now, but time was a crucial factor, he was telling himself

> _I pretty understand the workflow I should take but hmmm,but 3 weeks doesn't seem a lot of time, so let me find a tool to help me speed up the process and doesn't hook me up into much networking details for now_

> _Let me take a nap for now_

It was 3:51 PM, and this time he woke up prepared, he took his pc while lying on his bed. The goal is to find a solution to help him speed up building the infrastructure, A quick search leads him to a few tools like Terraform, Ansible, CloudFormation, Puppet, Vagrant

But it didn't seem the final result, Terraform & CloudFormation were IAC( Infrastructure as Code) tools an abstraction over cloud provider Rest API, Ansible and Puppet were Configuration Management tools he doesn't need for now and a vagrant is a virtualization tool which was away from his needs.

<figure class="kg-card kg-image-card"><img src="/assets/img/Screenshot_2023-03-08_14-42-23.png" class="kg-image" alt loading="lazy" ></figure>

In the same way, he was eager to find a tool to work on top of CloudFormation, he was typing all sorts of keywords around CloudFormation and IAC tools like "Tools that use CloudFormation, AWS open source projects for building networking infrastructure, What better than CloudFormation, CloudFormation Cons, CloudFormation Alternatives"

As a consequence, the results were somehow near his needs, he found a few tools to compare them and decide which one to go with

- [AWS CDK](https://docs.aws.amazon.com/cdk/v2/guide/about_examples.html)
- [HashiCorp WAYPOINT](https://www.waypointproject.io/)
- [AWS Copilot](https://aws.github.io/copilot-cli/docs)
- [AWS Amplify](https://aws.amazon.com/amplify/)
- [Fissaa](https://github.com/hamzabouissi/fissaa)

How extendible the tool is, How much is supported by the community if it's open source(GitHub Starts, pull request, issues), learning curve, have additional and unique features. Those were the criteria he judged upon choosing his next tool, for better clarity he created a table where he assigned each one a score

<figure class="kg-card kg-image-card"><img src="/assets/img/image-4.png" class="kg-image" alt loading="lazy"></figure>

After calculating the scores, AWS Copilot won the battle by how much is can be extendible into adding CloudFormation files, the community is quite good with 2.7k stars and few issues and on the pull requests side people seems interested in developing features on it, the learning curve scored 6 because documentation was pretty clear, and they have tutorial section and AWS is investing some videos also, it scored the highest on unique features because it helps deploy storage services(S3, Aurora, DynamoDB), customizing the underlying infrastructure like Subnet, NAT, Load Balancer...etc.

It's 7:19 PM, and the alarm rings saying "30-min Walk"...