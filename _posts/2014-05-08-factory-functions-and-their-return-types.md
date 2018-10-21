---
layout: post
title: Factory functions and their return types
date: '2014-05-08T14:05:00.001-07:00'
author: Eugene Yakubovich
tags:
- C++
- C++11
modified_time: '2014-05-09T09:13:32.529-07:00'
---

Suppose we need to write a factory function that constructs a runtime polymorphic object.
For the purposes of this post, let's say we want to construct a concrete shape object -- a rectangle, triangle, or an ellipse.
Here are our basic declarations:

{% highlight c++ linenos %}
{% raw %}
struct shape {
  virtual ~shape() {}
};

struct ellipse : shape {
  ellipse(int rad_a, int rad_b) {}
};

struct triangle : shape {
  triangle(int base, int height) {}
};

struct rectangle : shape {
  rectangle(int width, int height) {}
};

{% endraw %}
{% endhighlight %}

Basic stuff.
Now for the factory function:

{% highlight c++ linenos %}
{% raw %}
enum class shape_type { ellipse, triangle, rectangle };
struct rect {
  int w, h;
};

??? make_shape(shape_type type, rect bounds);
{% endraw %}
{% endhighlight %}

What should `make_shape` return.
A pointer to shape, of course, but which kind.
Should it be a raw pointer or a smart pointer like `std::unique_ptr` and `std::shared_ptr`.
C++11 heavily advocates against raw pointers and I completely agree.
That leaves us with a `unique_ptr` or a `shared_ptr`.
I believe that in vast majority of situations there's a single owner of an object so that begs for returning `unique_ptr`.
At least few other people are of the same opinion: [here](http://stackoverflow.com/questions/13062106/returning-pointer-from-factory) and [here](http://herbsutter.com/2013/05/30/gotw-90-solution-factories/).
The argument goes that a `shared_ptr` can be constructed from a `unique_ptr` so this will also work just fine for the less common shared ownership cases:

{% highlight c++ linenos %}
{% raw %}
std::shared_ptr<shape> s = make_shape(shape_type::ellipse, { 3, 5 });
{% endraw %}
{% endhighlight %}

While that is certainly true, there is a performance problem with this.
C++11 encourages us to use `std::make_shared` to construct shared ownership objects.
Most `std::make_shared` implementations use a single dynamic memory allocation for both the object and the pointer control block (that stores the ref count).
Not only does that save on overhead of calling 'new' twice, it also improves the cache locality by keeping the two close.
That benefit is clearly lost with conversion from `unique_ptr` to `shared_ptr`.
I would therefore argue that factory functions should come in two flavors: a unique and a shared kind:

{% highlight c++ linenos %}
{% raw %}
std::unique_ptr<shape> make_unique_shape(shape_type type, rect bounds);
std::shared_ptr<shape> make_shared_shape(shape_type type, rect bounds);
{% endraw %}
{% endhighlight %}

We now have two functions that do almost identical work.
To avoid code duplication, we should factor out common behavior, right?
Right but it turns out to be trickier than I expected.
What we want is a helper function that is parameterized on `make_shared` or `make_unique` (or similar till will have it in C++14).
The solution I came up with uses good old tag dispatching.

First, declare the tags but have them also know their associated smart pointer type:

{% highlight c++ linenos %}
{% raw %}
struct shared_ownership {
  template <typename T> using ptr_t = std::shared_ptr<T>;
};

struct unique_ownership {
  template <typename T> using ptr_t = std::unique_ptr<T>;
};
{% endraw %}
{% endhighlight %}

Next, we add two overloads to do the actual construction:

{% highlight c++ linenos %}
{% raw %}
template <typename T, typename... Args>
std::unique_ptr<T> make_with_ownership(unique_ownership, Args... args) {
  return std::make_unique<T>(std::forward<Args>(args)...);
}

template <typename T, typename... Args>
std::shared_ptr<T> make_with_ownership(shared_ownership, Args... args) {
  return std::make_shared<T>(std::forward<Args>(args)...);
}
{% endraw %}
{% endhighlight %}

Finally, we can put it all together to create a generic `make_shape` along with `make_unique_shape` and `make_shared_shape`:

{% highlight c++ linenos %}
{% raw %}
template <typename OwnTag>
typename OwnTag::template ptr_t<shape> make_shape(shape_type type, rect bounds, OwnTag owntag) {
  switch( type ) {
    case shape_type::ellipse:
      return make_with_ownership<ellipse>(owntag, bounds.w / 2, bounds.h / 2);
    case shape_type::triangle:
      return make_with_ownership<triangle>(owntag, bounds.w, bounds.h);
    case shape_type::rectangle:
      return make_with_ownership<rectangle>(owntag, bounds.w, bounds.h);
  }
}

inline std::unique_ptr<shape> make_unique_shape(shape_type type, rect bounds) {
  return make_shape(type, bounds, unique_ownership());
}

inline std::shared_ptr<shape> make_shared_shape(shape_type type, rect bounds) {
  return make_shape(type, bounds, shared_ownership());
}
{% endraw %}
{% endhighlight %}
 
If you look at the return type of `make_shape`, it should make you cringe with disgust.
Yeah, no bonus points for elegant syntax here.
I also dislike the verbose name `make_with_ownership`.
Nevertheless, I believe having a generic function for both unique and shared construction is extremely valuable.
I would love to hear proposals for a better implementation and suggestions for a more concise name.
As always, the code is available on [GitHub](https://github.com/eyakubovich/blog-code/tree/master/factory).
