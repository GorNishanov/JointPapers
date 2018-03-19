
| Document Number: | D0XXXR0                                         |
| -----------------|-------------------------------------------------|
| Date:            | 2018-03-18                                      |
| Audience:        | Evolution                                       |
| Revises:         | none                                            |
| Reply to:        | Richard Smith, Gor Nishanov (gorn@microsoft.com)               |

# HALO: Coroutine Heap Allocation eLision Optimization: The joint response.

## Summary

During the discussion of the coroutine in a Coroutine Tuesday session in Jacksonville 2018, we had the following exchange:
> Richard Smith: I have concerns about the generator example. ... The optimization
> relies on `accumulate` being inlinable or optimizable. ...
 You need to inline at least `begin` to know that the value of the handle is not changed.

> Gor Nishanov: Let's take this offline and come back with the joint response.

This document is that join response. It evaluates the necessary conditions for heap allocation elision
to work. After careful study, we conclude that:
1. Heap allocation elision optimization DOES NOT require inlining potentially unbounded amount of code, such as coroutine body OR the algorithms using the coroutine (such as `accumulate` in the generator example).
2. It DOES REQUIRE inlining some compiler synthesized code (such as coroutine ramp) and inlining bounded amount of code from some wrapper/glue code that gives the coroutine the desired semantics (such as `begin` in the generator example).

The rest of the document goes over details of what exactly needs to be inlined and explores generator and task examples.

## Appendix

### Terminology

**Coroutine frame**: a memory location where state of the coroutine that has to be preserved across suspend points is stored. Coroutine frame can reside on the heap or on the stack, depending on whether coroutine heap elision is in effect or whether a custom allocator is provided by the user to override default allocation of the coroutine state.

**User-authored coroutine body**: A *function-body* of the coroutine. 

**Coroutine State Machine**: a compiler created transformation of the *user-authored coroutine body*

**Coroutine ramp function**: a compiler synthesized ramp/trampoline/thunk that creates coroutine state and starts the coroutine state machine.

**Coroutine handle**: An object of a coroutine_handle type that points at a coroutine state machine.

**Coroutine type**: A wrapper around the coroutine handle that has necessary bindings to give the corotuine the desired semantics. For example: a generator<T> or a task<T>.

**Coroutine escaping its caller**: A call to a function the body of which is not available to a compiler for the examination that may modify the coroutine handle stored in the coroutine type. For example: invocation std::move on the generator may result in "nulling" out the coroutine handle stored in the coroutine type.

### Generator case

Let's consider the example presented in P0978R0:

```c++
generator<int> range(int from, int to) { // user-authored coroutine body starts
   for (int i = from; i < to; ++i)
      co_yield i;
} // user authored coroutine body ends

int main() {
  auto s = range(1, 10);
  return std::accumulate(s.begin(), s.end(), 0);
}
```

Conceptually, the coroutine transformation will transform `range(int,int)` into something like this:
```c++
struct range$state-machine { ... transformed user-authored-body-is-here ... };

generator<int> range(int from, int to) { // coroutine ramp/thunk/trampoline
   auto p = new range$state-machine(from, to);
   return p->get_return_object();
}
```

For the coroutine heap allocation elision optimization to work we need the following to be available
for inlining:

* coroutine ramp function
* get_return_object()
* begin of the generator
* constructor and move constructor of the generator
* destructor of the generator

It DOES NOT need to inline the body of the coroutine or the algorithm `accumulate`.

To give a feel of how big the function that needs to be inlined are, here is a typical generator implementation:

```c++
struct generator {
  struct promise_type { ... };
  ...
  coroutine_handle<promise_type> h; // wrapper around void* trival to construct/destruct
};
```

Constructors that need to be inlinable:
```c++
generator(coroutine_handle<promise_type> h) : h(h) {}
generator(generator&& rhs) : h(rhs.h) { rhs.h = nullptr; }
```

Destructor must be inlinable:
```c++
~generator() { if (h) h.destroy(); }
```

get_return_object must be inlinable:
```c++
generator promise_type::get_return_object() {
  return {coroutine_handle::from_promise(*this)}; }
```

begin() must be inlinable
```c++
iterator begin() { return {h}; }
```

We find that the functions that need to be inlined for coroutine heap allocation elision optimization
to work are tiny and we would want them inlined anyways irrespectively whether they are needed for
heap allocation elision to work or not.

<!--
https://godbolt.org/g/2FaMtz (heap allocation elision with most of the function bodies removed)

https://godbolt.org/g/6qRsCT (original)

https://godbolt.org/g/PXaJHD (task)
-->

### Task example

Let's consider the big_task example from P0978R0 with a small addition of a call to an executor (as the question was raised whether a coroutine posted to execute on an executor can have their allocation elided).

```c++
task<> subtask(Executor ex) {
  ...
  co_await execute_on(ex);
  ...
}

task<> big_task(Executor ex) {
  ...
  co_await subtask(ex); // subtask frame allocation elided
  ...
}
```

Similar to the generator case, only a small number of library functions needs to be available for
inlining:

* coroutine ramp function
* get_return_object()
* `await_suspend`/`await_ready`/`await_resume` of the task
* constructor and move constructor of the task
* destructor of the task

It DOES NOT need to inline:
* executor .execute / on_execute
* body of the subtask or big_task

To give a feel of how big the function that needs to be inlined are, here is a typical task implementation:

```c++
struct task {
  struct promise_type { ... };
  ...
  coroutine_handle<promise_type> h; // wrapper around void* trival to construct/destruct
};
```

Constructors:
```c++
task(coroutine_handle<promise_type> h) : h(h) {}
task(task&& rhs) : h(rhs.h) { rhs.h = nullptr; }
```

get_return_object:
```c++
task promise_type::get_return_object() {
  return {coroutine_handle::from_promise(*this)}; }
```

Destructor:
```c++
~task() { if(h) h.destroy(); }
```****

await_suspend:
```c++
auto await_suspend(coroutine_handle<promise_type> h) {
  h.promise().waiter = h;
  return h;
}
```
await_ready/await_resume:
```c++
constexpr bool await_ready() const noexcept { return false; }
constexpr void await_resyme() const noexcept {}
```

We find that the functions that need to be inlined for coroutine heap allocation elision optimization to work are tiny and we would want them inlined anyways irrespectively whether they are needed for heap allocation elision to work or not.

