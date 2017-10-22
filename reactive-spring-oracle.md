# Reactive Spring

- the need for async, reactive IO
- the missing idiom
- the reactive streams initiative
- towards higher order operators w/ Reactor
- the reactive Spring ecosystem
- writing some sample data  w/ reactor and dependent calls
- data access
- spring mvc style controllers
- functional reactive endpoints
- security
- testing w/ the webtest client

# The Need for Async IO
At pivtoal we see a lot of organizations moving to miccroservices. the arhcitecture emphasizes small, singly focused, independently deplotable -piecs of functinoality. Theyre small because small implies a smaller workforcec working on the codebase. theyre singly focused becuase its easier to stay small that way. theyre independenly deploabble because tht means youre never blocked by other teams in the organization. the architecture has a lot of benefits; thye support organiztional agility. that said, they do have some drawbacks. the approach impies network distribution. the diestribution in turn invites architectural complexity and things get difficult when dependencies fails. things get difficul t when networks fail. and, specifically for our purposes in this discusion, things get diffiult when the conduction of data between nodes necoems overwhelming.

the overhwelming condduction of data between nodes is a harder problem to solve than most, espeically on the JVM, wehre the abstractions supporing IO  are   blocking in the original design in JDK 1.0. when we say blocking we mean that for a client to read a byte from a file or a socket or whatever, it needs to wait for the byte to arrvie. the thread in which the read is happeing is idle, but unusable. this is a pity because it doesnt need to be thiis way. at lower levels in the OS, network IO is typiccally asynchronous. POSIX complpian OSes have supported the ability to poll filedesritptors ro IO activity and dispatch when there is someting to do.   the classic `select(fd*)` call supports this.

JDK 1.4 saw the introduction of NIO, the _new_ IO packages. NIO gaves us channels which allow asyncchronous IO. this sia marked improvement but youll notice that it hasnt exactly become the pervasive metaphor for computation. these ddays almost alll of the APIS typical of enterpriee  computing are blocking by default and you probabyl dont even realize it. hopefully, most of us arent wriing custom database drivers, http sweb stacks, and security technologies ourselves. were relying on tried and true implementations that sit below the value line, below the line of things that we feel the need to expend cognitive effort trying to understand. our business application i swhat matters, and so most frameworks let us think about the world in terms of our business domain, relegating IO concersn to the murky depths of the of the framework and app stack.

wee think about the world in terms of instances of ourdomain model entities, `T`, or collections of `T`. those are synchronous views on data. the `CompletableFuture<T>` gives us asyncchronous views of data. it lets us say that, eventually, we'll have a reference to a `t`. But we havent had for the longest time a way to say that we have a potentially infinite collection of type `T` that may or may not arrive asynchronously.

Such a metaphor would need to be coupled with a way to gate the production and consumption of values. In the world of finite data sets, producers may occasionally overwhelm consumers but a consumer has various knobs and levers it can pull to buffer extra work, allowing it to eventually get to it. This works because the buffer is sufficient to cocntain the overwhelming demand. But what happens if the production is infinite? eventually, as with a floodded boat whose occupants are desperately trying to bail the ocean's worth of water,   the consumer will sink. This is called flow control, and it needs to be a first class feature of any computational metaphor  we choose.

# The Missing Metaphor

