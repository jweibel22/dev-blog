---
title: "DDD: What is it good for?"
description: Building maintainable microservices - part 1
date: 2024-06-22
tags:
  - Domain Driven Design
  - Microservices
---

This is part 1 in a series of blog posts focused on maintainability of microservice based backends.
The series will discuss issues around maintainability at different levels, we start at the lowest level, the code, and move upwards through the organisation.

## Introduction

I often encounter the attitude that DDD is not worth the effort and in my opinion those developers are underestimating the impact it can have. When developing a distributed backend, I believe that DDD is a vital tool for decomposing the backend into smaller decoupled and autonomous components (e.g. microservices) and ensure clear and stable responsibility boundaries. In my opinion, building a microservice backend without an up front and continuous effort into understanding the problem space, e.g. by employing DDD, is a blatant mistake. A mistake that I see happening too often. Obviously the real world does not allow for idealism. Time to market and budgets demands short term focus rather than long term planning but it is the responsibility of the software developer or architect to make good and conscious decisions and I believe that even small efforts can have a compounding effect that will help scalability and maintainability on the mid to long term. Conversely, neglecting it will have adverse effects, impacting the ability to evolve the software and react to changing requirements eventually leading to stagnating productivity.

DDD is a very large topic. This post will not be going into details about DDD, and although some familiarity with DDD concepts is a good prerequisite, the text can be read without prior knowledge, but the reader is encouraged to read up on DDD later. The post will not discuss all benefits and consequences of applying DDD, rather it will focus on a single aspect only, the maintainability/adaptability of the software.

## Problem space and solution space

There are, at all times, two models at play. There's a model of the problem space, which is a model of the business problem you're trying to solve and a model of the solution space which is your current code base. The model of the problem space may not be explicitly defined. It is often implicit and different people may have a different understanding of it. When the code is written this understanding of the problem space is used by the developers to model the code. It is often quite complicated and driven by the business so there is a high risk that the developers do not have a proper understanding, i.e. they end up implementing something else than what the business expected. A design approach called Domain Driven Design (DDD) has been invented to tackle this common issue by giving the developers a set of tools and design principles to help develop and maintain an explicit model of the domain (the problem space) which is aligned with the understanding of the problem space by the stakeholders and product owners. For example, developers might invite relevant subject matter experts to event storming sessions where they in collaboration come to a common understanding of the business processes and capabilities that define the domain they need to model. They'll develop artifacts, e.g. notes, descriptions, diagrams, wireframes etc. that describe the domain model, i.e. makes it explicit. DDD also contains many best practices that aims at aligning the solution space (the code) as much as possible to the problem space, e.g. by introducing a common "ubiquitous" language which is shared between people from the business and developers and that is used throughout the code base.

The domain model (whether explicit or implicit) is not static. Like the business, it will evolve over time. New or changing requirements will continuously alter the domain model and as it changes the developers will need to evolve the code base to fit the new requirements. The concepts making up the domain model, like actors, entities, processes etc. are all related to each other, some more than others. When following a DDD approach, once all business processes has been sketched out in an event storming session it should stand out how the concepts are coupled to each other. The degree of coupling of all those relationships should be respected in the solution space. If something is closely related in the problem space that should also be the case in the code you write and if they're independent they should not be coupled in the code. This is in fact what is being expressed by Conway's law which you may already be familiar with:

{% quote 'Conway\'s law', 'Melvin E. Conway', 'How Do Committees Invent?' %}

Organizations which design systems (in the broad sense used here) are constrained to produce designs which are copies of the communication structures of these organizations.

{% endquote %}

If there's a misalignment between the problem space and the solution space, i.e. your code and the domain model are structured differently, simple changes in the problem space can turn out to be much harder to carry out in the solution space than they should be. To make this point let's turn to some of the quality metrics of software design that you're most likely already familiar with.

## Coupling and cohesion

You're probably already familiar with the term **single responsibility** as it's often mentioned as an important quality attribute of software solutions. However, there can be different interpretations of what this concept means, so in order to avoid any confusion let's do a short recap here and establish what the author means when referring to those concepts.

The **single responsibility principle** states that a component (a class, module, microservice etc.) should have one and only one responsibility. What is meant by this? When does a component have multiple responsibilities?

