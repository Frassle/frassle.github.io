---
layout: post
title:  "Generator.send as monadic bind"
date:   2021-01-22 21:52:00 +0000
categories: programming python
---

I'm a functional programmer at heart, so I'm quite happy thinking in terms of things like functors and monads.
I use F# a lot which has _computation expressions_ which is nice syntax for using monads without having to write bind continuations everywhere.

{% highlight fsharp %}
let final = Result {
    let! x = some_function_that_returns_result()
    let y = some_normal_function(x)
    return y
}
{% endhighlight %}

`final` in the above code will either be `Error` if `some_function_that_returns_result` returned an error, or it will be `Ok` with the value of y.

Today I was pointed at [generator.send](https://docs.python.org/3/reference/expressions.html#generator.send) in Python. Which you can ~~abuse~~ use to get a pretty similar flow.

{% highlight python %}
class Result(object):
    __none = object()

    def __init__(self, ok=__none, error=__none):
        self.ok = ok
        self.error = error

    @staticmethod
    def monadic(func):
        def wrapper(*args, **kwargs):
            gen = func(*args, **kwargs)
            value = None
            while True:
                try:
                    result = gen.send(value)
                    if result.ok is not Result.__none:
                        value = result.ok
                    else:
                        return result
                except StopIteration as ex:
                    return Result(ok=ex.value)

        return wrapper

    def __str__(self):
        if self.ok is not Result.__none:
            return "Ok({})".format(self.ok)
        else:
            return "Error({})".format(self.error)

def do_something(value):
    print("Do something, got value {}".format(value))
    return Result(ok=value+4)

def do_something_else(value):
    print("Do something else, got value {}".format(value))
    return Result(ok=value*2)

def do_something_and_maybe_fail(value):
    print("Do something and maybe fail, got value {}".format(value))
    if value <= 10:
        print("Don't fail")
        return Result(ok=value)
    else:
        print("Do fail")
        return Result(error="Some bad error")


@Result.monadic
def example(value):
    x = yield do_something(value)
    y = yield do_something_and_maybe_fail(x)
    z = yield do_something_else(y)
    return z

if __name__ == "__main__":
    print("Run passing example")
    result = example(1)
    print("Final result = {}".format(result))
    print()
    print("Run failing example")
    result = example(10)
    print("Final result = {}".format(result))
{% endhighlight %}

If you execute the above python you'll get the following output:
```
Run passing example
Do something, got value 1
Do something and maybe fail, got value 5
Don't fail
Do something else, got value 5
Final result = Ok(10)

Run failing example
Do something, got value 10
Do something and maybe fail, got value 14
Do fail
Final result = Error(Some bad error)
```

Isn't that fun! It's monadic bind in python and it doesn't even look that bad.