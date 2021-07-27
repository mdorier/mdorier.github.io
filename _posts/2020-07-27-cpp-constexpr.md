---
layout: post
author: Matthieu
title: Using C++17 metaprogramming to compute a function's signature at compile time
---

Constructing a string that represents a function's signature in C++ is a common problem,
often showing up when wrapping low-level C libraries with higher-level, heavily-templated
C++ code. One way of getting a type name in C++ is as follows:

{% highlight cpp %}
auto name = typeid(x).name();
{% endhighlight %}

Where `x` is the variable whose type name you want. Let's use that for a function:

{% highlight cpp %}
void f(const int&, char&&, long*, char[3][4]);
auto name = typeid(f).name();
{% endhighlight %}

When we print out the `name` however, we get `FvRKiOcPlPA4_cE`. Ouch!
Quoting [cppreference](https://en.cppreference.com/w/cpp/types/type_info/name):

_Some implementations (such as MSVC, IBM, Oracle) produce a human-readable type name.
Others, most notably gcc and clang, return the mangled name, which is specified by the Itanium C++ ABI._

While we could go an use something like `boost::core::demangle` to produce a
readable name, the whole type_info/demangle machinery works at run time. What
if we want the string _at compile time_? And what if we want to customize it?
An example would be returning `"%d, %f -> %c"` when given a function
that looks like `char f(int, float)`, as a way to produce a formatting string
for a C-style logger.

This problem got me interested, so in this post I'll show you how to produce 
a function's signature at compile time in C++17, using a mix of _templates_,
_constexpr_, and _type traits_.

### Writing a `type_name` structure

The problem we want to solve consists of writing a `type_name` template structure
with a `get` static function, such that for any type `T`, `type_name<T>::get()`
will return a string name. Let's start with its declaration.


{% highlight cpp %}
template<typename... T>
struct type_name;
{% endhighlight %}

The use of a variadic template will make sense later. We now need to
specialize this template structure for various types, and provide the `get`
function in these specializations.

In C++17, `std::string` doesn't have constexpr constructors, so we can't have `get`
return an `std::string`. We cannot return a `char[]` either. So we first need to
write a little `constexpr_string` structure to hold strings that can be constructed
and returned in a constexpr context.

{% highlight cpp %}
template <unsigned N>
struct constexpr_string {
    char value[N];
};
{% endhighlight %}

Since we are not going to use `typeid`, we need to provide the name of simple
types ourselves. Let's specialize our `type_name` structure for simple types,
for example `int`:

{% highlight cpp %}
template<>
struct type_name<int> {
    static constexpr auto get() {
        return constexpr_string<4>{"int"};
    }
};
{% endhighlight %}

In fact, let's make this a macro, so we can call it for other basic types
(or any classes to which we want to provide a name).

{% highlight cpp %}
#define REGISTER_TYPE_NAME(__type__) \
    template<> struct type_name<__type__> { \
        constexpr static auto get() { \
            return constexpr_string<sizeof #__type__>{#__type__}; \
        } \
    }

REGISTER_TYPE_NAME(void);
REGISTER_TYPE_NAME(int);
REGISTER_TYPE_NAME(char);
REGISTER_TYPE_NAME(long);
REGISTER_TYPE_NAME(double);
REGISTER_TYPE_NAME(float);
{% endhighlight %}

We could register names for a bunch of other basic types, but let's leave it at
that for the moment.

Let's also provide a specialization for any type not registered:

{% highlight cpp %}
template<typename T>
struct type_name<T> {
    constexpr static auto get() {
        return constexpr_string<10>{"<unknown>"};
    }
};
{% endhighlight %}

### Concatenating `constexpr_string`s

If we want to display multiple type, as is typical in a function signature, we
will need to concatenate `constexpr_string`s. This can be done in a constexpr
context as follows.

{% highlight cpp %}
template<unsigned ...L>
constexpr auto cat(const char (&...strings)[L]) {
    constexpr unsigned N = (... + L) - sizeof...(L);
    constexpr_string<N + 1> result = {};
    result.value[N] = '\0';

    char* dst = result.value;
    for (const char* src : {strings...}) {
        for (; *src != '\0'; src++, dst++) {
            *dst = *src;
        }
    }
    return result;
}
{% endhighlight %}

Let's dive a little into this function: it takes as parameters
an arbitrary number of `char` arrays (we will use that so we can directly
pass string litterals as well). It then computes `N` as the sum of the lengths
of all the input strings (sing a fold-expression) , minus the number of
strings (to remove the trailing null-terminator of each string).
It initializes the `result` string with a
length of `N+1` (to accomodate for `result`'s own null-terminator, which
is assigned next). It then iterates over all the input strings, and over
all their characters, to copy them into the result. All of this can be done
at compile time.

### Printing a list of type names

Using the above `cat` function, we can now specialize our `type_name` structure
to handle a list of template parameters with more than one type. In this example
we will add a `", "` separator between them.

{% highlight cpp %}
template<typename T1, typename ... Other>
struct type_name<T1, Other...> {
    constexpr static auto get() {
        return cat(type_name<T1>::get().value, ", ",
                   type_name<Other...>::get().value);
    }
};
{% endhighlight %}

We can now specialize the `type_name` for functions by using the above specialization
to get a comma-separated list of argument types, preceded and followed by parenthesis,
and follows by the type of the return value.

{% highlight cpp %}
template<typename R, typename ... T>
struct type_name<R(T...)> {
    constexpr static auto get() {
        return cat("(", type_name<T...>::get().value, ")",
                   " -> ", type_name<R>::get().value);
    }
};
{% endhighlight %}

Now if we declare `double f(int, char)` and call `type_name<decltype(f)>::get().value`
we obtain `"(int, char) -> double"`.

### Handling const, references, pointers, etc.

If we try what we have coded so far with `double& f(const int&, char&&, float*, long[2][3])`,
we obtain... `"(<unknown>, <unknown>, <unknown>, <unknown>) -> <unknown>"`. Very helpful...
This is because while we have registered the basic types, we haven't registered references
to these types, or pointers, or arrays. We can fix that we going back to our specialization
of `type_name` that took a single `T` parameter, and use type traits along with constexpr if
statements to detect these modifiers.

{% highlight cpp %}
template<typename T>
struct type_name<T> {
    constexpr static auto get() {
        if constexpr (std::is_pointer_v<T>) {
            return cat(type_name<std::remove_pointer_t<T>>::get().value, "*");
        } else if constexpr (std::is_const_v<T>) {
            return cat("const ", type_name<std::remove_const_t<T>>::get().value);
        } else if constexpr (std::is_rvalue_reference_v<T>) {
            return cat(type_name<std::remove_reference_t<T>>::get().value, "&&");
        } else if constexpr (std::is_lvalue_reference_v<T>) {
            return cat(type_name<std::remove_reference_t<T>>::get().value, "&");
        } else {
            return constexpr_string<10>{"<unknown>"};
        }
    }
};
{% endhighlight %}

Now we get `"(const int&, char&&, float*, <unknown>*) -> double&"`. Why is `long[2][3]`
transformed into `<unknown>*`? Well, because `long[2][3]` decayes into `long[2]*`, not
into `long**`, and our code doesn't know how to handle arrays. We need to add support for
arrays, as follows.

{% highlight cpp %}
template<typename T>
struct type_name<T[]> {
    constexpr static auto get() {
        return cat(type_name<T>::get().value, "[]");
    }
};

template<typename T, std::size_t N>
struct type_name<T[N]> {
    constexpr static auto get() {
        return cat(type_name<T>::get().value, "[]");
    }
};
{% endhighlight %}

We now have the following signature: `"(const int&, char&&, float*, long[]*) -> double&"`.
This mechanism ignores the length of the array `N` and just prints `[]`. I leave it as
an exercise to the reader to come up with a mechanism that builds a `constexpr_string`
decimal representation of `N` in a constexpr context. If you understood the code above,
and have some knowledge of constexpr, templates, and C++17 metaprogramming in general,
you will surely find the solution.

There are of course some more details to take into considerations, such as handling
member object/function pointers (with `std::is_member_object_pointer` and
`std::is_member_function_pointer`), or properly displaying functions that take
a function pointer as argument (for instance `void g(int (*)(char, char))`
is shown to have the type `"((char, char) -> int*) -> void"`, which looks
like it takes a function that returns an `int*`, instead of a pointer to a function
that returns an `int`).

If you want to try the code of this blog post for yourself, you can head
<a href="https://godbolt.org/z/vh7zaW5sn" target="_blank">here</a>
and you'll see from the assembly that all of this does compute the signature
at compile time.