Not for want of trying, of course.. a lot of differnent orgs have tried to address this gap. microsoft kocked things off w the RX extensions for .NET. netflix ported that to java and created rxjava. (rxjava has in turn inspired countless lcones in othe rlanguages and platforms like Swift, NOde.js, and Ruby.) the spring team launched a project called Reactor. Lightbend (ne' Typesafe) tried to support this space in Akka and the Akka Streamsproject. Tim Fox, first at VMWare, and then at RedHat, launched the vert.x project. all of these attempt to address the same usecases: fill in the gap in our idioms and then, tow whatever extn possible, develop an ecosyste of tools on top of the new idiom that suppots modern application development concerns.

# The Reactive Streams Initiative
theres common groun dacross these different approaches. so the aforementioned vendors worked together to develop a de-facto standard, the [reactive streams initiative](http://www.reactive-streams.org/). The reactive streams initative defines four types:

the `Publisher<T>` is the computational metaphor we've been looking for. it defines the missing piece; a producer of values that may eventuallt arrive. a publisher produces values of type `T`

.`Publisher<T>``
<!--  -->

the `Subscriber` susbscribes to a Publisher, receiving notifications on any new values f type `T`

.`Subscriber`
<!--  -->

When a `Subscriber` subscribes to a `Publisher`, it results in a `Subscription<T>`

.`Subscription<T>`
<!--  -->

A `Publisher<T>` that is also a `Subscriber<T>` is called a `Processor<T>`

.`Processor<T>`
<!--  -->

The `Publisher<T>` has one method, `Publisher#subscribe(Subscriber<T>)`. The `Subscriber` has a callback method, `Subscriber#onSubscribe(Subscription)`. that is invoked by the `Publisher<T>`. This is the first and last interaction between the `Publisher<T>` and the `Subscriber<T>` _until_ the subscriber calls the `Subscription#request(long numberOfRecordsRequested)` method. Here, the `Subscriber<T>` signals to the `Publisher<T>` how many more records, `T`, it is prepared to handle. The `Subscriber<T>` can not be overwhelmed - it will receive only as many records as it can handle. Then, and only then, will the `Subscriber#onNext(T)` method be called, as many times as requested. This dynamic  - where the producer gates the production based on capacity - is called _backpressure_; it's a way of managing flow.

The Reactive Sstreams specification serves as a foundation for interoperability. it provides a common way to address this missing metaphor. The specification is not mean tot be a prescription for the implementation APIs, it instead defines types for interoperability. Implementors can either build their implementations on top of the types or at least be able to consume and produce  references to the types as sort of adapters.

The Reactive Streams types eventually found their way into Java 9 as 1:1 semantically equivalent interfaces in the `java.util.concurrent.Flow` class. The Reactive Streams initiative, for the moment, remains a sepeate set of types but the expectation is that all implementations will also support Java 9 as soon as possible.

# Reactor

Are the types in the Reactive Streams initiative enough? I'd say _no_! The reactive streams specification is a bit like the plain vanille array in the JDK. It provides a basis on which to support higher order computations. Most of us don't use arrays, we use something that extends from `java.util.Collection`. The same is true for the basic reactive streams types. In orer to support filering, transforming, and iteration, youll need DSLS and thats where one of the RS implementations can really help.

Pivotal's Reactor project is a good choice here. its built on top of the Reactive Streams specification. It provides two specializations of the `Publisher<T>` type. The first, `Flux<T>`, is a Publisher that produces 0 or more values. It's unbounded. The second, `Mono<T>`, is a `Publisher<T>` that produces zero or one value. They're both publishers and you can treat them that way, but they go much further than the RS spec. They both provide ooperators - ways to  process the stream of values. Reactor types compose nicely - the output of one thing can be the input to another.

# Reactive Spring

As useful as Reactor is, it doesn't give us much on its own if we're trying to build a reactive web application talking to a database. Or make HTTP requests. Or support authentication and authorization. Or ..whatever. While Reactor gives us the missing metaphor, Spring helps us all speak the same language.  Spring framework was release in September 2017. It builds on Reactor and the RS specification. It includes a new reactive runtime and component model called Spring WebFlux. Spring WebFlux does not depend or require the Servlet APIs to work. It ships with adapters that allow it to work on top of a Servlet-engine, if need be, but it's not required. It also provides a Netty-based web server. Spring Framework 5 is the foundation for changes in much of the Spring ecosystem. Spring Fraework 5 has a Java 8 baseline; it works with

Spring Data Kay supports reactive data stores like MongoDB, Cassandra, Redis and COuchbase and introduces new reactive repository variants and templates. Spring Security 5.0 integrates security for Spring WebFlux based applications and introduces a new hierarchy to support authentication and authorization in a reactive, non-blocking application.

All of this rolls up into Spring Boot 2.0. Spring Boot 2.0 provides auto-configurations for all the aforementioned pieces and crucially adapts Spring Boot-only components like the Actuator - a set of endpoints designed to surface information about the application - to be web-runtime agnostic. The Spring Boot Actuator now works with Spring MVC, JAX-RS, and Spring WebFlux.

Spring Cloud Finchley builds on Spring Boot 2.0, and updates a number of different APIs, where appropriate, to leverage the reactive paradigm. things like sevice registration and discovery work in   Sspring webflux based applications.   spring cloud commons supports client-side load-balancing for the Spring Framework `WebClient`, the new reactive HTTP client. Spring cloud netlix Hystrix circuit breakers have always worked naturally with RxJava, which in turn can interop with Reactie Streams `Publisher<T>`s. This interop is even easier now.  Spring Cloud Stream supports working with pubishers to describe how messages arrive and are sent to messaging subsraits like RabbitMQ, Apache Kafka or Redis. Spring Cloud Gateway is a new reactive API gateway project that supports HTTP and websocket  request proxying, rewriting, load-balancing, circuitbreaking, rate limiting and much more. Sspring Cloud Sleuth has been updated to support distributed tracing across reactive boundaries.

# A Bootiful application

Let's look at an example. We'll build a simple Spring Boot 2.0 application.  Say, how about we build a service to manage books? We could call the project Bibliothech or Library or something ostentatious like that.

## the Spring Initializr

Go to the [Spring Initializr](http://start.spring.io). Make sure that some version of Spring Boot 2.0, or later, is selected in thhe version drop down menu. We're writing a service to manage access to libraries, so gice this project the artifact ID `library-service`. Select `Reactive Web`, `Actuator`, `Reactive MongoDB`, `Reactive Security`, and `Lombok`. Then, click _Generate_. You'll be given an archive; unzip it and open it in your favorite IDE that supports Java 8 (or later) and Maven (though we could've chosen Gradle at the SPSring Initializr, I'm assuming you've selected Maven for the purposes of this article.

Our stock standard Spring Boot application looks like this:

.the empty husk of a new Sprign Boot project
<!--  -->

## Data Access with Reactive Spring Data  modules

We've got reactive MongoDB on the classpath so lets use it to manage some data. create a new entity called `Book`.

.a MongoDB `@Document` entity
<!--  -->

create a Spring Data repository to support the data management lifecycle of the entity. this proecess should ook very familiar to anybody whos ever used Spring Data, except that the repository supports _reactive_ interactions. methods return `Publisher` types, and input can be given as a `Publisher<T>`.

.a reacive Spring Data MongoDB repository
<!--  -->

With that we have enough to install some smaple data (just for our demo). create an `ApplicationRunner` component that deletes all the data in the DB; then emits out a few book titles, maps them to `Book` entities, and then persists those books; then query all
the records in the DB and prints outt everything with the subscribe call.

## installation of some sample data

.an ApplicationRunner to write data
<!--  -->


its important to understand that this is a pipeline. each step in the code listing defines a stage in a pipeline. the pipeline is not _hot_, or eager. we need to trgger the pipeline. u can do this by subscribing. the `Publisher<T>`  api only defines one type of subscription, but Reactor defines a few overloads that support consumers and callbacks. there are other overloads that suport providing lambas to  a) process any of the emitted values with a `Consumer<T>` b) process any throwables thrown during procesing c) a Runnable to be  run when the `Publisher<T>` has finished emitting values d) a consumer that is given the initial subscription that is created after the subriber has subscribed to a pulbisher. these lambdas provide the same lifecycle hooks as overriding individual methods in a fully formed `Subscriber<T>` would.

