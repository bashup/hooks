# Hooks, event listeners, and AOP for bash

This small (~1K, ~30 line) bash module provides tools for modifying bash functions in such a way as to allow using them as hooks, events, or AOP-style "advice".  It is written using only bash builtins, and so has no dependencies.

### Installation, Requirements And Use

Copy and paste the [code](bashup.hooks) into your script, or place it on `PATH` and `source bashup.hooks`.  (If you have [basher](https://github.com/basherpm/basher), you can `basher install bashup/hooks` to get it installed on your `PATH`.)  The code is licensed [CC0](http://creativecommons.org/publicdomain/zero/1.0/), so you are not required to add any attribution or copyright notices to your project.

```shell
    $ source bashup.hooks
```

### Events and Hooks

The main API functions are `hook.before`, `hook.after`, and `hook.without`.  Each takes a function name, followed by a command or function name and optional arguments.  The function named by `$1` is then modified accordingly (or created, if it doesn't exist):

* `hook.before` *func cmd...* modifies *func* to run *cmd...* **before** its existing code
* `hook.after` *func cmd...* modifies *func* to run *cmd...* **after** its existing code
* `hook.without` *func cmd...* modifies *func* to **remove** *cmd...* from its existing code

Using these functions, you can then implement an event-listener or hook pattern like this:

```shell
# Check for function existence with hook.exists:
    $ if ! hook.exists event1; then echo "not yet"; fi
    not yet

# Subscribe to events using hook.after (which auto-creates the function)
    $ hook.after event1 echo "got event1"
    $ hook.after event1 echo "isn't this cool?"

    $ if hook.exists event1; then echo "event1 exists!"; fi
    event1 exists!

# Fire events by calling the function:
    $ event1
    got event1
    isn't this cool?

# Unsubscribe using hook.without:
    $ hook.without event1 echo "got event1"
    $ event1
    isn't this cool?

# Removing everything from a function leaves a no-op (:)
    $ hook.without event1 echo "isn't this cool?"
    $ declare -f event1 | sed 's/ $//'
    event1 ()
    {
        :
    }
```

#### Passing Arguments

Note that the *cmd...* passed to these functions does not have access to any arguments passed to the altered function.  You can work around this by setting variables inside the hook/event function, which are then accessible from any listener functions:

```shell
# Expose arguments as variables:
    $ my-hook() { local SOME_ARG=$1 OTHER_ARG=$2; }

# Use a function to access call-time values:
    $ listener1() { echo "listener1: '$SOME_ARG'"; }
    $ hook.after my-hook listener1
    $ my-hook foo bar
    listener1: 'foo'
```

Note that it does *not* work to reference these variables when *registering* the funtion, e.g.:

```shell
# WRONG; echos the listen-time value!
    $ hook.after my-hook echo "listener2: '$SOME_ARG'"
    $ my-hook baz
    listener1: 'baz'
    listener2: ''
```

This is because bash will interpolate the value of `$SOME_ARG` that exists at the time of registration, rather than at the time of invocation.

(If you want to have listeners be able to receive arguments directly, you may want to use `hook.eval-after` instead, as described in the next section.)

### AOP and Monkeypatching

Sometimes, the function you need to modify is not one whose source you control, so you can't rely on it supplying its arguments in a particular form, or you need to inject code that's not a function call, etc.

For these scenarios, you can use `hook.eval-before`, `hook.eval-after`, and `hook.eval-without`.  These functions take a function name and a string; the string is then injected (or removed) from the function source as-is.  (The non-`eval` variants of these functions actually work by invoking the `eval` variants with their arguments combined into a properly-escaped string using `hook.quote-args`.)

(Note: all of these functions require their string argument to be syntactically valid as one or more *complete* bash statements, or a syntax error will be the result.)

As an example, we can use `hook.eval-after` to make a listener that reads its arguments directly:

```shell
# remove the broken hook from the previous example
    $ hook.without my-hook echo "listener2: ''"

# replace it with one that works
    $ hook.eval-after my-hook $'echo "listener2: \'$1\'"'
    $ my-hook spam
    listener1: 'spam'
    listener2: 'spam'
```
Of course, this requires you to quote and escape the code correctly, and to remove a listener you have to carefullly match the original string:

```shell
    $ hook.eval-without my-hook $'echo "listener2: \'$1\'"'
    $ my-hook again
    listener1: 'again'
```

So be sure to carefully test any code that directly uses the `hook.eval-` variants.

### "Around" Advice

Sometimes, what you really want is to wrap a function with some logic, like a condition or loop.  Or perhaps post-process its results, or change its arguments. The  `hook.around` function lets you do this, by defining a second function as a *template* for how the first function will be modified.  For example, suppose we are using this function:

```shell
    $ hello() { echo "hello $1!"; }
```

And we would like it to default to `world` as its first argument, if no arguments are given.  We can do that like so:

```shell
    $ hello-with-default() {
    >     if (($#)); then
    >         hello-without-default "$@"
    >     else
    >         hello-without-default "world"
    >     fi
    > }

# Wrap hello's old definition as 'hello-without-default` inside a copy of 'hello-with-default':
    $ hook.around hello hello-with-default hello-without-default
```

`hook.around` will then rewrite the body of the `hello` function so it looks like this:

```bash
    $ declare -f hello | sed 's/ $//'
    hello ()
    {
        function hello-without-default ()
        {
            echo "hello $1!"
        };
        if (($#)); then
            hello-without-default "$@";
        else
            hello-without-default "world";
        fi
    }
```

It's a good idea to name the wrapped function something that reflects the intent of your wrapper, so as to avoid collisions with other function names.  It's a bad idea to use generic names like `original-foo` or `wrapped-foo`, because more than one piece of advice might be named the same way, resulting in one overwriting the other.

### Utilities

#### hook.exists

`hook.exists` *function* returns truth if a function named *function* exists, and false otherwise.

#### hook.quote-args

`hook.quote-args` *variable args...* sets *variable* to a space-separated list of the given arguments in shell-quoted form (using `printf %q`.  The resulting string is safe to `eval`, in the sense that the arguments are guaranteed to expand to the same values (and number of valus) as were originally given.  (The resulting string also begins with a space, so if you're using it for something other than `eval` you may need to remove it.)

#### hook.body

`hook.body` *function defaultbody [before [after]]* sets `REPLY` to the body of *function*'s source code, optionally replacing the opening brace with *before* and the closing brace with *after*.  If *before* is not supplied, it defaults to `{`; if *after* is not supplied, it defaults to `}`.  If *function* doesn't exist, it's treated as if it did exist, and had *defaultbody* inside the braces.

## License

<p xmlns:dct="http://purl.org/dc/terms/" xmlns:vcard="http://www.w3.org/2001/vcard-rdf/3.0#">
  <a rel="license" href="http://creativecommons.org/publicdomain/zero/1.0/"><img src="https://licensebuttons.net/p/zero/1.0/80x15.png" style="border-style: none;" alt="CC0" /></a><br />
  To the extent possible under law, <a rel="dct:publisher" href="https://github.com/pjeby"><span property="dct:title">PJ Eby</span></a>
  has waived all copyright and related or neighboring rights to <span property="dct:title">bashup/hooks</span>.
This work is published from: <span property="vcard:Country" datatype="dct:ISO3166" content="US" about="https://github.com/bashup/realpaths">United States</span>.
</p>