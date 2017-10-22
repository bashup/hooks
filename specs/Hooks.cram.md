# Event Hooks and AOP in bash

    $ source "$TESTDIR/../bashup.hooks"

## 'before' hooks

Because the docs don't really test "before" functions, let's do one now:

    $ my-hook() { local SOME_ARG=$1 OTHER_ARG=$2; }
    $ listener1() { echo "listener1: '$SOME_ARG'"; }
    $ hook.after my-hook listener1
    $ my-hook foo bar
    listener1: 'foo'

    $ listener3() { echo "listener3: '$SOME_ARG'"; }
    $ hook.before listener1 listener3
    $ my-hook bar
    listener3: 'bar'
    listener1: 'bar'

## Empty/Edge Cases

It should be possible to "before" an empty function:

    $ hook.before nosuch foo
    $ declare -f nosuch | sed 's/ $//'
    nosuch ()
    {
        foo
    }

Or "around" one:

    $ hook.around target nosuch foo
    $ declare -f target | sed 's/ $//'
    target ()
    {
        function foo ()
        {
            :
        };
        foo
    }

Or "around" an empty function with another empty:

    $ hook.around null1 null2 proceed
    $ declare -f null1 | sed 's/ $//'
    null1 ()
    {
        function proceed ()
        {
            :
        };
        proceed
    }

And it should be possible to eval code with braces in it, even in empty functions:

    $ hook.eval-before null3 '{ true || false; }'
    $ hook.eval-after  null4 '{ true || false; }'

And hook.body should handle braces in a default:

    $ hook.body null5 '{ true || false; }' '{xxx' 'yyy;}'; echo "$REPLY"
    {xxx
    { true || false; }
    yyy;}

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
