.. _akka-testkit:

#####################
Testing Actor Systems
#####################

.. toctree::

   testkit-example

.. sidebar:: Contents

   .. contents:: :local:

.. module:: akka-testkit
   :synopsis: Tools for Testing Actor Systems
.. moduleauthor:: Roland Kuhn
.. versionadded:: 1.0
.. versionchanged:: 1.1
   added :class:`TestActorRef`

As with any piece of software, automated tests are a very important part of the
development cycle. The actor model presents a different view on how units of
code are delimited and how they interact, which has an influence on how to
perform tests.

Akka comes with a dedicated module :mod:`akka-testkit` for supporting tests at
different levels, which fall into two clearly distinct categories:

 - Testing isolated pieces of code without involving the actor model, meaning
   without multiple threads; this implies completely deterministic behavior
   concerning the ordering of events and no concurrency concerns and will be
   called **Unit Testing** in the following.
 - Testing (multiple) encapsulated actors including multi-threaded scheduling;
   this implies non-deterministic order of events but shielding from
   concurrency concerns by the actor model and will be called **Integration
   Testing** in the following.

There are of course variations on the granularity of tests in both categories,
where unit testing reaches down to white-box tests and integration testing can
encompass functional tests of complete actor networks. The important
distinction lies in whether concurrency concerns are part of the test or not.
The tools offered are described in detail in the following sections.

.. note::

   Be sure to add the module :mod:`akka-testkit` to your dependencies.

Unit Testing with :class:`TestActorRef`
=======================================

Testing the business logic inside :class:`Actor` classes can be divided into
two parts: first, each atomic operation must work in isolation, then sequences
of incoming events must be processed correctly, even in the presence of some
possible variability in the ordering of events. The former is the primary use
case for single-threaded unit testing, while the latter can only be verified in
integration tests.

Normally, the :class:`ActorRef` shields the underlying :class:`Actor` instance
from the outside, the only communications channel is the actor's mailbox. This
restriction is an impediment to unit testing, which led to the inception of the
:class:`TestActorRef`. This special type of reference is designed specifically
for test purposes and allows access to the actor in two ways: either by
obtaining a reference to the underlying actor instance, or by invoking or
querying the actor's behaviour (:meth:`receive`). Each one warrants its own
section below.

Obtaining a Reference to an :class:`Actor`
------------------------------------------

Having access to the actual :class:`Actor` object allows application of all
traditional unit testing techniques on the contained methods. Obtaining a
reference is done like this:

.. code-block:: scala

   import akka.testkit.TestActorRef

   val actorRef = TestActorRef[MyActor]
   val actor = actorRef.underlyingActor

Since :class:`TestActorRef` is generic in the actor type it returns the
underlying actor with its proper static type. From this point on you may bring
any unit testing tool to bear on your actor as usual.

Testing the Actor's Behavior
----------------------------

When the dispatcher invokes the processing behavior of an actor on a message,
it actually calls :meth:`apply` on the current behavior registered for the
actor. This starts out with the return value of the declared :meth:`receive`
method, but it may also be changed using :meth:`become` and :meth:`unbecome`,
both of which have corresponding message equivalents, meaning that the behavior
may be changed from the outside. All of this contributes to the overall actor
behavior and it does not lend itself to easy testing on the :class:`Actor`
itself. Therefore the :class:`TestActorRef` offers a different mode of
operation to complement the :class:`Actor` testing: it supports all operations
also valid on normal :class:`ActorRef`. Messages sent to the actor are
processed synchronously on the current thread and answers may be sent back as
usual. This trick is made possible by the :class:`CallingThreadDispatcher`
described below; this dispatcher is set implicitly for any actor instantiated
into a :class:`TestActorRef`.

.. code-block:: scala

   val actorRef = TestActorRef(new MyActor)
   val result = actorRef !! Say42 // hypothetical message stimulating a '42' answer
   result must be (42)

As the :class:`TestActorRef` is a subclass of :class:`LocalActorRef` with a few
special extras, also aspects like linking to a supervisor and restarting work
properly, as long as all actors involved use the
:class:`CallingThreadDispatcher`. As soon as you add elements which include
more sophisticated scheduling you leave the realm of unit testing as you then
need to think about proper synchronization again (in most cases the problem of
waiting until the desired effect had a chance to happen).

