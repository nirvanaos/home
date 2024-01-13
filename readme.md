# Introduction

## What is Nirvana

"Nirvana" is a research open-source project of the operating system,
based on some new principles.

<div class="note">

If you are afraid of the words "operating system", you can consider it
as just a framework, especially since the current implementation of the
Nirvana core works over the Windows host.

</div>

## Key features

### Actor model

This is probably the first OS based on the [actor
model](https://en.wikipedia.org/wiki/Actor_model).

### Everything is an object, every object is an actor

Nirvana implements the actor model implicitly. A programmer just writes
code as usual C++ classes and describes the interfaces in IDL (Interface
Definition Language). IDL compiler creates a proxy code that wraps the
object interface and converts the object into the actor. Each outgoing
call to other object is internally converted into sending the request
message and receiving the reply message. Each incoming call from other
object is converted into receiving the request message and sending the
reply message.

Each object method described in IDL can be called as usual or called
asynchronously via asynchronous method invocation (AMI) mechanism
provided by system.

So programming for Nirvana does not require changing the programming
paradigm as for other actor frameworks. You can use OOP paradigm, as you
are used to. This significantly decreases the entry threshold.

### CORBA

Nirvana implements the Common Object Broker Architecture (CORBA) - an
amazing specification of the inter-object communication framework.
Unlike other CORBA-compliant systems, oriented to the enterprise level
network systems, Nirvana is designed for usage in any size systems from
the large enterprise clusters to IoT controllers, because it provides
efficient communications between objects placed on the same computer.

### Distributed Computing

Nirvana is an implementation of the CORBA standard. So it may seamless
communicate via CORBA IIOP (Internet Inter-ORB Protocol) not only with
other Nirvana systems but also with any other CORBA implementations.

The Nirvana objects may seamless call remote objects from other CORBA
implementations and vice versa.

### Service Oriented Architecture

You do not have processes. You can not create threads. You do not use
mutexes or other synchronization objects. Your code just processes
requests.

A Nirvana application does not have a `main()` function that does all
the work. You have to create a class instead.

    class MyCoolProgram {
    public:
      int main (std::vector <std::string> args);
    };

### Real-Time

Many years ago, software was divided into package processing and
real-time systems. Today this division is obsolete. I hope that each
interactive system is a real-time system because it has to react to user
input at an acceptable time. Because of this, Nirvana was designed as a
real-time system with an accent on the reaction time.

Nirvana core contains two different schedulers: real-time and
background.

#### Real-Time Scheduler

Messages sent by the actors have deadlines. Queued messages are
prioritized according to deadlines. In this way, the real-time scheduler
implements the [Earliest Deadline
First](https://en.wikipedia.org/wiki/Earliest_deadline_first_scheduling)
scheduling algorithm.

We do not have priorities (firstly because we do not have threads).
Instead of the usual division on high-priority and low-priority threads,
we just assign the deadline for finishing processing each incoming
event, based on the required reaction time for a particular event.

#### Non-preemptive Multitasking

Preemptive multitasking is a legacy of the dark ages when we had only
one processor core for several tasks or even for several users. And the
main issue of the operating system was to distribute the processor
resources between many consumers.

Now we have a lot of processor cores for one user, and the main issue is
to distribute the work over these cores. It is an opposite issue and it
requires the opposite approach.

Nirvana uses non-preemptive multitasking for real-time activities. When
one actor claims a processor core for the incoming message processing,
it would not be preempted until it finished processing the current
message or sent a message to another actor.

### Stateless Objects and Background Scheduler

If an object does not have a state (internal variables), it does not
need any synchronization. Such objects can process multiple messages
simultaneously.

If a message processed by a stateless object has infinite deadline, it
is processed in the background. Background processing is scheduled by
the separate background scheduler with preemptive multitasking and
executed only when some processor core is free and no real-time messages
in the real-time scheduler queue.

### New Memory Management

All current memory management is based on the two very old functions:
`malloc` and `free`. Modern OS and CPUs provide advanced features like
paging, protection, and page sharing.

Different OS-es provide the proprietary API for advanced memory
management (VirtualAlloc, mmap). These API often looks like clutches and
rarely used in real programming.

In Nirvana there is the Memory interface - the try to create a modern,
consistent and portable memory management API, suitable for any hardware
platform.

### Modularity

One of the Nirvana architecture design goals was the software modularity
with maximal binary reuse. Binary modules (class libraries) may be built
with different compilers and different programming languages. Binary
module exports and imports some interfaces defined with IDL. IDL
provides standard ABI (Application Binary Interface) independent of the
particular compiler. All compiler-specific things, like the
implementation of exceptions and virtual tables, are hidden inside the
module.

The export and import module metadata contains version information.
During the module loading, the system checks version compatibility. This
lets to keep multiple versions of the module and provide backward
compatibility.

### Distributed Garbage Collection

CORBA specification has a serious lack - the absence of distributed
garbage collection.

Nirvana provides a solution for the DGC problem. The main idea was
borrowed from the DCOM protocol. Unlike DCOM, the programmer can
explicitly enable or disable the DGC policy for objects at the creation
time. If DGC policy is not explicitly specified, the system makes the
decision by default, based on the context. In most cases, it is enough
and does not require additional effort from a programmer.

The distributed garbage collector is also an exported object. Every
server that participates in the distributed garbage collection needs to
export a similar object. It is possible to have heterogeneous
distributed garbage collection by having servers in other platforms and
languages export a similar distributed garbage collecting object. For
the details see NirvanaDGC.idl file.

When Nirvana interacts with other CORBA systems, that do not support
DGC, it uses the usual CORBA connection-oriented garbage collection
strategy: when a connection is alive, all objects grabbed by this
connection are considered as alive.

To deal with clients that have crashed holding references to objects,
the Transport Heartbeats are used (see [Fault Tolerant
CORBA](https://www.omg.org/spec/FT/1.0/)). When a server has not
received a ping message after some configurable amount of time, the
server assumes that the client no longer exists and furthermore
decrements the reference counts of the objects that the particular
client has references to.

### Legacy Mode

If you have a legacy POSIX-compatible application with a main() function
and you do not want to convert it to Nirvana class, you might just
compile it as usually and run in Nirvana legacy mode.

Legacy mode is a special separate environment that lets to run legacy
applications in a usual way. Legacy applications can not create Nirvana
objects directly. But legacy applications can create objects via object
factories and call the object's operations.

Legacy application have background priority and execute only when there
are processor cores free from performing the Nirvana requests. Legacy
mode provides preemptive multitasking. If Nirvana runs over a host OS,
the background threads have default priority for this OS.

## Philosophy

### Programmer-oriented Environment

Today's operating systems are performance-oriented. They offer a variety
of performance-enhancing tools, such as threading. But the use of these
tools requires a lot of attention and accuracy from the programmer. The
well-known technical writer Don Box, in one of his articles, compared
streams with chainsaws, which can be very useful, but, if used
carelessly, can cause great harm to both the user and others. Developing
this idea, we can say that the developers of modern operating systems
offer programmers more and more powerful tools, not caring about whether
their use will be safe and convenient. The results of this are felt by
the user when he suddenly, for unknown reasons, the program hangs or
falls. The easiest way to explain everything is the programmer's lack of
qualifications. However, firstly, the human brain is ill-equipped to
solve certain types of problems, in particular, analysis of the
interaction of parallel processes.

An ideal system should maximize the efficiency of the entire software
life cycle, from development to support. Such a system should take on
routine functions, such as optimal computational parallelization and
locking, freeing the programmer for more creative work. Using the same
analogy, the chainsaw should be replaced by an intelligent robot, able
to cut a set number of trees, spending a minimum of energy and not
endangering anyone. And the actor model provides the solution.

Joe Armstrong, the creator of the Erlang:

> I also suspect that the advent of true parallel CPU cores will make
> programming parallel systems using conventional mutexes and shared
> data structures almost impossibly difficult, and that the pure
> message-passing systems will become the dominant way to program
> parallel systems.

Brian Storti, the author of the Kubernetes in Practice:

> Our CPUs are not getting any faster. What’s happening is that we now
> have multiple cores on them. If we want to take advantage of all this
> hardware we have available now, we need a way to run our code
> concurrently. Decades of untraceable bugs and developers’ depression
> have shown that threads are not the way to go. But fear not, there are
> great alternatives out there and today I want to show you one of them:
> The actor model.

### Why Nirvana?

I am working in the programming industry for over 35 years and no one of
the programming environments was satisfied me completely. Because of
this, many years ago, I have decided to try to imagine how the ideal
programming environment might be. Reflections on it have brought me to a
pleasant state, like Nirvana. After a lot of thinking I started to
implement it. Because I started to implement it completely from scratch,
I did not be bounded by the restrictions of any programming environment.
And I could think about the right implementation for an unlimited time.
Nobody has noting me for the terms. I felt completely free and happy.

At first, when you every day have a deal with a non-ideal API and
non-ideal code every, you sometimes become unhappy. So I needed some
relaxation in my free time. And the best way to relax for me is writing
a perfect code. So in the evenings, I had let myself relax and dive into
my Nirvana. I do it just for a fun.

Second, I try to create an ideal programming environment, where the
programmers will feel happy.

## Portability

Nirvana is implemented in C++ and can be ported to any architecture
which has C++ 11 compiler and provides lock-free compare-and-swap
operation on the pointer type.

Implementation is divided into three parts:

Core  
The Nirvana core. Portable.

Library  
The common library for Nirvana applications. Portable. Interacts with
Core only.

Port  
The layer between the Core and the underlying host. Underlying host may
be the other operating system or the micro kernel. Currently implemented
the port for the Windows only.

For porting to a new host, the new Port library has to be created.

## Current state

Development is still at the pre-MVP stage. Currently are implemented:

- Runtime library

- Memory manager

- Scheduler

- The IDL compiler

- Object-oriented access to file system as a part of the CORBA Naming
  Service.

All that you can do currently is build and run tests.

The main supermodule for building with MS Visual Studio 2019 is
[here](https://github.com/nirvanaos/nirvana.vc).

Also you can have a look on [IDL compiler front-end
library](https://github.com/nirvanaos/idlfe).

I'm very interested in the enthusiasts who will help me in the
development. If you want to participate, e-mail me:
\<popov.nirvana@gmail.com\>.
