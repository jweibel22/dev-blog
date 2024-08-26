---
title: Reasons to use a saga library
date: 2024-09-27
tags:
  - Microservices
---

When building a backend using the microservices architecture you'll most likely find yourself in a situation where you need to execute a transaction or a business workflow that spans multiple services. Since each service has its own data store maintaining consistency is non trivial, if one of the individual transactions that make up the distributed transaction fails it is necessary to roll back everything to get back to a consistent state. A few data stores provides distributed ACID transactions via 2PC (two phase commits) but 2PC are notoriously difficult to work with, breaks service encapsulation and can have a serious impact on performance due to lock congestion. For these reasons it is common practice to embrace eventual consistency and implement distributed transactions using the saga pattern.

{% note 'Terminology' %}

In the definition of sagas used by [Chris Richardson]((https://microservices.io/patterns/data/saga.html)) (and others) sagas can either be orchestrated or choreographed. The implementation of a choreographed saga is distributed amongst the participating services, each service is subscribing to a set of events and will reactively execute side effects and possibly publish new events to the event bus that will move the process forward. This is an event driven approach that fosters a loose coupling between the services. Conversely an orchestrated saga is implemented by a single service that has the responsibility of keeping track of the state of the process, making calls to other services in a sequence of steps. This can result in a more tightly coupled architecture but the fact that the responsibility of the flow is anchored inside a single service makes it easier to debug and operate the sagas. Both approaches have their merits but this particular post is about orchestrating sagas.

Sagas are often referred to as [process managers](https://learn.microsoft.com/en-us/previous-versions/msp-n-p/jj591569(v=pandp.10)?redirectedfrom=MSDN#what-is-a-process-manager) in the DDD community. Introducing a process manager into an architecture is a decision to model the process explicitly using the process manager rather than having the process modelled implicitly by having the aggregates communicate directly via commands and events. In the terminology of Chris Richardson you can therefore think of the process manager as an orchestrating saga.

{% endnote %}

## Sagas as first class citizens

I was once working on a project where our aggregates were modelled using event sourcing and sagas were implemented as any other aggregate. Often an aggregate representing a business entity would also act as a saga, taking responsibility of calling other aggregates. The remote calls were carried out by event handlers on the domain events of the saga and  when the remote call finished a new domain event was written to the event store causing the next event handler to be triggered. There were a few problems with this approach.

First of all an orchestrating saga is essentially a DAG consisting of the set of tasks that needs to be taken and ideally the code should reflect this. However in the approach depicted above the DAG was essentially modelled using reactive code, each task being represented by an event handler, and progressing to the next step was done by writing an event which then reactively triggered the next task. This impedance mismatch, the misalignment between what is being modelled and how it is represented in the code made it hard to read. It was quite difficult to get a holistic overview of the DAG because it required jumping around the code base between different events and event handlers. One symptom of this was that the team were maintaining UML diagrams of all the sagas that were implemented. These diagrams were necessary to keep up to date to help the developers understand the workflow, because it was simply too difficult (time consuming) to obtain a mental image of the workflow by reading the code itself.

Secondly since the business entity and the saga was modelled by the same object it became difficult to separate orchestration problems from business problems. E.g. some remote calls were async which meant that it was necessary to write two events related to the call to the event store. First an event representing the intention to make the call and the following representing the result of the call. Obviously this meant that if we needed to make changes to the orchestration for technical reasons, replacing an async API with a sync API, implementing the change would involving making changes to the event stream which was also used to represent the state of the business entity.

Another problem was how to handle event replay. Replaying events is a common practice in event sourcing which allows e.g. to rebuild views that are based on the events from the event store, however since the saga actions were implemented as event handlers on those events this could result in unwanted side effects as already completed sagas would then start making remote calls.

Lastly implementing orchestrating sagas is non trivial. It involves keeping track of the saga state, implementing retries and keeping track of stuck or failed sagas and we discovered that operating the sagas involved many commonalities. A saga could get stuck due to some remote system not responding or an expected event not arriving, they could end up in an unexpected situation from which they couldn't progress needing human assistance. We found ourselves implementing the same  patterns over and over again, e.g. implementing observability and endpoints that allowed us to resume a stuck saga.

In the end we decided to model sagas explicitly and keep all orchestration responsibilities out of the business entities. Since modelling the sagas were now independent of the domain logic it allowed us to implement a library providing generic saga functionality that could be reused across domains. In the following sections I'll go through some of this common functionality.

### Observability

Keeping track of failed or stuck sagas is obviously quite important. The saga library maintained a database table listing all sagas, a row in the table showed the active task and a lastUpdated timestamp for a saga. Failed sagas were sagas that had ended up in the special `FailWithUnknownError` state and stuck sagas were sagas that had not made a state change in a while. The information in this table was used as the basis for prometheus metrics that reported on stuck and failed transfers and once alerted we could consult this table to get an overview of which sagas were having issues.

The saga library was built using an in-house event sourcing library, that choice was not so much based on an opinion that modelling sagas using event sourcing was a good approach, but more on the fact that it was commonly used and we had a lot of experience with it. Using an event sourcing library meant that the entire saga history was automatically saved in the event store. This meant that in a debug scenario, apart from consulting the logs, if more details were needed regarding what data was sent or IDs used the saga event history could be consulted.

### Ops

Many different types of orchestration related errors can happen in production. Some of them may cause a saga to become stuck or take wrong decisions and it must be possible to rectify these situations. E.g. a bug in the saga code, e.g. it was not able to recognize a valid response code from an API, could cause it to end up in `FailWithUnknownError`. In this case the corrective action would be to fix the bug in the saga code and move the saga back to the task that failed to execute. Of course the bug that caused the saga to misbehave could also originate from one of the remote systems that the saga interacts with. This could e.g. cause an API call to fail unexpectedly making the saga jump to the special `FailWithUnknownError` state. Again this must be fixed by fixing the bug in the remote system and move the saga back to the task that failed to execute.

Much more complicated issues can arise. E.g. a bug can cause the saga to take wrong decisions and make incorrect remote calls, causing the distributed system to end up in an inconsistent state. In such situations it is often better to simply force complete the saga, and have a developer take over the responsibility for the state normally managed by the saga. If the APIs of the remote systems are available to be called manually the developer can make the corrective actions manually. The alternative, to enable the saga to get out of the situation by coding in the required flow will add complexity to the saga but since the situation was caused by a bug it is unlikely to ever end up in the same situation again so this part of the saga will not be useful later and only cause confusion to future developers who may not be familiar with the particular incident and understand why that particular behaviour is implemented.

A `mission control` API could be used to manually control the saga behaviour at runtime. Here we list some of the available methods.

- **Next Task**: Move the saga with the specified type and ID to the specified task.
- **Force Complete**: Force completes the saga.
- **Retry Task**: A saga is stuck in a task (it never moved to the next task, this can happen when using an `asyncTask`) and we'd like to manually retry it.
- **Update Data**: A bug in the saga code caused it to write some wrong data to its state. This method can be used to mutate the saga data.

The methods were exposed in our run script setup which has been described here <a href="/blog/reducing_run/">Running fast</a> This allowed us e.g. to quickly handle cases where many sagas had failed or gotten stuck due to the same reason, this was a simple matter of looping through the sagas in the saga overview table and calling the mission control function.

### Versioning

It is quite common to end up in a situation where it is necessary to make breaking changes to the saga flow. In this context breaking changes means a change to the saga flow that is incompatible with the existing flow, which means, if code changes were to be deployed while sagas using the old saga implementation were in-flight this would cause wrong behaviour. E.g.  changing the order of tasks or replacing one remote API with a another API could in some situations be breaking changes.

We supported versioning by attaching a version property to a DAG and allowing to register multiple DAGs of different versions to the same saga. A saga was triggered by the `sagas.Start` method which takes in a version parameter. To make breaking changes to a saga the developer workflow was to create a new version of the DAG and register it on the saga, change the code that starts the saga to use the new version, wait for all sagas that were based on the deprecated DAG to complete in production and finally remove the old DAG from the code.

If we had not built or used a saga framework then the saga implementations would have been entangled with the domain specific business logic. This means that figuring out how to make a breaking change to a saga flow would be a domain specific problem and would thus have to solved, in a new way, every time the problem arose.

### Postponement

Sometimes our sagas needed to wait for a period of time before continuing. This was also turned into a feature in the saga library that allowed a task to postpone itself for a specified period of time.

## The code

This post is not really a presentation of the specific library, but for completeness I'll show some examples.

In the below example we're constructing a saga that will carry out some action (e.g. call some remote system) and if it fails it will carry out some compensating action. The example is a bit contrived, normally you wouldn't need to carry out a compensating action unless some previous actions actually succeeded, however to keep the example code short we skipped that part. The `done` task is a task provided by the saga library that will simply complete the saga.

```go

  type SagaData struct {
    RememberThis string
  }

  var (
    sagaName            = types.Name("mysaga")
    saga                = factory.NewSaga(sagaName)
    someAction          = tasks.NewSomeAction()
    compensatingAction  = tasks.NewCompensatingAction()
    done                = commontasks.NewComplete(saga)
  )

  dag := sagas.NewDag[SagaData](types.UnspecifiedVersion)
  dag.StartFrom(someAction).
    Transitions(
      someAction.OnSuccess.GoTo(done),
      someAction.OnFailure.GoTo(compensatingAction),
      compensatingAction.OnSuccess.GoTo(done),
  )

  saga.RegisterDag(dag)
```

Notice how the DAG is explicitly expressed in the code. Also notice that each task is implemented as a separate type. We keep each task in a separate go file and this means that it is easy to get an overview of which tasks exist when looking at the code in the tree navigator. It becomes immediately apparent to the reader that there is a concept called a saga and a task when looking at the folder structure, which was certainly not the case previously where the saga and tasks were buried within events and event handlers. This is an example of applying the principle of [screaming architectures](https://blog.cleancoder.com/uncle-bob/2011/09/30/Screaming-Architecture.html)

Also notice how the saga and DAG construction is separated. A DAG is constructed and registered on the saga. We will get back to this when we talk about versioning in a later section.

Next let's take a look at how the tasks are implemented. The `someAction` task declares two connectors, OnSuccess and OnFailure. In the implementation of Execute it will, based on the outcome of the side effect that is executed, decide where to go next, to the OnSuccess or to the OnFailure task. The connectors has two type arguments, one being the SagaData type which must correspond the SagaData type on the saga and an Args type which is the type of arguments that the task can take. The connectors allow us to reuse a saga task implementation in multiple sagas by simply plugging in different tasks in the connectors when constructing the DAGs.

```go
type someAction struct {
  *sagas.Task[sagas.EmptyArgs, SagaData]

  OnSuccess *sagas.Connector[sagas.EmptyArgs, SagaData]
  OnFailure *sagas.Connector[sagas.EmptyArgs, SagaData]
}

func NewSomeAction(
) *someAction {
  task := &someAction{
    Task: sagas.NewTask[sagas.EmptyArgs, SagaData](),
  }
  task.OnSuccess = sagas.NewConnector[sagas.EmptyArgs, SagaData](task)
  task.OnFailure = sagas.NewConnector[sagas.EmptyArgs, SagaData](task)

  return task
}

func (i *someAction) Execute(ctx context.Context) sagas.Decider[sagas.EmptyArgs, SagaData] {
  return func(_ sagas.EmptyArgs, state *saga.State) (saga.Command, error) {
    if rand.Intn(100) < 50 {
      // well that didn't work, we must follow the rainy day path
      return sagas.Next(i.OnFailure.Resolve(state), args.Empty) 
    }
    // all good, follow the happy path
    return sagas.Next(i.OnSuccess.Resolve(state), args.Empty)
  }
}
```

The `compensatingAction` task has a single connector, OnSuccess, there is no OnFailure connector as there is no expected way it can fail. If it fails for unexpected reasons the task will move the saga into the `FailedWithUnknownError` state. This is a special state that all sagas can end up in. When a saga is in that state it must be moved out of the state manually by calling one of the methods on the `mission control` API that we will discuss in detail in a later section.

The `FailedWithUnknownError` state is used to handle all the situations that cannot be handled automatically, either because they were unexpected and therefore the saga code did not take that scenario into consideration, or because there simply is no supported way to handle the situation automatically. We will discuss this in more detail later.


```go
type compensatingAction struct {
  *sagas.Task[sagas.EmptyArgs, SagaData]

  OnSuccess *sagas.Connector[sagas.EmptyArgs, SagaData]
}

func NewCompensatingAction(
) *compensatingAction {
  task := &compensatingAction{
    Task: sagas.NewTask[sagas.EmptyArgs, SagaData](),
  }
  task.OnSuccess = sagas.NewConnector[sagas.EmptyArgs, SagaData](task)

  return task
}

func (i *compensatingAction) Execute(ctx context.Context) sagas.Decider[sagas.EmptyArgs, SagaData] {
  return func(_ sagas.EmptyArgs, state *saga.State) (saga.Command, error) {
    err := makeRemoteCall()
    if err != nil {
      if errors.Is(err, BadRequest) {
        // something really unexpected happened, I have no idea what to do now!
        return sagas.FailWithUnknownError[SagaData](err.Error())
      }
      // this error is probably retriable
      return sagas.Retry[SagaData](err.Error())
    }    

    return sagas.Next(i.OnSuccess.Resolve(state), args.Empty)
  }
}
```

Notice that if for some reason the saga framework fails to persist the decision about where to go next this will also result in the call to Execute to be automatically retried. Therefore it is essential that the side effect carried out in the Execute method is idempotent.

A task can mutate the saga data. In this example it could set the value of the `RememberThis` property and that value would be available to subsequent tasks. It is also possible to pass on arguments to tasks in the sagas.Next method. In this example the tasks did not declare any arguments so the special value `args.Empty` was used.

## Conclusion

As this post hopefully demonstrates orchestration is a non trivial problem and many of the challenges that arise are of a generic nature. Moving away from a domain specific approach to using a generic library allowed us to save a lot of time and if you have to deal with a lot of orchestration it is something that I can definitely recommend.