One more special aspect which is overridden for single-threaded tests is the
:meth:`receiveTimeout`, as including that would entail asynchronous queuing of
:obj:`ReceiveTimeout` messages, violating the synchronous contract.

.. warning::

   To summarize: :class:`TestActorRef` overwrites two fields: it sets the
   dispatcher to :obj:`CallingThreadDispatcher.global` and it sets the
   :obj:`receiveTimeout` to zero.

The Way In-Between
------------------

If you want to test the actor behavior, including hotswapping, but without
involving a dispatcher and without having the :class:`TestActorRef` swallow
any thrown exceptions, then there is another mode available for you: just use
the :class:`TestActorRef` as a partial function, the calls to
:meth:`isDefinedAt` and :meth:`apply` will be forwarded to the underlying
actor:

.. code-block:: scala

   val ref = TestActorRef[MyActor]
   ref.isDefinedAt('unknown) must be (false)
   intercept[IllegalActorStateException] { ref(RequestReply) }

Use Cases
---------

You may of course mix and match both modi operandi of :class:`TestActorRef` as
suits your test needs:

 - one common use case is setting up the actor into a specific internal state
   before sending the test message
 - another is to verify correct internal state transitions after having sent
   the test message

Feel free to experiment with the possibilities, and if you find useful
patterns, don't hesitate to let the Akka forums know about them! Who knows,
common operations might even be worked into nice DSLs.

Integration Testing with :class:`TestKit`
=========================================

When you are reasonably sure that your actor's business logic is correct, the
next step is verifying that it works correctly within its intended environment
(if the individual actors are simple enough, possibly because they use the
:mod:`FSM` module, this might also be the first step). The definition of the
environment depends of course very much on the problem at hand and the level at
which you intend to test, ranging for functional/integration tests to full
system tests. The minimal setup consists of the test procedure, which provides
the desired stimuli, the actor under test, and an actor receiving replies.
Bigger systems replace the actor under test with a network of actors, apply
stimuli at varying injection points and arrange results to be sent from
different emission points, but the basic principle stays the same in that a
single procedure drives the test.

The :class:`TestKit` trait contains a collection of tools which makes this
common task easy:

.. code-block:: scala

   import akka.testkit.TestKit
   import org.scalatest.WordSpec
   import org.scalatest.matchers.MustMatchers

   class MySpec extends WordSpec with MustMatchers with TestKit {

     "An Echo actor" must {

       "send back messages unchanged" in {
         
         val echo = Actor.actorOf[EchoActor].start()
         echo ! "hello world"
         expectMsg("hello world")

       }

     }

   }

The :class:`TestKit` contains an actor named :obj:`testActor` which is
implicitly used as sender reference when dispatching messages from the test
procedure. This enables replies to be received by this internal actor, whose
only function is to queue them so that interrogation methods like
:meth:`expectMsg` can examine them. The :obj:`testActor` may also be passed to
other actors as usual, usually subscribing it as notification listener. There
is a whole set of examination methods, e.g. receiving all consecutive messages
matching certain criteria, receiving a whole sequence of fixed messages or
classes, receiving nothing for some time, etc.

.. note::

   The test actor shuts itself down by default after 5 seconds (configurable)
   of inactivity, relieving you of the duty of explicitly managing it.

Another important part of functional testing concerns timing: certain events
must not happen immediately (like a timer), others need to happen before a
deadline. Therefore, all examination methods accept an upper time limit within
the positive or negative result must be obtained. Lower time limits need to be
checked external to the examination, which is facilitated by a new construct
for managing time constraints:

.. code-block:: scala

   within([min, ]max) {
     ...
   }

The block given to :meth:`within` must complete after a :ref:`Duration` which
is between :obj:`min` and :obj:`max`, where the former defaults to zero. The
deadline calculated by adding the :obj:`max` parameter to the block's start
time is implicitly available within the block to all examination methods, if
you do not specify it, is is inherited from the innermost enclosing
:meth:`within` block. It should be noted that using :meth:`expectNoMsg` will
terminate upon reception of a message or at the deadline, whichever occurs
first; it follows that this examination usually is the last statement in a
:meth:`within` block.

.. code-block:: scala

  class SomeSpec extends WordSpec with MustMatchers with TestKit {
    "A Worker" must {
      "send timely replies" in {
        val worker = actorOf(...)
        within (50 millis) {
          worker ! "some work"
          expectMsg("some result")
          expectNoMsg
        }
      }
    }
  }

.. note::

   All times are measured using ``System.nanoTime``, meaning that they describe
   wall time, not CPU time.

Ray Roestenburg has written a great article on using the TestKit:
`<http://roestenburg.agilesquad.com/2011/02/unit-testing-akka-actors-with-testkit_12.html>`_.
His full example is also available :ref:`here <testkit-example>`.

CallingThreadDispatcher
=======================

The :class:`CallingThreadDispatcher` serves good purposes in unit testing, as
described above, but originally it was conceived in order to allow contiguous
stack traces to be generated in case of an error. As this special dispatcher
runs everything which would normally be queued directly on the current thread,
the full history of a message's processing chain is recorded on the call stack,
so long as all intervening actors run on this dispatcher.

How to use it
-------------

Just set the dispatcher as you normally would, either from within the actor

.. code-block:: scala

   import akka.testkit.CallingThreadDispatcher

   class MyActor extends Actor {
     self.dispatcher = CallingThreadDispatcher.global
     ...
   }

or from the client code

.. code-block:: scala

   val ref = Actor.actorOf[MyActor]
   ref.dispatcher = CallingThreadDispatcher.global
   ref.start()

As the :class:`CallingThreadDispatcher` does not have any configurable state,
you may always use the (lazily) preallocated one as shown in the examples.

How it works
------------

When receiving an invocation, the :class:`CallingThreadDispatcher` checks
whether the receiving actor is already active on the current thread. The
simplest example for this situation is an actor which sends a message to
itself. In this case, processing cannot continue immediately as that would
violate the actor model, so the invocation is queued and will be processed when
the active invocation on that actor finishes its processing; thus, it will be
processed on the calling thread, but simply after the actor finishes its
previous work. In the other case, the invocation is simply processed
immediately on the current thread. Futures scheduled via this dispatcher are
also executed immediately.

This scheme makes the :class:`CallingThreadDispatcher` work like a general
purpose dispatcher for any actors which never block on external events.

In the presence of multiple threads it may happen that two invocations of an
actor running on this dispatcher happen on two different threads at the same
time. In this case, both will be processed directly on their respective
threads, where both compete for the actor's lock and the loser has to wait.
Thus, the actor model is left intact, but the price is loss of concurrency due
to limited scheduling. In a sense this is equivalent to traditional mutex style
concurrency.

The other remaining difficulty is correct handling of suspend and resume: when
an actor is suspended, subsequent invocations will be queued in thread-local
queues (the same ones used for queuing in the normal case). The call to
:meth:`resume`, however, is done by one specific thread, and all other threads
in the system will probably not be executing this specific actor, which leads
to the problem that the thread-local queues cannot be emptied by their native
threads. Hence, the thread calling :meth:`resume` will collect all currently
queued invocations from all threads into its own queue and process them.

Limitations
-----------

If an actor's behavior blocks on a something which would normally be affected
by the calling actor after having sent the message, this will obviously
dead-lock when using this dispatcher. This is a common scenario in actor tests
based on :class:`CountDownLatch` for synchronization:

.. code-block:: scala

   val latch = new CountDownLatch(1)
   actor ! startWorkAfter(latch)   // actor will call latch.await() before proceeding
   doSomeSetupStuff()
   latch.countDown()

The example would hang indefinitely within the message processing initiated on
the second line and never reach the fourth line, which would unblock it on a
normal dispatcher.

Thus, keep in mind that the :class:`CallingThreadDispatcher` is not a
general-purpose replacement for the normal dispatchers. On the other hand it
may be quite useful to run your actor network on it for testing, because if it
runs without dead-locking chances are very high that it will not dead-lock in
production.

.. warning::

   The above sentence is unfortunately not a strong guarantee, because your
   code might directly or indirectly change its behavior when running on a
   different dispatcher. If you are looking for a tool to help you debug
   dead-locks, the :class:`CallingThreadDispatcher` may help with certain error
   scenarios, but keep in mind that it has may give false negatives as well as
   false positives.

Benefits
--------

To summarize, these are the features with the :class:`CallingThreadDispatcher`
has to offer:

 - Deterministic execution of single-threaded tests while retaining nearly full
   actor semantics
 - Full message processing history leading up to the point of failure in
   exception stack traces
 - Exclusion of certain classes of dead-lock scenarios

