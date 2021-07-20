---
layout: post
author: Matthieu
title: Python directives and domain-specific language
---

If you work in high-performance computing, you probably know
the [spack](https://spack.readthedocs.io/en/latest/) package
manager. In spack, packages are defined as python classes,
using what looks like a domain-specific language, as examplified bellow.

{% highlight python %}
class Zlib(Package):
    """A free, general-purpose, legally unencumbered lossless data-compression library."""

    homepage = "http://zlib.net"
    url = "http://zlib.net/fossils/zlib-1.2.11.tar.gz"

    version('1.2.11', sha256='c3e5e9fdd5004dcb542feda5ee4f0ff0744628baf8ed2dd5d66f8ca1197cb1a1')
    version('1.2.8', sha256='36658cb768a54c1d4dec43c3116c27ed893e88b02ecfcb44f2166f9c0b7f2a0d')
    version('1.2.3', sha256='1795c7d067a43174113fdf03447532f373e1c6c57c08d61d9e4e9be5e244b05e')

    variant('pic', default=True,
            description='Produce position-independent code (for shared libs)')
    variant('shared', default=True,
            description='Enables the build of shared libraries.')
    variant('optimize', default=True,
            description='Enable -O2 for a more optimized lib')

    patch('w_patch.patch', when="@1.2.11%cce")

    ...
{% endhighlight %}

The developers of spack took inspiration from [homebrew](https://brew.sh/),
another package manager, which uses Ruby for its domain specific language.
While Ruby is very good for this purpose, Python is much harder to use,
and I always wondered how spack packages worked. Today I dug into the code,
and managed to reproduce what spack does. So let's dive into it.

Let's assume we want to implement the `version` and `variant` directives,
and make them available within a `MyPackage` class.

{% highlight python %}
class MyPackage(object):
    variant('x')
    version('1')
{% endhighlight %}

Executing this code will, of course,  produce an error:

{% highlight txt %}
NameError: name 'variant' is not defined
{% endhighlight %}

So we need to define these functions.

{% highlight python %}
def variant(v):
   print('Calling variant with {}'.format(v))

def version(v):
   print('Calling version with {}'.format(v))

class MyPackage(object):
    variant('x')
    version('1')
{% endhighlight %}

Now our code prints this:

{% highlight txt %}
Calling variant with x
Calling version with 1
{% endhighlight %}

Great... but how do we modify the `MyPackage` class to keep track
of the versions and variants? Because after all, that's what we
are after. Well, we need to know when the class is being created
by Python, and for this, we have _metaclasses_. Let's write one,
and have `MyPackage` use it.

{% highlight python %}
from six import with_metaclass

class PackageMeta(type):

    def __new__(cls, name, bases, attr_dict):
        print('Using {} to create class {} with bases {} and attributes {}'.format(
            cls.__name__, name, bases, attr_dict))
        return super(PackageMeta, cls).__new__(cls, name, bases, attr_dict)

def variant(v):
   print('Calling variant with {}'.format(v))

def version(v):
   print('Calling version with {}'.format(v))

class MyPackage(with_metaclass(PackageMeta)):
    variant('x')
    version('1')
{% endhighlight %}

The code above now prints the following:

{% highlight txt %}
Calling variant with x
Calling version with 1
Using PackageMeta to create class MyPackage with bases () and attributes {'__module__': '__main__'}
{% endhighlight %}

We can see that the directives inside `MyPackage` are called first, then the `__new__`
method of the metaclass. We can use this to our advantage by having the `variant` and `version`
functions modify some static attributes of the metaclass, then use these static attributes during
the creation of the `MyPackage` class.

{% highlight python %}
from six import with_metaclass

class PackageMeta(type):

    versions_to_add = []
    variants_to_add = []

    def __new__(cls, name, bases, attr_dict):
        attr_dict['versions'] = PackageMeta.versions_to_add
        attr_dict['variants'] = PackageMeta.variants_to_add
        PackageMeta.versions_to_add = []
        PackageMeta.variants_to_add = []
        return super(PackageMeta, cls).__new__(cls, name, bases, attr_dict)

def variant(v):
    PackageMeta.variants_to_add.append(v)

def version(v):
    PackageMeta.versions_to_add.append(v)

class MyPackage(with_metaclass(PackageMeta)):
    variant('x')
    version('1')

print('MyPackage has versions {}'.format(MyPackage.versions))
print('MyPackage has variants {}'.format(MyPackage.variants))
{% endhighlight %}

This code now successfully adds the `versions` and `variants` static attributes
to the class, and will print the following:

{% highlight txt %}
MyPackage has versions ['1']
MyPackage has variants ['x']
{% endhighlight %}

We can make this code a little safer by using `_versions_to_add` instead
of `versions_to_add` (private attribute), by defining a `version` static method
in `PackageMeta`, and by adding `version = PackageMeta.version` right outside
of the `PackagetMeta` class to make it usable at global scope.

## Going one step further

The part of spack that implements the mechanism above is in the
[directives.py](https://github.com/spack/spack/blob/develop/lib/spack/spack/directives.py) file.
But if you read it, you will find that it doesn't look very much like what I have
shown above. This is because the spack developers have gone one step further
by using decorators and functional programming to allow adding new directives as needed.
We can try to do something similar:

{% highlight python %}
from six import with_metaclass

class PackageMeta(type):

    list_names = set()
    directives_to_execute = []

    def __new__(cls, name, bases, attr_dict):
        for name in PackageMeta.list_names:
            attr_dict[name] = []
        return super(PackageMeta, cls).__new__(cls, name, bases, attr_dict)

    def __init__(pkg, name, bases, attr_dict):
        for directive in PackageMeta.directives_to_execute:
            directive(pkg)
        PackageMeta.directives_to_execute = []

def directive(list_name):
    PackageMeta.list_names.add(list_name)
    def _decorator(func):
        def _wrapper(*args, **kwargs):
            result = func(*args, **kwargs)
            PackageMeta.directives_to_execute.append(result)
        return _wrapper
    return _decorator

@directive('variants')
def variant(v):
    def _execute_variant(pkg):
        pkg.variants.append(v)
    return _execute_variant

@directive('versions')
def version(v):
    def _execute_version(pkg):
        pkg.versions.append(v)
    return _execute_version

class MyPackage(with_metaclass(PackageMeta)):
    variant('x')
    version('1')

print('MyPackage has versions {}'.format(MyPackage.versions))
print('MyPackage has variants {}'.format(MyPackage.variants))
{% endhighlight %}

This code is, again, a simplification of what Spack does, but let's dive into it,
focusing on the `variant` directive (`version` works in exactly the same way).

The first thing Python encounters is the `directive` decorator wrapping the `variant`
function. This directive takes the name of a list (`"variants"`): this is the list
in which we will want to add variants. We pass this name to the decorator so that
it can add this name to `PackageMeta.list_names`, a list of list names that
`PackageMeta` will have to use when creating a class' attributes.

The second thing the `directive` decorator does it return a `_wrapper` for the
decorated function. At this point, the name `variant` does not refer to the
`variant` function anymore, but to a wrapper around it, and `PackageMeta.list_names`
contains `"variants"`. More directives (like `version`) are added the same way.

Next, we enter the `MyPackage` definition, and call `variant('x')`. This calls the
`_wrapper` function, which executes the actual `variant` function. The `variant`
function returns the `_execute_version` function, which is stored in
`PackageMeta.directives_to_execute`.

When, at last, `PackageMeta.__new__` is called, `PackageMeta.list_names` contains
the list of attributes that we have to create. This is done by adding them
to the `attr_dict` variable. At this point, all is left is to call the actual
directives. But we cannot do that before the class is created, since the
directives take a class as argument. Hence, we have to call these directives
in `PackageMeta.__new__`, which is called with the created class. This function
goes through `PackageMeta.directives_to_execute`, finds `_execute_variant` in
it and calls it, which leads to the variant being added to `MyPackage.variants`.

And voilà!
