---
layout: post
title: "Making unittests better with Docker"
description: "How to use docker for unit testing"
category: 
tags: ["docker","python","testing"]
---

#My unittests are already awesome. 
I know, I like mine as well. In this post I’ll share how we make unittesting a little more awesome by launching a fresh environment fast with [Docker](https://www.docker.com/).   

#Huh?   

Maintaining a dev environment that is consistent with your production environment is hard, Docker makes that a little easier. In a nutshell, Docker lets you start very light weight virtual machines called containers. You can set them up to hold pretty much anything, like your custom configured postgres server or a rabbitmq host.    

I fell in love with Docker because it allowed us to develop an app and it’s environment locally and move the whole thing to production with a guarantee that the environments match.     

We recently found a tangent use case. While revamping a large  ETL project we found the need to have a few daemons up, the usual suspects like postgres, memcached and rabbitmq. Having these installed locally started to slow us down, since these resources were shared by all the locally developed projects that consumed them. We would have to battle permissions on postgres and make sure all of our rabbit queues were flushed or else face mysterious artifacts from unrelated applications. In addition, having all of these daemons up slowed boot up times and ate away at resources. No one wants to wait for mongodb to start when we’re not going to use it.     

We started using Docker to load these daemons ”on-demand” and easily shut them down when we were done. We defined environments in a simple yaml syntax and loaded them in one line. This way we’d have all of the resources we needed up, fresh and not clashing with previous development we did. 
We saw that this was good and wanted to automate the process so that our tests would also have a fresh environment. Here’s a quick walkthrough of how we do that.     

#Enough history, what’d you do?     

The beauty of unittests are in their automation. The lazy developer need only launch the test and review the results, letting the test create all of the resources it needs. Obviously, manually setting up and reinitializing your env on every test is a no no, and even calling [docker-compose](https://docs.docker.com/compose/) misses the point of a fully automated test. So let's leverage the docker python client to build and tear down our environment on each test run.    

First, we’ll need to install the [docker-py](https://github.com/docker/docker-py) client with pip

First, we'll need the docker-py client so just
{% highlight bash %}
pip install docker-py
{% endhighlight %}

Next, we want to define the containers we are working with. We can make a dictionary for each container with its details (or read the docker-compose.yml)). For this example, I want postgres, rabbitmq (with management plugins) and redis. I’ll make sure ports are exposed where I expect them with the ports list.
{% highlight python %}
postgres =dict(image="postgres", #Use the latest postgres image from dockerhub
               ports=["5432:5432",], #Map the ports
               environment={ #Set environement variables
                   'POSTGRES_USER':'test',
                    'POSTGRES_PASSWORD':'test'
               }
               )
rabbit = dict(image="rabbitmq:3-management",
              ports=["15672:15672","5672:5672"]
              )
redis = dict(image="redis",
             ports=["6379:6379"])

containers = [rabbit,redis,postgres]
{% endhighlight %}



Setting up the tests
--------------------

Let's get our tests up and running. We need to pull any images not on our host then create and run a container for each. 
Since we don't want to do this for each test, we'll override the setUpClass and tearDownClass classmethods. This way, our testcase will set up all of the containers and they will be available for all of the individual tests we're running.  
Lets define those containers at a class level, pull them and make a container for each one. 
{% highlight python %}
import docker
class DockerUnittestExample(unittest.TestCase):
    @classmethod
    def initContainers(cls):
        postgres =dict(image="postgres", #Use the latest postgres image from dockerhub
               ports=["5432:5432",], #Map the ports
               environment={ #Set environement variables
                   'POSTGRES_USER':'test',
                    'POSTGRES_PASSWORD':'test'
               }
               )
        rabbit = dict(image="rabbitmq:3-management",
                      ports=["15672:15672","5672:5672"]
                      )
        redis = dict(image="redis",
                     ports=["6379:6379"])

        containers = [rabbit,redis,postgres]

        for container in containers:
            cls.dockerClient.pull(container['image'])
            cls.containers.append(cls.dockerClient.create_container(**container))
{% endhighlight %}

We need to start each container, by passing its ID to the client.  When we're done we'll need to stop each container. 
{% highlight python %}
        
        @classmethod
    def runContainers(cls):
        for container in cls.containers:
            cls.dockerClient.start(container["Id"])
    @classmethod
    def stopContainers(cls):
        for container in cls.containers:
            cls.dockerClient.stop(container["Id"])
{% endhighlight %}
Finnaly, we define our setUpClass and tearDownClass classmethods, which are called by the unitest framework when we run our tests.
{% highlight python %}
    @classmethod
    def setUpClass(cls):
        cls.containers =[]
        cls.dockerClient =docker.Client()
        cls.dockerClient .login('dockerhubuser','dockerhubpw')
        cls.initContainers()
        cls.runContainers()
    @classmethod
    def tearDownClass(cls):
        cls.stopContainers()
{% endhighlight %}

Put it all together, add some tests and rock and roll:

{% highlight python %}
__author__ = 'tal'
import docker
import unittest

class DockerUnittestExample(unittest.TestCase):
    @classmethod
    def initContainers(cls):
        postgres =dict(image="postgres", #Use the latest postgres image from dockerhub
               ports=["5432:5432",], #Map the ports
               environment={ #Set environement variables
                   'POSTGRES_USER':'test',
                    'POSTGRES_PASSWORD':'test'
               }
               )
        rabbit = dict(image="rabbitmq:3-management",
                      ports=["15672:15672","5672:5672"]
                      )
        redis = dict(image="redis",
                     ports=["6379:6379"])

        containers = [rabbit,redis,postgres]

        for container in containers:
            #cls.dockerClient.pull(container['image'])
            cls.containers.append(cls.dockerClient.create_container(**container))
    @classmethod
    def runContainers(cls):
        for container in cls.containers:
            cls.dockerClient.start(container["Id"])
    @classmethod
    def stopContainers(cls):
        for container in cls.containers:
            cls.dockerClient.stop(container["Id"])
    @classmethod
    def setUpClass(cls):
        cls.containers =[]
        cls.dockerClient =docker.Client('unix://var/run/docker.sock')
        cls.initContainers()
        cls.runContainers()
    @classmethod
    def tearDownClass(cls):
        cls.stopContainers()

    def test_postgres(self):
        #Test something with postgres
        pass
    def test_redis(self):
        #Test something with redis
        pass
    def test_rabbit(self):
        #Test something with redis
        pass
    def test_all(self):
        #Test something with all of them together
        pass
if __name__ == '__main__':
    unittest.main()
{% endhighlight %}

A few details   
-------------
1. You'll want to make sure you're logged in to dockerhub to be able to pull images. I added the login call here, though we use environment variables instead of plain text user and password. This makes it more portable to a CI server.     
2. The pull step is slow. Thats tolerable on a CI server but very annoying locally. I'm still thinking of ways to get over that hurdle and would love some input.    

3. tearDownClass is not called when tests fail with an error. That means that your containers stay up if your test fails with error and you won't be able to run a new one until you tear it down. This is not necessarily a bad thing, as sometimes you'll want to jump into that container and see what was happening inside to understand the error.     

4. Often you'll want to use your own image instead of the ones dockerhub provides. For example, the default postgres image does not allow connections from outside the host machine to the database. Build an image with the configurations you need and use that. 








