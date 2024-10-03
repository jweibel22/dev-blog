---
title: How to run faster
description: Sharing an example of how to build tooling that will make your run work faster
date: 2024-07-24
tags:
  - Developer productivity
---

Some systems are of a complexity that a certain baseline of `run work` is unavoidable. If the system is large or complex or if there's a lot of ongoing feature development, issues will arise. In addition, sometimes systems are plagued with legacy and getting out of that situation is not something that can be achieved in the short to mid term.

{% note 'Change - Improve - Run' %}

It is common practice to split the work of a software engineer into three types: `change`, `run` and `improve`.
`Change` denotes feature development, `run` is work needed to keep the system running and `improve` work are tasks that reduce the amount of run. Fixing a bug that is causing issues in production is an example of improve work because it will remove the run work associated with handling the incidents. Project managers will often pay attention to the proportion of run work that is required by a team and prioritize improve work accordingly. If run work is taking up a large amount of time it could be an indicator that improve work is not being sufficiently prioritized.

{% endnote %}

If you find yourself in a situation where run work is taking up a significant chunk of your time and there is nothing you can do about it from a project management perspective you should look at how you can optimize your run workflow. If you spend a lot of your time digging through logs, logging into various systems, copy pasting strings between applications etc. it might be worth considering how you can reduce this overhead. In this post I give an example of what I did on a project to help myself solve the run tasks faster.

DISCLAIMER: The code samples shown in this post are for illustration purposes and are not extracted from any real system.

## Context

The backend was built using a microservice architecture, the services were running in kubernetes. Databases were running in AWS RDS. It was possible to get access to the database of a service via psql in the terminal. If an incident had caused bad state to be written to the database it was common practice to implement so called `mission control` endpoints for the service that could be used to execute a corrective action. It was also possible to request write access to the database and correct the data directly via sql. The mission control endpoints were usually implemented using graphql and engineers would execute their mission control endpoints via the graphql playground UI directly in the browser.

There were several annoyances with this setup:

- It was necessary to have a terminal open for each service database that an engineer would need to access, or alternatively log out and log back in via psql, and data queried from one service database could not be joined with data queried from another service database.
- Executing the mission control endpoints were done in the browser, so all parameters would need to be passed into the playground UI. Sometimes an incident had caused issues related to many entities so that could entail constructing long list of entity IDs that would need to be pasted in. This was time consuming and error prone.
- The centralised log were accessed via the browser. It was common to use the logs to obtain IDs of entities that would need to be inspected in the database. The IDs from the logs would need to be copied and pasted into psql as valid SQL, also a time consuming and error prone task.

## Scripting

We wanted to find a way to replace the manual process with a scripting approach. Scripting run work has the obvious benefit that the code becomes an artifact and can be submitted to a git repository like all other code.

- A run script can be reviewed before being executed. This can be useful depending on the risk factor of the specific run script.
- Previous run work serves as documentation. It can be used as inspiration on how to solve a run task.
- Previous run work is automatically documented. If a database update carried out by a mission control endpoint is later observed by an engineer it can be traced back to a run script which can provide context.

## Pycharm

We were looking for a unified environment where an engineer could log in once and gain access to all service databases and mission control endpoints and where data could be joined across databases and systems. Python is one of the most popular scripting languages and it turned out that the PyCharm IDE had everything we needed. We created a simple folder structure, one folder per week, and when an engineer was solving a run task he would put the corresponding script into the folder of the week. Quite often a run task would be similar to other run tasks and any reusable code was put into another package that could be referenced from all the run scripts.

{% image "./run_folder.png", "" %}

Pycharm provides a REPL (called python console) and Jupyter notebooks out of the box. Both are very suitable when solving run tasks and we will get into more details about this later.

## Everything is a dataframe

