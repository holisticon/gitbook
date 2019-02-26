# Design Principles: put your code into the right place

Sometimes it is rather difficult to write programs using frameworks if you don't understand the original intent of the authors. This can happen if you have a different technical background and are missing experiences in solving a specific type of problems. In the following I’ll summarize some important patterns, rules and anti-patterns, I learned during my study of CQRS in general and AxonFramework in particular.

## Patterns / Intended Usage

This section consists of a short list of useful patterns or design principles which should be applied during system construction. These are missing in the original AxonFramework documentation and are probably obvious for the advanced users but often got wrong by the beginners.

### Every state change or request is a command.

Every intent of a state change in the application is encapsulated in a **Command.** The command is usually a POJO, containing the information to execute the intended change. A **Command** is created and sent to a special component called **Command Bus**, which is responsible for delivering it to a corresponding **Command Handler**.

AxonFramework provides different **Command Bus** implementations and allows for annotating regular methods with `@CommandHandler` annotation, so the commands get delivered. In order to abstract from that, you may use the **Command Gateway**.

### Create on check fail command pattern.

Imagine a Command Handler that is annotated on the constructor of an aggregate \(pretty straight forward\). In your scenario you want to make sure, an aggregate exists, so on some client code you send the command to the **Command Gateway**. The problem is, that if the aggregate already exists, it will not create a second one but throw an exception \(unique constraint violation\).

You may consider using the following pattern. Create a second `CheckCommand` that has a corresponding `@CommandHandler` inside the same aggregate. Now send the `CheckCommand` with the same `@TargetAggregateIdentifier` and, if no aggregate can be found, send the `CreateCommand` in the fail block. The code might look like this \(Spring Stack\):

```text
class Client {

   @Autowired
   CommandGateway gateway;

   void checkOrCreate(CreateCommand command) {
       gateway.send(new CheckCommand(command.getId()), new CommandCallback<CheckCommand, Object>() {
           public void onSuccess(CommandMessage<CheckCommand> c, Object result) {
           }
           public void onFailure(CommandMessage<CheckCommand> c, Throwable cause) {
               gateway.send(command);
           }               
       });
   }
}
```

### The entire state is stored in Aggregate.

