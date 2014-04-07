Probe
=========

Probe: systematic capture for dynamic program state

## Introduction

Text-based logging is an unfortunate historical artifact. A log
statement serializes a small portion of dynamic program state in a
human-readable string. While these strings are trivial to store,
route, manipulate, and inspect - in modern systems they require a
great deal of development and operations work to aggregate and
programmatically analyze.  What if we could capture and manipulate
traces of program behavior in a more accessible manner?

Moving from logs to 'probes', or capturing the state of a program
value at a point in time, enables us to attach a wide variety of
comptuational stages between the probe point and human-consumption of
that state.  This works particularly well for functional systems where
state is predominantly immutable.

This library facilitates the insertion of 'probe' points into your
source code as well as various dynamic contexts (such as state change
events or function invocations).  A probe point generates an arbitrary
'state map' of program state as well as facilities for 'subscribing' subsets
of state across the program.  A flexible tagging system allows you to
link probes together across namespaces, run them through a set of
filters or other transforms, and to any number of different
destination 'sinks'.

The state map produced by probe points can serve as logging
statements, but may also be used for a wide variety of additional
purposes such as profiling, auditing, forensics, usage statistics,
etc.  Probe also provides a simple logging API compability layer
supporting the Pedestal-style structured logging interface.
(Clojure.tools.logging compability coming soon).

