---
layout: post
title:  "Migrating Stack from GCP to AWS Part 1 (The Awakening)"
date:   2022-05-29 21:20:09 +0100
categories: Cloud
tags: AWS, Cloud, GCP
author: hamza bou issa
---

<img src="/assets/img/journey-chap1.jpg">



<a href="pengguin.com">Pengguin</a> is an online platform for teaching languages while playing games, the platform was hosted on GCP during the past 3 months until the CTO walked in asking to change into AWS and if this decision can benefits pengguin on the long term.

I present this journey as an old traveler(me) who decided to leave his hometown(GCP) looking for a better place(AWS) to live on, but he will face many challenges and hardship(new things to learn) that we will tackle together. 

- [Reason of leaving (GCP)](#reason-of-leaving-gcp)
- [Exploring the landscape (GCP)](#exploring-the-landscape-gcp)
- [Comparing The desired destination](#comparing-the-desired-destination)
- [Conclusion](#conclusion)
- [What we gonna do next](#what-we-gonna-do-next)


Enjoy the music while reading 🎵
<iframe style="border-radius:12px" src="https://open.spotify.com/embed/playlist/7tspLoZXjrHIPAbHnSmlwj?utm_source=generator" width="100%" height="380" frameBorder="0" allowfullscreen="" allow="autoplay; clipboard-write; encrypted-media; fullscreen; picture-in-picture"></iframe>


# Reason of leaving (GCP)

GCP pros are the ease of use, whenever you want to deploy services, set up databases, or Redis, all this can be set with a few clicks or a simple terraform file without understanding the complication of networks. But the ease of use can be a curse when you want to customize your infrastructure adding up the poor documentation, it may be painful to think about customizing.
On the other side, GCP didn't give us more money credits to host our application (that one was the biggest red flag we thought about migrating). 

# Exploring the landscape (GCP)

DevOps lifecycle is a combination of different phases of continuous software development, integration, testing, deployment, and monitoring. A competent DevOps lifecycle is necessary to leverage the full benefits of the DevOps methodology. So let's walk through each phases and see what services were used to fullfil the needs 

1. **Continuous Development**
   
   This cycle involves planning and coding the software. Here we decide at the end of every sprint which tasks are finished and we measure our success rate and what blocked us.
   There is no GCP services fullfil this phase so we have been using asana most of the time but recently we moved into Jira because of the ability to assign "story point estimate" on every task.
   
2. **Continuous Integration**
   
   Continuous integration (CI) is the practice of automating the integration of code changes from multiple contributors into a single software project. allowing developers to frequently merge code changes into a central repository where builds and tests.
   We have been using Cloud build to build and package our code into docker container and store them on Artifact Registry
   

3. **Continuous Testing**
   
   Continuous testing is an organizational process within software development in which business-critical software is verified for correctness, quality, and performance. 
   we can use Cloud Build to run the tests, we define additional step on the Cloud Build Configuration to run the "pytest" 

4. **Continuous Deployment** 
   
   Continuous Deployment is the art of seeing your code changes flow into your desired environment without intervening. 
   for this step, we didn't use Cloud Deploy as service to automate the deployment into production, instead we created Cloud Run instances and hosted our apps then we added another step on Cloud build that update Cloud Run Services . yeah I know this not the best practice but it worked for us :smile:.
   

5. **Continuous Monitoring**
   
   Continuous monitoring in DevOps is the process of identifying threats to the security and compliance rules of a software development cycle and architecture.
   This step required 2 different services from GCP, which we have been using:
    - Google Cloud Monitoring (including Google Cloud Managed Service for Prometheus & Grafana) for gaining visibility into the performance, availability, and health of applications and infrastructure.
    - Google Cloud Logging for real-time log management
  
6. **Continuous Feedback**
   
   Continuous feedback is essential to ascertain and analyze the final outcome of the application monitoring phase, this phase builds up a set of information about our next decision into improving the application.
   One type of feedback we collect:
   
   1. User Interaction:
      Flexibility is the main key feature we're looking for in our platform, and we want our client to experience that as well, that's why we use a set of tools(<a href='https://www.hotjar.com/'>hotjar</a>) to analyze and study website errors and malfunction during client surfing.





# Comparing The desired destination

If you’re leaving your home, you will think at least of replicating the same living standards you had but with less cost, it will be advantageous if you had a chance to obtain more benefits (better job, better home, better company, better weather ).

The same applies on our journey, we’re going to compare each GCP service to its AWS alternative and try at least to build the same functionalities we had, then later we think about reducing money and work time.

A note aside: we’re limited by time, because we’re running out of money on GCP, so we would like to migrate as fast as we can. We couldn't find the exact match service on AWS because they’re using different workflow to deploy applications, but we will try to do our best to design the process of build-test-deploy.


|Services \ Platform   | GCP                        |       AWS         |
-----------------------|----------------------------|-------------------|
|Continuous Integration| Cloud Build(Triggers)      | CodeBuild/CodeStar|
|Continuous Testing    | Cloud Build                | CodeBuild         |
|Continuous Deployment | Cloud Deploy               | CodeDeploy        |
|Continuous Monitoring | Cloud Monitoring           | CloudWatch        |

AWS pack all the recent service we mention above into one unit called **CodePipline**, usually we build a cloudformation file and containing the CodePipeline and we describe each phase.

# Conclusion

In this article, we talked about our old stack architecture on GCP , why we decide to leave into another platform (AWS) and how the desired destination should be.

# What we gonna do next
The road of migration can be difficult without carrying a set of tools(water, knife,..etc).
On the next chapter, I will tell you what tool I have been using to survive the hardship of the journey  