The [**Aggregate**](https://martinfowler.com/bliki/DDD_Aggregate.html) is a term invented by the DDD guys and denote the unit in your software responsible for holding the state. Generally, there are different ways to store the state of an aggregate, but the interesting one is to use [**Event Sourcing**](https://martinfowler.com/eaaDev/EventSourcing.html). This focuses on a storage of the entire history of events instead of the last state. Aggregate root is the only entry point to the Aggregate - this should be able to receive Commands and coordinates the state change.

In AxonFramework, you can simply annotate a class with `@Aggregate` and therby denote it as the Aggregate Root. Usually the **Command Handler** is implemented inside of the **Aggregate \(Root\)** class. The Aggregate denotes a unique business entity, which is identified by an **Aggregate Identifier**. This means, that for given Aggregate Identifier there may be exactly one or no aggregate at place.

### Communicate state change by events.

Every change of the state inside of the **Aggregate** that is relevant outside of the aggregate should be communicated by the corresponding events. In case of **Event Sourcing** the internal state changes should be reflected by events, in order to be able to recover the internal state based on the event stream.

### Command Handler may emit events.

The Command is used to indicate the intent of state change inside of the Aggregate. Essentially, an **Aggregate** transforms the received **Command** into one or multiple **Events**, which are send to **Event Bus**. Events are then delivered to **Event Listeners** \(`@EventHandler`\), which may reflect the state change. The Aggregate itself may contain so-called **Event Sourcing Listener**\(`@EventSourcingHandler`\), to react on **Events,** it fires and change its own state. The Query part of the application registers **Event Listeners** to get informed about the state change.

### Saga is a transaction coordinator responsible for orchestration.

A term **Saga** is a SOA Term for a business transaction. It is adopted by CQRS to denote an orchestration of multiple events. Usually, a Saga is special class providing multiple **Saga Event Listeners** and reacting to certain state changes by sending Commands. Saga keeps track of internal state of the transaction it coordinates.

In AxonFramework, a Saga is a simple class with one or multiple `@SagaEventListener` methods. The entry point into Saga is annotated with `@StartSaga` annotation. A Saga is assigned to a particular business entity identified by a property of the Event it is started from. This means that the transaction gets assigned to business context by this assignment and therefor there can be multiple running Sagas in the same time - each to its own business context.

### Distribute using Event Bus.

If you have to distribute your application, the easiest place to do so is along the **Event Bus**. Generally, event listeners are de-coupled from the event source \(Command handlers in the Aggregate\), so it is a good idea to distribute there. There are several implementations of event buses available for different messaging technologies, e.g. AMQP, JMS or others.

### Use Tracking processors for replay-aware event listeners.

There are two ways how **Event Listeners** receive events.

**Monitoring Event Listeners** get informed synchronously on Event receipt by the Event Bus. This means that even when using messaging like Rabbit MQ, the event is _sent_ synchronously. If the Event Listener was not available, as the Event occurred \(system component was e.g. down\), the event will not get re-delivered to it \(of course your messaging can provide QoS and delivery guarantees, but this is beyond of AxonFramework\). Hense, the event will be lost.

In contrast to Monitoring Event Listeners explained above **Tracking Event Listeners** get informed asynchronously by polling the **Event Store**. If the Event Listener misses the event, the event will get automatically delivered after it recovers. This happens because AxonFramework maintains the information about the Events delivered to every **Tracking Event Listener**. If you rely on at least-one delivery **Tracking Event Listeners** is your choice.

AxonFramework offers methods to control the so-called **Processor** to be **Monitoring** or **Tracking**. A processor denotes a group of event listeners - it defaults to the package of event listener, but can be named random \(the business context is a good idea\).

## Anti-patterns

A short list of don't and smells which are not always impossible to implement, but will probably break the design of your system.

### The state change MUST NOT be executed inside of the CommandHandler, but should be delegated to EventSourcingListener.

#### Related Principles: Separation of Concerns

The reason for this is simple – an aggregate may be destroyed and restored at some point of time. Restore means a replay of events relevant for the current aggregate \(Commands are not replayed\), so if the state change has been executed during command handling it will not be restored.

### The EventListener SHOULD NOT fire Events, if using Event Sourcing and Tracking Processors.

#### Related Principles: Separation of Concerns

An EventListener is executed on arrival of a certain Event. If you are using Event Sourcing, events are first class citizen: they are stored in an Event Store and used for replay. Replay is a process of state recovery which is performed by emitting the stored events in their original order. Imagine the listener is registered for event A and emits B and C. On replay, it will first replay A, which will cause the event listener to emit B and C again and then replay B and C from the previous run. Luckily, AxonFramework prevents the events from being published twice, so nothing really bad happes, but it is a smell.

From the design perspective there should be no reason to send events on event receive. If you intend to execute a state change on receipt of events, think of sending commands and use Sagas for this.

### The EventListener SHOULD NOT send Commands, if using Event Sourcing and Tracking Processors.

#### Related Principles: Separation of Concerns

For the same reason as with events, it is a smell to send commands from event listeners. Event listeners are used to react on EXTERNAL change and reflect is in term of an INTERNAL state. CQRS fosters strong Separation of Concerns. If instead of internal state change, you want to coordinate external activities, move this logic to separate place. The concept used for this is called Sagas and represent externalized orchrestration logic \(some people call it Transaction Manager\). In some situations, you may want to do so if your Aggregate gets more complex, but try to avoid this.

### Saga SHOULD NOT coordinate activities for a single aggregate.

#### Related Principles: Information Hiding

A Saga should coordinate activities between DIFFERENT aggregates. If you have a complex Aggregate and there is some activity needs to be coordinated inside of it, try to put the logic into the Aggregate itself.

