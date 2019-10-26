---
title: "GoLab 2019"
date: 2019-10-23T22:23:55+02:00
---

### Intro

In this post I'll try to write about my experience at GoLab 2019.

As per the 2018 edition, I had the honor to give a talk during the Italian Go conference [GoLab](https://golab.io/).
There are many reasons why I loved the conference:

- The organization was flawless. The people from [Develer](https://www.develer.com/) have been organizing conferences for a long time and it's pretty clear they know how to do them.

- It was my first conference with a Red Hat (well, not literally, but as a Red Hatter)

- The size of the conference was quite big. More than 500 attendees, three tracks for two days, several workshops. A lot of them were from abroad. I bet it's a good excuse to visit a nice city like Florence and to taste some good food (the food was the best one I ever had at a conference).

![](/images/golab/florence.jpg)

# My talk

I spoke about some of the advanced but less known features of grpc in my Rpc with steroids with Go and Grpc ([slides here](https://speakerdeck.com/fedepaol/rpc-on-steroids-with-go-and-grpc)), in what a colleague of mine describe as "conference driven development", meaning that when I submitted the abstract of the talk I barely knew about the existence of those features. I think it's a good but daunting way to force yourself learn new stuff :-).

I won't provide more details about the talk here because I plan to write a series of posts about the topic.

My experience was pleasant, despite being a bit nervous during the preparation. Hope I provided some good time and some food for thought to the attendees.

# The talks

The level of the talks was actually really good. A lot of well know speakers combined together with a good selection of new comers (including yours truly) made the conference really interesting.

### Eleanor McHugh - Intro to functional programming in Go

Despite the fact that hearing a talk about pure functions and monads in the morning not the easiest thing to follow, Eleanor made a good intro on what functional programming is and how to approach it in Go.

She spoke about how to give functions a state (sort of), and the aha moment prize goes to discovering that in Go we can add methods to any type, including functions.

I feel like Go has good support in regard of the functional approach (hello closures) which is probably something to stop and think about when programming imperatively. I also had reminiscence of when I was tinkering with RxJava in the Android world (not that I miss java).

### Aaron Schlesinger - The Athens Project: A proxy server for Go modules

This is a talk I was especially fond of, not because I did not know the subject but because Athens was the first Go open source project I contributed to (I even gave a light talk about it at [Golab 2018](https://youtu.be/yH3XPAQJTA8?t=553)).

Aaron made an excellent job in explaining how the proxy works, why we need immutability and all the technical nuances around it (including a live demo). 

But the part I liked the most was the one about being in control of your dependencies:

- not relying on the vcs
- not relying on a registry maintained by some company (more or less notable)
- why we need an independent mean to distribute our dependencies

And also, the fact that by using a Go proxy and leveraging the Go modules protocol, we can enforce the decentralization of ownership of the dependencies (think of eMule but for Go dependencies :-) ).

![](/images/golab/athens.jpg)


### Fabio Falzoi - An insight into the Go garbage collector

Fabio did an awesome job in describing how the Go garbage collector works and how and why it has an impact to the performance of our applications. 

He described thoroughly how the mark & sweep algorithm work, why the mark phase is expensive and the iterations that were done to mitigate this impact.

If there is one takeway from this talk is, in order to avoid performance impacts try to keep the variables on the stack as much as you can (but do this only after *measuring* the impact on the performances given by the gc). Having the variables on the heap has a big impact, not because of the allocation costs but because of the costs induced by the gc.

### Roberto Clapis - It starts with a problem

Another really really inspiring talk on how to grow and nurture a personal project, mixing advices on how to maintain and keep an opensource project, together with super useful hints on how to write and grow your codebase from simple idea to full fledged project.

# The gophers dinner

A fantastic dinner in a fancy location in the heart of Florence. Despite being really tired, I also greatly enjoyed the walk from the venue to the restaurant.

![](/images/golab/dinner.jpg)


### Bill Kennedy - You want to build a service?

I would keep listening to Bill's talk for hours. Each time I attend one I feel like I am a better Go developer, and this one makes no exception. In this talk Bill lives code his way through setting up a service. A lot of useful informations on how to develop a service and still keep your health.

Honorable mention to the expvar package together with pprof. A neat way to have metrics right out of your application's box, I did not know about that.

### Hadas Yaakobovitch - Migrating a mission critical service in Go

A real life story of how AppsFlyer migrated one of their most critical services from Clojure to Go. It was really interesting to hear how the the transition was handled team wise.

### Alan Braithwite - Advanced Testing tecniques in Go

A full lenght talk about testing in Go and some tips on how to handle it properly. I liked how he promote the use of dependency injection and how to use reasonable abstractions to perform your tests. This includes the usage of the [httptest package](https://golang.org/pkg/net/http/httptest/).

### Gabriele Vaccari - Building an event driven notification system with Go & RabbitMQ

The "how we did it" type of talk is one of my favourite and here Gabriele told another real life story on how Nulab split their monolith in microservices using Go, grpc and rabbitmq.
The first part of the talk focused on the evolution of the architecture, what types of messages and interactions their system has. The second part was a nice (and soft) introduction to the Amqp protocol and how to avoid common pitfalls while using the amqp Go client.

### Antony Seure - Developing a Go client, do's and Don't

A talk about how hard is the life of the maintainer of a Go client library compared to the other languages, and some valuable tips & tricks on how to mitigate this problem.

### Francesco Romani - KubeVirt: bringing virtual machines in a Kubernetes world

This is another talk I am fond of, not only because Francesco is an old friend of mine, but also because KubeVirt is the project I am currently working on at Red Hat. In his talk, Francesco described the why and the how KubeVirt allows us to run virtual machines on kubernetes clusters, and in the second part described a couple of examples of how Go made this task possible.

![](/images/golab/fromani.jpg)


### Steve Francia - The Legacy of Go

I really liked Steve's history lesson on how the evolution of different "language branches" contributed to the origins of Go and why it is the way it is (and why we love it!).

# Wrap up

A really recommended conference. The talks were good, the organization was spotless and October is the perfect season for visiting Florence (given the presence of so many people from abroad, I think this is well known).

Despite being so near my hometown it felt like a real international environment, and the Go community confirms itself to be as pleasant as always. Can't wait to be here again next year!

