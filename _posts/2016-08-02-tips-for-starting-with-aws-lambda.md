---
layout: post
title: "Four ways AWS Lambda makes me happy"
description: ""
category: Serverless
tags: ["Serverless","AWS","node"]
---
# Intro

## What is lambda
Side projects are my way of learning new technologies. One that I've been anxious to try is AWS Lambda and I finaly got the chance. There is always room for improvement, but this post will focus on the things that make Lambda a great service in my opinion.

For the uninitiated, Lambda is a service that allows you to essentially upload a function and AWS will make sure the hardware is there to run it. You pay for the compute time in hundred millisecond increments instead of by the hour, and you can run as many copies of your lambda function as needed. 


You can think of Lambda as the natural extension to containers. Containers (like Docker) allow you to easily deploy multiple workloads to a fleet of servers, you no longer deploy to the server, you deploy to the fleet and if there is enough room in the fleet your container runs. Lambda takes this one step further by abstracting away the management of the underlying server fleet and containerization. You just upload code, AWS containerizes it and puts it on their fleet. 

## Why did I choose lambda?
A bit of background. My latest side project is [Smart Scribe](http://smart-scribe-fe-test.s3-website-us-east-1.amazonaws.com/), an automated transcription service. The service processes large media files in memory and I realized that building a service that would overcome the memory bottleneck on traditional architecture would be difficult and expensive. I knew that using a serverless architecture would help overcome that bottleneck. 
Also, I hate paying for things I don't use, so buying by 100 milisecond of compute was attractive. 

# How AWS Lambda makes me happy


## It's very cheap

I love to invest my time in side projects, I get to create and learn. Perhaps irrationaly, I don't like to put a lot of money into them from the get go. When I start building a project I want it up all the time so that I can show it around. On the other hand I know that 98% of the time, my resources will not be used. 

Serverless infrastrucutre saves me that 98% by allowing me to pay by the millisecond instead of by the hour. 98% is a lot of savings by any account


## I don't have to think about servers
As I mentioned, I like to invest my time in side projects but I don't like to invest it in maintainging or configuring infrastructure. A thousand little things can go wrong on your server and any one of those will bring your product to a halt. I'm more than happy to never think about another server again.

To illustrate this point, here are a few things that have slowed me down before that Lambda, and the architecture that it induces, have abstracted away: 

1. Having to reconfigure because I forgot to set the IP address of an instance to elastic and the address went away when I stopped it (to save money)
2. Worrying about disk space. Sometimes I write big temp files concurrently, sometimes I output large logs locally. Having AWS take care of the infrastructure lets me abuse the file system as I wish, and someone else takes care of it
3. Running out of memory. This is a fine point because a single lamabda function can only use 1.5 G of memory. Applications that need to hold large data sets in memory might not benefit from lambda, but applications that need to hold many small to medium sized data sets in memory or prime candidates.

Smart Scribe works with fairly large media files and we need to store them in memory with overhead. Even a few concurrent users can easily lead to problems with available memory - even with a swap file (and we hate configuring servers so we don't want one). Lambda guarantees  that every call to my endpoints will receive the requisite amount of memory. That's priceless. 

## Deployments are fast

I use [Apex](http://apex.run/) to deploy my functions, which happens in one line

{% highlight bash %}
apex deploy
{% endhighlight %}    

Apex is smart enough to only deploy the functions that have changed. And in that one line, my changes and only them reach every "server" I have. Compare that to the time it takes to do a blue green deployment or, heaven horbid, sshing into your server and pulling the latest changes.

But wait, there is more. Pardon last years buzzword, but AWS Lambda induces or at least encourages a microservice architecture. Since each function exists as its own unit, testing becomes much easier and more isolated which saves loads of time. 

## Tight integration with other AWS services
What makes Microservices hard is the overhead of orchestration and communications between all of the services in your system. What makes lambda so convenient is that it integrates with other AWS services, abstracting away that overhead. 

Having AWS invoke my functions based on an event in S3 or SNS means that I don't have to create some channel of communcation between these services, nor monitor that channel. 
I think that this fact is what makes Lambda so convenient, the overhead you pay for a scalable, maintainable and simple code base is virtually nullified. 

# The punch line

One of the deep axioms of the world is "Good, Fast, Cheap : Choose two". AWS Lambda takes a stab at challenging that axiom. 
