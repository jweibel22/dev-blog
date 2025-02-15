---
title: "Building maintainable microservices - Part 2: Architecture level"
description: A look at responsibility boundaries
date: 2024-06-23
tags:
  - Domain Driven Design
  - Microservices
---

This is part 2 in a series of blog posts focused on maintainability of microservice based backends.
The series will discuss issues around maintainability at different levels. In the <a href="/blog/maintainability_part1/">first post</a> we were looking at the design level, in this post we're at the technical architecture level.

## Introduction

When a system based on a microservice architecture is implemented without a proper understanding of the problem space it will result in accidental complexity, unintentional coupling and a tightly coupled architecture in general. This has many negative consequences but, as mentioned, in this post we will focus on maintainability. When the misalignment is allowed to exist you'll often be able to notice the consequences by observing the responsibility boundaries that exist between the bounded contexts in your backend. Some bounded contexts will have multiple responsibilities, some bounded contexts will share responsibilities with each other, and some boundaries will simply be unclear.

{% note 'Bounded context' %}

Throughout this blog post I will be using the term `bounded context`. The term comes from DDD and defines the boundary inside of which the "ubiquitous" language is well defined, i.e. where terms from the language has a single definition. Outside of it a term from the language may have a different meaning. In a microservice architecture the ownership of the solution is usually separated into multiple independent teams, as having the ability to do so is usually one of the primary reasons for choosing a microservice architecture. As each team is independent it follows that teams will not be sharing bounded contexts as that would require the teams to agree on every term used in their codes bases. On the other hand, it is possible for a single team to have multiple bounded contexts, e.g. they could be owning two services that both refer to the term `Product` but with different meanings. In this text, when the term `bounded context` is used you can also think "team" as a bounded context originates from a single team.

{% endnote %}

In this post, we'll be focusing on challenges that arise when building systems using a microservice architecture. From the perspective of the single responsibility principle a microservice is just like any other component (e.g. classes or modules). However, microservice boundaries are a lot more rigid because they involve network and often team boundaries. This means that the single responsibility principle becomes even more important, because once you break it, it is much harder to refactor and correct the issue.

There are three common cases of bad responsibility boundaries that you'll encounter, let's go through them here.

### Violation of the single responsibility principle

**A single component has the responsibility of multiple independent business capabilities or processes from the problem space.**

Because the implementation of the business capabilities are entangled in the code it will be difficult to evolve them independently. This can happen on many levels, it could be an aggregate inside a service taking up too much responsibility, it could be a service or a bounded context.

### Shared responsibility

**The implementation of a single responsibility from the problem space is scattered between two or more independent components in the solution space.**

In a microservice architecture the components could be two bounded contexts owned by separate teams.

From the perspective of the owners this means that implementing a change request is hard because it requires the coordination of multiple teams. They must align roadmaps and all teams will depend on each other while implementing and testing the new changes.

From the perspective of dependent teams that must implement business rules or processes that are based on entities or events coming from the domain it will be difficult to implement and maintain their solutions. Since domain events are coming from multiple bounded contexts how do they know that they have consumed all relevant events, and how can they be sure which events are relevant, the publishers might use different names for the same thing, or the same name for different things. Likewise how could they ensure that their solution will continue to subscribe to all events in the future, if ownership is already scattered, what's to stop yet another team from publishing new events belonging to the domain in the future? Similar problems arise when relying on scattered APIs.

### Inconsistent responsibility boundaries

**There are clear responsibility boundaries but they are drawn inconsistently across another orthogonal dimension.**

A Business capability is owned by a single team A, except for the cases where the users logged in are coming from country C, in those cases the business capability is owned by team B.

This will obviously make it harder to evolve the business capability because both team A and team B will need to be involved, at least if feature parity is a concern. It will also likely cause a lot of duplicated effort.

### Chaos

**Responsibilities boundaries are not based on the problem space at all. They're based on competencies or are accidental**

In this scenario the problem space is not even taken into consideration. This is the most extreme situation and it is total chaos. In this situation all 3 scenarios mentioned above are present at the same time. It is a very common situation and the default state many organisations get into when they have no knowledge of DDD or team topologies.

Teams could e.g. be based on different competencies, there's a frontend team, a backend team, a database team etc and each team must know about all aspects of the problem space and how it intersects with the layer they're responsible for.

In other cases teams are formed without much consideration and responsibility boundaries are purely accidental. When your architect utters the sentence "for historical reasons" several times a day you're most likely facing this scenario.

## Anti-patterns

