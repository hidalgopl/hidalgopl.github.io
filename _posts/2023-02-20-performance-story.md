---
layout: post
title:  "Fixing performance bottlenecks: Joe's or Bob's way"
date:   2023-02-20 21:12:11 +0200
author: "Paweł Bojanowski"
categories: performance stories json profiler flamegraph
---

After watching a video lecture by Casey Muratori about [Philosophies of Optimization](), I recalled a story that happened to me in one of the previous gigs which is, in my eyes, worth sharing.
If you haven't watched the video, I'll link it at the end of this video. It's worth watching; to give you more context Casey starts with naming three philosophies of optimization: 1) optimization 2) non-pessimization and 3) fake optimization.
To set the stage for the story, let's introduce our two main characters in the story: Bob and Joe, two senior software engineers, my team-mates. As one of the most junior people in the team, I took the role of curious observer, which benefited me in becoming a better engineer plus being able to share this story now, as a narrator.

## The problem

Our system was a bunch of distributed micro macro micro services. 
We got a report that the average request processing visibly increased and we were task to hunt down the root cause of it. 
After initial investigation, we found a bottleneck: microservice called **OmegaStar**&trade; (not really, but I watched way too many **KRAZAM** [videos](https://www.youtube.com/@KRAZAM) recently). 
Bob and Joe volunteered to investigate further, separately and get back together after some time, to compare the findings.
Bob was a guy who was always up-to-date with the language tooling, hyped technologies and so on. Besides, he was a really intelligent and knowledgeable engineer. 
He had professional experience working with codebases that were optimized for blazing fast execution. 
He used his tooling knowledge to employ **ptracing profiler**, which didn't require any modification in the microservice's code. He collected sample data and created a bunch of cool-looking flame graphs. His data-driven approach led him to the following conclusion: within this microservice, the main offender on the response time is a JSON serializer library that we were using. It seemed to be especially slow for one of the microservice endpoints. Bob, as a hands-on guy, immediately identified another library which was claiming to be 10x faster and replaced the old one in one PR. Bob took a path of local optimization.
Joe was a bit of a different type of character. He wasn't up-to-date with the language, framework and libraries that the microservice was written in. He was more of a tech generalist, able to write comprehensive code in many languages. He tackled the problem from a different angle: microservice codebase was quite small, as it was providing just a couple of http endpoints. He checked other microservices code to understand which ones call those endpoints. He also grabbed a data collection of metrics and tried to see when the average response times increased and what changes were made to the whole system in that week. He noticed something interesting: another microservice (**Bingo**&trade;) which used to be singleton, was now running multiple replicas. That other microservice was using our **OmegaStar**&trade;'s endpoint to collect a big set of data. He dug into the git history to understand the motivation of running more than one **Bingo**&trade; replica. It turned out that **Bingo**&trade; was asking **OmegaStar**&trade; about the list of objects representing the current state of our internal cloud, then it was dividing the response data into three buckets, based on a single property of each object. This was happening when **Bingo**&trade; was singleton, but later on, the decision was made to run multiple replicas of **Bingo**&trade;, so each replica can query **OmegaStar**&trade;, but act only on the subset of objects (so each replica was interested only in objects with the different value of property X).However, it tripled the number of requests to **OmegaStar**&trade;. Joe understood what was the mistake: each **Bingo**&trade; replica was asking **OmegaStar**&trade; about the whole dataset. It didn't make sense, because **Bingo**&trade; was only interested in objects with a given property. Joe concluded that **OmegaStar**&trade; endpoint needs a query parameter, so **Bingo**&trade; can ask about less data and **OmegaStar**&trade; has less data to serialize.It was a relatively easy fix once he knew, so he opened two PRs: one in Bing, another in **OmegaStar**&trade;. Joe, having the high level overview of the situation, realized that we need to fix the code, so we don't pessimize.

## The result

We combined Joe's and Bob's changes, and deployed them. It decreased average user response time to the levels that were much lower than the ones before the problem occurred. 
It was noticeable to the end-user, we even got several reports that "our SaaS got recently faster". For myself it was great to observe and learn.

## Which type are you?

Joe's change had a way bigger impact than Bob's. Joe took a higher-level, pragmatic approach to address the problem. Today, after a few years of experience, I'm trying to tackle problems in a similar way. However, Bob's approach has some takeaways as well; it's always good to know tools. Data-driven approach can reveal many insights about the way a system works; however, insight from an experienced engineer that knows the system well is hard to beat. I sometimes recall this story, because there are many lessons to learn from it, but my favorite part is challenging the status quo (why this part of the system is doing this particular thing) over trying to do this particular thing faster.
