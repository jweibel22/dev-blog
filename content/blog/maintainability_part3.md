---
title: "Building maintainable microservices - Part 3: Management level"
description: Agility
date: 2024-06-24
tags:
  - Domain Driven Design
  - Microservices
---

This is the final part in a series of blog posts focused on maintainability of microservice based backends.
The series will discuss issues around maintainability at different levels. In the <a href="/blog/maintainability_part1/">first post</a> we were looking at the design level, in the <a href="/blog/maintainability_part2/">second post</a> we were at the technical architecture level, and in this final post we're at the management level.

## Introduction

In organisations characterised by short term focus and where activities concerned with gaining an understanding of the problem space are not valued or prioritised project management will often take decisions that will make the situation worse. Badly informed architecture decisions will be taken which will lessen the maintainability of the system over time and progress will slow to a grinding halt. What can the engineers do about it?

## Accumulating tech debt

Unpaid technical debt is a major culprit of bad maintainability. It can come in many shapes and forms.

- **The system is buggy:** It can lack support for edge cases which means that engineers will need to intervene whenever those edge cases are hit in production. It can also have bad reliability causing incidents that will also eat away at your time.
- **The system is badly designed:** Perhaps the requirements changed but the appropriate change to the design that should have been done to better accommodate the new requirements were never done, but instead hacks were added here and there to make it work. This make the solution harder to maintain.

Sometimes technical debt is accumulated with your eyes open. You sprint towards a deadline and take short cuts. At other times technical debt arises because your understanding of the problem you were trying to solve was inadequate. We discussed this issue in detail in the <a href="/blog/maintainability_part1/">first post</a> in the series.

If technical debt is managed properly it can be a powerful tool to steer velocity, to push when the opportunity is ripe, and to slow down at less opportune times to pay off accumulated debt. Your company will have a product/sales department that are constantly pushing the gas pedal and a tech department that are pushing the break pedal, and hopefully the management at your company has a good working relationship and are able to handle these two opposing forces in an optimal way.

Communicating technical debt to management is notoriously difficult. One common approach is to annotate engineering work as either run, improve or change. Management can then follow along and intervene if run grows too large.

{% note 'Explainer: Change - Improve - Run' %}

It is common practice to split the work of a software engineer into three types: `change`, `run` and `improve`.
`Change` denotes feature development, `run` is work needed to keep the system running and `improve` work are tasks that reduce the amount of run. Fixing a bug that is causing issues in production is an example of improve work because it will remove the run work associated with handling the incidents. Project managers will often pay attention to the proportion of run work that is required by a team and prioritize improve work accordingly. If run work is taking up a large amount of time it could be an indicator that improve work is not being sufficiently prioritized.

{% endnote %}

It is of course not a sufficient approach. The consequences of accumulating technical debt often doesn't appear until much later and will creep in without management noticing. Quantifying technical debt is very difficult, a possible alternative approach of communicating technical debt to management is to look at known antipatterns that causes technical debt. When these arise you can point them out to management. The <a href="/blog/responsibility_boundaries/">second post</a> in the series listed several examples.

## Responsibility boundaries

Bad responsibility boundaries is another major culprit of bad maintainability, especially in microservice based architectures. We discussed this issue in detail in the <a href="/blog/responsibility_boundaries/">second post</a> in the series. This section lists some common anti patterns that cause bad responsibility boundaries. When you identify one of these antipatterns you can point them out. The list is by no means exhaustive.

### Unclear ownership

