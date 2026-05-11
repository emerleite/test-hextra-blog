---
title: "Elixir Video User Profile Service for the Olympics — Application teardown"
date: "2016-09-21T17:39:46Z"
weight: 5
draft: false
tags:
  - elixir
  - software-development
  - case-study
  - software-architecture
  - ruby-on-rails
description: "Elixir Video User Profile Service for the Olympics — Application teardown"
author: "Emerson Macedo"
canonicalUrl: "https://blog.emerleite.com/elixir-video-user-profile-service-for-the-olympics-application-teardown-56ac3e103d1a"
medium:
  originalUrl: "https://blog.emerleite.com/elixir-video-user-profile-service-for-the-olympics-application-teardown-56ac3e103d1a"
---

![](images/1_t0uN_qHprJBcTeapfUoIhg.png)

I recently published [an article about our experience scaling the Video User Profile Service we built for the Olympics](https://medium.com/software-sandwich/how-elixir-helped-us-to-scale-our-video-user-profile-service-for-the-olympics-dd7fbba1ad4e#.d1ap2vjcr). In that article, I focused on the overall experience and then a little bit about the Elixir solution. Now it's time to explain the details of our Elixir application.

### Software rewrite (from Ruby to Elixir)

It's common sense that a [software rewrite is often really a bad idea](http://www.joelonsoftware.com/articles/fog0000000069.html). Despite this, [as we've seen in the previous article](https://medium.com/software-sandwich/how-elixir-helped-us-to-scale-our-video-user-profile-service-for-the-olympics-dd7fbba1ad4e#.j3qibnezo), sometimes it is hard to scale past a given point. I believe that, for cases like this, software rewrites can be a natural move for a technology stack so that it can meet its new requirements. In our case, the application **was written 4 years ago, and it did the job very well until the traffic increased 6x**.

### The Ruby endpoint we choose to rewrite

The *Video User Profile Service* has an endpoint that saves many logged user actions, like "**Add to favorites"** and "**Watch Later".** It also tracks how much of a video the user watched **in** their "**Watch History"** list, which allows users to resume watching a video from the where they left it. This tracking of the *watched percentage* is somehow special, because our [video player](https://github.com/clappr/clappr/) sends this information to our endpoint **every 10 seconds**, via a HTTP POST call. This endpoint, implemented in Ruby, represents **80% of Video User Profile Service throughput**. Simplifying it a bit, we have a split of ***READ(20%) and WRITE(80%)*** operations, as we can see below:

![](images/1_6f2PamxeGz8k02wlZq9XIw.png)

*Simplified version — Ruby for read and Elixir for write*

### Explaining the Ruby (POST) version

The original system implementing this endpoint is a traditional [RoR application](http://rubyonrails.org/). After saving a video in someone's list, we need to **update the list counter** for that specific video, because we need to know how many times a video was added to favorites. Say we have **video 1234** and **ten users** adds it to their favorites, so the **overall COUNTER** for **video 1234** must be **10**.

![](images/1_LKMcsvdHoiuvgHXXX1c50w.png)

*Counter for Favorites*

As it happens with traditional Rails applications, the original system would always block its entire process for each request. Given that *Update Counter* is a slow operation, the usual solution for RoR applications is to use a [Background Job](https://en.wikipedia.org/wiki/Background_process), using something like [Resque](http://resque.github.io/) or [Sidekiq](http://sidekiq.org/). Both solutions use Redis as a message queue to allow workers to execute jobs. I consider this solution a workaround because the platform does not have a built-in solution. This is illustrated in the diagram below:

![](images/1__Lx2z8vwOpo6GbJJdLHf6A.png)

*Ruby application needs another "application" to deal with async jobs*

### Creating the Elixir version

*Elixir* has similar tools to *Ruby*. In the *Ruby ecosystem,* we use *Rake*, *Bundler* (with *Gemfiles*), etc. *Elixir* has [Mix](http://elixir-lang.org/getting-started/mix-otp/introduction-to-mix.html). *Mix* got all years experience from Rails and Ruby community and combines almost everything in a single, fast and well-designed tool. From starting a project with **mix new**, to release a version with **mix release**, you can install dependencies, run tests and your project configuration also stays on a "mix file", called **mix.ex**.

With the tools, the syntax is very similar and the move is natural. I think the hardest part is to change from [Object-Oriented](https://en.wikipedia.org/wiki/Object-oriented_programming) paradigm to [Functional](https://en.wikipedia.org/wiki/Functional_programming) paradigm, but if you're already familiar with [Functional Programming](https://en.wikipedia.org/wiki/Functional_programming), it's very easy.

With the Functional Programming paradigm, Elixir also has the [OTP](http://learnyousomeerlang.com/what-is-otp), inherited from the [Erlang Platform](https://www.erlang.org/). This environment deserves an exclusive article, but quickly, it is a conjunction of abstractions to allow developers create concurrent, parallel and distributed software, with application and supervision capabilities, dealing with all processes lifecycle, from spawn to exit, dealing with failure restart, allowing an easy implementation of the [Crash-only software philosophy](https://en.wikipedia.org/wiki/Crash-only_software).

#### The Architectural Changes

An important change migrating the Ruby POST endpoint to Elixir was no longer needing a [Resque Application](http://resque.github.io/) to deal with [Async Tasks](https://en.wikipedia.org/wiki/Asynchronous_method_invocation). With this change, **we removed 20 docker containers**, responsible for dealing with *Update Counter Job*. Elixir has the [Task Module](http://elixir-lang.org/docs/stable/elixir/Task.html), which provides us [Async Tasks](https://en.wikipedia.org/wiki/Asynchronous_method_invocation), with the option to be supervised, in the case of failure. Combined with [Poolboy](https://github.com/devinus/poolboy), we can throttle the throughput of *Update Counter Job* using a [Pool of Objects](https://en.wikipedia.org/wiki/Object_pool_pattern). The main difference from the Ruby solution is that everything is part of Elixir and OTP architecture, which is very tested and mature. That's the final result:

![](images/1_0ap0_B--vlxsGAytSuHcrg.png)

*With Elixir, we do not need Resque anymore*

#### Frameworks and Libraries

To develop the POST Endpoint, we choose to use [Phoenix Framework](http://www.phoenixframework.org/), which is similar to [Rails API](http://guides.rubyonrails.org/api_app.html) and [Rails](http://guides.rubyonrails.org/) itself.

To deal with [cache](https://en.wikipedia.org/wiki/Cache_(computing)), the best option we found was [CacheX](https://zackehh.github.io/cachex/), that uses the [ETS](http://elixir-lang.org/getting-started/mix-otp/ets.html) and can replicate across [Nodes](http://elixir-lang.org/docs/stable/elixir/Node.html). ETS uses the BEAM VM Memory, so it's the fastest way to have an object to be fetched when needed. If we compare to a cache layer using [Redis](http://redis.io/) or [Memcached](https://memcached.org/), it's almost the same as comparing L1 cache with RAM access (of course I'm overreacting). The use of [CacheX](https://zackehh.github.io/cachex/) is working perfectly since we put our application into production.

For tests, Elixir provides [ExUnit](http://elixir-lang.org/docs/stable/ex_unit/ExUnit.html), a great tool, fully [integrated with Mix](http://elixir-lang.org/docs/stable/mix/Mix.Tasks.Test.html), so you do not need to install another one. It's something like [RSpec](http://rspec.info/) and Test::Unit. IMO, it's a great tool. With [ExUnit](http://elixir-lang.org/docs/stable/ex_unit/ExUnit.html), we're using [FakeServer](https://github.com/bernardolins/fake_server) (created by my friend and coworker [Bernardo Lins](https://github.com/bernardolins)) for HTTP Request Stubs.

To access MongoDB, we had some issues. There [is a mongo driver](https://github.com/ericmj/mongodb), but it's not evolving and lacks support for [Replica Sets](https://docs.mongodb.com/manual/replication/). I started a Fork, called [MongoX](https://github.com/emerleite/mongox). It integrates with [Ecto 1.1.9](https://hexdocs.pm/ecto/1.1.9/Ecto.html) using [mongox_ecto](https://github.com/emerleite/mongox_ecto). It needs improvements, but has full [Replica Sets](https://docs.mongodb.com/manual/replication/) support and we're using it in production.

We're using other libraries as listed below:

- [Corsica](https://github.com/whatyouhide/corsica) for [CORS (Cross-origin resource sharing)](https://en.wikipedia.org/wiki/Cross-origin_resource_sharing)
- [Exometer](https://github.com/Feuerlabs/exometer_core) to send metrics to [Logstash](https://www.elastic.co/products/logstash) and [Grafana](http://grafana.org/) (Miss you [New Relic](https://newrelic.com/))
- [HTTPoison](https://github.com/edgurgel/httpoison) for HTTP Requests

### The current version (Ruby and Elixir)

The *Video User Profile Service* now have a mixed solution, using Ruby for queries and Elixir for commands. It's something like [CQRS (Command Query Responsibility Segregation)](http://martinfowler.com/bliki/CQRS.html), but not so canonical, because although we have the Ruby application reading and the Elixir writing, we didn't isolate the Command Model and the Query Model. The structure is identical, but we're moving to isolate them. It needs some improvements, and we're doing it right now. The next picture shows the current state:

![](images/1_wE-fisbpGvDzJki4qTniqQ.png)

*Current CQRS Ruby and Elixir Video User Profile Service*

### Conclusion

After I wrote the [first article](https://medium.com/software-sandwich/how-elixir-helped-us-to-scale-our-video-user-profile-service-for-the-olympics-dd7fbba1ad4e#.37tn6rtvb), I received lots of questions about the implementation details. I hope this article can be a good answer to everyone interested in this case. Of course, there will be always uncovered questions, so feel free to ask for anything.

### Update

In 2017, we [rewrote the core engine using GenStage](https://blog.emerleite.com/using-elixir-genstage-to-track-video-watch-progress-9b114786c604).
