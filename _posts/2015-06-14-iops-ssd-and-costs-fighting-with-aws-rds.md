---
layout: post
title: "Adding millions of slugs to an RDS instance"
description: ""
category: AWS
tags: ["AWS","RDS","Django","Costs"]
---
tl;dr
-----
I needed to add millions of slug fields to a Django site with a small RDS database. I ran into problems with IOPS on SSD's and solved it by temporarily switching to magnetic storage. 


It all started when
-------------------
[WhatTheyPlayed](http://www.whattheyplayed.com) started off as a tool for me, where I could look up tracklists and listen to individual tracks from DJ sets I liked. The data follows a relational structure, 

*    A Track is by an Artist
*    A DJ is an artist
*    A DJSet is played by a DJ
*    Many tracks make up a DJSet

Each object has its own page, showing information about that object (Tracks by and Artist, Sets by a DJ, Tracks played for Tracklists and Tracklists that played the Track for a Track. ) and of course, each page had its own URL. I originally set the url's to be the objects name followed by a __*object_id* which led to monstrosities like ***Mano%20le%20tough%20live%20at%20waren%20beach%20club%2012__2782***

When we started thinking about the sites usability, my friend Boaz point out that this is hideous and is bad UX and should be fixed immediately. What should have been a simple feature in the beginning (The addition of a Slug field to any relevant model) became an annoying and non trivial chore.

Django-Autoslug to the rescue
-----------------------------

[Andy Mikhailenko](http://neithere.net/) Made the wonderful Django-Autoslug module that will automatically slugify a model instance. It comes with built in Slugging logic or you can role our own, though Andy's does a great job and I didn't need any. 

After installing simply importing the Autoslugfield

{% highlight python %}
from autoslug.fields import AutoSlugField
{% endhighlight %}

and adding it to my models 
{% highlight python %}
    slug = AutoSlugField(populate_from='name',null=True,db_index=True)
{% endhighlight %}
had my models readyy to add a slug to any * new * instance. 

Since all of the relevant models had a name field and some functionality related to them, I actually had a Base model for them, so I only had to add the slug once

{% highlight python %}
class NamedModelBase(models.Model):
	name = models.CharField(max_length=500,db_index=True)
	name_length = models.IntegerField(blank=True,null=True,default=None)
        slug = AutoSlugField(populate_from='name',null=True,db_index=True)
    class Meta:
        abstract =True
    def __str__(self):
        return str(self.name)
    def __unicode__(self):
        return str(self.name)
    

    def save(self, *args, **kwargs):
        if self.name_length is None:
            self.name_length = len(self.name)
        return super(NamedModelBase, self).save(*args, **kwargs)
{% endhighlight %}    

I set null=True to accommodate the 1m+ rows of existing data that needed the name set. Which leads us to where things got tricky, populating the slug for all of the existing rows.

Updating the existing data 
--------------------------
To update the existing data, all I needed to do was call  ** save()** on each object instance and Django Autoslug would populate the field. This had a few caveats

*  There were quite a few models I needed to do this to and I did not want to sit around typing for loops over querysets and waiting for them to complete.

*  Each such queryset was big and it would not fit in memory all at once, since iterating over a queryset requires retrieving it.

*  As with any "long running task", it could fail in the middle, leaving the data in some mixed state and mandating that I monitor it. No thanks. 

So instead, I broke down the task into smaller pieces and ran it across a few EC2 instances with Celery. 

{% highlight python %}
@cellery_tasks.task(bind=True)
def SluggifyModel(*args,**kwargs):

    min = kwargs.get('min')
    max = kwargs.get('max')
    mod = kwargs.get('mod')
    for obj in mod.objects.filter(pk__lte=max,pk__gt=min,slug=None):
        try:
            obj.save()
        except Exception as e:
            logger.exception(e)
{% endhighlight %}

This little celery task would get a Model class (mod) and a range of primary keys (All sequential id's) and retrieve all of the model instances in that range that did not have a slug set. It would then iterate over the model instances in the queryset and save each one, thus adding the slug field. 

I made another task to break the work down into such chunks

{% highlight python %}
@cellery_tasks.task(bind=True)
def ScheduleSlugification(*args,**kwargs):
    CHUNK_SIZE=1000
    for mod in NamedModelBase.__subclasses__():
        count = mod.objects.aggregate(Max('pk'))['pk__max']
        i = 0
        while i < count:
            min =i
            max =i+CHUNK_SIZE
            SluggifyModel.apply_async(kwargs={'min':min,'max':max,'mod':mod})
            i = max
{% endhighlight %}

The line 
{% highlight python %}
for mod in NamedModelBase.__subclasses__(): 
{% endhighlight %}

Shows a really great feature of python, the ability to iterate over all classes that inherit from some Base class. Since I needed to run this task over all Models that inherited from NamedModelBase, iterating over its children reduced the code I needed to write and made sure I didn't accidentally miss a class. 

If only it were that easy
-------------------------
At the time, WhaTheyPlayed was perfectly content running on Amazon's free tier which provided a t2.micro EC@ instances and a 15 GB RDS instance also backed by a t2.micro with SSD.

t2.micro's are "Burstable instances", meaning that the instance has  certain amount of credits it can use to consume CPU and earns more (slowly) as it sits idle. Whenever I needed to do heavier loads, like the one above, I would spin up a stronger instance (Get them cheap with [Spot Intances](http://aws.amazon.com/ec2/purchasing-options/spot-instances/). 

Well, armed with Django Autoslug, my Celery tasks and a dirt cheap C4.xlarge instance I thought I was good to go. But then, after a few minutes of running the task nothing happened anymore. The CPU on the instances was barely moving and the network IO between the DB and the instances went to nearly 0. Which was strange because the worker processes were running and their were plenty of tasks in the queue (which was not depleting). What could have happened?

Remembering that the RDS instance was also backed by a t2.micro I checked if it had depleted it's CPU credits via the RDS console. But it hadn't, the task at hand was simple IO and their was no reason the CPU credits would be depleted.

The curse of IOPS with SSD
--------------------------
When provisions an RDS or EC2 instance you can choose if your storage will be either Magnetic, SSD or provisioned IOPS.

** IOPS = IO Per Second - How many data operations you can perform, your instance is allocated a certain amount of IOPS and gathers credits when not using them under some voodoo accounting scheme Amazon has possibly documented somewhere.  **

* Magnetic disks are the cheapest, at half the price of an SSD instance. They're  slower and you pay an additional $0.05 per million IO operations. They have no limitations on IOPS

* Provisioned IOPS are the other end of the spectrum, you pay the most per GB of storage and you pay for each IOPS you provision, and thus are guaranteed a consistent IO rate.

* The middle ground, and default choice are general purpose SSDs. Here you pay just for the storage (usually twice what you pay for magnetic) and you receive 3 IOPS  per provisioned gigabyte. These are usually great as they are super fast and you don't pay per IO operations like you do on SSD.

So know that we know what all about AWS storage we can guess why all my tasks stopped working. My RDS instance was on the free tier, with 15 GB of SSD storage which gave me 45 IOPS. 
It turned out that the sudden burst of IO from retrieving model instances and writing to them burnt out my IO credits and so the instance slowed to a crawl as it waited to be fed a credit and instantly used it. This was very annoying.

The solution
------------
Having realized/guessed that my problem was IOPS depletion I took inventory of possible remedies

* The obvious first move was to switch to provisioned IOPS, finish the task at hand and revert back. But provisioned IOPS has a minimum of 100 GB storage, and once you increase the storage size of you instance you can't go back. I had no intention of paying for an extra 80 GB of storage I was not going to use any time soon.

* The next option was to increase the size of the SSD volume to get more IOPS. The math here is IOPS= 3*(Provisioned Storage) so this is a good solution. This has proven itself in other situations where the extra spend was negligible, but in this case I did not want to start paying for storage I was not going to use.

* The final course, and the one I chose was to revert to magnetic storage. Sure this would slow down the instance, and I'd have to pay for the IO (about the cost of a stick of gum) but magnetic instances don't dabble in IOPS accounting, they just work - however slowly.

Once I switched to magnetic storage, the tasks started running and all of my model instances had slug fields. A happy day for all. 

Caveats
-------

* Since WTP was pretty low traffic back then, I did this on the production DB. In a more user aware situation, you'd have to take into account both the user experience as well as avoid disabling your log in system if its DB related. I expect that a Multi AZ deployment would solve that but haven't thought it through. 



