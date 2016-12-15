---
layout: post
title:  "From Zero to Hero : Building a team of powerful developers"
date:   2016-12-15 08:42:27 +0100
categories: team building update
---

One year ago, I joined a small team of developers. We were three developers in charge of building 
a new application based on a legacy PHP website. This old application was in charge of managing 
rights and roles of users for others applications in the whole company. So, you can virtually 
have access to any application by granting you the rights to connect through this application. 
A very sensitive one!


The context was tense due to the sensitivity of data. The application must :
 - Be secured. You should not allowed unauthorized person to access others applications
 - Enforced traceability. We should be able to link who has access to what and when
 - Functionality compatible with the process and infrastructure. The application relies on a LDAP directory with strange ACL and construction, a legacy of 10 years of evolution.


Among all of those points, the main challenge was the team. The core team was made of three 
developers and one business owner. 


I’m a 9-year java experienced developer, the two others were junior, less than 2 years. 
One of the two developers was against the use of TDD. None of them has been in an agile 
team before or familiar with CI/CD principle.


We goes from there to an Agile team familiar with TDD, ATDD/BDD, DDD, Continuous Delivery 
and user-focused which was able to produce a product that our users will delight. Here are 
feedbacks and advices about how we manage to You have to build the team and understand people with 
whom you are going to work for the next months and at the same time, you would have to change habits. 
Don’t forget it is also a long-run, so you will have not to demotivate yourself.


# Building the team


**“Individuals and interactions over processes and tools”**


## You are a member of a team


The Man is a social animal. Projects are more and more complicated. Some many skills are needed 
if you want to build a good application : developer, designer, tester, security expert, … Even 
if you see yourself as Superman, you will sooner or later need help.


First thing to do is you. You have to be ready to work in a team. You have to discipline yourself. 
You have to learn to work with others persons. Don’t be afraid to open to others and share what you feel.


This is the most complicated part. Working in a team will make you discover a lot of things about yourself.


## Creating links : Sharing & Trust


**Build projects around motivated individuals. Give them the environment and
support they need, and trust them to get the job done.**


You have to trust each others. Trust is really a key in empowering the team. Trust means knowing how people 
will react and not using it as a way to manipulate them. Take time to listen about what everyone wants and 
looks in the project (You can use the [moving motivator][moving-motivator] activity) AND outside the project. 
For example, share a coffee, eat together during lunch time, …


I recommend the great book “Five dysfunctions of a team” from Patrick Lencioni, which gives good advices about dynamic of a team.


Learn to celebrate small victories. It is a good way to create link between people and to relax after hard work. 
After the demo and before the retrospective, we used to celebrate our victory with a small cake.


## Giving responsibilities


*The best architectures, requirements, and designs emerge from self-organizing teams.*


If you are in charge of building the team, remember that everything should not rely on your shoulders. 
If so, you would be seen as “a project manager” instead of “a team member”. An agile team is self-organized. 
You have to trust people and let them take the lead. 


In the team, I was designed as the project manager, because of my experiences. Don’t take such attributions 
as a matter of fact. When someone suggests, encourage him or her to do it. That why I encourage a lot people 
to experiment and try. They know what is good for the project


People will have to self-discipline. It was more easier than I thought, because team members are keen to take 
initiatives and want to progress. During the retrospective, it is good to give an honest feedback on the 
improvements of everyone. Having recognition is a good motivator.


## Why do we need this team ?


*Our highest priority is to satisfy the customer through early and continuous delivery of valuable software.*


The team exists for a reason. The reason must be known by everyone. Write it at the top of your board, talk 
about it, … You have to understand that you are building the software for someone. So, be close with your the 
product owner. Sometimes, the product owner is a “proxy”. In this case, take time to meet the real users. Don’t 
be afraid to see them directly. You should have free access to them. Understand their pains, their motivations, … 
It can be a good start if you want to use personnaes.


With the product owner, define in a sentence what is the global objective. Define also which business requirements 
are the most important (reliability, security, design, …).


The next step will be to understand the domain. If it is a new domain, you will have to discover the “vocabulary”. 
Start a glossary and share the same words with the business. Also, learn the “grammar”: which processes are involved? 


During the project, you should challenge the business about their needs. Ask what is expected in terms of business 
outputs : more customers? More money? … The most you know about the business, the better you would be in anticipating their needs.


For example, during the project, we had regular meetings with the business. Before starting the implementation of a user 
story, the business has to be precise about the reason (namely the “in order to” of user story).

