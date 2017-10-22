# Event Hooks and AOP in bash

    $ source "$TESTDIR/../bashup.hooks"

## Test Documentation Examples

First, We run the docs with mdsh, and the results should be as described:

    $ eval "$(mdsh --compile "$TESTDIR/../README.md")"
    got event1
    isn't this cool?
    isn't this cool?
    listener1: 'foo'
    listener1: 'baz'
    listener2: ''

In adddition, the `hello` function should look like that example's output:

    $ declare -f hello | sed 's/ $//'
    hello ()
    {
        function defaulted-hello ()
        {
            echo "hello $1!"
        };
        if (($#)); then
            defaulted-hello "$@";
        else
            defaulted-hello "world";
        fi
    }

## 'before' hooks

Next, because the docs don't really test "before" functions, let's do some now:

    $ listener3() { echo "listener3: '$SOME_ARG'"; }
    $ hook.before listener1 listener3
    $ my-hook bar
    listener3: 'bar'
    listener1: 'bar'
    listener2: ''

    $ hook.without my-hook echo "listener2: ''"
    $ my-hook test
    listener3: 'test'
    listener1: 'test'


## 'eval' Variants and 'without' Syntax

It should be possible to remove stuff from the start of a function:

    $ hook.without listener1 listener3
    $ my-hook 'again!'
    listener1: 'again!'

And to add or remove arbitrary statements with the eval variants, even with different spacing, etc.:

    $ hook.eval-before listener1 'if [[ $SOME_ARG == hello ]]; then echo world; fi'
    $ my-hook hello
    world
    listener1: 'hello'

    $ hook.eval-without listener1 'if [[ $SOME_ARG == hello ]];then echo world ; fi'
    $ my-hook hello
    listener1: 'hello'

    $ hook.eval-after listener1 'if [[ $SOME_ARG == hello ]]; then echo world; fi'
    $ my-hook hello
    listener1: 'hello'
    world

    $ hook.eval-without listener1 'if [[ $SOME_ARG == hello ]];  then echo world; fi'
    $ my-hook hello
    listener1: 'hello'
