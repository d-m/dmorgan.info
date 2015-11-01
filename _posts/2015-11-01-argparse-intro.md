---
layout: post
title: "argparse: A quick overview of command line interfaces in Python"
modified: 2015-12-01 10:02:07 -0400
excerpt: Create a simple Python command line utility using the argparse module.
category: posts
tags: [argparse, python, command line, programming, utilities, terminal]
image:
  feature:
  credit:
  creditlink:
comments:
share:
---

<section id="table-of-contents" class="toc">
  <header>
    <h3>Overview</h3>
  </header>
<div id="drawer" markdown="1">
*  Auto generated table of contents
{:toc}
</div>
</section>

The [`argparse`][2] module comes as part of the Python standard library and is used for creating command-line interfaces. You can use it to easily add arguments and create complex argument parsing rules for your command-line utility. This module helps out by:

  1. automatically generating and formatting your command line utility's usage, help messages, and description.
  2. parsing and validating user input to your command line programs.
  3. raising appropriate error messages and displaying help text when user input doesn't match what the program is expecting.

The `argparse` module has many options and can be overwhelming when beginning to code command-line utilities. However, you can create a functional, user-friendly command-line interface by utilizing a small subset of this package.

## The ArgumentParser ##
All `argparse` command line interfaces begin with an instance of `ArgumentParser`.

### The simplest parser ###
The simplest proof of concept can be written in two lines:

{% highlight python %}
import argparse
parser = argparse.ArgumentParser()
{% endhighlight %}

Testing this code in a [Jupyter notebook][1] prints:

{% highlight console %}
>>> parser.parse_args(['--help'])
usage: __main__.py [-h]

optional arguments:
  -h, --help  show this help message and exit

SystemExit: 0
{% endhighlight %}

Let's go over this output quickly. the `ArgumentParser.parse_args` method takes a list of strings and validates it as command-line input. By default, `ArgumentParser` instances include the `--help` optional argument. When this argument is present in the input, the parser prints out usage information consisting of the name of the utility and any arguments. Below that, the parser prints each argument and its associated help text. Internally, this is done by calling `ArgumentParser.print_usage` and then `ArgumentParser.print_help`.

The program name is shown as `__main__.py`. This is an artifact of the Jupyter notebook. By default, `argparse` derives its name from the first item in `sys.argv` and the Jupyter notebook starts the Python kernel by executing the kernel's `__main__.py`. Getting your program name from `sys.argv` is usually what you want; by using `sys.argv` your usage information matches your user's command-line invocation of your utility.

The last line, `SystemExit: 0`, is not included in the help output. It is also part of the Jupyter notebook output and shows the resulting exit code when parsing arguments.

### Making your utility descriptive ###

`ArgumentParser` instances have a few options available for increasing the descriptiveness of your command line utility:

{% highlight python %}
parser.prog = 'PROG'
parser.description = "This is where the command-line utility's description goes."
parser.epilog = "This is where the command-line utility's epilog goes."
{% endhighlight %}

Now when we print the help text we get:

{% highlight console %}
>>> parser.parse_args(['--help'])
usage: PROG [-h]

This is where the command-line utility's description goes.

optional arguments:
  -h, --help  show this help message and exit

This is where the command-line utility's epilog goes.

SystemExit: 0
{% endhighlight %}

Now our parser is a little more descriptive.

## Adding arguments ##

The `ArgumentParser.add_argument` method adds positional or optional arguments to the parser. These types of arguments allow users to interact with your program using the command line.

### Positional arguments ###

Positional arguments are required arguments for running your command line utility. We can add one to our parser with:

{% highlight python %}
...
parser.add_argument('foo')
{% endhighlight %}

Now when we print the help text we get:

{% highlight console %}
>>> parser.parse_args(['--help'])
usage: PROG [-h] foo

This is where the command-line utility's description goes.

positional arguments:
  foo

optional arguments:
  -h, --help  show this help message and exit
{% endhighlight %}

The usage information updates to show the new invocation for our utility and the help text now has a "positional arguments" section listing our argument.

We can add a second argument `bar` similarly:

{% highlight python %}
parser.add_argument('bar', help="bar help")
{% endhighlight %}

This argument has additional help text passed in as a keyword argument that will be shown next to the argument with the rest of the help output:

{% highlight console %}
>>> parser.parse_args(['--help'])
usage: PROG [-h] foo bar

This is where the command-line utility's description goes.

positional arguments:
  foo
  bar         bar help

optional arguments:
  -h, --help  show this help message and exit

This is where the command-line utility's epilog goes.
{% endhighlight %}

Once again, the usage information updates to incorporate the `bar` argument and the help text we supplied appears in the "positional arguments" section.

### Optional arguments ###

