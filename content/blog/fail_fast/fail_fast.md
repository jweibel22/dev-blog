---
title: Reduce time to recover and embrace failing
description: A collection of ideas on how to design your system in a way that reduces the operational burden
date: 2024-07-04
tags:
  - Developer productivity
---
When your greenfield project finally hits production reality will bite you. Bugs will surface, edge cases you never thought about will emerge, the system will get into a bad state and you'll need to diagnose and correct the issues when they arise. This work, often referred to as "run" work, will be important and urgent and it will take up a lot of your time.

Did you build your solution in a way that makes it easy to operate? If you didn't, this kind of work can take up much more of your time than it should, time that you could otherwise have spent on building new features and (hopefully) make an impact.

## Functional requirements

Are you keeping a critical eye on the functional requirements? Are they reasonable and suitable for a first release of the product? Or are they just making the solution more difficult without much or any benefit?

{% note '' %}

One of the initial requirements I was presented with during the early days of implementing a data platform was the need for having BI dashboards updated in real-time. It was important for the people to know the KPIs of the business and the fresher the data the better!

At the time implementing support for real-time data transformations was much more challenging than supporting batch. In batch mode you could simply run the SQL transformations directly on the data warehouse at regular intervals, in real-time mode you would need to leverage technologies such as flink or spark streaming. The transformations were built by the users of the platform and could be of varying quality resulting in a heavy ops burden when the those transformations went rogue.

After taking a critical look at the requirement for real-time updated dashboards it turned out that there were no real use cases requiring it. Data updated at hour intervals was good enough for all known use cases. Having dashboards updated in real-time was "cool" but at the end of the day there was no real need. Cutting away this requirement allowed for a much simpler initial solution and therefore also less potential run work.

{% endnote %}

Perhaps there are some small alterations you can make to the functional requirements that will have little impact on the product but a huge impact on the complexity of the solution? A good example of this is the one given by Martin Kleppmann in "Designing Data-Intensive Applications". In the example a booking system has relaxed the requirement of strong consistency and instead opted for eventual consistency. This means that two users could be booking the same tickets at the same time without knowing. The proposal is to allow for this to happen and then notify the users over email afterwards that their booking didn't go through after all. Depending on how likely such a scenario is to happen and how much negative impact it has on the users it can be a reasonable choice to make. Guaranteeing strong consistency in a distributed system is a hard problem after all.

{% note '' %}

Once I was working in a company where the primary product was a mobile app. In one scenario, a user would input an "order" in the app which would trigger an orchestration process on the backend. While the backend process was running the user was presented with a "spinner" in the app. If the system was under heavy load some of the services involved in the orchestration process could take a long time to respond. This would mean that the user would be stuck with the spinner in the app not knowing what was going on. The solution was to not block the user for the duration of the orchestration process, but instead return immediately when the backend had received the order request from the user and show an "in progress" item in the app that would change state asynchronously when the backend process had ended.

The initial design resulted in very bad user experience. It was also the cause of a lot of run due to users inputting duplicate orders, since users would eventually lose patience with the spinner, restart the app and redo their order not knowing the original order was still getting processed.

{% endnote %}

