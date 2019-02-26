# Understanding Sagas

## Basic Saga concept

Handling orchestration based on several events may seem challenging. In order to ease this, a concept of **Sagas** has been implemented in AxonFramework.

Sagas should be used to coordinate activities between _several_ aggregates. This means that they receive events from one or multiple aggregates and send commands to one or multiple aggregates. In doing so, they maintain the state of the business transaction and ensure its consistency.

In contrast to an **Aggregate** which exists only once for every aggregate identifier (in fact representing the unique domain entity), the Sagas don't have their own identity, but use the identity of the events. 

Technically, a Saga is a simple class containing one or several methods annotated with `@SagaEventHandler` annotation. One of those methods is usually the start of the Saga and is denoted with `@StartSaga`annotation. There can be an `@EndSaga` method, or you may use the `SagaLifecycle` helper class to control it.

This is a typical Saga:

```
class MyBusinessTransaction {

  @Autowired
  CommandBus commandBus;

  @StartSaga
  @SagaEventHandler(associationProperty = "eventId")
  public void on(FirstEvent e) {
    // ...
    String anotherId = createAnotherId();
    associateWith("anotherId", anotherId)
    commandBus.send(SomeCommand(anotherId));
  }


  @SagaEventHandler(associationProperty = "id", keyName = "anotherId")
  public void on(AnotherEvent e) {
    // ...
    commandBus.send(AnotherCommand());
    end();
  }

}

class FirstEvent() {
  String eventId;
  // getters, setters
}

class AnotherEvent() {
  String id;
  // getters, setters
}

```

The resulting coordination looks as following: on receipt of `FirstEvent` start a new Saga, and associate it to the value of the `e.eventId` property. Then send `SomeCommand`. If afterwards an `AnotherEvent` with the same value of `e.id` as the value associated to Saga occurs, send `AnotherCommand` and end the saga execution.

You can put it in BPMN and it will result in:

![Saga Example](/CQRS/saga_example.png)

In the above example I allowed some minor violations in modelling of the current context in BPMN 2.x. AxonFramework events should be modelled as Signals, since they are not directed, but I used the Messages, since this BPMN symbol is more famous, intuitive and clear to people not familiar with the BPMN 2.x specification.

## Saga Association 

From the point of view of a Saga, it doesn't matter how the event sources of `FirstEvent` and `AnotherEvent` know that they have to send the same value in their `eventId` and `id` event fields. But this is an important consideration that has to be made during system construction. Since the usual event source and the place where we put `@CommandHandler` is an **Aggregate**, let's assume that events are not just appearing on Event Bus and the commands are not just sent to Command Bus. I'll extend my example by adding the aggregates responsible for the command processing and event emitting.

```
class FirstAggregate {

  @AggregateIdentifier
  String id;
  
  @CommandHandler
  FirstAggregate(Create c) {
    apply(new FirstEvent(c.getId()))
  }
  
  @EventSourcingHandler
  void on(FirstEvent e) {
    this.id = e.getEventId();
  }
  
  @CommandHandler
  void handle(AnotherCommand c) {
    // do something
  }  
  
  @CommandHandler
  void handle(JustAnotherCommand c) {
    // do something else
  }  
}

class AnotherAggregate {
  @AggregateIdentifier
  String id;
  
  @CommandHandler
  AnotherAggregate(SomeCommand c) {
    apply(new AnotherEvent(c.getId()))
  }
  
  @EventSourcingHandler
  void on(AnotherEvent e) {
    this.id = e.getId();
  }
}

class Create {
  String id;
  // getters, setters  
}

class SomeCommand() {
  String id;
  // getters, setters  
}

```

Let's see how this looks like if added to the diagram. For this purpose, I'll remove the event bus and the command bus, but put the aggregates at place.

![Extended Saga Example](/CQRS/saga_example_extended.png)

The `FirstAggregate` gets created (e.g. by receiving the `Create` command) and emits the `FirstEvent` which starts the Saga. The Saga creates an `AnotherAggregate` by sending the corresponding command, receives the `AnotherEvent` and finally sends the `SomeCommand` to the `FirstAggreggate` and terminates. 

The important question in this picture is, how AxonFramework assigns the events to the correct Saga instance (there may be several Sagas in place for different transactions). The answer is: it uses the **Association Properties**. 

The Saga sent the `SomeCommand` with an id it generated and associated using "anotherId". It listens to an event `AnotherEvent` having the bean property id which contains the value of this remembered id. To deliver this, the Aggregate handling the `SomeCommand` should supply this value in the `AnotherEvent` to target the originated Saga.










