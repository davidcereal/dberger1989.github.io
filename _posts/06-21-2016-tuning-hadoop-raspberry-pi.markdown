---
title: "Building and Tuning a Hadoop Cluster Using 4 Raspberry Pis"
subtitle:
layout: post
date: 2016-06-26 22:48
image: /assets/images/markdown.jpg
headerImage: false
tag:
- distributed computing
- hadoop
- big data
- raspberry pi
blog: true
author: davidberger
description:    
---
### Comum Elements
- [Pi Intro](#baking-some-hadoop-pi)
- [Hadoop’s Basic Configuration Principles](#hadoops-basic-configuration-principles)
- [Configuring an example data node](#configuring-an-example-data-node)
- [Running on Raspberry Pi](#running-on-raspberry-pi)
- [Block-size woes](#block-size-woes)
- [Optimizing on memory allocation](#optimizing-on-memory-allocation)
- [Takeaways](#takeaways)



## Baking some Hadoop Pi
I’ve been eager to try out distributed computing and storage, and figured there was no better place to start than the big yellow elephant in the room—Hadoop.

There was something that seemed really cool about having my own physical cluster of computers, so I bought four Raspberry Pi 3 computers, a mini storage tower for them, some 32gb micro SD cards, an ethernet switch box to connect all the computers, and a 5-port usb power-bank so I’d only need to use one power outlet. Putting it all together was super fun! Here’s how it turned out:

<img src ="/assets/images/post_images/picluster.jpeg" style="width:560px"/>

There are plenty of tutorials on how to set up a Raspberry Pi, and I don’t really have much to add, aside from the fact that a nice shortcut to getting one up and running is by connecting it directly to a Mac through the ethernet port and enabling ethernet internet sharing. By doing it this way, I didn't need to connect a monitor, mouse, or keyboard to the Pis since I was connecting to them from my Mac through SSH. Hadoop’s web user interface is very useful for debugging (although I think it’s better to learn without it first), so I hooked up screen sharing between my Mac and the Pis using tightvncserver, and the setup was complete.

<img src ="/assets/images/post_images/screensharing.png" style="width:560px"/>
<figcaption class="caption" style="margin-top:-20px">Connecting through SSH and viewing the screenshare<br><br></figcaption>

The operating system I installed was Raspbian Jesse, a variation of Debian Linux built specifically for the Raspberry Pi. Next I downloaded the latest (at time of writing) version of Hadoop, 2.7.2, and went through all the necessary steps to set up the file system and turn the 4 Pis into a cluster. [Hadoop: the Definitive Guide, 4th Edition](https://www.amazon.com/Hadoop-Definitive-Guide-Tom-White/dp/1491901632) was really helpful in its guidance.

The cluster is made up of 4 nodes: 1 NameNode, and 3 Data Nodes. The NameNode keeps a directory of the filesystem tree and tracks where in the cluster all the data is kept, as well as its metadata. No data is actually stored locally on the NameNode. Instead, the data is distributed across the DataNodes, and usually replicated across multiple machines for resiliency. Like a standard filesystem, the Hadoop Distributed File System (HDFS) stores data in blocks. Hadoop’s default block size is 128MB, which will become an important piece of information later on. 


## Hadoop’s Basic Configuration Principles

The art of tuning Hadoop’s config parameters is vital to ensuring good performance. The difference between a poorly tuned and well tuned cluster can be enormous. Furthermore, because the Raspberry Pi's hardware specs is so limited, proper tuning was necessary to stop MapReduce jobs from crashing because of memory issues.

Hadoop’s computing performance is generally constrained by 4 components, listed in no particular order:

1. CPU power
2. Memory
3. Storage I/O speed
4. Network bandwidth 

Ignoring any of these factors will cause unnecessary overhead, severe bottlenecks, and generally underutilized hardware resources. As I’ll describe in the paragraphs below, all of them needed to be accounted for as I configured the cluster. 

Tuning the parameters is done by configuring Yarn, Hadoop’s resource negotiator. This is done by defining configuration properties in the `yarn-site.xml` file. And since we’ll be running a MapReduce application, the MapReduce config file, `mapreduce-site.xml needs to have the appropriate variables as well. 

On each data node, Yarn divides up units of work into containers. This is determined based on how many resources are available to Yarn in general, and how many resources are needed to run the application tasks (in our case, MapReduce). If we set up Yarn to have 4 GB of memory available to it, and each map or reduce task requires 512MB of memory, then Yarn will at maximum allocate a 8 containers for that application. Not only memory, but cpu cores can also determine how many containers are deployed. If, we set up Yarn so that it has 4 cores available to it, and each mapper and reducer is allocated 1 core, then at most, we can have 4 containers running at the same time. 

So the question arises, how many containers do we want? The number of containers is usually set to the number of cores, giving each container its own core. The intuition here is that we want to maximize the number of containers so as to maximize on computational power. 

While using all of our cores is a good goal to have, we can’t blindly ignore disc read/write limitations. During the mapping phase of MapReduce, each record is read and given to the mapper class. Lets say we had a datanode with 4 cores and one disc. We could give one core per container, and we would thus have 4 containers. 4 containers for one disc is not too big a stretch, and read/write speeds might be able to keep up. But if we had 48 cores and used 48 containers and still had only 1 disc, the number of mappers being read would not be enough to keep up with how many are being processed. In this specific case, I/O speed would a significant bottleneck.

## Configuring an example data node

Now that we have a decent idea how to determine how many containers we’d like to use, we need to configure memory allocation. Lets use this data node as an example, and point to the corresponding configuration settings along the way. 

<img src ="/assets/images/post_images/datanodeexample1.svg" style="width:560px"/>


Since we have 12 cores and 4 discs, we’re willing to give each core its own container. This shouldn't be too taxing on the discs. We give 2 cores to the system, which leaves 10 left, and that means 10 containers. We would thus configure `yarn.nodemanager.resource.cpu-vcores` to be equal to 10 in the yarn-site.xml file:

```html
<property>
 <name>yarn.nodemanager.resource.cpu-vcores</name>
 <value>10</value>
</property>
<property>
 <name>yarn.scheduler.minimum-allocation-vcores</name>
 <value>1</value>
</property>
<property>
 <name>yarn.scheduler.maximum-allocation-vcores</name>
 <value>10</value>
```

But how much memory should each Yarn container get? 

First we need to determine how much memory Yarn will have to work with in general. This is easily determined by calculating how much memory the datanode has in total and subtracting how much memory the system and Hadoop use when not running Yarn. In our case it’s `8192MB - 512MB = 7680MB`

The minimum amount of memory we want to allocate is determined by dividing the number of potential containers, 11, by the memory available to Yarn: `7860MB / 10 = 768MB`

We set the core allocation properties accordingly in `yarn-site.xml` accordingly:

```html
<property>
 <name>yarn.nodemanager.resource.memory-mb</name>
 <value>7680</value>
</property>
<property>
 <name>yarn.scheduler.minimum-allocation-mb</name>
 <value>768</value>
</property>
<property>
 <name>yarn.scheduler.maximum-allocation-mb</name>
 <value>7680</value>
</property>
```


The next step is to decide how much memory to make available for the different tasks that will be done by the application. In MapReduce, the basic tasks are mapping and reducing. We set the Map memory to the same as the Yarn minimum allocation memory, `768 MB` . We can’t allocate less memory than a Yarn’s container minimum, and we don’t want to allocate more, because then we wouldn’t be activating as many containers as we possibly could, and thus wouldn’t be capitalizing on our computing power. It’s important to remember this last point. We calculated the number of containers we wanted because we wanted to have as many possible under the constraints of cores and I/O speed. Once an appropriate number is determined, we want to be using all of them. 


Because reduce takes in the results of map and holds them in memory, it’s generally advised that twice as much memory should be allocated for the reduce phase as the map phase, although as we’ll see later, this is not always the case. So the reducer memory allocation in our current example would be `2 * 768 MB = 1536 MB`

This is what our mapred-site.xml file would look like:

```html 
<property>
<name>mapreduce.map.memory.mb</name>
   <value>768</value>
</property>
<property>
   <name>mapreduce.map.java.opts</name>
   <value>-Xmx204m</value>
</property>
<property>
   <name>mapreduce.map.cpu.vcores</name>
   <value>614</value>
</property>
<property>
   <name>mapreduce.reduce.memory.mb</name>
   <value>1536</value>
</property>
<property>
   <name>mapreduce.reduce.java.opts</name>
   <value>-Xmx1228m</value>
</property>
<property>
   <name>mapreduce.reduce.cpu.vcores</name>
   <value>1</value>
</property>
```
The properties with .java.opts extensions are heap-size limits for the corresponding tasks. It’s usually advised to make this .8 of the total memory of the container, since we also want to leave 20 percent for the executing java code. 


The last memory configuration allocation we need to do is for the Application Master (AM). The AM negotiates with Yarn for resources and containers, and is specific to each application, since each application needs containers for different purposes. For example, the MapReduce application requires Map containers and Reduce containers. Only 1 AM is necessary for each application running, so 1 MapReduce job will only need 1 application. This value is usually advised to be twice the minimum memory allocation, so in our case it would be `2 * 768 MB = 1536 MB`


```html 
<property>
    <name>yarn.app.mapreduce.am.resource.mb</name>
    <value>1536</value>
</property>
<property>
    <name>yarn.app.mapreduce.am.command-opts</name>
    <value>-Xmx1228</value>
</property>
<property>
    <name>yarn.app.mapreduce.am.resource.cpu-vcores</name>
    <value>1</value>
</property>
```

In our example above, it’s important to note that while the diagram depicts 4 mappers and 2 reducers, there could have been any permutation within the limit of 7680MB and 10 cores. The mappers will always be in increments of 768MB and the reducers in increments of 1536MB. There could have been 10 mappers and 0 reducers, as would happen in the beginning of a MapReduce application, when only the mappers are at work (and if the App Master is not on the datanode). There could have also been 0 mappers and 4 reducers, as might be the case at the end of the application. In much of the lifecycle of a MapReduce job, a datanode will have both mappers and reducers, and in such a case we might get 8 mappers and 1 reducer, or perhaps the configuration we saw above, where we have 4 mappers and 2 reducers, as well as the AM.

In our example above, the data node had 4 mappers, 2 reducers, and the AM container. The memory and cores would be allocated as:

<img src ="/assets/images/post_images/datanodecontainers.svg" style="width:560px"/>

But the node might also have looked like this: 

<img src ="/assets/images/post_images/datanodecontainers2.svg" style="width:560px"/>

Note that in the second example, there are 10 containers, and since we have 10 available after accounting for the background system, each receives one core. In the example above it, memory allocation only allows for 7 containers, and therefore 3 of the containers are able to recieve an extra core.



## Running on Raspberry Pi

Now that we’ve gone through the theory behind tuning a cluster, lets actually do it. 

Each Raspberry Pi has 1GB RAM, 1 micro SD for storage, and 4 processing cores.

I figured I’d leave1 core to the system and reserve the other 3 for Yarn. I decided I wanted 3 containers, one per core, hoping 3 containers wouldn’t strain the drive too much. With HDFS running but not Yarn, I had a little over 771MB of memory free. So I decided to leave 256 for the system and allocate 768 to Yarn. 

Since I had the potential for 3 containers baed on how I allocated the cores, I set the Yarn minimum memory allocation to 256. This way, if the maximum number of containers were deployed, they would utilize all 768MB given to Yarn. 

I gave map tasks the Yarn minimum I set of 256MB, while giving 512MB to reduce, since, as explained above, it’s common practice to give twice as much memory to reducers as mappers.

I also allocated 1 core and 128MB to the Application Master (as noted previously, only one node in the cluster will have an AM). 

<img src ="/assets/images/post_images/raspberrypidatanode1-wrong.svg" style="width:560px"/>

The diagram above breaks down the Pi's config settings. I left the App Master off the illustration because with this configuration, there would not be enough memory for a map container, a reduce container, and the App Master container. 

## Block-size woes

We’ll soon see that these initial settings were far from optimal. I ran 3 MapReduce applications on a 300MB text file I created by concatenating various books from projectgutenberg.org. 

The application maxed out the memory right away and crashed. It also only ran on one node. After doing some debugging I saw that it was because my task sizes were too large, and this was a product of my HDFS block sizes being too large.

HDFS’s block size has a default value of 128MB. Since MapReduce splits the tasks up 1 per block, a single Map container was being fed a 128MB block, and the container’s memory of 256MB couldn’t handle it. So I made the HDFS block size smaller. You can do this by configuring the file system itself to split files up into smaller blocks in the hdfs-site.xml config file, or by setting the block size to a particular file when you upload it:

`hadoop fs -Ddfs.block.size= 5242880 -put ~/data/test-300mb.txt /test-300mb-15mbBS.txt`


In the case of the Raspberry Pi’s I started from 50mb and worked downward. I also increased the size of the file I was using to 300MB so I could have more room to toy with the block size settings. Things started to get manageable for the container RAM around 20MB. At 20 MB block sizes, I was only getting a max of 2 failed map tasks out of the 15 that were launched, although sometimes none would fail at all. I didn’t want to increase the memory allocated to each node’s containers, because I already had it set to 256MB, and since I had 768MB in total to allocate to Yarn applications, increasing the memory allocation would have meant I would have less than 3 containers, running, and would thus not have been utilizing all 3 of the cores available. 

15 MB was the point at which containers almost virtually never ran out of memory, but in setting the block splits to such a granular level, I was also causing the number of tasks to increase, and putting a strain on the 4th of the key configuration components, network bandwidth. Smaller blocks mean a greater number of tasks, and while the tasks will be smaller, extra tasks mean extra bandwidth accumulated. The right block-size is this thus an optimization where you maximize balance across nodes while minimizing bandwidth overhead, and also ensure that they're not too big for you container’s memory to handle. 
 
To illustrate how the bandwidth bottlenecks performance when the block size is very small, I ran MapReduce jobs on 3 different block-size versions of the same 300mb text file. For each version of the file, I ran the test 5 times and averaged the results:


| Block Size        | Completion Time (m:s)  | 
| ------------- |:-------------:|
| 5MB      | 7:05 |
| 10MB     | 7:37     |
| 15MB | 8:13 |


As we can see, the more granular the block splits, the more tasks need to be run, and for each task, there is an extra delay because of the network’s bandwidth.

## Optimizing on memory allocation

I also noticed a mismatch in the map memory allocation and reduce memory allocation configurations. I had originally set my Map memory allocation to 256 MB and Reduce to twice that number, 512 MB, as is almost universally recommended. However, I noticed that once my reducers started running, my mapping progress would immediately slow down. But the reducers themselves are very quick, and when mapping is done, the reducers finish up very quickly. This told me that the reducers may have been allocated too much memory. Of course, it was possible that if I lowered reducer memory allocation, their containers would run out, but I was willing to tinker with it and see what happened. 

Low and behold, I saw roughly 1 minute 30 second speed increase when I lowered the reduce allocation to 256MB, and then another roughly 1.5 minute speed increase when I lowered it again, this time to 128MB. This was because at the 512MB mark, when a reducer was running, there was only room for 1 more 256MB sized container since Yarn and MapReduce only had 768MB to work with. Thus, if I had a reducer container running, I could only be running 1 mapper on the same node, and I’d only be utilizing a maximum 2 of the 3 potential containers available on my Node, which also means only 2 of the 3 cores available to MapReduce. When I lowered Reducer memory to 128MB, this made it possible to have 2 mappers and 1 reducer, or 2 reducers and one mapper, both of which combinations would have utilized all 3 containers. I ran each setting 3 times, and these were the average results:

| Reducer Memory Allocation        | Completion Time (m:s)  | 
| ------------- |:-------------:|
| 512MB      | 8:11 |
| 256MB     | 6:36     |
| 128MB | 5:02 |

Now that we know all about containers and cores, it makes perfect sense that when I lowered the reducer container to 128, I saw a drastic performance increase (60 percent!). The node went from being able to run only 1 map container while a reducer was running to 2, and when the mapping was finished and only reducers were running, the node was able to deploy 3 reducers rather than just one (at 128MB each, there would be enough memory for more, but remember Yarn only has 3 cores to work with). So this is what my optimized datanode setup looks like: 

<img src ="/assets/images/post_images/picontainers.svg" style="width:560px"/>


To be sure, the situation described above was idiosyncratic to my file only being 300mb large. In a production environment where there are many more nodes and much larger files, the reduce tasks can be significantly more memory intensive than the mapping tasks, as they have to hold the aggregated mapping output in memory as they reduce. 

## Takeaways

So in the end, I couldn’t simply follow the configuration playbook. I had to adjust the HDFS block size of my input, and I also had to tinker with the reducer container memory allocation. Understanding the nuts and bolts of how to tune a cluster and the reasons for those configurations is essential to making sure your Hadoop applications run to their maximal performance. 

Setting up this Hadoop cluster has been an extremely rewarding entry into distributed computing. Look out for an upcoming blog post about writing custom MapReduce applications! 