As the business expands some teams will no longer be able to deal with the cognitive load and responsibility boundaries will need to be altered. Sometimes this is done by placing new feature development with a different team and leaving the old team with the maintenance of the old solution. Often there is no agreed plan on when and how to migrate the old responsibilities to the new team and this results in a fragmented solution space that suffers from e.g. [Inconsistent responsibility boundaries](../maintainability_part2#inconsistent-responsibility-boundaries). Quite often the reason for not prioritizing the alignment is that the consequences of not doing so are underestimated.

{% note 'Real world example' %}

A company entered a new country and needed to generalise the existing features to work in the new country but also build new country specific features. The  responsibilities for the existing solution were split between a stream-aligned team and a `complicated subsystems` team. When development started on adding support for the new country it was decided that it would be better for the complicated subsystems team to be a stream-aligned team, and a new solution based on this decision was built for the new country. Years passed, and the misalignment of the architectures for the different countries persisted. In the new country the team acted as a stream-aligned team and in the old country that role was still assigned to the old team. This caused confusion for the developers and all stakeholders. E.g. Triaging incoming service desk tickets was a big challenge because it was quite difficult for non-technical staff to determine ownership of specific tickets.

{% endnote %}

### Responsibility is delegated based on roadmap bandwidth

In an organisation that has no clear idea of the domain landscape in their problem space and where this is not taken into consideration from project management and leadership responsibility of new projects are often delegated based on team availability. This is the most obvious way to delegate tasks as it optimises utilisation of manpower, but there is obviously no correlation between roadmap bandwidth and domain ownership, so this naturally leads to a scattered ownership landscape, e.g. [Inconsistent responsibility boundaries](../maintainability_part2#inconsistent-responsibility-boundaries). There is nothing inherently wrong in taking roadmap bandwidth into consideration but if there is no anchoring in domain ownership to help steer the decisions it quickly leads to entropy.

{% note 'Real world example' %}

A team was responsible for a domain but only for the users located in some of the countries. For users from other countries the responsibility for the domain lied within a different team, even though the problem space (in this example) did not distinguish between users from different countries. This situation can e.g. occur when different countries requires different integrations and the implementation of those integrations are carried out over time correlated to when the company enters those countries. In those situations there is a possibility that the team that owns the domain does not have room in their roadmap when the time comes and the implementation task is then handed over to another team. This leads to the domain responsibility being scattered. Since, in this example, the country boundary does not exist in the domain, product evolution will happen across this boundary going forward, i.e. new features will/should be introduced and changed for all countries at the same time, however since the responsibility for all countries are not owned by a single team, implementing the new functionality would involve multiple teams leading to project and roadmap coordination work and possibly duplicated effort for the implementation.

{% endnote %}

### Responsibility is delegated based on technology familiarity

When the implementation of a new business capability involves a specific technology already known by a subset of teams one of those teams will be selected for carrying out the implementation while completely disregarding if the new business capability belongs to a domain that the team is already responsible for. This leads to [Shared responsibility](../maintainability_part2#shared-responsibility) or  [Inconsistent responsibility boundaries](../maintainability_part2#inconsistent-responsibility-boundaries). Responsibilities of new capabilities that belong to domains owned by other teams wíll get assigned to the team that has the experience with the technology and the team is effectively taking slices of the responsibilities that should belong to the other teams.

It is not necessary for technology based knowledge to be tied to specific teams. The experienced team can act as mentors and share knowledge with other teams, or they can build generic (i.e. not domain specific) components, like libraries or services, that can be used by other teams that need to use the technology.

### Responsibility is delegated based on solution space

A component of the system is taking on too much responsibility and is often involved when new features are developed. For this reason the team owning the component will often be delegated responsibility of business processes/capabilities that belong to domains owned by other teams. This situation often occur because developers are not even aware that multiple domains are involved. It can also occur because the ramifications of ending up in this situation is greatly underestimated.

This is often seen when complex subsystem teams or platform teams builds generic components that gets entangled with the business domains. E.g. it could be a team owning a CRM product or a customer service product. If care is not taken to allow other teams to plug in to the generic component often the team owning the generic component ends up becoming a bottleneck and ends up taking slices of the domains of other teams, leading to [Shared responsibility](../maintainability_part2#shared-responsibility) or  [Inconsistent responsibility boundaries](../maintainability_part2#inconsistent-responsibility-boundaries).

## Why is this so hard?

People think they can take an iterative just-in-time approach with regards to requirement gathering (i.e domain understanding). They have an agile mindset and they think they can POC it, put it in production, gather feedback and then continuously re-iterate. There is nothing wrong with being agile but they don't realise the huge costs associated with rewriting the software. This is especially true for microservice architectures. If the refactoring goes beyond the boundaries of bounded contexts this means changing APIs and coordinating the new release with all involved teams. Perhaps even migrating existing state between services. If the architecture is event oriented it might also involve recreating the new events for historic data and reconstituting to all consumers.

It is very difficult to detect the misalignment in realtime because quite often the consequences of the misinformed decisions do not appear until later when the solution must be evolved and it is no longer a good fit. By the time that the symptoms, such as a tightly coupled architecture, unclear responsibility boundaries, the constant need for refactoring projects and the build up of legacy code becomes visible, it is far too late.

The business is not run by tech. There is often a big pressure from management to focus efforts on feature development and it can be a challenge to communicate the value of working on other tasks because they are often very technical in nature. This is a well known problem when dealing with e.g tech debt and it is also problem here.

## How to fix this issue?

As a first measure you should try to assess your situation. If you're not currently following DDD or similar practices chances are that your solution is misaligned with the problem space. Take notice of people from the organisation speaking of areas of responsibilities that you were not aware of. "I really want X (person or group of people) to take responsibility for Y". If this does not align with your reality then this is an indication that their view of the world is different than yours, that is, your understanding of the problem space conflicts with theirs. In this case your solution might be fitted well to your understanding of the problem space, but that understanding could be wrong or just not aligned with the rest of the organisation.

The business or management might need some convincing before they will allow you to invest resources into improving your understanding of the problem space. The responsibility for doing this usually belongs to a head-of-tech or a CTO if such a role exists in your company. Try to give examples of projects from the past that were harder and took longer than they should have. Projects that were simple in the problem space but became very complex in the solution space. Perhaps you can find examples of projects that involved many teams and required coordination and project management and were difficult to roll out and test due to all the interdependencies even though the problem belonged to a single domain? Being able to deliver projects faster and give more reliable estimates constitute convincing arguments. You could also try to highlight the amount of legacy code that has been generated.

Finally you might need to make adjustments to the organisation. Every domain should have a designated product owner who has the responsibility of keeping the understanding of the problem space up to date. That involves understanding how the business currently operates and how it is expected to evolve in the near or mid term future. The company should have designated architects. The architects will be in close contact with the stakeholders and product owners and keep an up to date model of the current and future domain landscape. They will understand the current architecture of the solution and be responsible for a target architecture that should strive to align the solution as much as possible with the domain landscape.

## Final thoughts

Software design is an inherent iterative process. Most of the industry have now moved on from the days of the waterfall process and learned that splitting the process into a design and an implementation phase is almost never feasible. The devil is in the details and even small development tasks can't be designed up front because often new details and unforeseen obstacles appear while coding. However, the fact that development is carried out in an agile fashion should not lead one to believe that gathering requirements and understanding the problem space can wait till the very last moment. The misalignment between the problem space and the solution space that this approach allows to exist will negatively impact maintainability and thus profitability in the long run.

You can build your design and abstractions based on your intuitions and, depending on how well you know the domain, they'll probably fit well enough, at least for some time, but if the product lives long enough, sooner or later it will catch up to you and the design overhaul will become a reality. This happens all the time. How many of the projects that you worked on were brownfield or greenfield projects? They're mostly greenfield on paper, but many of them are just a new generation of the old solution that no longer was able to fit the problem space. (there are of course other reasons why projects needs to be rebuilt, like deprecated tech stacks, mergers etc.) Again, this is not a big tragedy, but it certainly is very wasteful and not at all cost efficient in the long run. Thing is, budgets are often short lived and time to market is vital. This is often the reality and those factors should of course be weighed in when considering if and how DDD could be relevant. Just remember that projects tend to stay around much longer than the initial budget and if you intend to stay around as well you'll do your future self a huge disservice if you don't think ahead.
