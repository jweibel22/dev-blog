---
title: How to reach production in less than 6 months
description: A collection of ideas on how to get from scratch to an MVP in the shortest possible time
date: 2024-08-02
draft: true
tags:
  - Data
  - Developer productivity
---

Some projects will have deadlines imposed by the business. They could originate from time-to-market considerations or from specific contract obligations, and for those projects delivery time is obviously important. However, I would ague that even on projects that have no hard deadlines, the time to reach production is still vital. Reaching production sooner rather than later builds trust with stakeholders and the developers will get a feeling that they're making an impact. It also closes the feedback loop. Getting feedback from experiments or test users from production is often the most valuable type of feedback and will help the agility of the project going forward.

In this post I share my experiences related to reducing the time to reach production on greenfield projects. In particular I will be referring to my experiences with building a data platform from scratch. It was a big undertaking, I was a one man army, and it was therefore absolutely vital to be pragmatic and diligent about prioritisation to ensure that the product could provide value to the business in a reasonable amount of time.

## What is the MVP

Obviously you'll need to define the minimal viable project. What is the smallest set of requirements that you can deliver while still having a viable value proposition? If your product is replacing an existing product you must find out what capabilities that product has and you must take care to deliver a better alternative, or at least not something that is significantly worse. If the existing product is comprehensive you could consider replacing only parts of it, and then carry out the sunsetting in iterations. This will of course need to be a collaboration between the development team and stakeholders and/or product owners. If you're building a platform product it could be feasible to set up regular feedback sessions with your target users to help prioritise and shape the MVP.

{% note '' %}

The company I was working with already had an existing product. The product consisted of a postgreql database, a BI tool, and a microservice that was subscribing to event data coming from other services from the backend and mapping the data into fact and dimension tables in the postgresql database.

The company was a startup and the initial motivation behind building the existing product was to make it possible for people from the business to keep track of various KPIs via the BI tool. The set of KPIs were fairly static so the solution worked ok for a while, but it didn't take long before other analytical use cases appeared. Those new use cases came with the need to get more data into the database and to execute queries that did not fit the facts and dimensions structure that had been built and did not utilize the indexes. Making these sorts of changes was done in the microservice. It was responsible for ingesting, cleaning and mapping the source data to the table structure in the database, and so to add a new data source a go developer was needed to add a subscription to the new event and create the needed mappings. This was painful, the go developer was not personally invested in the analytical use cases and this felt like a chore to him, and the analyst was stuck waiting for the go developer to prepare the tables. To make matters worse, most often it was impossible to backfill data, so when a new subscription to an event was created it was not possible to get hold of all the historical data that had been published before the subscription was created. This meant in practice that for data to be available for an analyst it would be required that he was able to predict which types of analytical queries he would be interested in making at the time when a new event started to be published from the backend. New analytical use cases pop up all the time so this is clearly not feasible.

What the company needed was a data warehouse that supported OLAP queries without requiring up front indexing. Such a data warehouse could obviously also easily solve the existing use case of making KPIs available to the business. Alleviating the pain associated with adding new data sources and supporting new queries and also solving the performance issues they were facing due to the postgresql database struggling to keep up with the large amounts of data was a very appealing value proposition and something "worth waiting for". However, the existing product had not taken years to build, so it would not be reasonable to spend years building a data platform without delivering a product that could provide some value. 

The absolute minimal set of requirements were determined to be:

1. A data warehouse.
2. An ingestion pipeline that ingests the data from the backend into the data warehouse and makes it available for analysts.
3. Tooling that will make it possible to refine the raw event data in the data warehouse into better suited table structures. The tooling must be SQL based so analysts can own this.

The initial product did not need to make ALL data from the backend available in the data warehouse, only the data that was currently in use was necessary. The SQL tooling did not need to be perfect, it just needed to be less painful than the existing solution, which was a very low bar.

{% endnote %}

## Compromise

You will need to cut corners. Identify what are need-to-haves and what you can be less diligent about. You probably want to ensure durability and avoid loosing data, but perhaps the UX/DX doesn't need to be perfect?

