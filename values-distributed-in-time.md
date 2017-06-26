
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
    * [safe composition and coordination](#safe-composition-and-coordination)
    * [lifetime](#lifetime)
    * [algorithms](#algorithms)
 * [IV. Impact On the Standard](#iv-impact-on-the-standard)
 * [V. Acknowledgments](#v-acknowledgments)

# II. Introduction

A lot of `promise`s have been made. Some `promise`s have been broken. However, there are more ways of representing a `future` value than have yet been dreamt of in C++ proposals.

Please step away from the variety of `promise`d implementations of a `future` value and journey instead through the algorithms for and sources of, values-distributed-in-time that will reveal the shape of the missing pieces that will bind them together. In this way, we tease fate towards the same happy outcome we enjoyed when the definition of the various kinds of; `Iterator`, algorithms and containers, successfully abstracted values-distributed-in-space. Perhaps we can even skip right to the composibility provided by the kinds of `Range` winding their way to the standard.

Too long have we mixed the container, algorithms and value into ungainly `promise`s and `future`s until they look like `string` or `iostream`. Please explore a brighter path..

# III. Motivation and Scope

## negative space

The STL added Iterators, Containers and algorithms together. In addition, the Iterators and Containers exceeded the [Rule of Twos](https://medium.com/capital-one-developers/rule-of-twos-and-microservice-architecture-3f57db7f6896) by including more than one Container implementation and Iterator implementation. This, in turn, positively affected the algorithm design. The negative space between various Container implementations and the algorithms exposed the shape of the Iterators needed.

A `promise` replacement should fill the negative space between multiple sources of values-distributed-in-time and a set of algorithms. Things like ux event callbacks, network callbacks, task scheduler callbacks, polling loops, event signals, interrupt signals, must all be supported behind a common interface that is also sufficient to build the algorithms.

## implementation vs concept

The [Rule of Twos](https://medium.com/capital-one-developers/rule-of-twos-and-microservice-architecture-3f57db7f6896) induced Containers and Iterators to be concepts instead of a single implementation. Conflict over implementation details of a `promise` replacement would be avoided if concepts that can be implemented with different strategies are specified instead of a single implementation.

## safe composition and coordination

Today there are as many patterns for delivering values-distributed-in-time in C/C++ as there are libraries. The effort to compose these together and safely integrate them with raw `thread`, `mutex`, `condition_variable` and `atomic` usage always results in a '`future`' of periodic wailing and gnashing of teeth.

In a brighter world, a standard shape and surface for values-distributed-in-time is defined. Many safe coordination algorithms are built. The algorithms support safe composition and are easily shared. Productivity increases vastly.

~~~C++
   http.get(url) |
      map([](auto r){return r.body;}) |
      switch_on_next() |
      retry(3) |
      timeout(10s) |
      subscribe(
          [](const auto& c){cout << c;},
          [](exception_ptr ep){cerr << endl << what(ep) << endl;});
~~~

> `operator|` is used to compose algorithms to match the range-v3 composition pattern.

## algorithms

The algorithms create most of the negative space that a replacement for `promise` needs to fill.

__existing__

Some algorithms are embedded into the `promise` implementation proposals. This makes them impossible to add or extend the set of algorithms.

*   `then` is overloaded to mean both `map()` and `flat_map()`
*   `when_all` is called `zip` elsewhere
*   `when_any` is called `amb` elsewhere
*   `make_ready_future` is called `just` elsewhere
*   `make_exceptional_future` is called `error` elsewhere

__missing__

Some of these missing algorithms change the shape of the negative space.

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

`promise`/`future` is `subject` elsewhere

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

This may not seem to match the interface of any proposed `promise`. It is perhaps confusing to see the same _Promise_ concept used for both producing the data and consuming the data. Yet, the only difference between consumption and production of data in a push-model is the implementation - the concept is shared. The data using a pull-model requires independent methods for get and set of each piece of data. The data using a push-model requires only set methods. The set methods in a push-model are implemented twice. Once for production and again for consumption. This uniformity provides natural composition since producers and consumers can be layered and interchanged at will.

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

Now other algorithms can be built on this implementation of the concepts. Since `catch` is a C++ keyword and `on_error_resume_next` is too long, `fallback` will be used instead.

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

This is a nice step towards the light. `fallback` is a transform on the error just like `then` is a transform on the value. Other algorithms require changes to the concepts. 

## lifetime 

Many algorithms need to have a signal that indicates that the lifetime has ended. This signal is used to cancel, close, and free resources. There is no C++ destructor that can provide this signal due to the requirement that producers and consumers share state. The producers indicate the end of the lifetime by calling `value()` or `error()` on the _Promise_. The consumers also need to send a signal to end the lifetime early. 

Given that there are multiple signals and that both producers and consumers may send them, handling the end of lifetime signal is always a race.

> `timeout`, `take_until` and `when_any`/`amb` are all examples of consumers that need to signal early termination. 

## virtuous procrastination 

Many algorithms need to be able to defer the work to start later or to restart later or to start in a different context.

> `subscribe_on` and `replay` are examples of consumers that need to control when and how the work is started. 

~~~C++
struct Lifetime
{
    bool is_stopped() const;
    void stop() const;
};

struct SingleObserver
{
    template<typename T>
    void value(T&& t) const;

    template<typename E>
    void error(E&& e) const;
};

struct SingleSubscriber
{
    Lifetime lifetime;
    SingleObserver observer;
};

struct Single 
{
    Lifetime subscribe(SingleSubscriber) const;
};
~~~

## many are greater than the sum

*   `amb` - [reactivex.io](http://reactivex.io/documentation/operators/amb.html)
*   `buffer` - [reactivex.io](http://reactivex.io/documentation/operators/buffer.html)
*   `combine_latest` - [rxmarbles.com](http://rxmarbles.com/#combineLastest)
*   `concat` - [rxmarbles.com](http://rxmarbles.com/#concat)
*   `concat_map`
*   `debounce` - [rxmarbles.com](http://rxmarbles.com/#debounce)
*   `delay` - [rxmarbles.com](http://rxmarbles.com/#delay)
*   `distinct` - [rxmarbles.com](http://rxmarbles.com/#distinct)
*   `distinct_until_changed` - [rxmarbles.com](http://rxmarbles.com/#distinctUntilChanged)
*   `element_at` - [rxmarbles.com](http://rxmarbles.com/#elementAt)
*   `filter` - [rxmarbles.com](http://rxmarbles.com/#filter)
*   `finally`
*   `flat_map` - [reactivex.io](http://reactivex.io/documentation/operators/flatmap.html)
*   `group_by` - [reactivex.io](http://reactivex.io/documentation/operators/groupby.html)
*   `ignore_elements` - [reactivex.io](http://reactivex.io/documentation/operators/ignoreelements.html)
*   `map` - [rxmarbles.com](http://rxmarbles.com/#map)
*   `merge` - [rxmarbles.com](http://rxmarbles.com/#merge)
*   `observe_on` - [reactivex.io](http://reactivex.io/documentation/operators/observeon.html)
*   `on_error_resume_next` - [reactivex.io](http://reactivex.io/documentation/operators/catch.html)
*   `pairwise`
*   `publish` - [reactivex.io](http://reactivex.io/documentation/operators/publish.html)
*   `reduce` - [rxmarbles.com](http://rxmarbles.com/#reduce)
*   `repeat`
*   `replay` - [reactivex.io](http://reactivex.io/documentation/operators/replay.html)
*   `retry` - [reactivex.io](http://reactivex.io/documentation/operators/retry.html)
*   `sample` - [rxmarbles.com](http://rxmarbles.com/#sample)
*   `scan` - [rxmarbles.com](http://rxmarbles.com/#scan)
*   `sequence_equal` - [reactivex.io](http://reactivex.io/documentation/operators/sequenceequal.html)
*   `skip` - [rxmarbles.com](http://rxmarbles.com/#skip)
*   `skip_last` - [rxmarbles.com](http://rxmarbles.com/#skipLast)
*   `skip_until` - [rxmarbles.com](http://rxmarbles.com/#skipUntil)
*   `start_with` - [rxmarbles.com](http://rxmarbles.com/#startWith)
*   `subscribe_on` - [reactivex.io](http://reactivex.io/documentation/operators/subscribeon.html)
*   `switch_if_empty` - [reactivex.io](http://reactivex.io/documentation/operators/switch.html)
*   `switch_on_next` - [reactivex.io](http://reactivex.io/documentation/operators/switch.html)
*   `take` - [rxmarbles.com](http://rxmarbles.com/#take)
*   `take_last` - [rxmarbles.com](http://rxmarbles.com/#takeLast)
*   `take_until` - [rxmarbles.com](http://rxmarbles.com/#takeUntil)
*   `take_while` - [reactivex.io](http://reactivex.io/documentation/operators/takewhile.html)
*   `time_interval` - [reactivex.io](http://reactivex.io/documentation/operators/timeinterval.html)
*   `timeout` - [reactivex.io](http://reactivex.io/documentation/operators/timeout.html)
*   `timestamp` - [reactivex.io](http://reactivex.io/documentation/operators/timestamp.html)
*   `window` - [reactivex.io](http://reactivex.io/documentation/operators/window.html)
*   `with_latest_from` - [rxmarbles.com](http://rxmarbles.com/#withLatestFrom)
*   `zip` - [rxmarbles.com](http://rxmarbles.com/#zip)


# IV. Impact On the Standard

# V. Acknowledgments
