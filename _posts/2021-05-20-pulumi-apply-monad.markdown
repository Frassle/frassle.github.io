---
layout: post
title: "Pulumi's apply with monads"
date: 2021-05-20 17:18:00 +0100
categories: programming pulumi
---

Inspired by a [blog post by Lee Briggs](https://www.leebriggs.co.uk/blog/2021/05/09/pulumi-apply.html) about understanding apply in Pulumi.

Lee was writing about apply and comparing it to how async types in modern languages work, e.g promise in JS, or Task in C#. But like those async types themselves all of these can all be described with monads.

A refresher but a monad is simply any type which has a `return` (or `unit`) and `bind` operation. Where return can _lift_ a value into the monad space, and bind can take a value from within the monad and apply it to a function to get a new monad value.

```
// Lift a value into the monad M
return : T → M<T>

// Take the T value from a monad, call a function to transform that T value into a
// new monad with a U value.
bind : M<T> → (T → M<U>) → M<U>
```

The monad in Pulumi's design isn't explicitly clear but instead split over two types, Input and Output. An `Input<T>` is either a plain `T` value, an `async T` value (i.e. `Task<T>` in C#), or an `Output<T>`. An `Input<T>` can be used to create an `Output<T>`, so this is just a long handed way of implementing monadic return. There is an interesting point here that we can lift `T` or `async T` into the Pulumi monad space rather than just `T`, this is because Outputs are kind of like a subtype of the async type (as Lee described in his blog post).

The second part of the monad is bind, and that's all apply is. It's just monadic bind, given an `Output<T>` and a function from `T` to `Output<U>` you can get an `Output<U>`. So Output is a monadic type.

I want to make it clear that Output and Pulumi's eventual consistency engine really are monadic. There's a warning in the [pulumi documentation](https://www.pulumi.com/docs/intro/concepts/inputs-outputs/):
> During some program executions, apply doesn’t run. For example, it won’t run during a preview, when resource
> output values may be unknown. Therefore, you should avoid side-effects within the callbacks. For this reason,
> you should not allocate new resources inside of your callbacks either, as it could lead to pulumi preview
> being wrong.

But preview being wrong is the only error that happens if you do make resources in apply, the types all match up and the Pulumi engine doesn't actually have any issue with going off and making these new resources and continuing the graph. In fact this trick is even used in some of the pulumi examples ([azure firewall example](https://github.com/pulumi/examples/blob/master/classic-azure-ts-appservice-devops/infra/index.ts#L106)).

This is more powerful and general than what tools like terraform and cloudformation are capable of. Terraform has some handling for creating a dynamic number of resources based on a collection ([for_each](https://www.terraform.io/docs/language/meta-arguments/for_each.html)), but this is not as general as what's possible with apply.

The downside to being monadic is that preview can be wrong. There is no way to statically analyse a monadic graph without actually running it, so at preview time there's no way for the Pulumi engine to _look inside_ the functions for apply to see what might be created.

For simple cases like the firewall example above it would probably be possible to add a special `for_each` function constrained like terraform is and because it's constrained it would be possible to show at preview time what was going to be created. Pulumi could also take ideas from [selective applicative functors](https://www.staff.ncl.ac.uk/andrey.mokhov/selective-functors.pdf) to cover conditional cases that would today be hidden behind a bind.

However in general there will always be some expressions that are possible with bind that aren't expressible any other way. It might be those expressions aren't actually ever needed in the programs people are writing to spin up infrastructure and Pulumi could become safer to use by removing apply and requiring all code to be written in a way that can be shown accurately at preview time. My feeling is it's probably best to let people have this power as an escape hatch for when it's needed but to provide and encourage them to use the safer alternatives where possible. You can comment on that idea at [#5464](https://github.com/pulumi/pulumi/issues/5464).