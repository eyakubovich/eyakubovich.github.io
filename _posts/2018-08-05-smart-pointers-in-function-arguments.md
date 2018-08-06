---
layout: post
title: Smart Pointers in Function Arguments
date: '2018-08-05T20:59:00.000-07:00'
author: Eugene Yakubovich
tags:
- C++
modified_time: '2018-08-05T20:59:46.484-07:00'
---

The guidance around function arguments and smart pointers is quite old, yet I still see it used incorrectly. In this post, we’ll explore the guidance and the costs of not following the advice.

C++ Core Guidelines make this point clear:

> [F.7: For general use, take `T*` or `T&` arguments rather than smart pointers](https://github.com/isocpp/CppCoreGuidelines/blob/master/CppCoreGuidelines.md#f7-for-general-use-take-t-or-t-arguments-rather-than-smart-pointers)
>
> *Reason*
> Passing a smart pointer transfers or shares ownership and should only be used when ownership semantics are intended (see R.30). Passing by smart pointer restricts the use of a function to callers that use smart pointers. Passing a shared smart pointer (e.g., std::shared_ptr) implies a run-time cost.

In other words, if the function just dereferences a pointer to get to the object, pass a reference (or a pointer if you want to allow nulls) to the underlying object. Here’s an example:

{% highlight c++ linenos %}
{% raw %}
struct employee {
  employee() = default;
  employee(std::string n, unsigned s) : name(std::move(n)), salary(s) {}

  std::string name;
  unsigned salary;
};

// GOOD
void print(const employee& emp) {
  std::cout << emp.name << ": " << emp.salary << std::endl;
}

auto emp = std::make_unique<employee>("John Doe", 42);
print(*emp);

// BAD
void print(std::unique_ptr<employee>& emp) {
  std::cout << emp->name << ": " << emp->salary << std::endl;
}

auto emp = std::make_unique<Employee>("John Doe", 42);
print(emp);
{% endraw %}
{% endhighlight %}

The observation is that the `print` function does not care about ownership -- the caller can own the object in any way it wants.
As such, the function becomes less general, if bound to a smart pointer, since it restricts it only to callers that have a unique ownership of the object.
Of course, the same issue exists with shared pointers as well.
This is the strongest argument of all for why avoiding smart pointers in function arguments is a good idea. But are there any other reasons?

## `std::unique_ptr`

You might be thinking that you don’t care to make `print` more general -- you know all the uses of it and they all involve a `unique_ptr`.
What other reasons might there be to avoid doing so? Let’s look at two.

### Issue 1: const-ness

In the "good" case, we passed the argument as a const reference.
This makes sure that the `print` function does not accidently modify the employee record.
In the "bad" case, the unique pointer is passed by a non-const reference. We could, of course, add a `const` in front of `unique_ptr` but it would make the pointer constant, not the object it points too. The following happily compiles:

{% highlight c++ linenos %}
{% raw %}
void print(const std::unique_ptr<employee>& emp) {
  emp->salary = 0;
}
{% endraw %}
{% endhighlight %}

What we really need to do is to also place the "const" in front of the "employee":

{% highlight c++ linenos %}
{% raw %}
void print(const std::unique_ptr<const employee>& emp) { ... }
{% endraw %}
{% endhighlight %}

Alas, the call to `print` will not compile -- there is no converting constructor from `std::unique_ptr<T>` to `std::unique_ptr<const T>`.

### Issue 2: Extra indirection

Because `std::unique_ptr` does not have a copy constructor, we’re forced to pass it by reference.
The compiler implements the reference as a pointer and we effectively end up with `Employee**`.
When it’s time to use the employee record, we’re forced to do an extra indirection.
In this example, the double indirection does not cost us much as the memory is hot in cache.
However if the parameter was passed deeper into functions before it was dereferenced, the cache could cool down, leading to a memory fetch.

In addition, if the `unique_ptr` was dereferenced in a loop, the compiler might not be able to cache the pointer value, resulting in a dereference on every iteration.
See disassembly for source line 12 in this example: <https://godbolt.org/g/W9eGb7>

## `std::shared_ptr`

The story with `std::shared_ptr` is better in some respects.
`std::shared_ptr<T>` is convertible to `std::shared_ptr<const T>` which eliminates the const-ness issue that plagues `std::unqiue_ptr`.
The indirection issue can be avoided by taking `std::shared_ptr` by value since it is copyable.
However the copy is usually quite expensive.

`std::shared_ptr` is usually implemented via a reference counting scheme (a linked list variant is possible but rarely used).
The count, however, must be incremented atomically to make it thread-safe.
The standard mandates that making copies of `std::shared_ptr` be thread safe.
While Intel (and most other multiprocessor architectures) offer atomic increment instructions, they can still be quite expensive to execute.
Here’s a blog post that goes into a bit of detail why it is expensive: <https://fgiesen.wordpress.com/2014/08/18/atomics-and-contention/>.
Even under the best of circumstances we’re looking at an order of magnitude difference.

## Conclusions

The performance costs may seem negligible and in an isolated case, they’re are.
But our programs are made from a multitude of decisions just like this one.
Each, on their own, is not crucial to the overall performance.
All together they compound to either make or break the performance of the final product.