## Spring WebFlux

now that weve got ddata in the database, lets stand up a rest api. well use spring webflux, a brand new reactive web runtime and component model. Soring WbFlux does not depend on the servelt specification . it can work independtly, with a Netty-based web server. it is designed, from the bottom up, to work with Publishers.

## REST w/ Sprign MVC style endpoints

we can use Spring MVC style controllers, like this:

.a Sprng MVC style REST api. make sure to run the sample code w/ the `mvc-style` profile
<!--
  this API should be
 -->

this api should loko familiar to anybody who has ever sed spring mvc. the class provides endpoint handlers and endpoint mappings through declarative annnotations. the annotations describe the endpont and how the routing should be handled. theyre sophistcated, but ultimately limited to whatever the framework itself can do with those annnotations. if u want morre flexible request matching capabilities, youo  can use  sprinng webflux functional reactive endpoints.

## Functional Reactive WebFlux Endpoints

.the same endpoints reworked as a funtional reactieve endpoints in java . make sure ot run this using the `frp-java` profile
<!--  -->

the functional reacitve style lets u express http endpoints as  request predicates mapped to a handler class. the handler class implementation is easily written using java lambdas, which can be very concise. u can use the default request predicates or, if u like, provide ur own, and gain full control over how reqwuests are matched. If youre using the Kotlin language, things can be even more concise thanks to a Kotlin-language DSL. Heres the same endpoints using the DSL.