You'll need to take calculated risks. E.g. if you release a component without testing it, what would be the impact to the users if a bug creeped into production, how would you resolve the issue and how long would it take to resolve it? It can be ok to fail in production if the impact is small and the time to recover is low, but if the production issues result in a large maintenance burden the ROI of implementing it and testing it properly before releasing it could be above 0.

The compromises you make will result in feature-incompleteness or perhaps a less reliable product. If you keep track of user/developer pains you'll be aware of these shortcomings and eventually they will get prioritised. However, you will feel a sting as it will conflict with your professional pride.

{% note '' %}

We chose a tool called DBT for providing SQL based transformation capabilities to the users of the data platform. DBT was in the early stages at the time (2019), there was no dbt cloud and so we had to roll our own execution environment. We did this by copying the user code into a docker image containing a dbt installation as part of our build pipeline, and then we would run this image on kubernetes (using airflow as a scheduler). The docker image was very simple, it contained an entrypoint.sh shell script that included a `dbt run` command.

As time passed new requirements meant that the simple `dbt run` command needed to be extended with more options and we did this by extending the shell script. There were no tests of the shell script and sometimes bugs would sneak into the script causing the execution of dbt to fail in airflow. This was obviously an inconvenience to users who more often than not had no clue what was causing the error to occur. Most often the error was detected and corrected in a few minutes but it was still causing developer pains and a great source of instability, yet it was allowed to stay this way for 6 months.

Eventually we finally got around to making this more robust. The shell script was converted to python code and acceptance tests were added.

{% endnote %}

## Ops

Going to production as early as possible entails having to operate immature software. This will most likely result in many production issues and a lot of related "run" work, and you'll need to have a plan for how to handle this. If you end up spending all your time doing run work the project will eventually stall.

{% note '' %}

One of the classical problems faced by data platforms is how to get hold of the schemas of the data that is being ingested. The data that was ingested was based on rabbitmq messages exchanged between the backend services written in go and the schema definitions were written in go code which was buried inside of a larger shared go project and since the data platform was not written in go, it was written in scala and python, the schemas were not directly accessible.

In a situation where schema definitions are not readily available getting hold of the schema definitions will require manual effort, and one must decide where this responsibility lies. Does the responsibility lie with the data producers or with the consumers, in this case the data platform? Generally, this responsibility should lie with the data producers because if producers are required to provide a valid schema it allows anyone to consume the data without hassle, but in this case all consumers except for the data platform were go microservices and were therefore able to use the schemas provided in the go project. Moving to a language agnostic schema format, would require a great deal of discussions, alignments and agreements across the entire organisation. At the time, there was no formal RFC process in place and the data platform team had no mandate to make such requirements. Changing the schema definition process would require a lot of work for all backend teams and there was simply no appetite to prioritise this as the pain was only felt by the data teams. For this reason the responsibility of getting hold of the schemas fell on the data platform.

One popular solution in these situations is to extract schemas using schema inference. With schema inference the schemas in the platform are automatically inferred and dynamically adapted to the data as it is being ingested. Schema inference causes a lot of headaches, especially if many heterogenous (polyglot) sources are at play. E.g. Is “7" an integer or a decimal number? (some languages/frameworks leaves out the decimal points when it is 0). Is “.” a decimal point or a thousands separator? These imperfections of schema inference inevitably results in inaccurate schemas and eventually in data that cannot be ingested because of mismatching schemas. Another common problem is that schemas can evolve in a non compatible way causing errors to occur when the schema inference process tries to register the new version of the schema.

Instead we chose to build tooling that could extract avro schemas from the aforementioned go project. The schemas in the go code were not defined in a declarative way, i.e. it was not as easy as scanning for structs and using the fully qualified name of the struct as the schema name. It was, however, possible to parse the go code of the project, traversed the AST and then by making a number of assumptions on the structure of the code extract the information that was needed from the code and generate the schemas in avro format. Luckily, it turned out that there was a lot of conformity among the developers who all adhered to the existing code structure when extending the code, which meant that the solution wasn’t as brittle as one might have feared. Nevertheless, because of the nature of how we had to extract the schemas and because we wanted to allow developer teams to experiment in dev/prod which might entail introducing breaking changes to schemas, there was a high probability that issues related to faulty schemas could cause ingestion of messages to halt and create noise in the logs that would require the attention of the engineers responsible for the platform and in many cases this work would be `urgent + not important` because the errors would be related to data that was not in use by the data analysts. We therefore chose not to automate the schema registration process but instead create a CLI that allowed a data analyst to generate and register a schema on demand using the tooling described above.

