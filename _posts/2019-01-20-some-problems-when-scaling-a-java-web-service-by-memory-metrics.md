---
title: "Some problems when scaling a java web service by memory metrics"
categories:
  - spring-boot
  - java
  - auto-scaling
---

Recently, I have a requirements about scaling a Java web service from my company. My first apporach to solve this requirements, is collect twom metrics: RAM usage (unit MB) and CPU utilization (unit %), from each web service instance, then set scale in/out threshold values for each metric. If average RAM usage **or** CPU utilization over threshold, a instance of web service will be spawed or deleted. With CPU metrics, i have no problem with it: When CPU utilization is high, it means that web service is in a high load. And when web services idle, CPU utilization is low. But with RAM usage metrics, I catch many problems when using it to scale web service.

Let start wih a real case. I set two threshold is 2048 MB and 4096 MB to scale in/out a Java application. It means that, if average RAM usage of all instances is below 2048 MB, the system will scale in web service and delete an instance. And if average RAM usage of all instances is large than 4096 MB, the system will scale out web service and spawn a new instance.

But ater some tests, I realized that RAM usage of a Java process is not a good metrics to decide scale in/out Java web service, because an important point: 

> Java application will not automatically release occupied RAM back to OS in all life time of it. 

This means that we will see average instance RAM usage is high even when our web service is idle and doesn't handle any request, and when RAM usage calculated is high, web service will be scaled out. This behavior will lead our system to a wrong state: web service will be scaled out, even when it is in a idle load. This isn't behavior we want.

Here is a my test to check RAM usage when web service is in heavy load state and when it is in idle state. You could see that, when system is idle (when thread number is low (the number of threads which application is used to handle requests)), OS RAM usage of process - the orange line, isn't decrease. 

![memory-usage-metric-scaling-problem.png]({{site.baseurl}}/assets/images/posts/memory-usage-metric-scaling-problem.png)


Because of this, I think that average RAM usage of instances of a Java web services is not a good metrics to system decide to scale in/out web services. 

In the next part of this blog, i will demonstrate that an other memory metrics of a Java application: jvm.memory.used - the blue line, also is not good metrics to scaling system.