.heres the same endpoints using the Kotlin anguage DSL
<!-- the Kotkin language example -->

## observability with Spring Boot 2's Actuator
// todo talk about how were chunking the results and so on
// intoduce the idea that u could handle all sorts of itnteractions ith the client in the same fashion since, at the end of the day, everything is a publisher. a publisher can handle websockets. it can handle server sent events. it can do everything.

Alright! Not bad. We've got a rest api, three different ways! canw e go to  production yet? Not yet. There are a number of concerns that need to be addressed in this. one is observability. observability is the insight that, no matter what technical stack ur using, no matter which business vertical ur software serves, no matter where ur buildign software, one thing is surprisingly consistnet: when the pager (or pagerduty, as much as antyhing these days) goes off, _somebody_ gets roused to a computer and has to get things back online. the goal is not root cause analysi, but to salve the wound. when the system is down, every second counts and we need to  support the remediation rpoccess with aas much insight into the operation of the system as possible. diffferent levels of the runtime will aid you here: a smart platform,  like Cloud Founddry, can gie ua  lot of insigth about the container running thw applicatoin and the runtime itself. the applicatoin is n the best situation to articalate its own state. we can support that articulatoin with the spring boot actuator.

the spring boot actuator isnt uniquely reactive. its been in spring boot from the beginning. the good news here is that, as of spring boot 2.0, it has been divorced frm any particualr web runtime. it no longer requires Spring MVC. it can work with Spring WebFlux, spring mvc, or jax-rs. specify that it export the http web endpoints with a bit of confiuration.

.configuration to expose the Actuator endpoints for http access
<!--  -->

w/ teh actuaot endpoints in place u can access all the usual suspects including the `applicatoin/metrics` endpoint and the `/applicatoin/health` endpoints.

.the health endponit
<!--  -->

the metrics endpoint is eworked in spring boot 2. its now backed by a new project  called [micrometer](http://micrometer.io). micrometer is an abstraction for dimensional metrics publication and accumulatoin. it integrates with time series databases like    Prometheus, Netflix Atlas, and Datadog. Support for InfluxDB, statsd, and Graphite are coming.  we designed micrometer to be framework agnostic, so u could use it independent of SPring Boot. that said, we hope youll agreeits even more compelling with.

.the metrics endpoint
<!--  -->

## Spring Security

I know what youre thinking, but its *still* nto quite ready fro production. we need to addrss security. well use spring security 5. spring security has been the bedrock for application security since its inception, providing a rich set of integrations with all manner of identity providers. it supports authentication and authorization. Spring Security supports authentication by propagating a security context so thrat applicatio level code - method invocations, http requests, etc. - have easy access to the context. the context has, historically, been implemented in terms of a threadlocal, which makes a lot of sense in a non-reactive world. in a reactive world, however, things arent's so simple. thankfully, Reactor provides a  `Context` object, which acts as a sort of dictionary. Sspring Seecurity has been reworked to propagate its security context using this. additionally, parraellel, reactive type hierarchies  have been introduced that support non-blocking interaction with the security apparata for ur application.  






<!--
  TODO:;
    - mention that there are a zillion other already possible things inlcuding using Reactor in Sprinng INtegraton
    - there's RSocket, which has a lot of possibility as well
    - theres websocket support
    - theres SSE support (thoug, the sse endpoints in this example should be used in the exampel above)
-->