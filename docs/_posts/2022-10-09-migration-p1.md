---
layout: post
title:  "Migrating Stack from GCP to AWS Part 1 (The Power of Imagination)"
date:   2022-10-09 18:29:00 +0100
categories: Cloud
tags: AWS, Cloud, GCP
author: hamza bou issa
---

<img src="/assets/img/imagination-1-chapter-2.jpg">

A day before choosing his new destiny, the wooden house was empty, the rats were digging the floor searching for a single grain of wheat, a breeze of wind fighting to extinguish the flames, that flames were warming his body while imaging his new destination, he was lying on the dry grass, hands crossed behind his gaunt face, looking at the sky while he paints how life can get better after reaching his wonderland.

A land where rice is everywhere, cows and milk, sheep and wool, neighbors to talk to, and a fair ruler to protect his people from thieves, but his smile was interrupted by travelers' stories about the road.

A road full of hideous and scary monsters, plenty of robbers. The fear rushed to seed the pessimistic thoughts into his imagination. Not enough food to feed him during the journey, he wasn't a good sword man to fight the Bandits, and he lacked knowledge about the road.

After shuffling his mind, he stood up clenching his fists saying to himself: "I'am committing to this journey" and he headed up to his house...

# My Skills 

Before committing to a long journey, I must understand myself first, I must understand my strength and weakness.

To start with my strong points, I have already good experience building services with AWS, I created an open-source project to challenge myself to learn AWS, also I was a Backend developer and I know some of the scripting languages like Bash, C#, Python, I'm competent enough in English to read articles and a good searching skills for the problems.

Now discussing my weakness, I sometimes get wrong prioritizing my tasks or splitting them into small ones, also my imagination plays a big role in feeding my ideas, but this can prevent me from moving and looking too far from my steps.

With all of that am willing to learn other things during the journey and challenge myself.

# The Journey's need

Studying the journey path and needs is a must, so I took the time to set and think about the upcoming obstacles.

One of the key points of the migration is GCP, we already have services running there, so having some knowledge about GCP's current workflow (CI/CD, Maintaining VMs,…) is a must, which is something I currently lack.

On the other side, I'm limited by time therefore I need to deploy a test version of the application as fast as possible which will prevent me from understanding every service thoroughly.

Finally, I must expect that there are no exact alternatives for GCP's services on AWS and some implementations on AWS have problems(lack of proper examples on documentation,...)


# The Bag

A weapon is a final piece before the departure, I need a weapon to polish my skills with...
So I was searching for a weapon that helps me deploy my containerized application without the headache of setting up the underlying infrastructure (VPC, Subnet, and Security Groups), alongside that I want it to be customizable to add my own services or customized logic, I have been using CloudFormation before, so I would like to customize using CloudFormation for less-learning time.

So the result of my searching ended up choosing  <a href="https://aws.github.io/copilot-cli/">AWS copilot</a>, it suits all my requirements
from setting up infra to deploying containers to ECS 


# What will be doing next 

He stepped out of his house back warding while keep looking at his house, he was holding his tears after the memories splashed into his eyes, he swept his tears, saying: "sorry, but it's for the family, dad"



<div class="PageNavigation">
  {% if page.next.url %}
    <a class="next" href="{{page.next.url}}">{{page.next.title}} &raquo;</a>
  {% endif %}
</div>
