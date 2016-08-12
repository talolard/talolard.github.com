---
layout: post
title: "Proccessing terabytes with celery and friends"
description: ""
category: 
tags: ["Celery","Docker","rabbitMQ",]
published: false
---

The Problem
-----------
We've been processing many small text files for a while and had a good pipeline laid down. One day we partnered with a new data supplier who increased our workload by 100x. Our partner delivered about 100 million files (each about 500k) over a few hundred compressed and encrypted archives, stored on S3.   
We obviously wanted to process this data set in a distributed manner and started thinking what the best way to go was. Our first tests showed that decrypting and decompressing an archive took up a third of the time needed to processes it, in other words we had a big IO pain. We also realized that HDFS was not much of an option, as we needed to decrypt and decompress the archives before getting them into HDFS. In addition, HDFS has an affinity for large chunks, leading to [The Small File Problem](http://blog.cloudera.com/blog/2009/02/the-small-files-problem/), so even if we would get the data into HDFS, it was not in a format that would leverage it well. 

The Solution
------------
We ended up using celery and rabbit to distribute the work amongst as many servers as we had availble. We used ec2 spot instances to get the capacity we needed on the cheap.  Our workflow looked something like this:

* A master process would check for new archives to process on S3. 
* Each archive would be assigned to a celery task that would download, decrypt and decompress it on a machine. All of these tasks went to a dedicated queue on rabbit, and each machine had a celery worker process running on it that would retrieve a task and process the archive. 
* Once opened, each file in the archive would be assigned a task for processing. Since their were many machines running, each one having only some of the files, we gave each machine its own queue from which only it would read tasks. 
* Store the ouput. This ended up being a challange in and of itself and I'll cover it on a seperate post.

The rest of this post will cover a few tips and tricks we picked up along the way.


Name Queues after the Host
--------------------------

So as I mentioned, after opening the archive, each file was assigned a task to be processed. The asks argument were the path to the file, and not the file contents as that would quickly overwhelm our message broker. Since different files resided on different machines we had to ensure that each machine had it's own queue that from which only it would read and write tasks. 

We did this by dynamically creating the queues in our celeryconfig.py file. In it we placed some code that would call boto and get the instance public dns, then define a queue named after that dns. This gave us a unique queue for each machine and a garauntee that each machine was only aware of its own queue. 

You might think getting the dns from boto is overkill and that it would suffice to use the hostname. We were using docker to deploy the workers, and so had no garauntee that seperate docker dameons (one on each instance) would give distinct hostnames to each machine.


