---
layout: post
title:  "Learning To Deploy Cloud Native Apps, Second Week(Part 1)"
date:   2022-05-29 21:20:09 +0100
categories: Cloud-Native apps on AWS
tags: AWS, Cloud, apps,ECS, Beanstalk
---

<img src="/images/chapter-2-explorer-1.jpg">

# Introduction
In this article, I will describe how I learned AWS ECS(Elastic Container Service) and tried to deploy  a demo application to satisify my curiosity :smile:<br>
Try to enjoy reading and to get a beautiful experience listen to this loved playlist 🎵
<iframe style="border-radius:12px" src="https://open.spotify.com/embed/playlist/1FfdbDiwCRjGNPu6nJJxGU?utm_source=generator" width="100%" height="80" frameBorder="0" allowfullscreen="" allow="autoplay; clipboard-write; encrypted-media; fullscreen; picture-in-picture"></iframe>

# Why we're here(Goals) 
Let me clear my mind, what I want is deploying  at least a demo application on ECS to understand the basics then I can customize more, replacing the demo  with my application.

# Reading AWS documentation
<img src="/images/chapter-2-reading.jpg">

Let's start exploring our beloved documentation, here is a link : <https://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html>.
Well scrolling through the documentation, give me nervous because there is no a simple way of knowing how to deploy things.
but wait ?! ahhhah I found you, there is tutorial page .

<img src="/images/tutorials_screen.png">

I found the first tutorial [Tutorial: Creating a cluster with a Fargate Linux task using the AWS CLI](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_AWSCLI_Fargate.html) interesting, as a first step so let's discover it ma son.

# Cutting the wires(Prerequisites)
A general look at the article, we see there are some prerequisites we need to accomplish first.
Sadly this will slow down our steps into deploying, but let's understand the documentation and walk throught it 
Note: Don't Forget to listen to music 🎵

<img src="/images/chapter-2-cutting-wires.jpg">

I will skip the first step(Installing the CLI) and start directly with:
- [The steps in Set up to use Amazon ECS have been completed](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/get-set-up-for-amazon-ecs.html).

Now, give yourself a try to understand the above page and comeback to compare our thoughts. here is a clue:<br>
**it's normal you don't understand everything**, just read what you need to deploy a demo. if you didn't know how, don't worry I will show you 😉

## Ready?, Let's compare thoughts 
<img src="/images/chapter-2-mans-talking.jpg">

Everytime you read AWS Documentation try to look at the right of the page to get a general view of what the page is showing.
for our case:

<img src="/images/chapter-2-prerequisite-page-list.png">

1. Create an IAM user <br>
On this part, I understand that we shouldn't from the security perspective use our root account to access created resources, alternatively we need to create an IAM user and give him the required permissions that he needs to manage AWS Services.<br>
While creating an IAM user, we need to download his creedentials so we can use them on AWS CLI that you installed on the previous step.

2. Create a key pair
Let's suppose you created an EC2 server and you wanna access it from your localhost, key pair come to  the rescue.
you  

1. Create a virtual private cloud

2. Create a Security Group





# Third Try: Understand how to read AWS documentation


# Conclusion