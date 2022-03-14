---
layout: post
title: Kubernetes Is Not Microservices, and Microservices Are Not A Magical Cure
tags:
  - kubernetes
  - aws
---

## The Story
As I've progressed in my career and met more colleagues, I've started to meet some people who are hardcore supporters of Kubernetes, containers, and microservices. Now, there's nothing wrong with this. These inventions have made life a lot easier on both the development and operations front, but it has also brought a lot of hype and misconceptions along with it.

## Pain
Here's how some of my discussions with my colleagues went:

### Discussion 1
Me: "We've been facing some performance issues, especially on the database side of things."

Colleague: "I think we should move to Kubernetes, it's great. It allows us to scale up and scale down easily".

### Discussion 2
Me: "It's the end of the year. Let's plan for some objectives to aim and work towards next year."

Colleague: "As we are experiencing customer growth, I think we should separate our web application and mobile application backends into microservices, so that we can scale."

![Pain](/images/microservices-is-not-a-magical-solution/1.jpg "Hide the pain Harold")

## The Misconceptions
### You need Kubernetes to scale
No, you do not.

Yes, Kubernetes helps you to scale very easily with its in-built horizontal pod autoscaler, but you have to realize that that's just the pods. Throw in the cluster autoscaler to scale your virtual machines (VM), and now you have a setup consisting of VMs and pods.

In the more traditional setup without Kubernetes, if you're using a cloud service like AWS, you scale your infrastructure by using auto scaling groups. When new VMs are added, more instances of your application are running. Depending on the way you've set up your application, it's possible to have multiple instances of an application running and serving requests within a single VM. These application instances can also be configured to scale, starting up and shutting down applications within the VM itself.

So, what do you have in the end? Sounds a lot like the "VMs and pods" in the Kubernetes solution, doesn't it?

Yes, there are differences in the speed of scaling. Yes, there are differences in the ease of setup. But ultimately, both methods of deployment allow you to scale your application.

### You need to separate your application into microservices to scale
No, you do not.

I think many people have the idea that you need microservices to be able to handle a lot of traffic. That's the wrong understanding of what microservices are about.

If you've always understood the idea of microservices to be "microservices = many separated servers", then you've completely misunderstood the purpose of microservices.

Microservices are about the logical separation of your code architecture. It is not about the physical separation of your server architecture.

If your application has always been a single monolithic service, splitting it into two separate services isn't going to magically improve the number of requests it can handle.

Think about it. There isn't much difference between running your single monolith on two VMs, and running your two separate services on one VM each. In fact, it can be argued that the single monolith has better performance, since there's less overhead and less wasted performance budget due to reservations (if service A is underutilized and service B is over utilized, then service B cannot make use of the spare performance budget that service A has reserved).

I think the worst part about my colleague's suggestion is that even if the application is separated into two microservices, what about the data? The data is tightly coupled and shared, so both of these microservices would be talking to the same database anyway. To do this properly, we would have to move the data to a new database, which would be a highly risky operation. Unless all other options are exhausted, moving to microservices should never be the first suggestion that comes into your mind. Work on optimizing your SQL statements, your code's space and time complexity, and possibly looking into frontend optimization features like pagination and hiding your load times behind animations.

## Enlightenment
### What Microservices provide
So if microservices aren't for scaling your servers for performance, then what are they for?

Well, applications nowadays are far bigger than those made decades ago. Development teams have also grown. What microservices help with is the logical separation and compartmentalization of application features, code, and infrastructure.

In a way, you could say that microservices are used to scale development. By cutting up your application into a smaller piece which can stand on its own, you could assign a small team of developers to take charge of this microservice. Repeat this across hundreds of microservices and development teams, and you would have an easier time managing the code and infrastructure. The only coordination needed between microservices would be the APIs and how they would interact with each other.

Microservices also scale development by allowing for small updates to the whole application. Instead of updating a single monolith, only one microservice needs to be updated. This makes updates faster to be rolled out to servers, and it also reduces the blast radius of any outage caused by a bad release.

Finally, similar to the previous point, microservices reduce the blast radius of any outage caused by whatever reason. Instead of your whole application going down, only part of it goes down. Not ideal, but better than nothing.

### What Kubernetes provides
One very important thing to understand about Kubernetes is that it's simply a container orchestration tool. What does that mean? It means that it manages your containers. That's it.

If you tell your Kubernetes cluster to spin up a deployment with three pods, an ingress, and an autoscaler to keep CPU utilization at 50%, then it will do just that.

Nowhere is Kubernetes described as purely a microservices tool. Nowhere does it say that Kubernetes is required to set up microservices. It's possible to set up microservices without containers. You can do it using VMs. You can do it using individual physical machines.

I believe that the draw of Kubernetes would be these:
- Infrastructure as code
- Self-healing
- Widely used in the industry
- Comes with all the basics to set up a production grade service (autoscaler, ingress, ingress controller, secrets, pod priorities)

With that in mind, Kubernetes would actually be a fantastic tool to use to set up monolithic services.

## Conclusion
I know it's tempting to want to use the latest and shiniest thing in the tech scene. Tech moves quickly, and us software engineers tend to like to keep up with the latest and greatest.

You can do that on your hobby projects, sure. But when it comes to working on an existing production service, sit down, take a minute, and think. What can this new technology or tool really provide to my code stack or infrastructure stack? What will I lose by moving to this new technology? What will I gain? What needs to be done to facilitate the move? How much time would it take? How much effort is needed? What are the risks involved in executing such a move?

Think deeper.
