# php-gambling-website

__Table of contents__

* [Overview](#overview)
* [Installation and requirements](#installation-and-requirements)
* [Context is king](#context-is-king)
  * [Chat](#chat)
  * [Common](#common)
  * [Connect Four](#connect-four)
  * [Identity](#identity)
  * [Web Interface](#web-interface)
* [Transition to Microservices](#transition-to-microservices)
* [Scale-Out the application](#scale-out-the-application)
* [Chosen technologies](#chosen-technologies)
* [A note on testing](#a-note-on-testing)

## Overview

This is my playground project to explore new ideas and concepts. It's a gambling website where people can play against
each other. Currently, the only game is [Connect Four](#connect-four) but I plan other games to show more concepts.

Before you start looking at the code, I recommend reading this documentation to understand what concepts I use
and why I apply these concepts for this particular application.

The application is built with a
[Microservice Architecture](https://martinfowler.com/articles/microservices.html),
concepts of
[Domain Driven Design](http://domainlanguage.com/ddd/reference/)
and Scale-Out techniques in mind.
The sections
[Context is king](#context-is-king),
[Transition to Microservices](#transition-to-microservices)
and
[Scale-Out the application](#scale-out-the-application)
describe whats done to apply these concepts.

To get a ready-to-start project, the code and the development environment are in one git repository.

## Installation and requirements

I recommend to use
[Docker](https://www.docker.com),
[Docker Compose](https://docs.docker.com/compose/).
You can certainly set up a server yourself.

Since I use the latest JavaScript techniques, like EcmaScript 6, it doesn't work in all browsers.
Currently, it's only tested with Safari.
Don't look at the front end design, this is by no means my domain.

### Development

```
git clone https://github.com/marein/php-gambling-website
cd php-gambling-website
./project build
```

There're several other handy commands in the project script, like running tests or a static analyzer. You see them with

```
./project help
```

__Note__  
If you update your local repository, you have to "./project build" again.
This project is in development and I don't pollute the code with schema changes at this stage.

### Production

The
[production images](https://hub.docker.com/r/marein/php-gambling-website/)
are built when pushed to git master. They always reflect the latest stable version.

```
git clone https://github.com/marein/php-gambling-website
cd php-gambling-website
docker-compose -f docker-compose.production.yml pull
docker-compose -f docker-compose.production.yml up
```

## Context is king

The application is split into several
[Bounded Contexts](https://martinfowler.com/bliki/BoundedContext.html).
I've chosen modeling techniques like
[Domain Driven Design](http://domainlanguage.com/ddd/reference/)
and
[Event Storming](https://en.wikipedia.org/wiki/Event_storming)
to define them. Well, you only see the result and not the breakthroughs I went through.
The following sections describe what techniques I use in the respective context.
Note that I've intentionally chosen a different approach in each context.

I presuppose you've read
[Implementing Domain Driven Design](https://vaughnvernon.co/?page_id=168#iddd)
by Vaughn Vernon and
[Tackling Complexity in the Heart of Software](http://dddcommunity.org/book/evans_2003/)
by Eric Evans.

### Chat

This context is very simple. To organize the business logic, the
[Chat](/code/src/Chat)
uses the 
[Transaction Script](https://martinfowler.com/eaaCatalog/transactionScript.html)
pattern. Its only job is to initiate chats, list messages by chat
and to write messages in the chat. If authors are assigned to a chat, only those authors can write and read messages.
The layering chosen in the other contexts isn't worthwhile here.

The public interface is formed by a
[controller](/code/src/Chat/Presentation/Http/ChatController.php),
which can be called up via http, and a
[command line task](/code/src/Chat/Presentation/Console/RabbitMqCommandListenerCommand.php),
which serves as an interface to a message broker.

This context publishes
[Domain Events](https://martinfowler.com/eaaDev/DomainEvent.html)
through the message broker to inform other contexts what's happened here.
First the domain events are stored to the event store.
This happens in the same transaction in which the commands are executed.
After that, a
[command line task](/code/src/Chat/Presentation/Console/PublishStoredEventsToRabbitMqCommand.php)
publish these stored events to the message broker.

I've chosen MySQL as the storage.

### Common

The
[Common](/code/src/Common)
folder provide reusable components. If the project is more advanced, I'll outsource them as libraries.
But there're already battle tested implementations out there (like a
[Bus](https://tactician.thephpleague.com)
by Tactician, or an
[Event Store](https://github.com/prooph/event-store)
by prooph). You may use them, instead of mine. The
[Event Store](/code/src/Common/EventStore)
implementation inside
[Common](/code/src/Common)
isn't used to be a storage for an
[Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html)
model. It's really just a store for events.

### Connect Four

The
[Connect Four](/code/src/ConnectFour)
is the context where I put the most effort in. The business logic is definitely worth building a proper
[Domain Model](https://martinfowler.com/eaaCatalog/domainModel.html). Players can open, join, abort and resign a game.
Of course they can also perform moves. The game can be aborted until the second move.
After the second move, players can only resign or finish the game. The referee, which sits near the game desks,
ensure that the people can talk to each other. This process is described below.

As the
[folder structure](/code/src/ConnectFour)
shows, this context uses the "Ports and Adapters" architecture. The
[Application Layer](https://martinfowler.com/eaaCatalog/serviceLayer.html)
uses a command and query bus. The opposite of this approach is the traditional application service I use in the
[Identity](#identity)
context. It boils down to one class with many methods vs. many classes with one method.

The public interface is formed by a
[controller](/code/src/ConnectFour/Port/Adapter/Http/GameController.php),
which can be called up via http.

This context publishes
[Domain Events](https://martinfowler.com/eaaDev/DomainEvent.html)
through the message broker to inform other contexts what's happened here.
First the domain events are stored to the event store.
This happens in the same transaction in which the commands are executed.
After that, a
[command line task](/code/src/ConnectFour/Port/Adapter/Console/PublishStoredEventsToRabbitMqCommand.php)
publish these stored events to the message broker.

The Connect Four context applies the
[CQRS](https://martinfowler.com/bliki/CQRS.html)
pattern. Not only the domain model is divided into command and query side, but also the storage layer.
The query model is stored in an
[eventual consistency](https://en.wikipedia.org/wiki/Eventual_consistency)
manner. A
[command line task](/code/src/ConnectFour/Port/Adapter/Console/BuildQueryModelCommand.php)
retrieves the stored events from the event store and then builds the query model.
This is done for scalability reasons. Why exactly this was done is described in the section
"[Scale-Out the application](#scale-out-the-application)".
Before you use it in your application, you should check if you really need it.
This model adds risky complexity to the codebase. Also note that nothing I've done here is required to
apply the basics of the CQRS pattern. Look at
"[Busting some CQRS myths](https://lostechies.com/jimmybogard/2012/08/22/busting-some-cqrs-myths/)"
by Jimmy Bogard for further reading.

There's also a
[Process Manager](http://www.enterpriseintegrationpatterns.com/patterns/messaging/ProcessManager.html)
involved.
Its name is referee and it's a
[command line task](/code/src/ConnectFour/Port/Adapter/Console/RefereeCommand.php).
The referee picks up a player joined event and ensures, that a chat is initiated.
When the chat is initiated, it assigns the chat to the game.
This is done, so the storage of games and chats can be on different MySQL instances.
This allows to scale the games and the chats independently, but ensures consistency with
[eventual consistency](https://en.wikipedia.org/wiki/Eventual_consistency).

I've chosen MySQL as the command side storage. Since the games are stored as json, MySQL is used as a document store.
On the query side, I've chosen Redis as the storage, since there are no relational queries to perform.
Note that on the command side you'll need a database that allows you to store the domain model as well as
the events in a single transaction. Another possibility is to use
[Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html)
as the storage model. With this model, only the events are saved.
The game aggregate is actually a good fit for the event sourcing approach.
Maybe I'll implement this storage model in the next game.

### Identity

This context is currently only set up. Nothing interacts with it.

Although the
[Identity](/code/src/Identity)
context is another simple context, like the chat context, I still use an ORM for it. In this case
I've chosen Doctrine because it's a really matured ORM that applies the
[Data Mapper](https://martinfowler.com/eaaCatalog/dataMapper.html)
pattern. The main responsibilities are that users can sign up, change username and change password.

As the
[folder structure](/code/src/Identity)
shows, this context uses the "Ports and Adapters" architecture. The
[Application Layer](https://martinfowler.com/eaaCatalog/serviceLayer.html)
uses a traditional application service. The opposite of this approach is the
command and query bus I use in the
[Connect Four](#connect-four)
context. It boils down to one class with many methods vs. many classes with one method.

The public interface is formed by a
[controller](/code/src/User/Port/Adapter/Http/UserController.php),
which can be called up via http.

### Web Interface

The
[Web Interface](/code/src/WebInterface)
acts like an
[Api Gateway](http://microservices.io/patterns/apigateway.html).
All browser interactions go through this context, because its main responsibility is the session management
and the aggregation of the data from the other contexts. The
[JavaScript](/code/src/WebInterface/Presentation/Http/JavaScript)
and
[StyleSheet](/code/src/WebInterface/Presentation/Http/StyleSheet)
are also defined here.

There're currently three pages.
1. The first page is the game lobby. Users can come together here to open or join games.
If John opens a game and Jane clicks on it, both have a game against each other.
If John clicks on his own game, the game will be aborted.
2. The second page is the game itself. The users play against each other and can write messages.
3. The third page is the user profile. Users can see a history of past games here.

## Transition to Microservices

This application matches all requirements to be a so called
[Microservice Architecture](https://martinfowler.com/articles/microservices.html),
except for deployment. This section describe the steps which needs to be done to fulfill the microservice requirement.
The true microservice approach isn't done, because I don't need it. I'm a single developer and it isn't worthwhile
here. The only requirement I've set myself is the high scalability which is described in the next section. However,
I've assigned an abstraction layer to the Web Interface context for easier migration to the microservice architecture.

To have single deployable units, the following steps needs to be done
1. Copy the folders (at
[config](/code/config),
[src](/code/src)
and
[tests](/code/tests))
in a separate application for each context or the context that's worthwhile to be a single deployable unit.
Don't forget to define the routing for the controllers in the new application.
2. The Web Interface is the only context which performs direct method invocations to the others.
This needs to be rewritten. You've to write new implementations for the interfaces at
[code/src/WebInterface/Application](/code/src/WebInterface/Application).
The current implementations are at
[code/src/WebInterface/Infrastructure/Integration](/code/src/WebInterface/Infrastructure/Integration).
They're need to make real http calls.
3. Be happy.

__Note that's totally fine to invoke the application layer of the other contexts directly.
It helps with type safety and adds other benefits that you get from a monolithic approach.
I just added this abstraction layer to write this section.__

## Scale-Out the application

Scale-Out is a technique where you can put as much servers in parallel as possible to handle high loads.
I've set this requirement for this application. This section describe how this requirement is fulfilled.

The application code is stateless. To scale that is very easy.
Just put a
[Load Balancer](https://en.wikipedia.org/wiki/Load_balancing_(computing))
in front of the application servers.

In the next step we have to scale the databases. We divide this into two parts
1. First thing we've todo is scale the databases which are accessed for read purposes. Since there's no concurrency
issue here, we can just add replications for the MySQL and Redis storages.
2. The second thing is a little bit trickier. We've to scale the databases for write purposes.
I will describe this using the Connect Four and the Chat context. I've put much effort to allow scaling the Connect Four
context properly. Because we use the CQRS pattern to decouple the queries that span multiple games,
such as counting running games or listing open games, it's easy to scale the command side. We've just to throw in a
technique called
[sharding](https://en.wikipedia.org/wiki/Shard_(database_architecture)).
The shard key is, of course, the UUID of the game. It's used to determine which shard to use.
I'll implement this later as an example. To anticipate it: Define two or more connections to the database and use
this as a pool to determine which shard to use based on the UUID. Voil??, you'll find the game you want.
To scale the command side at the Chat context, nothing needs to be done. There are no queries that span multiple chats.
Just be sure you store the chat and the related messages to the same shard. But since we use the UUID of the chat
in every query and command, this should be easy. The rest is as described above.

__Note that the sharding technique described here is a manual technique. The application decides which shard to use.
There are certainly other variants.__

You may have seen that currently only one MySQL and one Redis instance is configured. This is done for simplicity.
I don't want to wait hours until the development environment is starting.
Of course, that's different in the production environment.
You can for sure configure different databases for the contexts. Have a look at the
[configuration file](/code/environment.env.dist).
There is an example. Have a look at the
[advanced production compose file](/docker-compose.production-advanced.yml).
We can split this even further.
For example, we can create a Redis instance per query model in the "Connect Four" context.
Of course, the code must be adapted. Whether it's worth it is another question.

## Chosen technologies

It's mainly written with PHP, but also JavaScript
for the frontend. I've chosen
[Symfony](https://symfony.com)
as the underlying framework, because I know it and I'm free to choose my application architecture,
or in this context, the directory structure.

Some other technologies:
* [MySQL](https://www.mysql.com) as the main storage of the contexts.
* [Redis](https://redis.io) for the query models or as a caching layer. Also the user sessions are stored here.
* [Rabbit Mq](https://www.rabbitmq.com) as the message broker.
* [Nchan](https://nchan.io) for real-time browser notifications.
* [Makefile](/code/Makefile) for concatenation of javascript and css.

## A note on testing

Not many tests have been written, but if you meet the requirements in
[Installation and requirements](#installation-and-requirements)
you can run them with

```
./project tests
```
