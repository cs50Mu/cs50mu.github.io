+++
title = "「Applying UML and Patterns」读书笔记"
date = "2021-02-05"
slug = "2021/02/05/reading-notes-of-applying-uml-and-patterns"
Categories = ["读书笔记"]
+++

## 「Applying UML and Patterns」读书笔记

### Inception

前期评估

the idea is to do just enough investigation to form a rational, justifiable opinion of the overall purpose and feasibility of the potential new system, and decide if it is worthwhile to invest in deeper exploration

**It is to decide if the project is worth a serious investigation (during elaboration), not to do that investigation.**

#### Use Cases

Informally, use cases are text stories of some actor using a system to meet goals.

the essence is discovering and recording functional requirements by writing stories of using a system to fulfill user goals; that is, cases of use

### Elaboration Iteration 1Basics - Domain Models

#### What is a Domain Model?

A domain model is a visual representation of conceptual classes or real-situation objects in a domain. 

Domain models have also been called conceptual models (the term used in the first edition of this book), domain object models, and analysis object models.

Applying UML notation, a domain model is illustrated **with a set of class diagrams in which no operations (method signatures) are defined.** It provides a conceptual perspective. It may show:

- domain objects or conceptual classes
- associations between conceptual classes
- attributes of conceptual classes

A domain model shows real-situation conceptual classes, **not software classes.**

#### How to Find Conceptual Classes?

- Reuse or modify existing models.
- Use a category list. 一些默认建议，非常好
- Finding Conceptual Classes with Noun Phrase Identification. 通过名词来找灵感
    - **use cases** are one rich source to mine for noun phrase identification.

> Perhaps the most common mistake when creating a domain model is to represent something as an attribute when it should have been a conceptual class.

If we do not think of some conceptual class X as a number or text in the real world, X is probably a conceptual class, not an attribute. 类比现实世界

#### Why Use 'Description' Classes?

![](/images/post/description-class.jpg)

item的描述是一直存在的，但item可能会卖光

The need for description classes is common in sales, product, and service domains. It is also common in manufacturing, which requires a description of a manufactured thing that is distinct from the thing itself.

#### Associations

An association is a relationship between classes (more precisely, instances of those classes) that indicates some meaningful and interesting connection

Multiplicity defines how many instances of a class A can be associated with one instance of a class B

![](/images/post/association-multiplicity.jpg)

#### Attributes

An attribute is a logical data value of an object.

Informally, most attribute types should be what are often thought of as "primitive" data types, such as numbers and booleans. The type of an attribute should not normally be a complex domain concept, such as a Sale or Airport. 类属性应该是基础类型，不应该是复杂的自定义领域类型（这种应该用Association来表达）

Relate conceptual classes with an association, not with an attribute.

#### Conclusion: Is the Domain Model Correct?

There is no such thing as a single correct domain model. All models are approximations of the domain we are attempting to understand; the domain model is primarily a tool of understanding and communication among a particular group. A useful domain model captures the essential abstractions and information required to understand the domain in the context of the current requirements, and aids people in understanding the domainits concepts, terminology, and relationships. 没有“正确”的模型

### Elaboration Iteration 1Basics - System Sequence Diagrams

#### What are System Sequence Diagrams?

A system sequence diagram is a picture that shows, for one particular scenario of a use case, the events that external actors generate, their order, and inter-system events. All systems are treated as a **black box**; the emphasis of the diagram is events that cross the system boundary from actors to systems.

System behavior is a description of what a system does, without explaining how it does it.  用于描述系统是做什么的，而不是如何做

### Elaboration Iteration 1Basics - Operation Contracts


### Elaboration Iteration 1Basics - Requirements to DesignIteratively

Iteratively Do the Right Thing, Do the Thing Right 做对的事 和 把事做对

The requirements and object-oriented analysis has focused on learning to do the right thing; that is, understanding some of the outstanding goals for the case studies, and related rules and constraints. By contrast, the following design work will stress do the thing right; that is, skillfully designing a solution to satisfy the requirements for this iteration.

