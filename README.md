Probe
=========

Probe: systematic capture for dynamic program state

## Introduction

Text-based logging infrastructure is an unfortunate historical
artifact. A log statement captures some aspect of dynamic program
state in a human-readable form. While these log strings are easy to
store, route, manipulate, and inspect - they take much more work to
programatically analyze and to connect back to the context in which
they occurred.  Modern systems are more than capable of “late binding”
the conversion of state from a native internal representation to a
human-readable format and provide facilities for processing streams of
state generated by program 'probe points' for many other purposes such
as profiling, auditing, forensics, etc.

[http://ianeslick.com/probe-a-library](Read the introductory post)

## Concepts

* Probe statement - Any program statement that extracts dynamic state during
  program execution. A probe will typically return a subset of
  lexical, dynamic, and host information as well as explicite
  user-provided data.  Probes can include function
  entry/exit/exceptions as well or tie into foundational notification
  mechanisms such as adding a 'watcher probe' to an agent or atom.
   * Function probe - Default probes that can be added to any Var with a function value
   * Log probe - Inject legacy library string logs into the probe system
* State - A map returned by a Probe statement
* Policy - A keyword name assigned to one or more operations that
  transform the probe State.  A policy is a singleton, seq of fns, seq
  of fn symbols, and/or other keywords naming a policy.  Policy seqs
  operate much like ring middleware, but are easier to modify at
  runtime to capture / ignore various subsets of state.
  * Sink - A function that takes a State element and routes it to some
    external system, typically media such as a File, a database, the Console, etc.
    and returns a null value indicating the policy chain is complete.
* Catalog - Global map storing policies
* Configuration - maps namespace + tags to a top-level policy, dynamically updatable

A policy function takes a single map, and returns an updated map.  Sinks return nil,
terminating policy execution.

Reserved keys:

- All probes: :msg, :ts, :ns, :line, :thread-id, :host, :tags
- Function probes: :fname, :fn, :args, :return, :exception

Reserved tags:

- Standard log hierarchy: :trace, :debug, :info, :warn, :error
- Function probes: :fn, :exit-fn, :enter-fn, :except-fn

Common user tags:

- causeid - Tyipcally a UUID used to filter/index a set of log
  statements related to a single high level action/event in a larger
  system.
- 

## Documentation by Example

    (use '[probe.core :as p])
	(use '[probe.sink :as sink])
	(use '[probe.policy :as policy])

Define a simple policy to print the raw state in a log-like format

	(p/set-policy! :console '[sink/console-log])

For probe points with :debug tags or above, handle them with the :console policy

    (p/set-config! 'user :debug :console)
    
    (p/probe #{:warn} :value 10)
	=> nil

    (p/probe #{:debug} :value 10)
    2013-05-10T22:59.212 user:1 {:thread-id 80 :value 10}
	=> nil

But we don't want the thread id, so let's clean up our default console pipeline

    (p/set-policy! :console ['(dissoc :thread-id) `sink/console-log])

    (p/probe #{:debug} :value 10)
    2013-05-10T22:59.212 user:1 {:value 10}
	=> nil

What does our global configuration look like now?

    (clojure.pprint/pprint @p/policies)
	=> {:console [(dissoc :thread-id) sink/console-log]
        :default [#<core$identity clojure.core$identity@789df8c>]}

    (clojure.pprint/pprint @p/config)
    => {user {#{:debug} :console}}

So we can create probe points anywhere in our code, and do anything we want
with the combination of lexical and dynamic context.  Of course good functional
programs already have wonderful probe points available, they're called functions.

    (def testprobe [a b]
	  (+ a b))

    (p/probe-fn! 'testprobe)

	(testprobe 1 2)
    => 3

    (p/set-config! 'user :enter-fn :console)

	(testprobe 1 2)
	2013-05-10T23:28.627 user:1 {:args (1 2), :fn :enter, :fname testprobe}
	=> 3

	(p/remove-config! 'user :enter-fn)
    (p/set-config! 'user :fn :console)
	(defn testprobe [a b] (+ 1 a b))
	(testprobe 1 2)
    2013-05-10T23:30.522 user:1 {:args (1 2), :fn :enter, :fname testprobe}
    2013-05-10T23:30.522 user:1 {:return 4, :args (1 2), :fn :exit, :fname testprobe}
	=> 3 ;; Wrapping survives redefinition

	(p/remove-config! 'user :fn)
    (p/set-config! 'user :exit-fn
       '[(select-keys [:fname :return :args]) sink/console-raw])

    (testprobe 1 2)
    {:fname testprobe, :args (1 2), :return 4}
    => 4

This demonstrates generating test vectors from runtime for any
function simply by probing the function and appropriately filtering
the result.  What if we want to capture these vectors into memory for
replay later?

    (def my-tests (atom nil))
    (p/set-config! 'user :exit-fn
	   '[(select-keys [:fname :return :args]) (sink/memory my-tests)])
    
    (testprobe 1 4) 
    (testprobe 1 5)
    => 7

    @my-tests
    ({:args (1 5), :return 7, :fname testprobe} {:args (1 4), :return 6, :fname testprobe})

We can also watch state elements like Refs and Vars:

    (def myatom {:test 1})
    (probe-state! identity myatom)
    (set-config! 'user :state
      [sink/console-raw])

    (swap! myatom update-in [:test] inc)
    => {:test 2}
    => {:ts 1384410782190, :ns user, :test 2, :thread-id 457, :tags [:state]}

Of course a flood of maps could get overwhelming in a real system,
even if you just turn on this ability for a short while.  We can
capture a random sub-sample of the traced data:

    (p/set-config! 'user :exit-fn
	   '[(policy/random-sample 0.01)
         (select-keys [:fname :return :args]) 
         (sink/memory my-tests)])

We can used a fixed length in-memory queue to keep the last N items

    (def my-tests (sink/make-memory))
    (p/set-config! 'user :exit-fn
	   '[(policy/random-sample 0.01)
         (select-keys [:fname :return :args]) 
         (sink/fixed-memory my-tests 5)])

    (dotimes [i 10]
      (testprobe i 10))

    (clojure.pprint/pprint (seq @my-tests))
    => ({:args (5 10), :return 16, :fname testprobe} 
        {:args (7 10), :return 17, :fname testprobe} 
        {:args (8 10), :return 18, :fname testprobe} 
        {:args (9 10), :return 19, :fname testprobe} 
        {:args (10 10), :return 20, :fname testprobe})

We can also select for specific functions in the namespace:
        
    (p/set-config! 'user :exit-fn
	   '[(policy/select-fn testprobe)
         (select-keys [:fname :return :args]) 
         (sink/memory my-tests)])
    
Or only collect data when there are failures:

    (p/remove-config! 'user :exit-fn)
    (p/set-config! 'user :except-fn
	   '[(select-keys [:ts :fname :return :args])
         sink/console-raw])

    (reset! my-tests (sink/make-memory))

    (defn testprobe [a b] (if (= a 2) (throw (Exception. "oops")) (+ a b 1)))
    (testprobe 1 3)
    => 5
	(testprobe 2 3)
    {:args (2 3), :fname testprobe}
    ; Evaluation aborted.

Finally, what if you just want to write a log in the (almost) traditional way?

    (use '[probe.logging :as log])
    (log/error :msg "This is an error" :exception e :value 10)

This uses the Pedestal convention promoted by Relevance, but captures
the structured data before converting it to a string.  This statement
is equivalent to:

    (probe [:error] :msg "This is an error" :exception e :value 10)

except that the probe statement is only called if the underlying
logger for that namespace is active.

## Discussion

One of my internal debates in the first version of this was whether to use
classic ring middleware that uses function composition rather than linear
sequences of symbols or the use of named policies to do composition.  It's
obviously more flexible, but it's also harder to reason about.  

## Future Work

Documented here are some opportunities for future work.

### Future Tasks (Minor)

* Add a policy to extract a clean stack trace from an exception
* Grab the stack state from a probe point (for profiling later)
* Add a simple dump of clojure data to a logfile, walk the data
  structure to ensure nothing unserializable causes errors?
* Add higher level targeted function tracing / collection facilities
  (e.g. trace 100 input/result vectors from function f or namespace ns)
* Better seatbelts in case of errors in policy functions, etc.
* Finish the rule-based configuration system allowing for more flexible
  enabling / disabling of different probe points in policy-rules.clj

### Future Tasks (Major)

* Performance.  There are quite a few places where the performance of
  the probe library can be improved.
* Probe log messages - Most systems will have legacy libraries that
     use one of the Java logging systems.  Create some namespaces that
     allow for injecting these log messages into clj-probe middleware.
* Adapt the function probes to also collect profile information

   	 