Another simple trick is to prefer asynchronous over synchronous communication. Is there a user waiting at the other end? If not, perhaps you should consider async request/response and event driven communication instead of synchronous communication. When a distributed system built on synchronous communication comes under heavy load it becomes very unpredictable. As resources are running out (cpu, memory, disk, caches and internal buffers spilling over etc.) the various services will start failing, perhaps only partially, become unresponsive or just slow, and this can have a cascading effect throughout the distributed system. This could result in a lot of run work. A really good source of examples on this is in the excellent book [Release it](https://www.amazon.com/Release-Production-Ready-Software-Pragmatic-Programmers/dp/0978739213) by Michael Nygaard.

## Non-functional requirements

One obvious way to reduce run work is to be tolerant about failures/degradation. When considering the non-functional requirements during the development phase be critical/realistic about need-to-have and nice-to-have. Make strict alerting on the need-to-haves but be more relaxed about nice-to-have. Perhaps you could make use of SLIs and SLOs to keep track of the service levels in general and react to those rather than individual incidents.

{% note '' %}

Some time ago I was working on an ingestion pipeline that had two separate parts. One part was responsible for ingesting the raw data into a datalake and the second part was responsible for applying a schema to the raw data and making it available for consumption in a data warehouse. If data was lost in the ingestion pipeline that ingested the raw data it was very hard or even impossible to replay the lost data from the source so it was absolutely critical to avoid data loss. In contrary, applying the schema to the raw data from the data lake was a repeatable process, if it failed it could be run again against the raw data from the datalake, and the consequence of any issues preventing the job from running would be delayed data delivery and not data loss. As a natural consequence of this difference we focused on making the ingestion of the raw data as robust as possible (implementing at-least-once delivery guarantee) and ensured that observability was in place even before going to production.

{% endnote %}

## Ops driven design

Let's say you're building a backend. How do you recover when some bug occurs and you accidentally called a service with some wrong data, how do you compensate for the mistake? Does it require you to make code changes or is it possible to do the compensating action by running a script? How do you clean up any bad state in the system databases or external systems caused by the incident? Again, how long does the recovery take? And how much time will you as an engineer spend on this? See <a href="/blog/sagas/">sagas</a> for an example on how we reduced run work by introducing a saga framework that allowed us to make generic tooling that could be used to control the sagas in production.

Let's say you're building a data pipeline. What happens if something unexpected happens and your pipeline stalls? After having fixed the issue can you resume from where it left off? Can you backfill data that was lost due to the incident? Can you backfill just the subset of the data you need or will you need to reprocess everything? How long will the recovery process take? If you have a plan, and the expected recovery time is reasonable, you can worry a little less about making mistakes and perhaps take mores risks. See <a href="/blog/data_platform/"> How to reach production in less than 6 months</a> on how I applied this principle on a data platform.

Design decisions you make early on can have a huge impact on how easy the solution will be to operate once it gets into production, therefore as you design your solution you should think about this from day 1. An analogy to this is code testability. If code is not written with testability in mind adding unit tests later will often require refactoring the code. This is one of the arguments made by TDD. By writing tests in parallel with the implementation this won't happen and your code will be testable (and better designed).

One way to achieve something akin to TDD could be to keep a staging environment running from the beginning of your development process and try to keep this environment in good state, i.e. if bad things happen you try to resolve any issues and recover the system in a similar way that you will do once you're in production. This way you'll find out if your design and the tooling you have in place is adequate to support the common run work that will occur. Obviously incidents that occur early on in the development process might not be representative to problems that you will see in the more mature system that reaches production so some level of pragmatism would be needed here. It would probably be a good idea to have a "wipe" tool that will restore the state of the system to a good state (perhaps just deleting all data) with a single touch of a magic wand.

## Alert strategy

Think carefully about your alert strategy. A very common default approach is to set up generic alerts that simply alerts on all kinds of errors, however this can lead to excessive alerting, forcing engineers to waste time manually segregating critical from non-critical alerts. Not only does this lead to alert fatigue it is also a huge waste of time. Investments into optimising this can potentially have a large impact as the overhead of filtering through the alert storm quickly compounds over time.

A good approach is to alert only on high severity or critical issues, i.e. issues that effect a large proportion of users or which could have fatal consequences if not dealt with immediately. These are the types of issues that demand the immediate attention of engineers and that you would want to be included in an on-call setup.  Everything else could be dealt with using a pull approach rather than a push approach. That way engineers can manage their own time better, do less context switching and be more efficient overall. You could set up a dashboard visualising the non-critical errors as they pile up in your backlog and regular check in on the dashboard to ensure issues are dealt with continuously. E.g. you could build a dashboard in grafana using panels that show up when issues arise:

{% image "./runboard.png", "", [900] %}

## Tooling

Do you have the ops tools in place that you need to solve the problems you'll be facing? Obviously you'll need observability, but once you realise that there is a problem how easy is it to diagnose what happened and resolve the issue? Run work will happen, and if you're spending an unreasonable large amount of time logging into various systems, accessing databases, copy pasting data between systems etc. the overhead will compound over time.

One thing you can do is make it easy to access the various sources of data you need to diagnose and resolve the issues. You could create a tool where you can access and correlate all the data and where all levers that can be pulled are also available. See <a href="/blog/reducing_run">Running fast</a> on ideas on how to make such tooling.