### Elaboration Iteration 1Basics - Logical Architecture and UML Package Diagrams

#### What is the Logical Architecture? And Layers?

The logical architecture is the large-scale organization of the software classes into packages (or namespaces), subsystems, and layers. **It's called the logical architecture because there's no decision about how these elements are deployed across different operating system processes or across physical computers in a network**

A layer is a very coarse-grained grouping of classes, packages, or subsystems that has cohesive responsibility for a major aspect of the system. Also, layers are organized such that "higher" layers (such as the UI layer) call upon services of "lower" layers, but not normally vice versa.

Typically layers in an OO system include:

- User Interface.
- Application Logic and Domain Objects 领域层
- Technical Services 
    - general purpose objects and subsystems that provide supporting technical services, such as interfacing with a database or error logging.
    - These services are usually application-independent and reusable across several systems.

> Guideline: Design with Layers

Applied to information systems, **typical layers** are illustrated and explained as follows:

- UI
- Application
    - handles presentation layer request
    - workflow
    - session state
- Domain (aka Business, Application logic)
    - implementation of domain rules
- Business Infrasture
    - very general low-level business services
    - used in many business domains
    - eg, CurrencyConverter
- Technial Services
    - (relatively) high-level technical services and frameworks
    - eg, persistence, security
- Foundation
    - low-level technical services, utilities and frameworks
    - eg, data structures, threads, math, file, DB and network I/O

> Guideline: Cohesive Responsibilities; Maintain a Separation of Concerns

每层干自己该干的事

The responsibilities of the objects in a layer should be strongly related to each other and should not be mixed with responsibilities of other layers. For example, objects in the UI layer should focus on UI work, such as creating windows and widgets, capturing mouse and keyboard events, and so forth. Objects in the application logic or "domain" layer should focus on application logic, such as calculating a sales total or taxes, or moving a piece on a game board.

UI objects should not do application logic. For example, a Java Swing JFrame (window) object should not contain logic to calculate taxes or move a game piece. And on the other hand, application logic classes should not trap UI mouse or keyboard events. That would violate a clear separation of concerns and maintaining high cohesion basic architectural principles.

> How do we design the application logic with objects?

To create software objects with names and information similar to the real-world domain, and **assign application logic responsibilities to them**. For example, in the real world of POS, there are sales and payments. So, in software, we create a Sale and Payment class, and give them application logic responsibilities. 

> Guideline: The Model-View Separation Principle