How about monitoring probe state across a distributed application?
Rather than using Scribe or Splunk to aggregate and parse text
strings, fire up [Reimann](http://reimann.io) and pipe probe state to
it or use a scalable data store like HBase, MongoDB, Cassandra, or
DynamoDB where historical probes can be indexed and analyzed as
needed?  Cassandra is especially nice as you can have it automatically
expire log data at different times based, perhaps, on a priority
field.

An alternative approach to this concept is
[Lamina](https://github.com/ztellman/lamina), introduced by a [nice
talk](http://vimeo.com/45132054#!) that closely shares the philosophy
behind Probe.  I wrote probe as a minimalist replacement for logging
infrastructure in a distributed application and think it is more
accessible than Lamina, but YMMV.  The more the merrier!

## Installation

Add Probe to your lein project.clj :dependencies

```clojure
[com.vitalreactor/probe "0.9.1"]
```

And use it from your applications:

```clojure
(:require [probe.core :as p]
          [probe.sink :as sink]
          [probe.logging :as log])
```
Probe and log statements look like this:

```clojure
(p/probe [:tag] :msg "This is a test" :value 10)
(log/info :msg "This is a test" :value 10)
```

See the examples below for details on how probe state is generated by
these statements, how to create sinks, and how to route probe state to
sinks with an optional transform channel.

## Concepts

* Probe statement - Any program statement that extracts dynamic state during
  program execution. A probe will typically return some subset of
  lexical, dynamic, and/or host information as well as explicit
  user-provided data.  Probes can include function
  entry/exit/exception events as well or tie into foundational notification
  mechanisms such as adding a 'watcher probe' to an agent or atom.
   * Function probe - Default probes that can be added to any Var holding a function value
   * Watcher probe - Probe state changes on any Atom, Ref, Var, or Agent.
* Probe state - A kv-map generated by a probe statement
* Tags - Probe expressions accept one or more tags to help filter and route state.
    Built-in probe statements generate a specific set of tags.
* Sink - A function that takes probe state values and disposes them in some
    way, typically to a logging subsystem, the console, memory or a database.
* Subscriptions - This is the library's glue, consisting of a Selector, an
    optional core.async Channel, and a sink name.
* Selector - a conjunction of tags that must be present for the probe state to
    be pushed onto the channel and on to the sink.

Reserved state keys

- Probe statements: :thread-id, :tags, :ns, :line, :ts
- Expression probe: :expr, :value
- Function probes: :fname, :fn, :args, :return, :exception

Reserved tags:

- Namespace tags: :ns/*
- Function probes: :probe/fn, :probe/fn-exit, :probe/fn-enter, :probe/fn-except
- Watcher probes: :probe/watch
- Standard logging levels: :trace, :debug, :info, :warn, :error

## Documentation by Example

```clojure
(require '[probe.core :as p])
(require '[probe.sink :as sink]
(require '[core.async :as async])
```
Start with a simple console sink

```clojure
(p/add-sink :printer sink/console-raw)
```
Let's watch some test probe points:

```clojure
(p/subscribe #{:test} :printer)

(p/probe [:test] :value 10)
=> nil
{:ts #inst "2013-11-19T01:21:57.109-00:00", :thread-id 307, :ns probe.core, :tags #{:test :ns/probe.core :ns/probe}, :line 1, :value 10}
```

Probe state is only sent to the sink when the selector matches the
tags.  In fact, the entire probe expression is conditional on there
being at least one matching probe for the tags.

```clojure
(p/probe [:foo] :value 10)
=> nil
```
We can use assign transform to watch just the values and timestamp:

```clojure
(p/subscribe #{:test} :printer :transform #(select-keys % [:ts :value]))

(p/probe #{:debug} :value 10)
=> nil
{:ts #inst "2013-11-19T01:25:37.348-00:00", :value 10}
```

What subscriptions do we have now?

```clojure
(p/subscriptions)
=> ([#{:test} :printer])
```
Notice that our update clobbered the prior subscription.  We can grab
the complete subscription or sink value to get a better sense of
internals:

```clojure
(p/get-subscription #{:test} :printer)
=> {:selector #{:test}, :channel #<ManyToManyChannel clojure.core.async.impl.channels.ManyToManyChannel@382226a7>, :sink :printer, :name #{:test}, :transform #<core$mk_transform_fn$fn__5471 probe.core$mk_transform_fn$fn__5471@5be470fd>}
```

Here we see a selector which determines whether probes are submitted
at all, the channel to push the state to, the sink that channel is
connected to, and the transform that will be applied to state.

```clojure
(p/get-sink :printer)
=> {:name :printer, :function #<sink$console_raw probe.sink$console_raw@1ff3ef9>, :in #<ManyToManyChannel clojure.core.async.impl.channels.ManyToManyChannel@7f528f>, :mix #<async$mix$reify__27625 clojure.core.async$mix$reify__27625@105559f>, :out #<ManyToManyChannel clojure.core.async.impl.channels.ManyToManyChannel@13874b7>}
```
Sinks uses a core.async mix to accept inputs from multiple
subscriptions and pipes them to the sink handler function.  If you
want to do something for every state submitted to a sink, you can just
wrap the sink function when creating the sink.  For any short-term or
source specific transforms, use subscription transform channels.  For
transforms performed system wide, write your own macro that wraps the
main probe macro used above and injects whatever data you care about,
or have a standard way of generating application-specific transform
channels.

Core.async channels are composable, so you can compose a set of standard
mapping, filtering channels into what you need for a specific purpose.

Let's explore some other probing conveniences.  For example, good
functional code comes pre-packaged with some wonderful probe points
called functions.

```clojure
(defn testprobe [a b]
 (+ a b))

(p/probe-fn! #{:test} 'testprobe)

(testprobe 1 2)
=> 3

(p/subscribe #{:test} :printer) ;; stomp on our earlier filter

(testprobe 1 2)
=> 3
{:ts #inst "2013-11-19T01:36:28.237-00:00", :thread-id 321, :ns probe.core, :tags #{:test :ns/probe.core :ns/probe :probe/fn-enter}, :args (1 2), :fn :enter, :line 1, :fname testprobe}
{:ts #inst "2013-11-19T01:36:28.237-00:00", :thread-id 321, :ns probe.core, :tags #{:probe/fn-exit :test :ns/probe.core :ns/probe}, :return 3, :args (1 2), :fn :exit, :line 1, :fname testprobe}
```
We can now magically trace input arguments and return values for every
expression.  How about just focusing on the input/outputs?  We can use
some channel builders from the probe.core package to make this more concise.

```clojure
(defn args-and-value [state] (select-keys state [:args :value :fname]))
(p/subscribe #{:test :probe/fn-exit} :printer :transform args-and-value)

(map #(testprobe 1 %) (repeat 0 10))
=> (1 2 3 4 5 6 7 8 9 10)
{:fname testprobe :args (1 0) :value 1}
{:fname testprobe :args (1 1) :value 1}
...
```
So far, we've just been printing stuff out.  Not much better than
current logging solutions.  What if we want to capture these vectors,
or a function trace from from deep inside a larger system for
interactive replay at the repl?

```clojure
(def my-trace (sink/make-memory))
(p/add-sink :accum (sink/memory-sink my-trace))
(p/subscribe #{:test :probe/fn-exit} :accum :transform args-and-value)

(map #(testprobe 1 %) (range 0 10))
=> (1 2 3 4 5 6 7 8 9 10)

(sink/scan-memory)
=> ({:fname testprobe, :value 1 :args (1 0)} {:fname testprobe, :value 2 :args (1 1)} ...)

(map :value (sink/scan-memory))
=> (1 2 3 4 5 6 7 8 9 10)

(def my-trace nil)  ;; remove state from namespace for GC
(unprobe-fn! 'testprobe) ;; Remove the function probe wrapper
(p/rem-sink :accum) ;; also removes the probe subscription for you
```
We can also watch state elements like Refs and Vars by applying a transform function that generates a probe by applying the transform-fn to the new value whenever the state is changed:

```clojure
(def myatom (atom {:test 1}))
(def myref (ref {:test 1}))
(probe-state! #{:test} identity #'myref)
(probe-state! #{:test} identity #'myatom)
(p/subscribe #{:test :probe/watch} :printer)

(swap! myatom update-in [:test] inc)
=> {:test 2}
{:ts #inst "2013-11-19T19:55:03.849-00:00", :ns probe.core, :test 2, :thread-id 97, :tags #{:test :probe/watch :ns/probe.core :ns/probe}}

(dosync
(commute myref update-in [:test] inc))
=> {:test 2}
{:ts #inst "2013-11-19T19:58:24.961-00:00", :ns probe.core, :test 2, :thread-id 103, :tags #{:test :probe/watch :ns/probe.core :ns/probe}}
```
Note: probing alter-var-root operations on namespace vars is still a little shaky so don't rely on this functionality yet.

### Subscriber Transforms

You can add a transform to a subscriber to alter the state map passed to the subscribers
sink, or set sample rate, filter out unwanted data, or compose a function that combines all
of the above.

```clojure
user> (p/add-sink :example sink/console-raw) ;add sink
=> {:name :example, :function #<sink$console_raw probe.sink$console_raw@47f3ed7f>, :in #<ManyToManyChannel clojure.core.async.impl.channels.ManyToManyChannel@6b8e2779>, :mix #<async$mix$reify__4555 clojure.core.async$mix$reify__4555@41c1b019>, :out #<ManyToManyChannel clojure.core.async.impl.channels.ManyToManyChannel@13105b09>, :policy-fn #<core$policy_all probe.core$policy_all@530db0f9>}

(p/subscribe #{:transform-example} :example :transform #(assoc % :foo "bar")) ;subscribe
=> {:selector #{:transform-example}, :channel #<ManyToManyChannel clojure.core.async.impl.channels.ManyToManyChannel@5d57f865>, :sink :example, :name #{:transform-example}, :transform #<user$eval10227$fn__10228 user$eval10227$fn__10228@4eccf230>}

(p/probe [:transform-example])
=> true
{:foo "bar", :ts #inst "2014-04-06T23:39:43.206-00:00", :thread-id 157, :ns user, :tags #{:transform-example :ns/user}, :line 1}
;:foo "bar" has been added to the state map passed to the sink
```

We can use the provided helper function to create a sample function if we only want a
fraction of state maps sent to the sink.

```clojure
(p/add-sink :sample-example sink/console-raw) ;add a new sink
=> {:name :sample-example, :function #<sink$console_raw probe.sink$console_raw@47f3ed7f>, :in #<ManyToManyChannel clojure.core.async.impl.channels.ManyToManyChannel@171ba877>, :mix #<async$mix$reify__4555 clojure.core.async$mix$reify__4555@18d1287b>, :out #<ManyToManyChannel clojure.core.async.impl.channels.ManyToManyChannel@4bb8aff7>, :policy-fn #<core$policy_all probe.core$policy_all@530db0f9>}

;lets take every 3rd state map
(p/subscribe #{:sample-tag} :sample-example :transform (p/mk-sample-fn 3))
=>{:selector #{:sample-tag}, :channel #<ManyToManyChannel clojure.core.async.impl.channels.ManyToManyChannel@bd35aa2>, :sink :sample-example, :name #{:sample-tag}, :transform #<core$sampler_fn$fn__9747 probe.core$sampler_fn$fn__9747@697b3ca3>}

(p/probe [:sample-tag])
=>true
(p/probe [:sample-tag])
=>true
(p/probe [:sample-tag])
=>true
{:ts #inst "2014-04-06T23:45:00.541-00:00", :thread-id 165, :ns user, :tags #{:ns/user :sample-tag}, :line 1}
(p/probe [:sample-tag])
=>true
(p/probe [:sample-tag])
=>true
(p/probe [:sample-tag])
=>true
{:ts #inst "2014-04-06T23:45:09.109-00:00", :thread-id 165, :ns user, :tags #{:ns/user :sample-tag}, :line 1}
;Every third state map, as expected.
```


### Data de-duplication policy

It is possible for multiple subscribers to send the same state map to the same sink,
resulting in duplicates.  By default all state maps are passed to a given sink, however
we can set a policy on the sink if we want to alter that behavior.  Three ready made
policies exist and are as follows:

 * `:all` -- default policy setting.  All state maps passed.
 * `:unique` -- A copy of each distinct state map is passed to the sink.
 * `:first` -- the first non nil state map found is passed to the sink.

You can optionally create your own policy function, provided that it returns a list
of maps of the form `{:sub <subscriber> :new-state <state after (:transform sub) is applied>}` and takes two args, state (map) and subscribers (seq of subscribers).

The policy on a sink can can be switched at any time with `probe.core/swap-sink-policy!`.

Lets walk through an example.

First, set up a console sink and add some subscribers.

```clojure
(p/add-sink :console sink/console-raw)

=> {:name :console, :function #<sink$console_raw probe.sink$console_raw@47f3ed7f>, :in #<ManyToManyChannel clojure.core.async.impl.channels.ManyToManyChannel@4dc834d6>, :mix #<async$mix$reify__4555 clojure.core.async$mix$reify__4555@1304f57f>, :out #<ManyToManyChannel clojure.core.async.impl.channels.ManyToManyChannel@2a916e9a>, :policy-fn #<core$policy_all probe.core$policy_all@3b7359cb>}

;add some subscribers

(p/subscribe #{:policy-ex1} :console)
=> {:selector #{:policy-ex1}, :channel #<ManyToManyChannel clojure.core.async.impl.channels.ManyToManyChannel@28657778>, :sink :console, :name #{:policy-ex1}, :transform #<core$identity clojure.core$identity@a8bed44>}

(p/subscribe #{:policy-ex2} :console :transform #(assoc % :assoced "value"))
=>{:selector #{:policy-ex2}, :channel #<ManyToManyChannel clojure.core.async.impl.channels.ManyToManyChannel@d9be234>, :sink :console, :name #{:policy-ex2}, :transform #<user$eval9211$fn__9212 user$eval9211$fn__9212@7a41fe1c>}

(p/subscribe #{:policy-ex3} :console)
=> {:selector #{:policy-ex3}, :channel #<ManyToManyChannel clojure.core.async.impl.channels.ManyToManyChannel@7dbd91bc>, :sink :console, :name #{:policy-ex3}, :transform #<core$identity clojure.core$identity@a8bed44>}
```

Now we have three subscribers all writing to the same sink.

```clojure
(p/probe [:policy-ex1 :policy-ex2 :policy-ex3])
=> true

{:ts #inst "2014-04-06T23:05:25.124-00:00", :thread-id 125, :ns user, :tags #{:policy-ex1 :policy-ex2 :policy-ex3 :ns/user}, :line 1}
{:ts #inst "2014-04-06T23:05:25.124-00:00", :thread-id 125, :ns user, :tags #{:policy-ex1 :policy-ex2 :policy-ex3 :ns/user}, :line 1}
{:assoced "value", :ts #inst "2014-04-06T23:05:25.124-00:00", :thread-id 125, :ns user, :tags #{:policy-ex1 :policy-ex2 :policy-ex3 :ns/user}, :line 1}
```
As expected, all the state maps from our subscribers were passed through to the sink.
Lets change `:console` to unique.

```clojure
(p/swap-sink-policy! :console :unique)
{:policy-fn #<core$policy_unique probe.core$policy_unique@484c3f0b>, :name :console, :function #<sink$console_raw probe.sink$console_raw@47f3ed7f>, :in #<ManyToManyChannel clojure.core.async.impl.channels.ManyToManyChannel@4dc834d6>, :mix #<async$mix$reify__4555 clojure.core.async$mix$reify__4555@1304f57f>, :out #<ManyToManyChannel clojure.core.async.impl.channels.ManyToManyChannel@2a916e9a>}

(p/probe [:policy-ex1 :policy-ex2 :policy-ex3])
=>true
{:ts #inst "2014-04-06T23:14:51.133-00:00", :thread-id 136, :ns user, :tags #{:policy-ex1 :policy-ex2 :policy-ex3 :ns/user}, :line 1}
{:assoced "value", :ts #inst "2014-04-06T23:14:51.133-00:00", :thread-id 136, :ns user, :tags #{:policy-ex1 :policy-ex2 :policy-ex3 :ns/user}, :line 1}
```

This time we only received two results.  The sink policy removed one of the two identical
state maps.

We can also set sink policy at sink creation

```clojure
(p/add-sink :new-console sink/console-raw :policy :unique)
=>{:name :new-console, :function #<sink$console_raw probe.sink$console_raw@47f3ed7f>, :in #<ManyToManyChannel clojure.core.async.impl.channels.ManyToManyChannel@2191fa12>, :mix #<async$mix$reify__4555 clojure.core.async$mix$reify__4555@76b8c4f5>, :out #<ManyToManyChannel clojure.core.async.impl.channels.ManyToManyChannel@5c8aedb7>, :policy-fn #<core$policy_unique probe.core$policy_unique@484c3f0b>}
```

Lets create our own policy and pass it. Lets create an easy function to stop any
state maps from reaching the sink by just passing back an empty list.

```clojure
user> (def off (fn [_ _] '()))
#'user/off
(p/swap-sink-policy! :console off)
=>{:policy-fn #<user$off user$off@98b1fdb>, :name :console, :function #<sink$console_raw probe.sink$console_raw@47f3ed7f>, :in #<ManyToManyChannel clojure.core.async.impl.channels.ManyToManyChannel@4dc834d6>, :mix #<async$mix$reify__4555 clojure.core.async$mix$reify__4555@1304f57f>, :out #<ManyToManyChannel clojure.core.async.impl.channels.ManyToManyChannel@2a916e9a>}

(p/probe [:policy-ex1 :policy-ex2 :policy-ex3])
=> true  ;;no state being passed to :console ! lets set it to :first and at least get one

(p/swap-sink-policy! :console :first)
=> {:policy-fn #<core$policy_first probe.core$policy_first@586f0f11>, :name :console, :function #<sink$console_raw probe.sink$console_raw@47f3ed7f>, :in #<ManyToManyChannel clojure.core.async.impl.channels.ManyToManyChannel@4dc834d6>, :mix #<async$mix$reify__4555 clojure.core.async$mix$reify__4555@1304f57f>, :out #<ManyToManyChannel clojure.core.async.impl.channels.ManyToManyChannel@2a916e9a>}

(p/probe [:policy-ex1 :policy-ex2 :policy-ex3])
=>true
{:ts #inst "2014-04-06T23:27:46.080-00:00", :thread-id 150, :ns user, :tags #{:policy-ex1 :policy-ex2 :policy-ex3 :ns/user}, :line 1}
```



Other implemented features we'll document soon:

- Using the logging namespace
- Capturing streams of exceptions for post-analysis
- Capturing dynamic bindings in probe points
- Sampling transform channels
- Using log sinks
- Creating a database sink example
- Other memory options (like a sliding window queue)
- Setting / unsetting probes on all publics in a namespace, etc
- Low level access via write-state

## Discussion

I moved to core.async in 0.9.0 because it provides a sound
infrastructure for assembling functions in topologies to operate over
streams of events.  I've picked a simple di-graph topology to keep
things simple for interactive use, but it would only take a little
extra code to support more complex DAG-style topologies.

## Future Work

Here are some opportunities to improve the library.

### Future Tasks (Minor)

* Add higher level channel constructor support
* Add a clojure EDN file sink
* Record the stack state at a probe point
* Add higher level targeted function tracing / collection facilities
  (e.g. trace 100 input/result vectors from function f or namespace ns)
* Add metadata so we can inspect what functions are probed
* Add a Reimann sink as probe.reimann usin the Reimann clojure client

### Major Tasks

* ~~Deduplication.  It's easy to create multiple paths to the same sink; how do we
     handle (or do we) deduplication particularly when subscription channels
     may transform the data invalidating an = comparison?~~
     state is dropped onto an async routing channel where all trasforms,
     filters, and sampling fns are evaluated, deduplicated (if applicable), and then
     passed to the appropriate subscribers channels and ultimately the subscribers sink.
* Complex topologies.  Right now we have a single transforming channel between
     a selector and a sink.  What if we wanted to share functionality across
     streams?  How would we specify, wire up, and control a more complex topology?
* Injest legacy logging messages - Most systems will have legacy libraries that
     use one of the Java logging systems.  Create some namespaces that
     allow for injecting these log messages into clj-probe middleware.  Ignore
     any that we inject using the log sink.  This may be non-trivial.
* Adapt the function probes to collect profile information
