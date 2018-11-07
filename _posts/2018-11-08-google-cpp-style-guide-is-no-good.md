---
layout: post
title: Google C++ Style Guide is No Good
date: '2018-11-06T20:59:00.000-07:00'
author: Eugene Yakubovich
tags:
- C++
modified_time: '2018-10-23T20:59:46.484-07:00'
---

The [Google Style Guide (GSG)](https://google.github.io/styleguide/cppguide.html) for C++ has become popular outside of Google.
But I don't like it at all. There... I said it.
I want to take this opportunity to explain why I feel so strongly about it.

## GSG "style"
The general observation is that the style guide is written in a prohibitive fashion.
Unlike C++ Core Guidelines that try to explain how to use the language effectively, GSG is about forbidding the use of certain features.
Many of the rules prohibit the use of a feature over the fear of confusion, abuse, ambiguity and bugs.
It is a well known fact that every feature (in every language) can be abused and misunderstood.
Every line of code (and feature) can be a source of bugs.
At the same time, every feature that went into the language was vetted for being useful.
It was deemed that without the said feature, the code was significantly worse (e.g more verbose, prone to bugs).
Therefore prohiting the use of some language feature just invites these problems into the codebase.
Additionaly the GSG prohibits constructs that are widely used in well-known codebases: stdlib implementations, Boost, cppreference.com examples, etc.

GSG is also concerned about some construct not being known to newcomers or even experienced C++ developers and outlaws such constructs based on these fears.
But how will developers learn new features if not from reading code that uses them?
Lastly, simple APIs often require complex implementations.
Consider, for example, the following simple code snippet:

{% highlight c++ linenos %}
{% raw %}
std::vector<int> v;
v.resize(10, 42);
auto it = v.begin();
std::advance(it, 5);
{% endraw %}
{% endhighlight %}

The above snippet invokes code paths that use constructs like `std::uninitialized_fill` (which uses placement new), `std::move_if_noexcept` (which uses SFINAE), implementation of random access iterator and tag dispatching.
Nevertheless, the API that is exposed is easy to use and understood by the beginner.

I can already hear an argument that there's a difference between "library writers" and "everyone else".
Maybe GSG should be waived for "library writers" who are expected to be language experts.
This is a dangerous world view.
We should not bifurcate developers into two camps as the reality exists in a continuum between these two extremes.
Every developer should be a "library writer" as it is the only way to decompose the code into manageable, testable  and reusable units.

## What's the alternative? What should a style guide look like?
It is certainly reasonable to establish rules around naming and formatting.
A style guide can also try to teach how to use the language effectively (like C++ Core Guidelines do).
At the same time, this subject is better presented in numerous books that have been published.
It would not be feasible to reproduce Scott Meyers' series of books in a style guide.
Perhaps the guide should just include a reading list for developers to hone their language skills.
A style guide can also prohibit anti-patterns.
For example, the use of naked `new` where `std::make_unique` and `std::make_shared` could be used.
Yet it shouldn't prohibit the use of a `new` operator.

## Concrete Grievances
Let's now look at some of the rules which particularly trouble me.

### [The `#define` Guard](https://google.github.io/styleguide/cppguide.html#The__define_Guard)
GSG prefers `#ifndef/#define` idiom over the simpler `#pragma once`.
Yes, `#pragma once` is non-standard but [widely supported](https://en.wikipedia.org/wiki/Pragma_once#Portability).

### [Forward Declarations](https://google.github.io/styleguide/cppguide.html#Forward_Declarations)
Although GSG does not forbid them categoricaly, it says to "Avoid using forward declarations where possible".

However forward declarations are useful in contexts like the following:
- Circular data structures:

{% highlight c++ linenos %}
{% raw %}
struct Container;
struct Node {
  Container* parent;
};
struct Container {
  Node node_a, node_b;
};

{% endraw %}
{% endhighlight %}

- Opaque types / pimpl idiom
{% highlight c++ linenos %}
{% raw %}
class Socket {
  std::unique_ptr<struct ControlBlock> cblk_;
};
{% endraw %}
{% endhighlight %}

Yes, both of these forward declarations are possible to avoid by type-punning through a `void*` but it is not a good pattern.

### [Inline Functions](https://google.github.io/styleguide/cppguide.html#Inline_Functions)
The guidance to "Define functions inline only when they are small, say, 10 lines or fewer." is sound.
However the guide fails to mention that:
- Many long functions (e.g. with long types) can end up generating little to no code.
- Marking the function "inline" lets the compiler make the decision. Not marking it inline is a sure way to prevent inlining, unless Link Time Optimizations are turned on.
- Inline functions are useful for header-only libraries.

### [Namespaces](https://google.github.io/styleguide/cppguide.html#Namespaces)
GSG says "With few exceptions, place code in a namespace.".
It does not specify which exceptions.

Namespaces are there to prevent collisions.
Library code should always be placed into a namespace.
Top level/application code has questionable value being placed in a namespace.
It is the code that is importing libraries and can trivially avoid name conflicts (even if names clash):

{% highlight c++ linenos %}
{% raw %}
// mathlib.h
namespace MathLib {
  double sqrt(double);
}

// app.h
#include "library.h"

double sqrt(double) { ... }
int main() {
  ::sqrt(3.0); // use app level one
  MathLib::sqrt(5.0); // use MathLib's version
  return 0;
}
{% endraw %}
{% endhighlight %}

GSG also prohibits the use of _using-directive_, e.g. `using namespace foo;`.
While the practice of dumping everything into your namespace (e.g. `using namespace std` at the top of every file) is bad, this ruling is overly prohibitive.
Here are some valid uses of of _using-directive_:

- In function scope:
{% highlight c++ linenos %}
{% raw %}
void foo() {
  using namespace std::placeholders;
  std::bind(bar, _1, 2);

  using namespace std::literals;
  auto s = "abc"sv;
}
{% endraw %}
{% endhighlight %}

- In versioning:
{% highlight c++ linenos %}
{% raw %}
namespace FancyProtocol {
  // bring in the current version of messages
  using namespace Messages::v5;
}
{% endraw %}
{% endhighlight %}

### [Unnamed Namespaces and Static Variables](https://google.github.io/styleguide/cppguide.html#Unnamed_Namespaces_and_Static_Variables)
GSG prohibits the use of static definitions and annonymous namespaces in header files.
How else do we declare constants?

### [Static and Global Variables](https://google.github.io/styleguide/cppguide.html#Static_and_Global_Variables)
The rule basically says that global/static variables with non-trivial constructors and destructors are not allowed.
While it's true that initialization/destruction order of globals *between* translation units is not defined and can pose problems, this rule is *overly* prohibitve.
Globals like this are not problematic and very useful:

{% highlight c++ linenos %}
{% raw %}
const std::unordered_map<std::string, Color> Colors = {
  { "red"s, rgb(255, 0, 0) },
  { "green"s, rgb(0, 255, 0) },
  { "blue"s, rgb(0, 0, 255) }
};
{% endraw %}
{% endhighlight %}

Even the inter-translation unit ordering is not a huge problem -- Stroustrup discusses it in his D&E book.

### [Implicit Conversions](https://google.github.io/styleguide/cppguide.html#Implicit_Conversions)
"Do not define implicit conversions".
I would urge the reader to consider what life would be like if `std::string(const char*)` constructor was marked `explicit` (especially in the absense of user defined literals, which GSG also outlaws).
Nuff said.

### [Copyable and Movable Types](https://google.github.io/styleguide/cppguide.html#Copyable_Movable_Types)
Lynchpin of C++ are value-types.
Such types should be copyable and moveable and the language automatically generates the necessary constructors and operators by default.
"a copyable class should explicitly declare the copy operations, a move-only class should explicitly declare the move operations" -- this goes against the language philosophy.

"A type should not be copyable/movable if the meaning of copying/moving is unclear to a casual user, or if it incurs unexpected costs":
- Because copies and moves can be elided, there can be no other meaning to copying/moving than the prescribed ones.
- I don't know what an average person expects the cost of copy/move to be.
However even expensive copies are not to be avoided. `std::vector` with a million items will be costly to copy.
It shouldn't cease being copyable.

### [Inheritance](https://google.github.io/styleguide/cppguide.html#Inheritance)
"All inheritance should be public. If you want to do private inheritance, you should be including an instance of the base class as a member instead."
Since private inheritance models "has-a" relationship and can often be replaced by a data member.
In certain cases, this has serious disadvantages.
Here's an example:

{% highlight c++ linenos %}
{% raw %}
class StrWriter {
public:
  StrWriter(char* buf);
  void writeBinary(int);
  ...
};

class StaticStrWriter :
  private std::array<char, 100>,
  public StrWriter {
public:
  StaticStrWriter() : StrWriter(data()) {}
};
{% endraw %}
{% endhighlight %}

In this pattern (which comes up often), the private base cannot be replaced by a data member because the base is constructed before the members.

"Multiple inheritance is permitted, but multiple _implementation_ inheritance is strongly discouraged."
The guide cites performance issues with multiple inheritance as the reason to avoid it.
I seriously doubt that the use of (non-virtual) multiple inheritance has serious performance overhead (it usually amounts to adjusting `this` pointer by a constant amount).
It does not make sense to say that a class can have an is-a relationship with only a single other one.

### [Operator Overloading](https://google.github.io/styleguide/cppguide.html#Operator_Overloading)
Although GSG does not forbid operator overloading, it cautiously allows it with the following restriction: "Define overloaded operators only if their meaning is obvious, unsurprising, and consistent with the corresponding built-in operators. For example, use | as a bitwise- or logical-or, not as a shell-style pipe."

What is obvious, unsurprising, and consistent meaning? Should `+` operator be commutative? If we say 'yes', we can then say goodbye to the string concatenation operator (`std::basic_string::operator+`).
The operator meaning is only known for a given type(s) and "projecting" built-in types' semantics onto user defined types is pointless.
Ben Deane recently gave an excellent talk on the subject at CppCon: [Operator Overloading: History, Principles and Practice](https://www.youtube.com/watch?v=zh4EgO13Etg).

I want to stress the importance of good notation, especially infix operators.
The math notation we all know is a rather modern invention.
For example, consider that in 628 AD, Brahmagupta, an Indian mathematician, gave the first explicit solution of the quadratic equation ax^2 + bx = c as follows: "To the absolute number multiplied by four times the [coefficient of the] square, add the square of the [coefficient of the] middle term; the square root of the same, less the [coefficient of the] middle term, being divided by twice the [coefficient of the] square is the value.
Compare this to the quadratic formula written the way you learned it in school.

Mathematicians, physicists, etc usually have the luxury of inventing new symbols or selecting from a wide range of available glyphs to denote operators.
We don't have such luxury in C++.
Nevertheless, coming up with a good notation using the set of symbols that we can use can be a huge positive for both writeablity and readability of code.
Depending on the context (types), having "unusual" meaning for existing operator symbols is rarely confusing.
For example, when dealing with grammars (e.g. in Boost.Spirit), using unary `*` operator to denote a Kleene star is natural.
Given the context (grammars), confusing it with the pointer dereference operator is not easy.

### [Reference Arguments](https://google.github.io/styleguide/cppguide.html#Reference_Arguments)
GSG's rule is "All parameters passed by lvalue reference must be labeled const."
This goes together with the stance that all [out and in/out parameters](https://google.github.io/styleguide/cppguide.html#Output_Parameters) must be passed by a pointer.

I think this rule mixes together two ideas for no good reason. C++ has a clear way to denote mutability -- the `const` qualifier.
Using pointer vs reference to signal mutability goes against the language.
It makes more sense to use a pointer when `nullptr` is allowed and a reference otherwise.
This would eliminate a runtime assertion for unexpected null pointer in the body of the function.

The reasoning behind this rule is to establish a convention that communicates mutability at the call site via the presence of the address-of operator.
In other words, the reader is expected to understand the semantics of an unknown function without consulting the declaration (or documentation).
Maybe it works for understanding mutability of arguments but I doubt the readers will get very far with just this knowledge.

### [Exceptions](https://google.github.io/styleguide/cppguide.html#Exceptions)
Exceptions vs error codes debate is much like space-vs-tabs so I will sit this one out.
GSG forbids exceptions on the ground of "Google's existing code is not exception-tolerant".
I'm not sure what that refers to.
Did they mark every function `noexcept` or are they using `goto fail` convention for cleanup?

What I find remarkable is that it offers no guideance on how to deal with errors in the absence of exceptions.
* Should developers use `errno` global?
* Should they use something like the proposed `std::expected`?
* Should every function return an error code and use out-parameters to return computed values?

### [Lambda expressions](https://google.github.io/styleguide/cppguide.html#Exceptions)
"Use lambda expressions where appropriate." -- probably a cheap shot, but what is the alternative? -- use them where it's not appropriate?

### [Template metaprogramming](https://google.github.io/styleguide/cppguide.html#Template_metaprogramming)
For the most part, GSG tries to scare anyone away from TMP.
Unfortunately Google is not alone here as TMP is often frowned upon in many C++ circles.
I have heard many opponents of TMP and their objections are almost always grounded in the fear of someone just using it to be clever.

It seems that many engineers simply don't understand what metaprogramming is and what it's used for.
When one needs to generate code (write a meta-program), it can either be done from within C++ (via TMP) or using an external script (e.g. written in Python) or a separate compiler (e.g. lex, yacc, swig).
I will assert that using an external script/compiler ends up being worse for a number of reasons:
* You need to integrate the tool into your toolchain
* You need to keep the data-structures in sync
* Crossing the boundary often requires going through a C interface
* Having scripts that output C++ code are not easy to develop or maintain

The use of a separate DSL does have advantages and can be justified if it will be a large part of the project.
For example, a compiler writer may invest in setting up lex and yacc for more natural syntanx and more informative error messages.
However almost every project can benefit from a little DSL (e.g. to format dates or match a regular expression).
The overhead of using a separate tool for such small tasks is probihitive.
Embedded DSLs shine in this space and such EDSLs have to be implemented using TMP.

### [Boost](https://google.github.io/styleguide/cppguide.html#Boost)
GSG whitelists only a handful of Boost libraries on the fears that it might encourage the developers to learn and use "advanced techniques".
Boost contains a wide array of useful high calliber libraries and their use should be supported.
It is certainly better than the alternative of coding the functionality yourself or copy/pasting code from some StackOverflow answer.

### [C++11](https://google.github.io/styleguide/cppguide.html#C++11)
It's 2018.