In this context, model is a synonym for the domain layer of objects (it's an old OO term from the late 1970s). View is a synonym for UI objects, such as windows, Web pages, applets, and reports.

**The Model-View Separation principle[2] states that model (domain) objects should not have direct knowledge of view (UI) objects, at least as view objects**. So, for example, a Register or Sale object should not directly send a message to a GUI window object ProcessSaleFrame, asking it to display something, change color, close, and so forth.

Model-View-Controller (MVC): The Model is the Domain Layer, the View is the UI Layer, and the Controllers are the workflow objects in the Application layer. 至今看到的对MVC的最完美的诠释吧

### Elaboration Iteration 1Basics - On to Object Design

#### Designing Objects: What are Static and Dynamic Modeling?

There are two kinds of object models: dynamic and static. **Dynamic models, such as UML interaction diagrams (sequence diagrams or communication diagrams), help design the logic, the behavior of the code or the method bodies.** They tend to be the more interesting, difficult, important diagrams to create. **Static models, such as UML class diagrams, help design the definition of packages, class names, attributes, and method signatures (but not method bodies).**

People new to UML tend to think that the important diagram is the static-view class diagram, but in fact, most of the challenging, interesting, useful design work happens while drawing the UML dynamic-view interaction diagrams. It's during dynamic object modeling (such as drawing sequence diagrams) that "the rubber hits the road" in terms of really thinking through the exact details of what objects need to exist and how they collaborate via messages and methods. 重点：先做Dynamic Modeling（新手会误认为要先做Static Modeling）

What's important is knowing how to think and design in objects, and apply object design best-practice patterns, which is a very different and much more valuable skill than knowing UML notation. 重点是OO设计，而不是UML

### Elaboration Iteration 1Basics - UML Interaction Diagrams

The term interaction diagram is a generalization of two more specialized UML diagram types:

- sequence diagrams
    - illustrate interactions in a kind of fence format, in which each new object is added to the right
- communication diagrams
    - illustrate object interactions in a graph or network format, in which objects can be placed anywhere on the diagram

![](/images/post/sequence-diagram.jpg)
![](/images/post/communication-diagram.jpg)

### Elaboration Iteration 1Basics - UML Class Diagrams


### Elaboration Iteration 1Basics - GRASP: Designing Objects with Responsibilities

**The critical design tool for software development is a mind well educated in design principles.** It is not the UML or any other technology.
 
> What Are Inputs to Object Design?

开始设计的前提是已做好需求分析

> Responsibilities and Responsibility-Driven Design

Basically, these responsibilities are of the following two types: doing and knowing.

Doing responsibilities of an object include:

- doing something itself, such as creating an object or doing a calculation
- initiating action in other objects
- controlling and coordinating activities in other objects

Knowing responsibilities of an object include:

- knowing about private encapsulated data
- knowing about related objects
- knowing about things it can derive or calculate

> What are Patterns?

In OO design, a pattern is a named description of a problem and solution that can be applied to new contexts; ideally, a pattern advises us on how to apply its solution in varying circumstances and considers the forces and trade-offs.

New pattern should be considered an oxymoron if it describes a new idea. The very term "pattern" suggests a long-repeating thing. **The point of design patterns is not to express new design ideas. Quite the oppositegreat patterns attempt to codify existing tried-and-true knowledge, idioms, and principles**; the more honed, old, and widely used, the better.

> A Short Example of Object Design with GRASP

There are nine GRASP patterns:

- Creator

关键词：mental model、low representational gap (LRG)

原则是要尽量降低人的心理模型跟design model之间的认知差异

- Information Expert

Problem： What is a basic principle by which to assign responsibilities to objects?

Solution： Assign a responsibility to the class that has the information needed to fulfill it.

- Low Coupling

Briefly and informally, coupling is a measure of how strongly one element is connected to, has knowledge of, or depends on other elements.

If there is coupling or dependency, then when the depended-upon element changes, the dependant may be affected. For example, a subclass is strongly coupled to a superclass. An object A that calls on the operations of object B has coupling to B's services.

It is not high coupling per se that is the problem; it is high coupling to elements that are unstable in some dimension, such as their interface, implementation, or mere presence.  不是耦合本身不好，而要看耦合的东西，若耦合的东西不稳定就不太好

- Controller

Problem：What first object beyond the UI layer receives and coordinates ("controls") a system operation?

Solution： Assign the responsibility to an object representing one of these choices:

1. Represents the overall "system," a "root object," a device that the software is running within, or a major subsystem (these are all variations of a facade controller).
2. Represents a use case scenario within which the system operation occurs (a use case or session controller)

有两种：

Facade controllers：门面，一切全包，比如TelecommSwitch、Phone、ChessGame

use case controller：按使用场景来分的，比如ProcessSaleHandler

- High Cohesion

In software design a basic quality known as cohesion informally measures how **functionally related** the operations of a software element are, and also measures how much work a software element is doing. 做的事要有相关性，一个类的方法集中如果有很多关系不大的事情，则这个类不是cohesive的

As a simple contrasting example, an object Big with 100 methods and 2,000 source lines of code (SLOC) is doing a lot more than an object Small with 10 methods and 200 source lines. **And if the 100 methods of Big are covering many different areas of responsibility (such as database access and random number generation), then Big has less focus or functional cohesion than Small.** In summary, both the amount of code and the relatedness of the code are an indicator of an object's cohesion.

Problem: How to keep objects focused, understandable, and manageable, and as a side effect, support Low Coupling?

Solution: Assign responsibilities so that cohesion remains high. Use this to evaluate alternatives.

### Elaboration Iteration 1Basics - Object Design Examples with GRASP

I wish to exhaustively illustrate that no "magic" is needed in object design

OO software design really can be **more science than art**, though there is plenty of room for creativity and elegant design.

实例讲解，非常详尽！

> The Command-Query Separation Principle

命令（有side effect的操作，比如update create等）与 查询 分离的原则

要这样：

```java
// style #1; used in the official solution
public void roll()
{
faceValue = // random num generation
}
public int getFaceValue() {
   return faceValue;
}
```
而不是这样：

```java
// style #2; why is this poor?
public int roll()
{
faceValue = // random num generation
   return faceValue;
}
```

为什么呢？

CQS is widely considered desirable in computer science theory **because with it, you can more easily reason about a program's state without simultaneously modifying that state. And it makes designs simpler to understand and anticipate.** For example, if an application consistently follows CQS, you know that a query or getter method isn't going to modify anything and a command isn't going to return anything. Simple pattern. This often turns out to be nice to rely on, as the alternative can be a nasty surpriseviolating the Principle of **Least Surprise** in software development

### Elaboration Iteration 1Basics - Designing for Visibility

Visibility is the ability of one object to see or have reference to another

There are four common ways that visibility can be achieved from object A to object B:

- Attribute visibility B is an attribute of A.
- Parameter visibility B is a parameter of a method of A.
- Local visibility B is a (non-parameter) local object in a method of A.
- Global visibility B is in some way globally visible.

### Elaboration Iteration 1Basics - Mapping Designs to Code

### Elaboration Iteration 1Basics - Test-Driven Development and Refactoring

Refactoring [Fowler99] is a structured, disciplined method to rewrite or restructure existing code without changing its external behavior, applying small transformation steps combined with re- executing tests each step.

### Elaboration Iteration 2 More Patterns - UML Tools and UML as Blueprint

### Elaboration Iteration 2 More Patterns - Quick Analysis Update

### Elaboration Iteration 2 More Patterns - Iteration 2More Patterns

### Elaboration Iteration 2 More Patterns - More Objects with Responsibilities

GRASP patterns:

- Polymorphism

多态

Alternatives based on type Conditional variation is a fundamental theme in programs. If a program is designed using if-then-else or case statement conditional logic, then if a new variation arises, it requires modification of the case logicoften in many places. This approach makes it difficult to easily extend a program with new variations because changes tend to be required in several placeswherever the conditional logic exists. if-else 问题

原则：不要用if-else或者case来解决，而是用多态来解决

- Pure Fabrication

Object-oriented designs are sometimes characterized by implementing as software classes representations of concepts in the real-world problem domain to lower the representational gap

Problem: What object should have the responsibility, when you do not want to violate High Cohesion and Low Coupling, or other goals, but solutions offered by Expert (for example) are not appropriate? there are many situations in which assigning responsibilities only to domain layer software classes leads to problems in terms of poor cohesion or coupling, or low reuse potential. 没办法跟Domain model（现实模型）对应上的object怎么设计？

Solution: Assign a highly cohesive set of responsibilities to **an artificial or convenience class that does not represent a problem domain concept something made up**, to support high cohesion, low coupling, and reuse.

- Indirection

Most problems in computer science can be solved by another level of indirection

- Protected Variations

Problem: How to design objects, subsystems, and systems so that the variations or instability in these elements does not have an undesirable impact on other elements?

Solution: Identify points of predicted variation or instability; **assign responsibilities to create a stable interface around them**.

虚拟机也是一个应用这个原则的例子

> The Liskov Substitution Principle (LSP)

LSP [Liskov88] formalizes the principle of protection against variations in different implementations of an interface, or subclass extensions of a superclass.

Informally, software (methods, classes, ...) that refers to a type T (some interface or abstract superclass) should work properly or as expected with any substituted implementation or subclass of Tcall it S. 

> Law of Demeter

Don't Talk to Strangers

违反`Law of Demeter`的代码示例：

```java
public void doX()
{
F someF = foo.getA().getB().getC().getD().getE().getF();
// ... 
}
```

**The design is coupled to a particular structure of how objects are connected.** The farther along a path the program traverses, the more fragile it is. Why? Because the object structure (the connections) may change.

Novice developers tend toward brittle designs, intermediate developers tend toward overly fancy and flexible, generalized ones (in ways that never get used). Expert designers choose with insight;  关键是这个度不好把握啊

> Open-Closed Principle

OCP and PV are essentially two expressions of the same principle, with different emphasis

### Elaboration Iteration 2 More Patterns - Applying GoF Design Patterns

> Adapter (GoF)

Problem:  How to resolve incompatible interfaces, or provide a stable interface to similar components with different interfaces?

Solution: Convert the original interface of a component into another interface, through an intermediate adapter object.

> Factory

The adapter raises a new problem in the design:

who creates the adapters? And how to determine which class of adapter to create, such as TaxMaster-Adapter or GoodAsGoldTaxProAdapter?

答案就是Factory

Problem: Who should be responsible for creating objects when there are special considerations, such as complex creation logic, a desire to separate the creation responsibilities for better cohesion, and so forth?

Solution: Create a Pure Fabrication object called a Factory that handles the creation.

![](/images/post/factory.jpg)

重点关注上图中，它是如何决定应该使用哪个Adapter的

Note that in the ServicesFactory, the logic to decide which class to create is resolved **by reading in the class name from an external source (for example, via a system property if Java is used) and then dynamically loading the class.** This is an example of a partial data-driven design.

> Singleton (GoF)

The ServicesFactory raises another new problem in the design: Who creates the factory itself, and how is it accessed? 汗死。。。  按下葫芦起了瓢的感觉

观察

First, observe that only one instance of the factory is needed within the process. Second, quick reflection suggests that the methods of this factory may need to be called from various places in the code, as different places need access to the adapters for calling on the external services.

Problem: Exactly one instance of a class is allowedit is a "singleton." Objects need a global and single point of access.

Solution: Define a static method of the class that returns the singleton.

![](/images/post/singleton.jpg)

> Strategy (GoF)

Problem: How to design for varying, but related, algorithms or policies? How to design for the ability to change these algorithms or policies?

Solution: Define each algorithm/policy/strategy in a separate class, with a common interface.

> Composite (GoF)

Problem：How to treat a group or composition structure of objects the same way (polymorphically) as a non-composite (atomic) object?

Solution: Define classes for composite and atomic objects so that they implement the same interface.

没有太理解清楚这个pattern

> Facade (GoF)

Problem: A common, unified interface to a disparate set of implementations or interfacessuch as within a subsystemis required. There may be undesirable coupling to many things in the subsystem, or the implementation of the subsystem may change. What to do?

Solution: Define a single point of contact to the subsystema facade object that wraps the subsystem. This facade object presents a single unified interface and is responsible for collaborating with the subsystem components.

> Observer (Publish-Subscribe)

Problem: Different kinds of subscriber objects are interested in the state changes or events of a publisher object, and want to react in their own unique way when the publisher generates an event. Moreover, the publisher wants to maintain low coupling to the subscribers. What to do?

Solution: Define a "subscriber" or "listener" interface. Subscribers implement this interface. The publisher can dynamically register subscribers who are interested in an event and notify them when an event occurs.

> Observer/Publish-Subscribe/Delegation Event Model (GoF)

都是一个意思

Problem：Different kinds of subscriber objects are interested in the state changes or events of a publisher object, and want to react in their own unique way when the publisher generates an event. Moreover, the publisher wants to maintain low coupling to the subscribers. What to do?

Solution: Define a "subscriber" or "listener" interface. Subscribers implement this interface. The publisher can dynamically register subscribers who are interested in an event and notify them when an event occurs.

Observer provides a way to loosely couple objects in terms of communication. Publishers know about subscribers only through an interface, and subscribers can register (or de-register) dynamically with the publisher.

### Elaboration Iteration 3 - Intermediate Topics

### Elaboration Iteration 3 - UML Activity Diagrams and Modeling

A UML activity diagram shows sequential and parallel activities in a process. They are useful for modeling business processes, workflows, data flows, and complex algorithms.

![](/images/post/activity-diagram.jpg)

### Elaboration Iteration 3 - UML State Machine Diagrams and Modeling

A UML state machine diagram, illustrates the interesting events and states of an object, and the behavior of an object in reaction to an event.

A state machine diagram shows the lifecycle of an object: what events it experiences, its transitions, and the states it is in between these events.

![](/images/post/state-machine.jpg)

### Elaboration Iteration 3 - Relating Use Cases

### Elaboration Iteration 3 - More SSDs and Contracts

### Elaboration Iteration 3 - Domain Model Refinement

### Elaboration Iteration 3 - Architectural Analysis

Architectural analysis can be viewed as a specialization of requirements analysis, with a focus on requirements that strongly influence the "architecture." For example, identifying the need for a highly-secure system.

The essence of architectural analysis is to **identify factors that should influence the architecture, understand their variability and priority, and resolve them.** The difficult part is knowing what questions to ask, weighing the trade-offs, and knowing the many ways to resolve an architecturally significant factor, ranging from benign neglect, to fancy designs, to third-party products

- variation point
    - Variations in the existing current system or requirements, such as the multiple tax calculator interfaces that must be supported.
- evolution point
    - Speculative points of variation that may arise in the future, but which are not present in the existing requirements.

非功能性分析，比如安全、性能等

### Elaboration Iteration 3 - Logical Architecture Refinement

![](/images/post/view-of-layers.jpg)

When the lower Application or Domain layer needs to communicate upward with the UI layer, it is usually via the Observer pattern.

it is not coupling per se that is a problem, but unnecessary coupling to variation and evolution points that are unstable and expensive to fix. 

> Is the Application Layer Optional?

If present, the Application layer contains objects responsible for knowing the session state of clients, mediating between the UI and Domain layers, and controlling the flow of work.

### Elaboration Iteration 3 - More Object Design with GoF Patterns

> Handling Failure

a common exception handling pattern:

- Convert Exceptions
    - Within a subsystem, avoid emitting lower level exceptions coming from lower subsystems or services. Rather, convert the lower level exception into one that is meaningful at the level of the subsystem. The higher level exception usually wraps the lower-level exception, and adds information, to make the exception more contextually meaningful to the higher level.

For example, the persistence subsystem catches a particular SQLException, and (assuming it can't handle it[2] ) throws a new DBUnavailableException, which contains the SQLException. Note that the DBProductAdapter is like a facade onto a logical subsystem for product information. Thus, the higher level DBProductAdapter (as the representative for a logical subsystem) catches the lower level DBUnavailableException and (assuming it can't handle it) throws a new ProductInfoUnavailableException, which wraps the DBUnavailableException.

- Name The Problem Not The Thrower
    - What to call an exception? Assign a name that describes why the exception is being thrown, not the thrower. The benefit is that it makes it easier for the programmer to understand the problem, and it the highlights the essential similarity of many classes of exceptions (in a way that naming the thrower does not).

- Centralized Error Logging
    - Use a Singleton-accessed central error logging object and report all exceptions to it. If it is a distributed system, each local singleton will collaborate with a central error logger. 

- Error Dialog

> Proxy

Problem: Direct access to a real subject object is not desired or possible. What to do?

Solution: Add a level of indirection with a surrogate proxy object that implements the same interface as the subject object, and is responsibility for controlling or enhancing access to it.

> Abstract Factory

Problem: How to create families of related classes that implement a common interface?

Solution: Define a factory interface (the abstract factory). Define a concrete factory class for each family of things to create. Optionally, define a true abstract class that implements the factory interface and provides common services to the concrete factories that extend it.    

### Elaboration Iteration 3 - Package Design

### Elaboration Iteration 3 - Designing a Persistence Framework with Patterns