Robert C. Martin has [expressed the principle](https://blog.cleancoder.com/uncle-bob/2014/05/08/SingleReponsibilityPrinciple.html) as **a class should have only one reason to change**. Another way to express this is to say that in order to comply with the single responsibility principle code that changes for the same reason should be close together (coupled) where as code that changes for different reasons should not.

{% simplequote 'Robert C. Martin', 'The Clean Code Blog' %}

If you think about this you’ll realize that this is just another way to define cohesion and coupling. We want to increase the cohesion between things that change for the same reasons, and we want to decrease the coupling between those things that change for different reasons.

{% endsimplequote %}

So what is a "reason to change" the code?

The reason to change the code comes from the stakeholders that operate in the problem space. They want new or improved features. The stakeholders are coming from the business and are organised using the business org chart. E.g. at Amazon there could be a stakeholder Aaron from the "product" department and another stakeholder Bridget from the "checkout" department. Aaron is concerned with how products are organised and presented on the Amazon site, Bridget is concerned with the user experience after a purchase decision has been made, e.g. what does the email receipt after the purchase look like? So if Aaron asked you to make a change to how product information is presented and you accidentally also changed how it was shown on the email receipt because the component you altered was being used in both places (i.e. it was responsible for delivering product information to both) then your component does not have a single responsibility. The product page and the email receipt are independent concepts in the problem space because the business has been organised in such a way that they are the responsibility of different departments that each have their own stakeholders. For that reason they will probably change independently and for different reasons. If Aaron wants the product information to look differently there is no reason to expect that Bridget will want the same change or even be aware that Aaron has requested the change.

So in this case the component was responsible for business logic that is coupled to multiple independent concepts in the problem space. The concepts are decoupled in the problem space but they were coupled in the solution space.

{% simplequote 'Robert C. Martin', 'The Clean Code Blog' %}

However, as you think about this principle, remember that the reasons for change are people. It is people who request changes. And you don’t want to confuse those people, or yourself, by mixing together the code that many different people care about for different reasons.

{% endsimplequote %}

Having this misalignment of the problem space and solution space means that changing your solution will be hard. As the problem space evolves, changes to different parts of the problem space (which are independent and decoupled) will be difficult because of the entangled mess. A change which is simple to express in the problem space will be very hard to do in the solution space because the component responsible for the concept is coupled to other concepts from the problem space.

You can experience the inverse problem as well. Your component shares its responsibilities with other components. Those other components might even be owned by other teams. So when Aaron wants his change you'll need to make changes in multiple places in order for the change to be implemented. If multiple teams are owning the components this means coordination, meetings, project management etc. which could otherwise have been avoided. This is what is referred to as low cohesion in any `introduction to programming` text book, and it is an undesirable characteristic.

So this is really all about cohesion and coupling. You want high cohesion, behaviour that is related should be close together, and inversely behaviour/features that are unrelated should not be close together in the code, that is, have low coupling.

The crucial point here is that you cannot comply with the single responsibility principle (or more generally keeping cohesion and coupling in the code just right) unless you have a proper understanding of the problem space because the changes to the code (solution space) are driven by changes in the problem space. In other words things that change for the same reason do so because they're related in the problem space, and things that change for different reasons (and should therefore not be coupled in the code) do so because they're independent concepts in the problem space. So if you don't have a proper understanding of how concepts are related in the problem space how could you possibly make good decisions on how to structure your code in such a way that you don't break the single responsibility principle?

## Why is this so hard?

This is all really quite obvious, isn't it? You need to understand the problem you're solving. So why is it so hard to get right?

You can spend your entire professional life confined within the solution space, adapting the solution iteratively as you learn more about the requirements during development. You can skip the event storming, the ubiquitous language and all of the other DDD practices. It will work and you'll be able to deliver the requested feature. Some new requirements will require a larger overhaul of the design because they cannot easily be adapted, the project will last several months/year but you'll eventually deliver the new features, while also having improved and cleaned up the solution, and everyone is happy. Little do the developers and stakeholders know that if only the solution had been a better fit to the problem space to begin with the larger refactoring project would not have been needed and the features could have been delivered in a few weeks. This is an alternative reality that never existed. Stakeholders usually don't have a good idea about what would be a reasonable amount of time to implement a feature. Sure it looks easy on paper, but IT is complex!

The misalignment between the problem space and solution space is not apparent when a developer is looking at the code. Even if the code looks SOLID it might very well be that the apparent single responsibility of a class in the code is actually multiple responsibilities in the problem space. Later when a change arrives in the problem space it can be hard to implement because the class implementing the related functionality is used to solve a different independent requirement in the problem space.

As mentioned, when your code base does not have the high cohesion/low coupling characteristics maintainability and evolution of the solution will be harder. However, in a distributed system the evolution of the solution might happen in another team that will then suffer the consequences of the bad choices made by your team. This is very common. This means that you will not yourself suffer the consequences of your bad choices. This can mean that teams do not learn from their mistakes. The price is paid in slower development and a less robust solution throughout the organisation, but it is not at all easy to see the cause and effect from a local perspective. Therefore architects with more of an overview are needed for sparring.

Every time you need to do a bug fix or refactoring task, think about how you ended up in this situation. Did it happen due to not putting enough consideration/effort into the original solution, e.g. because of time constraints, or was the original solution actually thorough, but the issue arose because of incomplete understanding of the problem that was being solved? If the latter case occurs often it is time to put more focus on practices aimed at improving your understanding of the problem space in due time (i.e. not just-in-time)
