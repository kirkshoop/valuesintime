
---
Values Distributed In Time
---

Document number: 

Date: 2017-06-19

Audience: Library Working Group

Reply-to:  Kirk Shoop kirk.shoop{at}gmail[dotcom]

# I. Table of Contents
 * [I. Table of Contents](#i-table-of-contents)
 * [II. Introduction](#ii-introduction)
 * [III. Motivation and Scope](#iii-motivation-and-scope)
    * [negative space](#negative-space)
    * [implementation vs concept](#implementation-vs-concept)
    * [safe composition and coordination](#safe-composition-and-coordination)
    * [algorithms](#algorithms)
    * [concepts hidden in the current proposals](#concepts-hidden-in-the-current-proposals)
    * [lifetime](#lifetime)
    * [virtuous procrastination](#virtuous-procrastination)
    * [Single](#Single)
    * [time and coordination](#time-and-coordination)
    * [many are greater than the sum](#many-are-greater-than-the-sum)
    * [Legion](#Legion)
 * [IV. Impact On the Standard](#iv-impact-on-the-standard)
 * [V. Acknowledgments](#v-acknowledgments)

# II. Introduction

A lot of `promise`s have been made. Some `promise`s have been broken. However, there are more ways of representing a `future` value than have yet been dreamt of in C++ proposals.

Please step away from the variety of `promise`d implementations of a `future` value and journey instead through the algorithms for and producers of, values-distributed-in-time that will reveal the shape of the missing pieces that will bind them together. In this way, we tease fate towards the same happy outcome we enjoyed when the definition of the various kinds of; `Iterator` along with algorithms and containers, successfully abstracted values-distributed-in-space. Perhaps we can even skip right to the composability provided by the kinds of `Range` winding their way to the standard.

Too long have we mixed the container, algorithms and value into ungainly `promise`s and `future`s until they look like `string` or `iostream`. Please explore a brighter path below..

# III. Motivation and Scope

## negative space

The STL added _Iterators_, _Containers_ and algorithms together. In addition, the _Iterators_ and _Containers_ exceeded the [Rule of Twos](https://medium.com/capital-one-developers/rule-of-twos-and-microservice-architecture-3f57db7f6896) by including more than one _Container_ implementation and _Iterator_ implementation. This, in turn, positively affected the algorithm design. The negative space between various _Container_ implementations and the algorithms exposed the shape of the several _Iterator_ needed.

A `promise` replacement should fill the negative space between multiple sources of values-distributed-in-time and a set of algorithms. Things like ux event callbacks, network callbacks, task scheduler callbacks, polling loops, event signals, interrupt signals, must all be supported behind a common interface that is also sufficient to build the algorithms.

## implementation vs concept

The [Rule of Twos](https://medium.com/capital-one-developers/rule-of-twos-and-microservice-architecture-3f57db7f6896) induced _Containers_ and _Iterators_ to be concepts instead of a single implementation. Conflict over implementation details of a `promise` replacement would be avoided if concepts that can be implemented with different strategies are specified instead of a single implementation.

## safe composition and coordination

Today there are as many patterns for delivering values-distributed-in-time in C/C++ as there are libraries. The effort to compose these together and safely integrate them with raw `thread`, `mutex`, `condition_variable` and `atomic` usage always results in a '`future`' of periodic wailing and gnashing of teeth.

In a brighter world, a standard shape and surface for values-distributed-in-time is defined. Many safe coordination algorithms are built. The algorithms support safe composition and are easily shared. Productivity increases vastly.

~~~C++
   http.get(url) |
      map([](auto r){return r.body;}) |
      switch_on_next() |
      retry(3) |
      timeout(10s) |
      observe_on(main_thread) |
      subscribe(
          [](const auto& c){cout << c;},
          [](exception_ptr ep){cerr << endl << what(ep) << endl;});
~~~

> `operator|` is used to compose algorithms to match the range-v3 composition pattern.

## algorithms

The algorithms create most of the negative space that a replacement for `promise` needs to fill.

__existing__

Some algorithms are embedded into the `promise` implementation proposals. This makes it impossible to add or extend the set of algorithms.

*   `then` is overloaded to mean both `map()` and `flat_map()` from other libraries
*   `when_all` is called `zip` in other libraries
*   `when_any` is called `amb` in other libraries
*   `make_ready_future` is called `just` in other libraries
*   `make_exceptional_future` is called `error` in other libraries

__missing__

Some of these missing algorithms change the shape of the negative space in ways that no existing `promise` proposal can fill.

*   `observe_on` - [reactivex.io](http://reactivex.io/documentation/operators/observeon.html)
*   `delay` - [rxmarbles.com](http://rxmarbles.com/#delay)
*   `finally`
*   `catch` - [reactivex.io](http://reactivex.io/documentation/operators/catch.html)
*   `replay` - [reactivex.io](http://reactivex.io/documentation/operators/replay.html)
*   `subscribe_on` - [reactivex.io](http://reactivex.io/documentation/operators/subscribeon.html)
*   `take_until` - [rxmarbles.com](http://rxmarbles.com/#takeUntil)
*   `timeout` - [reactivex.io](http://reactivex.io/documentation/operators/timeout.html)
*   `timestamp` - [reactivex.io](http://reactivex.io/documentation/operators/timestamp.html)
*   `never`
*   `timer`

## concepts hidden in the current proposals

`promise`/`future` is similar to `subject` in other libraries. The `promise` presents two faces, one is used by the producer and the other by the consumer.

The existing `promise` proposals can be represented as implementations of the following concepts

~~~C++
template<typename T>
struct Promise
{
    void value(T t) const;

    void error(exception_ptr e) const;
};

template<typename T>
struct Future 
{
    void then(Promise<T>) const;
};
~~~

This may not seem to match the interface of any proposed `promise`. It is perhaps confusing to see the same _Promise_ concept used for both producing the data and consuming the data. Yet, the only difference between consumption and production of data in a push-model is the implementation - the concept is shared. 

The data using the more familiar pull-model, requires independent methods for get and set of each piece of data. 

The data using a push-model requires only set methods. The set methods in a push-model are implemented twice. Once for production and again for consumption. 

The uniformity of the push-model provides natural composition since producers and consumers can be layered and interchanged at will.

To see how these concepts can be used to build a `promise`, see the following partial implementations of `create` and `then` for a hypothetical `promise`.

~~~C++
template<typename T>
struct type_erased_promise // model of Promise
{
    virtual void value(T v);
    virtual void error(exception_ptr e);
    // type-erase implementation omitted
};

template<typename T>
struct shared // model of Promise and model of Future
{
    void value(T v) {
        // if state is a promise, call value(v) on that promise
        // set state to v
    }
    void error(exception_ptr e) {
        // if state is a promise, call error(e) on that promise
        // set state to e
    }
    template<typename Promise>
    void then(Promise p) {
        // if state is T, call p.value(v)
        // else if state is exception_ptr, call p.error(e)
        // else set state to type-erased p
    }
    shared_ptr<variant<T, exception_ptr, type_erased_promise<T>>> state;
};

template<typename T>
struct promise // model of Promise
{
    void value(T v) {
        s.value(v);
    }
    void error(exception_ptr e) {
        s.error(e);
    }
private:
    shared<T> s;
};

template<typename T>
struct future // model of Future
{
    template<typename Promise>
    void then(Promise p) {
        s.then(p);
    }
private:
    shared<T> s;
};

template<typename T, typename Fn>
future<T> create(Fn fn) {
    shared<T> s;
    fn(promise<T>{s});
    return future<T>{s};
}

template<typename Future, typename T, typename Fn, typename R = . . .>
auto then(Future f, Fn fn) {
    shared<R> s;
    promise<R> r{s};
    type_erased_promise<T> p{
        [r](T v) {r.value(fn(v));}, 
        [r](exception_ptr e){r.error(e);}};
    f.then(p);
    return future<R>{s};
}
~~~

Now other algorithms can be built on this implementation of the concepts. Since `catch` is a C++ keyword and `on_error_resume_next` is too long, this algorithm is named `fallback` instead.

~~~C++
template<typename Future, typename T, typename Fn>
auto fallback(Future f, Fn fn) {
    shared<T> s;
    promise<T> r{s};
    type_erased_promise<T> p{
        [r](T v) {r.value(v);}, 
        [r](exception_ptr e){fn(e).then(r);}};
    f.then(p);
    return future<T>{s};
}
~~~

This is a nice step towards the light. `fallback` is a transform on the error just like `then` is a transform on the value. Other algorithms require changes to the concepts. Brighter lights ahead!

## lifetime 

Many algorithms need to have a signal that indicates that the lifetime has ended. This signal is used to cancel, close, and free resources. There is no C++ destructor that can provide this signal due to the requirement that producers and consumers share state. The producers indicate the end of the lifetime by calling `value()` or `error()` on the _Promise_. Many consumers also need to send a signal to end the lifetime early. 

Given that there are multiple signals and that both producers and consumers may send them, handling the end-of-lifetime signal is always a race.

> `timeout`, `take_until` and `when_any`/`amb` are all examples of consumers that need to signal early termination. 

The minimal _Lifetime_ concept is simple. 

~~~C++
struct Lifetime
{
    bool is_stopped() const;
    void stop() const;
};
~~~

Due to the inherent race condition

* calls to `stop()` after the first one are ignored
* `is_stopped() == true` does __not__ mean that the _Lifetime_ is in-scope
* `is_stopped() == false` does mean that the _Lifetime_ is out-of-scope

> Adding in an arena allocator would allow for algorithms to efficiently keep state for the current scope without using a lot of `shared_ptr<>`.

## virtuous procrastination 

Many algorithms need to be able to defer the work to start later or to restart later or to start in a different context.

> `subscribe_on` and `replay` are examples of consumers that need to control when and how the work is started. 

## Single

In an attempt to avoid potentially loaded terms like; _Promise_, _Observer_ and _Observable_, different nouns will be used to define concepts that fill the negative space.

~~~C++
struct Single
{
    template<typename T>
    void value(T&& t) const;

    template<typename E>
    void error(E&& e) const;
};

struct SingleSubscription
{
    Lifetime lifetime;
    Single destination;
};

struct SingleDeferred
{
    Lifetime subscribe(SingleSubscription) const;
};
~~~

The concept definitions also have additional operational constraints. These are added to make usage and implementation easier to reason about.

* _Single_ implementations are guaranteed one of the following
    * `value()` will be called once
    * `error()` will be called once
    * neither will be called.
* _SingleDeferred_ implementations must support multiple calls to `subscribe()`. This can
    * repeat the same values to each _SingleSubscription_ even when the lifetimes do not overlap in time. This is referred to as COLD.
    * allow multiple _SingleSubscription_ to share the same values while their lifetimes overlap in time. This is referred to as multicast or HOT. 
* `Single::next()` can be called with different types at different times. 
* `Single::error()` can be called with different types at different times. 

> Examples of producers that are naturally HOT include; mouse events, sensor measurements and hardware random number device.

> Examples of producers that are naturally COLD include; http requests, file reads and timers.

A COLD producer can be made HOT using the `publish()` algorithm.

A HOT producer can be made COLD using the `replay()` algorithm.

## time and coordination

Many algorithms need to sample a clock and coordinate signals and time.

> `subscribe_on`, `observe_on`, `delay`, `take_until`, `timestamp` and `timeout` are examples of consumers that need to coordinate signals and time. 

~~~C++
struct Clock
{
    time_point now() const;
};

struct Repeater
{
    Lifetime lifetime;
    Clock clock;
    void repeat_at(time_point at) const;
};

struct ActionDeferred 
{
    virtual void act(Repeater);
};

struct Strand
{
    Lifetime lifetime;
    Clock clock;
    virtual void defer_at(time_point at, ActionDeferred what) const;
};

struct Coordinator
{
    Clock clock;
    Strand create_strand(Lifetime) const;
};
~~~

* _Clock_ is an instance with a member `now()` function. In addition to the default clock (that just calls through to `system_clock` or `steady_clock`) a virtual-time clock can be built that has member functions that explicitly advance the value returned from `now()`. One virtual-time clock is the `test_clock` that is used by the `test_coordinator` to run time-sensitive tests that are reproducible and quick.
* _Strand_ is a work queue that guarantees 
    * only one _ActionDeferred_ will run at a time 
    * all _ActionDeferred_ given the same `time_point` value will run in FIFO order.
* _Coordinator_ is a _Strand_ factory and owns the clock instance that is shared across all the _Strand_ that it creates.
* _ActionDeferred_ is a function that does the work and takes a _Repeater_
* _Repeater_ allows the current _ActionDeferred_ to `repeat_at()` a new time. More than just a convenience, the implementation of `repeat_at()` can prevent unneeded queue push/pop, avoid virtual function calls and prevent stack recursion.

Concurrency is optional. _Coordinator_ implementations allow concurrency to be introduced. 
 * `subscribe_on(Coordinator)` will run the producer and `Lifetime::stop()` on a newly created _Strand_
 * `observe_on(Coordinator)` will run the consumer on a newly created _Strand_
 * `take_until(Coordinator, SingleDeferred trigger)` will run the consumers of both the trigger and the parent _SingleDeferred_ on one newly created _Strand_, which means that the `take_until` implementation does not use any raw concurrency primitives. 
 
 The algorithms are intentionally non-thread-safe by default. Thread safety adds overhead and is therefore added only as needed.

## many are greater than the sum

_Single_ is great, but there are many more algorithms to add when there is more than a single value in the _Lifetime_ of a subscription.

*   `buffer` - [reactivex.io](http://reactivex.io/documentation/operators/buffer.html)
*   `combine_latest` - [rxmarbles.com](http://rxmarbles.com/#combineLastest)
*   `concat` - [rxmarbles.com](http://rxmarbles.com/#concat)
*   `concat_map`
*   `debounce` - [rxmarbles.com](http://rxmarbles.com/#debounce)
*   `distinct` - [rxmarbles.com](http://rxmarbles.com/#distinct)
*   `distinct_until_changed` - [rxmarbles.com](http://rxmarbles.com/#distinctUntilChanged)
*   `element_at` - [rxmarbles.com](http://rxmarbles.com/#elementAt)
*   `filter` - [rxmarbles.com](http://rxmarbles.com/#filter)
*   `flat_map` - [reactivex.io](http://reactivex.io/documentation/operators/flatmap.html)
*   `group_by` - [reactivex.io](http://reactivex.io/documentation/operators/groupby.html)
*   `ignore_elements` - [reactivex.io](http://reactivex.io/documentation/operators/ignoreelements.html)
*   `map` - [rxmarbles.com](http://rxmarbles.com/#map)
*   `merge` - [rxmarbles.com](http://rxmarbles.com/#merge)
*   `pairwise`
*   `publish` - [reactivex.io](http://reactivex.io/documentation/operators/publish.html)
*   `reduce` - [rxmarbles.com](http://rxmarbles.com/#reduce)
*   `repeat`
*   `sample` - [rxmarbles.com](http://rxmarbles.com/#sample)
*   `scan` - [rxmarbles.com](http://rxmarbles.com/#scan)
*   `sequence_equal` - [reactivex.io](http://reactivex.io/documentation/operators/sequenceequal.html)
*   `skip` - [rxmarbles.com](http://rxmarbles.com/#skip)
*   `skip_last` - [rxmarbles.com](http://rxmarbles.com/#skipLast)
*   `skip_until` - [rxmarbles.com](http://rxmarbles.com/#skipUntil)
*   `start_with` - [rxmarbles.com](http://rxmarbles.com/#startWith)
*   `switch_if_empty` - [reactivex.io](http://reactivex.io/documentation/operators/switch.html)
*   `switch_on_next` - [reactivex.io](http://reactivex.io/documentation/operators/switch.html)
*   `take` - [rxmarbles.com](http://rxmarbles.com/#take)
*   `take_last` - [rxmarbles.com](http://rxmarbles.com/#takeLast)
*   `take_until` - [rxmarbles.com](http://rxmarbles.com/#takeUntil)
*   `take_while` - [reactivex.io](http://reactivex.io/documentation/operators/takewhile.html)
*   `time_interval` - [reactivex.io](http://reactivex.io/documentation/operators/timeinterval.html)
*   `window` - [reactivex.io](http://reactivex.io/documentation/operators/window.html)
*   `with_latest_from` - [rxmarbles.com](http://rxmarbles.com/#withLatestFrom)

## Legion

> My name is Legion .. because we are many

~~~C++
struct Legion
{
    template<typename T>
    void value(T&& t) const;

    template<typename E>
    void error(E&& e) const;

    void complete() const;
};

struct LegionSubscription
{
    Lifetime lifetime;
    Legion destination;
};

struct LegionDeferred
{
    Lifetime subscribe(LegionSubscription) const;
};
~~~

These concepts allow a call to `subscribe()` to deliver 0, 1 or N values, followed by an error or a completion signal.

_Legion_ allows for many additional algorithms and many additional producers. An http Request may produce a single http Response, but within that response there can be many http Headers and many lines of text in the http body. These concepts allow these nested value producers to be implemented and composed.

# IV. Impact On the Standard

The concepts, implementations and algorithms would be all new text in the standard.

# V. Acknowledgments

Erik Meijer, Gor Nishanov, David Sankel, Ben Christensen, Aaron Lahman, etc..