Optional arguments are preceded by a `--` and added to an `ArgumentParser` instance using the same `add_argument` method as positional arguments. These types of arguments can also include a short alias. A short alias is a single letter preceded by a `-`. It is a more convenient but less descriptive way for your user to pass in your optional argument.

We can add an optional argument and accompanying short alias using:

{% highlight python %}
parser.add_argument('-b', '--baz', help="baz help")
{% endhighlight %}

The short alias and the full name of your option can be supplied in either order. However, the short alias should come first to be consistent with how the `--help` argument is printed in the help text. The help text now reads:

{% highlight console %}
>>>parser.parse_args(['-h'])
usage: PROG [-h] [-b BAZ] foo bar

This is where the command-line utility's description goes.

positional arguments:
  foo
  bar                bar help

optional arguments:
  -h, --help         show this help message and exit
  -b BAZ, --baz BAZ  baz help

This is where the command-line utility's epilog goes.
{% endhighlight %}

The usage information now includes the new argument surrounded in square brackets to indicate that it is optional. The "optional arguments" section also shows both ways to set this argument.

The help text for our `--baz` argument is followed by `BAZ` unlike the text for the `--help` argument. This is because our `--baz` argument expects input while the `--help` argument does not. Optional arguments like '--help' function as booleans. In `argparse` this is accomplished by supplying the `add_argument` method the keyword argument `action='store_true'`.

## Parsing arguments ##

Now that we have a simple command line utility we're ready to parse some arguments. `ArgumentParser` instances validate input using the `parse_args` method. Throughout this post we've been using this method to parse `-h` optional argument to display help text. The `parse_args` method expects a list of strings to use as input. Typically, this list will come from `sys.argv` but this list can be passed in using an interpreter like iPython or a Jupyter Notebook as well.

Submitting a single argument "abc" results in:

{% highlight python %}
>>> parser.parse_args('abc'.split())
usage: PROG [-h] [-b BAZ] foo bar
PROG: error: too few arguments
{% endhighlight %}

This is because we have two required positional arguments and only supplied one to our utility.

If we submit two, we get the result:

{% highlight python %}
>>> namespace = parser.parse_args('abc xyz'.split())
>>> print(namespace)
Namespace(bar='xyz', baz=None, foo='abc')
{% endhighlight %}

The `argparse` module parsed our valid input and returned a `Namespace` object. A `Namespace` instance is similar to a `dict`: it contains key-value pairs of arguments and their values. However, these keys are properties of the `Namespace` object and are accessed using dot syntax:

{% highlight python %}
>>> print(namespace.foo)
abc
>>> print(namespace.bar)
xyz
>>> print(namespace.baz)
None
{% endhighlight %}

 Our utility assigned the first input argument to `foo` and the second to `bar`. The `--baz` argument wasn't used so its value is `None`.

Let's include `--baz` now:

{% highlight python %}
>>> parser.parse_args('abc xyz --baz'.split())
usage: PROG [-h] [-b BAZ] foo bar
PROG: error: argument -b/--baz: expected one argument
{% endhighlight %}

Whoops, we get the usage text and an error saying that `--baz` expected an argument.

If we resubmit our input with another argument, we get:

{% highlight python %}
>>> parser.parse_args('abc xyz --baz 123'.split())
Namespace(bar='xyz', baz='123', foo='abc')
{% endhighlight %}

The `--baz` argument gets assigned the value "123" as expected. Since optional arguments are preceded by `-` or `--` they can go anywhere in the list.

Arguments are assigned to properties in the resulting namespace based on their name. For optional arguments that also have a short alias, the namespace property matches the longer name without the `--`.

Now that `argparse` is taking care of the input validation for our utility we can make a script and focus on the business logic.

## Making a script ##

Here's a simple script for our parser:

{% highlight python %}
import argparse
import sys

# parser setup
parser = argparse.ArgumentParser()
parser.prog = 'PROG'
parser.description = "This is where the command-line utility's description goes."
parser.epilog = "This is where the command-line utility's epilog goes."
parser.add_argument('foo')
parser.add_argument('bar', help="bar help")
parser.add_argument('-b', '--baz', help="baz help")

# input parsing
namespace = parser.parse_args(sys.argv[1:])

# business logic
if namespace.baz:
    print(
        'You provided {}, {}, and {}'.format(
            namespace.foo, namespace.bar, namespace.baz
        )
    )
else:
    print('You provided {} and {}'.format(namespace.foo, namespace.bar))
{% endhighlight %}

This script includes some toy logic to show how we can use the `Namespace` object to change how our utility operates based on user input. We can run this script from the terminal like so:

{% highlight console %}
$ python test.py abc xyz
You provided abc and xyz  
{% endhighlight %}

or:

{% highlight console %}
$ python test.py abc xyz --baz 123
You provided abc, xyz, and 123  
{% endhighlight %}

[1]: https://jupyter.org "Jupyter Notebook home page"
[2]: https://docs.python.org/3/library/argparse.html