Let's examine a set of common anti-patterns related to maintainability that are frequently observed in systems where the understanding of the problem space has been lacking. The list is not exhaustive.

### Anti-pattern 1: Bad fit

As the problem space evolves it is natural to consider reusing or extending existing components to support the new business processes/capabilities. It might be possible to support the new requirements using an existing component by making minor extensions, e.g. by making functionality more generic and configurable or adding branching logic. If one does not consider whether the new business capability/process is part of the same domain as the existing functionality this can lead to [Violation of the single responsibility principle](#violation-of-the-single-responsibility-principle).

{% note 'Real world example' %}

A team was responsible for a core domain that was depended on by many other domains. Those other domains represented the products delivered to the users of the system. The company was using stream aligned teams (i.e. teams with vertical/fullstack ownership) and the team could in the terminology used in team topologies be characterised as a complicated subsystems team. However, the team was also itself using the core domain to implement a product and was thus also acting as a stream aligned team.
The team had built a BFF to enable the front-end of their product. The BFF exposed an API, used by the mobile app, that would authenticate the user and then delegate to the underlying backend service. It was also responsible for how the product was shown in the UI and for sending out notifications to users and this was implemented in an event driven way, by consuming event messages published by the backend service and then reactively refreshing the UI or pushing notifications.

The first few products built by other teams had no special requirements with regards to the UI and notifications, and it was therefore decided that they could be relying on this BFF to handle that part of the functionality for them. Later products had slightly different requirements. E.g. one product needed to show a different text in the notifications and that functionality was therefore implemented in the service belonging to that product and a corresponding `if` statement was added to the BFF to exclude the notifications for that specific service. The product also had one difference in the functionality in the UI, the item shown in the UI could not be deleted for this product which was possible in the common functionality, thus another `if` statement was added here. Another product did not exist in the app at all, it was provided to a different user base that did not use the app. Again, appropriate `if` statements were added in the BFF to exclude the handling of the UI and notifications for this product.

Do you see where this is going? The BFF service broke the single responsibility principle, it implemented common functionality for products that were not related. Every time new products were added, or existing products were altered, one needed to consider whether that product feature set would fit into the BFF's common functionality and if not make special cases to exclude certain parts. Likewise when making changes to the functionality within the BFF service a developer would need to consider the entire set of products that was using the BFF service to ensure that the changes made did not break any existing functionality in any of those products. This was a near impossible task as every product was different and the product catalog spanned many domains. To make matters worse, the list of products using the BFF was not readily visible in the code but could only be obtained by asking developers that had the historical context or looking at call graphs or logs. The initial intention of being more efficient by reusing existing functionality so that product teams did not have to write code that already existed eventually led to a brittle and hard to maintain system.

Investigating the domain should have made it apparent that current and future products were not identical. Also, since the products were user facing, as time goes by it is only natural that requirements for each product diverges because product owners will want to iterate on their products making them more sophisticated. If this had been realised from the beginning developers could have chosen to either not reuse any functionality at all or, if feasible, put reusable parts of the code into a library that the other product teams could use to implement their product features.

{% endnote %}

### Anti-pattern 2: Ghost domain

As the business evolves and is expanded with new capabilities/processes new domains can emerge. If the new domain is not identified in time, implementation of the related capabilities/process could get scattered all over the organisation and ending up in multiple bounded contexts leading to [Shared responsibility](#shared-responsibility).

{% note 'Real world example' %}

A company would charge fees for the different products/services they provided to their customers. There were many products/services and they were implemented by the various microservices in their microservice architecture and initially each microservice would directly charge a fee if they had carried out an action that required it. After some time it became apparent that there were many business rules and processes related to the management of fees. Some types of fees should only be charged if the user was in a certain membership tier, there were rules about discounts, e.g. if many fees had been charged in a short period. The company was also required by law to send out yearly fee statements to their customers so they could see which fees had been charged. Implementing these business processes quickly became a nightmare due to the scattered implementation of the fees. While some of the requirements related to the fee domain might not have been known initially, the requirement dictated by law should have been anticipated and would naturally have let to the discovery of the fee domain early in the development process.

The solution was to specify different types of fees and have each microservice notify the fee service about fees that needed to be charged but let the fee service take responsibility for handling the fees. This kept the decision on which specific action required fees to be charged within the service that implemented them, ensuring a low coupling of the fee service to the other services while keeping the responsibility of handling the fees and all related business processes within the fee service.

{% endnote %}

### Anti-pattern 3: Entanglement

In this scenario the representation of the problem space in the code is entangled with technical concepts from the solution space. This means that when the solution changes for purely technical reasons, e.g. because of a non functional requirement, code containing business logic will likely need to be refactored. This is wasteful and, as always, comes with the risk of introducing bugs. If the solution specific concepts has bled into the APIs or events this could also effect dependent services.

Business logic from the domain model leaks into technical components or technical components leak into the domain model. This violation of separation of concerns means that when you need to change your code for technical reasons (i.e. not to change business logic), e.g. fixing a bug, upgrading a library/framework to a newer version or simply refactoring the code you risk making unintended changes to the business logic. Likewise, understanding the business logic that comprises the domain model becomes harder because the code is entangled with code that is only there for technical reasons and does not exist in the problem space.

{% note 'Real world example' %}

A team was responsible for modelling a `SalesOrder` entity. As part of managing the SalesOrder process the team was also doing a lot of orchestration, calling other services and updating the state of the SalesOrder entity accordingly. In this particular case the orchestration process was purely there to implement a distributed transaction and did not represent a business workflow and as such only lived in the solution space. However, they had not separated the orchestration process from the modelling of the SalesOrder entity and this resulted in details related to the orchestration process to leak into the domain model. The domain model was responsible for storing information such as "async request to service X has been initiated, waiting for a response". Whenever a change to the orchestration process was needed, e.g. a service that was depended upon was replaced with another, this meant touching code that was located in the same place as the domain model, and therefore also risking making unintended changes. Sometimes the orchestration process (saga) would get into a bad state because something unexpected happened and it was needed to get it back on track. This was done by introducing new steps in the process that were only there because of the specific incident that caused the saga to end up in the bad state. Over time this polluted the saga code, and therefore also the domain model code with logic that made very little sense without the context of the historic incident, leading to a bloated domain model that was hard to understand and maintain. The sagas in this example were breaking the single responsibility principle. They were responsible for implementing the business process AND for implementing the orchestration.

{% endnote %}

As an example of entanglement consider a service that publishes an event with information needed for a salesforce integration, perhaps the event contains a salesforce related ID. The event is being used as a domain event also. When the partnership with salesforce ends it will not be possible to stop publishing the event because it is used elsewhere.

Another example of this could be a single concept (e.g. a product catalog) being spread out over multiple event definitions, because they happen to be published by two different microservices in the solution space. This will make it very hard to consume the data. How do I know that I have consumed all the data related to product catalogs? How do I know whether this business rule that I must implement is taking all instances of an enitity into consideration if events pertaining to that entity are being published in many places in the backend, and how do I know that yet another event will not be created in the future thus breaking the correctness of the implementation because it will be ignorant of this new event? It will also be brittle, because when someone wants to refactor the backend for technical reasons (not because of changes to the problem space), e.g. merging two microservices into one, the event definitions might change causing dependent teams extra work to change how they consume the data. This would not have happened if they were proper domain events because proper domain events will only need to change when there are changes in the problem space.

### Anti-pattern 4: Misaligned abstraction

A bounded context has implemented an abstraction which does not exist in the problem space. Changes to requirements breaks the abstraction forcing the owning team and all dependent teams to refactor.

An important aspect of modelling is abstractions. Developers come up with abstractions that will represent some concept or reusable functionality. What happens if the abstractions are not grounded in the problem space? Remember, the requirements are born in the problem space and they will tickle down and hit the abstraction, but if the abstraction is not something that exists in the problem space there is no guarantee that these requirements will align with it. The addition of a new requirement might not fit within the abstraction, forcing the developer to rethink the abstraction or create another. It is therefore a good idea to think about your abstractions from the perspective of the problem space. Did you come up with the abstraction yourself while coding or do the business stakeholders know what this is when you mention its name? Is it something that exist in some shape or form in the problem space, either concretely as a physical object or something less tangible like a concept people from the problem space are using in conversations. As always, a stint of pragmatism is good, your abstractions need not all be grounded in the problem space, but it is always a good idea to be conscious about these things and accepting the risks that comes with your misaligned abstractions.

### Anti-pattern 5: Dominant partner

A system integrates with a vendor or partner that provides capabilities to the company that spans multiple of the company's domains and this violation of single responsibility bleeds into the architecture of the system. 

In this scenario a single team is given the responsibility of integrating with the vendor/partner and that single team implements all the business processes that are related to those capabilities. That team now has responsibilities that are belonging to all those domains. Depending on the size of those domains this can end up being too much responsibility for a single team to handle. As the company evolves and expands into new teams that will take over some of those domains, the responsibility of the existing code will need to be handed over, otherwise the initial team will now have a slice of the responsibility of all those domains, leading to a shared responsibility with the other teams. If the initial team had segregated the code belonging to the different domains into separate microservices handing over the code is a matter of handing over the responsibility of those service to the respective squads. However if the initial team had coupled together the code from the various domains, untangling it and handing it over can be a sizeable task, most likely something that will be postponed. This leads to [Violation of the single responsibility principle](#violation-of-the-single-responsibility-principle), [Shared responsibility](#shared-responsibility) and [Inconsistent responsibility boundaries](#inconsistent-responsibility-boundaries).

This situation often occur because developers are not even aware that multiple domains are involved. It can also occur because the ramifications of ending up in this situation is greatly underestimated.

It is very similar to the well known Blob (or God object) antipattern. The team and their code is ubiquitous. The team members are required in most meetings and your architect will go around the office and utter sentences such as "this team is the beating heart of the company", and will likely not realise that this is a very bad thing. Worse still, those team members will become highly esteemed and have a high degree of influence, a situation that can be desirable for an engineer and there is a risk that those engineers will not be motivated to push for changes.

### Anti-pattern 6: Leaking complexity

While having a good understanding of your own domain is paramount it is not sufficient. In order to build good APIs and good integrations with other bounded contexts it is necessary to have some understanding of the business processes that are depending on your domain. Having this understanding enables you to design APIs that are fit for purpose. Not having this understanding usually leads to teams building APIs that reflect their domain model directly. If the domain is a complex one this will force dependent teams to understand and interact with the domain model in its full complexity even if their use case is a simple one. E.g. in an event driven architecture external events would reflect internal domain events directly and in a complex domain there would be many events and many details and relations to understand and writing and maintaining the code that consumes the data can be a big task.

## Why is this so hard?

A primary reason why these things happen is because the developers has insufficient understanding of the problem space and/or do not understand the importance of aligning the problem space with the solution space, as discussed in the <a href="/blog/maintainability_part1/">first post</a>.

Another common reason is being too strict about adhering to the DRY principle, don't repeat yourself. 

{% note 'The DRY principle' %}

If code is duplicated, maintenance becomes difficult. When making changes you need to remember to update the code in all the places it's duplicated and there's a chance you'll miss some of it, leading to inconsistent code and bugs! Therefore we should move the duplicated code into a function or a class and then reference that. 

{% endnote %}

As a developer this is one of the first principles you learn. It is easy to grasp and makes so much sense! This is really what programming is all about, creating reusable functions and classes that can be composed into larger structures. Even your IDE reminds you about this, if you have duplicated code it will tell you. The duplicated code is staring you right in the face! Just remember one thing, the DRY principle is defined within the confines of your solution space. What if that seemingly duplicated code is actually representing two different concepts in the problem space that, for the time being, just happens to have the same definition in the solution space? Think about the product catalog and the checkout domains from the Amazon example. They both have a product entity and they have the same definition but when Aaron asks you to add a "related products" property to the product catalog and Bridget does not, they are no longer the same. If you understand the problem space you also understand that the product catalog and the checkout process are two very different concepts and you will realise that when you look at your code. You'll know that they'll likely diverge in the future and will resist the temptation to apply the DRY principle and merge together the two identical product classes into one. In this example it is very obvious that there are two representations of a product, but your particular problem space might be much more complex or a lot less intuitive than this toy example. If you don't understand the problem space and is not constantly aware of its presence you'll likely miss this and make the obvious choice and apply the DRY principle instead.

The DRY principle only makes sense to adhere to if the duplicated code is duplicating concepts from your problem space. Let's say you had implemented a business capability in a service and you needed to support another capability that had the same functionality up to that point but where the remaining work would diverge, then there is nothing fundamentally wrong about copying a service consisting of 1000s of lines of code and handing it over to another team to allow them to build that new capability while allowing you to evolve the existing. This doesn't happen often in practice, but consider it. It goes against your gut to copy 1000s of lines of code, but your gut feeling could be wrong.

Sometimes pragmatism wins and reusing a service even if it is not fit for purpose is still the right choice. Time to market, resource allocation constraints etc. should of course always be factored in. However, there is a big difference between making this choice with open eyes, knowing that this could cause problems later as the solution evolves and taking necessary precautions, and on the other hand not realising that there even is an issue and end up in a bad situation later when the solution is not fit for the new requirements.
