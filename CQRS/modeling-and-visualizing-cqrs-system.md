# Designing, Modeling and Visualizing a CQRS system

Building application using AxonFramework involves software design of different parts of the CQRS/ES (**C**ommand **Q**uery **R**esponsibility **S**egregation using **E**vent **S**ouring) system. AxonFramework provides a perfect abstraction for the entire infrastructure, required to run the application, so the developer may focus on the design of the domain logic.

This means a developer or a team of developers must find correct abstractions for **Command**, **Event** and **Aggregate** in the system and be able communicate, visualize and discuss the resulting design. This article summarizes my thoughts on it. In addition I focus on **Saga** modeling and visualization.

## Static model

A static model of the system represent structural elements: classes, their properties and dependencies between them. Static model focuses on capturing the **What**-aspects of the design which are the main structural building blocks of the system. 

### Command

A Command is a simple data class. It should be immutable and be able to be validated (e.g. applying some type validation like BeanValidation). For this purpose, it is very handy to build Command classes in a language/framework which can guarantee these properties. Think of Scala case classes, Kotlin data classes, Immutables Framework or Project Lombok. 

Commands can be visualized by simple UML class diagrams, denoting only the contained fields. I believe it is a good idea to use domain data types inside the Commands to express the domain intent as precise as possible. For naming Commands I like the verb/object schema "Do thing" like "CreateInvoice" or "DeleteCustomer".

### Event

An Event has similar properties as a Command. It is a simple data class, immutable, but since it is created inside application code (and not by the client) the validation is desired, but not mandatory. 

For Events same rules as for Commands applies. Simple UML class diagrams denoting the properties and usage of the domain data types can be used for modeling and visualization.

### Aggregate

In contrast to Command and Event which represent static immutable _data_, an Aggregate is representing internal state and _behavior_. In doing so, an aggregate class offers basically two types of methods: Command Handlers and Event Sourcing Handlers. A good overview may be a UML class diagram of the aggregate class, displaying the internal state variables and dependencies to other aggregate members. Instead of method listings for all handlers, you can use dependencies from the Aggregates to Command and to Event classes.

### Summary 
In order to display the static overview of the CQRS application, an UML class diagram seems to be useful. Here is an example, capturing Commands, Events and an Aggregate.

![Example UML class diagram](/CQRS/cqrs_class_diagram.PNG)

## Dynamic model

The dynamic model focuses on interaction between different parts of the application. Modeling and visualizing the dynamics of the system is crucial for the development team in order to adequately communicate, discuss and argue about the system.

### Aggregate

Every Aggregate denotes a state machine allowing state transitions triggered by received Commands and emitting Events during those transitions. There are multiple ways to visualize state machines, e.g. UML state diagram.

### Saga

Sagas are responsible for implementing orchestration of state change in CQRS systems. A Saga is used to coordinate different Aggregate state changes to achieve a certain goal and can be seen as a kind of transaction. This is pure dynamics, so the only thing that needs to be modeled about a Saga is its behavior during orchestration of Events and Commands.

My first attempt to model and visualize it was to use UML sequence diagram, focusing on interaction between components. This approach looked very promising and helped me to understand how Saga interacts with Aggregates. In my model I omited the Command Bus, since Commands are usually directed to the Aggregates, but modeled the Event Bus, because of multiple event delivery (An event can be delivered to multiple subscribers). Here is an bigger example from a project called "Ranked" focusing on calculation of scores and ranking of table soccer players:

![Saga message exchange](/CQRS/sequencediagramm.org.png)

In the diagram above you can see how the Aggregate _Match_ is created upon the customer request and how it emit Events to the _Event Bus_. There are two participants interesting in the receipt of these events: the _EloHandlingSaga_ and the _MatchWinningSaga_ which send Commands to _Player_ Aggregates and communicate with further services. You can observe message exchange required for calculation of the new Elo ranking. 

The problem with this model is that it shows exactly one possible interaction between the components. In the depicted example case it focuses on the message exchange required for successful Elo ranking calculation. In terms of flow models there is exactly one (possible) flow depicted here. For some Sagas this may be sufficient, but in general you may be interested in an additional representations. For example, there can be some complex conditions resulting in different flows, errors in execution may require compensational message exchange and so on. 

Since a Saga is a transaction, my second naive idea was to use BPMN to display it. The following example is taken from [Understanding Sagas](/CQRS/understanding-sagas.md).

![BPMN diagram of CQRS Saga](/CQRS/saga_example_extended.png)

BPMN seems to fit better, for presenting the entire behavior of Saga. You can display two aspect in the same diagram simultaneously: exchange of the messages between Saga and the Aggregates and the logic inside of the Saga (e.g. some conditions, etc).

There are some certain patterns of modelling style which needs to be applied to preserve clean and understandable models. Here are some, I've detected so far:

#### Model only flow required for orchestration

Aggregates may get rather complex in real business scenarios regarding the internal state and conditions for state changes. Even if this is relevant for the design of the Aggregate itself, this internal details should be ommited when displaying its interaction with Sagas. In other words, it is ok to model the entire state machine of the Aggregate as a UML state diagram or BPMN, but use _reduced_ model of it depicting only branches and state changes relevant for interaction with Saga. (There is a possibility to skip the details of the Aggregate logic and model messages denoting the Aggregate as a collapsed pool only, but I think this is useful only in simple cases in which the Aggregate provides a very simple state machine.)

        
In the same time 
as soon as you have more then one command handler. The resulting model can distinguish between the first command (where the command handler is a constructor of the aggregate) and all following command, which needs to be modelled as a branch on the by event-based gateway, since commands are 
unconditional in CQRS. 