To ensure data from multiple data sources could be joined and correlated and to ensure a uniform experience we adopted an `everything is a dataframe` principle in the code base, i.e. all functions that return data should return the data in the form of a [pandas](https://pandas.pydata.org/) dataframe. This way the capabilities of pandas can be used to join, group, aggregate, plot and export data.

## Dry run

A global `dry_run` boolean variable is defined that defaults to `True`. When enabled all functions that perform side effects should run in `dry run` mode, i.e. not perform the side effects but instead print out the intended action. This will allow engineers to test their scripts and the associated side effects before executing it. It will also make it possible to create a PR containing the test script including its output that can be reviewed before it is executed. We will get into more details about how this works in the [Notebooks](#notebooks) section.

## Console

Pycharm provides a REPL (called python console) out of the box. The interactive nature of the console makes it perfect for exploration and debugging scenarios. E.g. sometimes you'll find that your logging verbosity is insufficient and you'll need to look at the actual state of your application to understand what has happened. We used the console as a way to gain access to all the databases of our services. In this section we'll go through an example scenario to illustrate the usefulness of the console.

Imagine you're looking at a dashboard and is discovering that some sagas have gotten stuck:

{% image "./runboard.png", "", [900] %}

The next step would be going to the python console in Pycharm and trying to find out what happened. When the console is opened the user will get logged in and a prompt appears.

{% image "./python_console.png", "", [900] %}

After looking up the failed sagas in the console it turns out that a few of them failed due to a null pointer exception. Turns out there was a bug in the code triggered by an unexpected edge case. We fix the bug in the code and redeploy. The sagas that failed with this particular error can now be retried.

{% image "./retry_sagas_2.png", "", [1000] %}

A `sagas.py` file, that was shared across all services, declared various functions that could be used to manage sagas. The reason the code in `sagas.py` could be shared across services was that the services were using the same saga library. The function for retrieving the set of failed sagas is very simple. It simply fetches the data from a database table and loads it into a pandas dataframe. Here's an excerpt from the `sagas.py` file:

```python
import pandas as pd
from utils import db

def get_failed(db_name):
    query = f"""select * from supervisor_view where completed=false and error_message is not null"""
    result = pd.read_sql(query, db.sqlalchemy_connect(db_name))
    return result
```


The `transfer_service.py` file called this function:

```python 
db_name = 'transferservice'

def get_failed_sagas():
    return sagas.get_failed(db_name)
```

Let's say you needed more domain insights. As an example, it could be necessary to find out the amount of funds that had been withheld due to the stuck transfers. This can be achieved by looking up details from the transfer aggregates.

*In this case the ID of the transfer aggregates are the same as the corresponding sagaID (there is one saga for each aggregate), and to get the state of an aggregate a function called `state_apply` can be called which will return one row for each aggregate. The name state_apply comes from the fact that we were using event sourcing to store the state of the aggregates so getting the state of an aggregate implies folding the events of the aggregate event stream. We'll not go into more detail about this here.*

{% image "./amount.png", "", [900] %}

Similary data can be fetched from other data sources, e.g. application logs, and because of the `everything is a dataframe` principle the data can be joined, grouped, aggregated and plotted freely.

## Notebooks

Pycharm will open the notebook view for files of type `ipynb` and will automatically start a local jupyter notebook server for you. Notebooks has a sequence of cells that contain the python code. If the python code in a cell returns anything or prints to standard out it will be printed in the notebook after the cell. Notebooks are very useful when the run script is complex or requires some diagnosis/analysis before the fix can be determined and applied, because the notebook "tells a story".

In the example given in the notebook below a saga got stuck because something unexpected happened, i.e. a service returned an error that the saga could not handle. In this case the easiest way to solve the issue was to do a manual corrective action, i.e. transfer some funds, and then force complete the saga. I.e. instead of trying to extend the saga implementation to be able to handle the pathological case the engineer takes over the responsibility of the specific flow from the saga by doing a manual action and then completing the saga.

{% image "./nb_top.png", "", [1000] %}
{% image "./nb_bottom.png", "", [1000] %}

Since this case involves transferring of funds it might be a good idea to have the script reviewed by a peer before executing it. That can be done by executing the notebook and answering yes in the input dialog that appears when the `dry_run.ask()` statement is executed. That way all cells will run and intended side effects will get printed as output but no side effects are executed. A PR containing the notebook output can then be reviewed by a peer before the notebook is finally executed with dry run disabled.

## Integrations

The backend APIs all required valid JWTs. Obtaining a valid jwt for the active user was fairly simple since the company was using okta. We defined a function `get_access_token` which would initiate a device authorization flow and fetch a new jwt token via the okta api, unless a valid token was already found locally. This function was used anywhere we needed a jwt token:

```python
  headers['authorization'] = f"Bearer {get_access_token()}"
```

Communication with the graphql APIs was done by forwarding to the remote port on kubernetes using the python kubernetes sdk. A configuration file `services.json` contained the names of all the backend services and a small script `generate_graphql.sh` could be used to generate client code for the graphql APIs of all the services.

Communication with the service databases was also achieved using port forwarding. Credentials were obtained via the AWS RDS python client. If the aws credentials of the user had expired the python code would automatically initiate a login flow for the user in the browser by running `aws sso login`.

## Caveats

The solution depicted in this post, the interactive console and the notebooks, works so well because the language supports it. Most dynamically typed languages are well suited but even some statically typed languages, e.g. Scala, has similar features. However, your backend team might be using a language where this sort of tooling is difficult to build. In our case the language in use was not python so building and using the tooling required team members to learn python at a basic level, including parts of the standard library (date time and string manipulation etc.). Understanding and using pandas dataframes is also not trivial for engineers with no experience in the python language or data analysis.

Another point is that interacting directly with database tables obviously breaks encapsulation, meaning that the scripts are likely to be quite brittle. This is not a huge concern as the scripts are part of run work, i.e. by definition a temporary solution to a temporary problem, so the code is expected to have a short timespan after which it is no longer needed. In addition, if the code breaks it is of less criticality as it is run manually by an engineer. Still it can be a challenge to keep the code working over time, and when a critical problem arises it can be very inconvenient if code needs to be refactored before a fix can be applied.