Because we needed to capture all the messages that were exchanged between the backend services and we required schemas to be defined upfront we needed to allow for schemaless data to exist in the data platform. To this end we decided to create two data lakes. An unstructured data lake and a structured data lake. The purpose of the unstructured data lake was to capture everything. It would contain all incoming data in its raw format (json in this case) without requiring a schema to be present. The structured data lake would contain the subset of data that had a schema defined.

Any issues related to schemas, e.g. issues with generating schemas from the go code, or schemas not being compatible with the ingested data, would now either occur in the tooling or in the data pipeline that applied the schema from the raw data. It would not effect the data pipeline that ingested the raw data into the unstructured data lake. I.e. the most critical requirement, not loosing any data, would be uncoupled from the inherent unstable process of applying schemas to the data, and thus those errors which we were expecting to see a lot of, especially in the beginning, were now no longer urgent and critical errors. Furthermore if schema related issues arose, users of the data platform could fix the invalid schema and run a backfill operation to correct the wrong data in the structured data lake, and all of this could be done without the presence of a member of the data platform team. Building two data lakes and two data pipelines rather than one is a larger effort, but the expectation was that the amount of avoided run work would quickly make up for this investment. (There are also many other advantages to making this split in the data lake that we will not get into in this post.)

{% endnote %}

See <a href="/blog/fail_fast/">Ops driven design</a> for a more general discussion on this topic.

## Single responsibility

As mentioned, it might make sense to consider cutting corners in order to reach production in a reasonable amount of time. Cutting corners will have consequences, such as less stability or usability of the product, which can be a reasonable compromise to make in certain situations. I would, however, advise against violating the single responsibility principle. It can be very tempting to cram together multiple concerns into a single service, tool or CLI. It is, after all, faster to build a single service and a single API rather than multiple ones and there will be less movable parts.

However, you should consider the consequences of doing so. If the idea is to move fast and reach production in the shortest possible time, then when you reach production you'll have an unfinished product that you will need to develop further. When you violate the single responsibility principle it will be more difficult to evolve the product because different concerns, that you'll most likely want to evolve independently, are entangled in the architecture. This will hinder further development as you'll often find you'll need to refactor a component before you can evolve it. Reaching production fast only to slow down significantly later is not a good strategy.

Instead try to understand your problem space and build up your solution based on components that each serve a specific purpose. That way, when the product needs to be evolved, the individual components can be replaced or extended independently.

{% note '' %}

As part of onboarding a new user on the data platform we needed to set up her development environment which included a personal database on a redshift  cluster. This personal database was used to run dbt locally as part of the development work flow (when developing new data models using dbt).
Setting this up involved executing some DDL statements on the redshift cluster for setting up the database, the user and permissions, and it would be necessary to repeat this if we wanted to provision a new redshift cluster, e.g. to increase capacity. We therefore wanted to automate this.

To comply with EU GDPR regulations it was important to limit access to PII (personally identifiable information). To this end we wanted to implement an RBAC mechanism, redshift did not support this at the time, where users would need to take on purpose specific roles in order to gain access to tables containing PII. To allow users to manage roles and permissions for those roles we needed a service that could take such a configuration and execute the required DDL statements to set up the users and permissions on the redshift database. This was a very similar problem to the problem described above of setting up personal databases.

We decided to implement a single service to handle both use cases. Users would maintain a single configuration file (protected by the 4 eyed principle) listing all users, all roles, and which users had access to which roles, and a backend service (a kubernetes controller to be exact) would apply the changes to the redshift clusters.

Now let's say we wanted to migrate to dbt cloud. All the logic related to setting up the development environment would need to be removed from the service, one would need to take care not to inadvertently change the rest of the logic. Similarly, let's say redshift implemented support for RBAC, we would no longer be required to maintain our own solution. One would need to remove that part from the service and again take great care not to touch the code that set up the development environment. Had each use case been implemented by separate services, or at least clearly separated in the service, evolving the product would have been easier.

{% endnote %}