```
**As** a right manager, **I want** to grant rights to an user **in order to** fulfill the request and allow him to use a managed application.
```


However, be also aware when you are doing shortcut or over-engineering. Always double-check with the business. 
In our DoD, we had to demonstrate the user story if we wanted to close it. At this moment, we were clarifying assumptions.


## Goodwill


Goodwill is a mind-spirit. Every time something wrong happens, don’t blame, don’t look for a culprit. Moreover, you can 
be the culprit, so don’t do finger pointing. 


Things are done with their best will. Mistakes will happen. And even if someone continues to do something wrong, you 
should remember it is a teamwork to prevent it.


When the CI server got broken, don’t blame. The whole team should stop, find the reason and solve it. Keep the incident 
somewhere and later, you can proceed a root cause analysis.


# Transform the team


**Continuous attention to technical excellence and good design enhances agility.**


## Educating


The first action was to introduce the team to Craft and Agility. I organized session to educate both developers and managers :
 - Classical presentations about TDD, Agility, CI, …
 - Hand-on sessions based on katas like gilded rose
 - Pair programming
 - Serious game 


There, you have to convince people about the advantages (and disadvantages) of methods and to listen their criticisms. 
This part is one of the most important steps. Be transparent, so people will trust you.


People were really keen on learning new domains. It took a lot of time. Giving lessons are important. But, you should 
know that understanding new concepts and using them can be really long.


## Habits


I proposed the Scrum methodology as our project methodology. Not only it is an efficient way of driving a project, 
it is also a good framework to create habits and to start for beginners. It is also a famous method, so management 
would be keen to let you use it and to support you.


The “Definition Of Done” for development was also created. It gave clear rules of what “Done” really means. I define 
the first version, but soon the team started to modify it, given them the opportunity to appropriate themselves this 
new rule. At the end, the DoD was often used as a checklist, fully caring its duties.


## Don’t oblige, propose


An agile team is defined by their capacity to be self-organized. To achieve it, you should propose instead of force.
For example, at the beginning of the project, junior developers were against the use of unit testing and TDD. 
So, I let them develop in their way. Soon, my tests started to be broken. They could not imagine that their developments 
could break something else. They understood that unit tests are a safety net. At the beginning, they were saying “Unit 
tests are useless” and they changed their mind to “Unit tests saved us!”. A great victory.
TDD was harder to deploy in the team. They were more using tests to protect them. The code was complex. Here, pair 
programming on complex user stories showed them that TDD helped them to write more efficient code.


# Marathon


**Agile processes promote sustainable development. The sponsors, developers, and users should be able to maintain a constant pace indefinitely.**


## Finding help


You should know that you don’t know everything. Sometimes, you can need help about Agility or team management. 
Don’t be afraid to look for help. In this project, I requested a agile coach which gave us good insights about 
our way of working. 
You should learn to work with managers. Often, they could facilitate the communications inside and outside the 
team, find new resources and help you defuse tense situations.


## Fighting boredom


Creating a product can be a long-term process. Sometimes months, sometimes years. When the team has passed the 
challenging part of understanding and mastering Agile, Craft, … boredom can become your new enemy. 


The user stories will look always the same, the business domain will not be tricky anymore, … Nothing to keep 
you in alert and give you the feeling that the team is improving. With boredom comes mistakes. You will start to 
be lazy, to have the feeling that you can do mistakes, ...


After the 5th sprint, we introduce more game & challenges during sprints. For example, we had a game to see who is 
writing the best code, the best documentation, … Also, you can use innovative retrospectives. People will be excited 
before. A good example is the [Star Wars retrospective][star-wars-retrospective]. I have also relied on 
the “[Agile retrospective kickstarter][agile-retrospective-kickstarter] by Alexey Krivitsky.





Building a team is challenging process. It requests a lot of energy and positive emotion. The final result was really good. 
People were ready to continue to work for months, they had acquired new and good habits… Remember that as a team builder, 
you would also need time to breath. It is a long run for you too!


Citations are from ["Agile Manifesto"][agile-manifesto] and ["Agile Manifesto
Principles"][agile-principles].


[agile-manifesto]: http://agilemanifesto.org/
[agile-principles]: http://agilemanifesto.org/principles.html
[moving-motivator]: https://management30.com/practice/moving-motivators/
[agile-retrospective-kickstarter]: https://leanpub.com/agile-retrospective-kickstarter/
[star-wars-retrospective]: http://blog.octo.com/retrospective-agile-sur-le-theme-star-wars/


