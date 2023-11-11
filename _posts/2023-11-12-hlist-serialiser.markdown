---
layout: post
title: "HList for serialization"
date: 2023-11-11 20:15:00 +0100
categories: programming pulumi
---

A lot of people are pretty negative about strong type systems, but I wanted to take a real world example from work where the use of a "fancy" type (in this case `HList`) allows the construction of a feature that supports some advanced use cases while still being type safe.

At work ([Pulumi](https://www.pulumi.com)) we've been thinking about how to build better frameworks for writing resource providers.

Two of the hardest parts of writing a resource provider are handling the serialization to and from the wire protocol to standard types, and writing a [Pulumi schema](https://www.pulumi.com/docs/using-pulumi/pulumi-packages/schema/) to describe the types to the engine and code generators.

# Generated providers

For the generated providers they don't need much built in support for these problems. As they're doing dynamic handling of the data from other source schemas they don't actually generate types internally to the provider, they just work directly with the dynamic values from the wire protocol and write the types they need directly to schema.

This isn't a bad framework for these providers, they only have to write their type information once (into the schema) and all their internal code is written dynamically anyway. We could probably make it a bit easier to generate the schemas, but if your writing providers at this level you probably want to care about schema rather than types in your providers implementation language.

But for manually written providers, where the programmer is wants to write just plain functions for each resource handling this stuff is a lot more effort. You end up having to write the type of your data into the schema, and then writing the type again in your provider source code. This is bad for two main reasons, firstly drift that what you write in one place might not match the other, secondly that manually writing schemas isn't a succinct way of writing types and requires you to know Pulumi schema.

# Reflection

The approach taken so far this problem has been to use reflection to go from types in the provider implementation to schema automatically. This is what the [Pulumi Go Provider](https://github.com/pulumi/pulumi-go-provider) does. You write a struct type like normal and add some annotations, for Go this looks very similar to JSON annotations.

```go
type RandomLoginArgs struct {
	PasswordLength pulumi.IntPtrInput `pulumi:"passwordLength"`
	PetName        bool               `pulumi:"petName"`
}
```

This works but it requires that the "shape" of your types in the provider match the "shape" on the wire. There's no way to customize this like you can for Go's JSON serialization where you can have `MarshalJSON` and `UnmarshalJSON` functions. But its a good start, and for a lot of cases is going to be enough, so when looking at building a similar system for DotNet it's where we started.

# DotNet 

The above in C# looks pretty similar, but using a C# class and attributes.

```csharp
public sealed class RandomLoginArgs : global::Pulumi.ResourceArgs
{
    [Input("passwordLength")]
    public Input<int>? PasswordLength = { get; set; } = null!;

    [Input("petName", required: true)]
    public Input<bool> PetName { get; set; } = null!;

    public RandomLoginArgs()
    {
    }
}
```

But again, we're limited to our domain objects being the same shape as the wire protocol.

# System.Text.Json.Serialization

This problem of converting between a wire type and a domain type looks a lot like the sort of conversion we do for domain objects and JSON. So the natural first place to go looking for ideas is [`System.Text.Json.Serialization`](https://learn.microsoft.com/en-us/dotnet/api/system.text.json.serialization), which adds extra attributes to tell the serialization framework about custom converters. A custom convert supplies two methods that tell the system how to read and write the value from and to JSON.

[Example](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/converters-how-to)
```csharp
public class DateTimeOffsetJsonConverter : JsonConverter<DateTimeOffset>
{
    public override DateTimeOffset Read(
        ref Utf8JsonReader reader,
        Type typeToConvert,
        JsonSerializerOptions options) =>
            DateTimeOffset.ParseExact(reader.GetString()!,
                "MM/dd/yyyy", CultureInfo.InvariantCulture);

    public override void Write(
        Utf8JsonWriter writer,
        DateTimeOffset dateTimeValue,
        JsonSerializerOptions options) =>
            writer.WriteStringValue(dateTimeValue.ToString(
                "MM/dd/yyyy", CultureInfo.InvariantCulture));
}
```

You could imagine a pretty similar API for converting between `PropertyValue` and `T` for our serialization system, and if all we cared about was converting values that would be fine. But we also want to generate Pulumi schema for these types. The above API doesn't have enough information for us to work that out automatically.

# Idea 1: Manual schema

So that's idea 1 for supporting custom serialization logic, just get the user to fill in the schema themselves. This idea might end up looking something like the following (using the same `DateTimeOffset` example):

```csharp
public class DateTimeOffsetPropertyValueConverter : PropertyValueConverter<DateTimeOffset>
{
    public override DateTimeOffset Read(
        PropertyValue value,
        Type typeToConvert,
        PropertyValueSerializerOptions options) =>
            DateTimeOffset.ParseExact(reader.GetString()!,
                "MM/dd/yyyy", CultureInfo.InvariantCulture);

    public override PropertyValue Write(
        DateTimeOffset dateTimeValue,
        PropertyValueSerializerOptions options) => 
            dateTimeValue.ToString(
                "MM/dd/yyyy", CultureInfo.InvariantCulture);

    public override string GetSchema() {
        "{\"type\": \"string\"}"
    }
}
```

It's workable, and if this was "really advanced users" only it would probably be acceptable. But it's still a bit sad because it reintroduces the two problems we we're trying to avoid, manually dealing with `PropertyValue` types and manually writing Pulumi schema.

# Idea 2: Thoth style coders

Inspired by [Thoth](https://thoth-org.github.io/Thoth.Json/) an F# JSON serialization framework one of our engineers suggested a similar approach for `PropertyValue`.

Rather than having a `Read` and `Write` methods we have methods to return an encoder and decoder. It takes away a lot of the manual dealing with `PropertyValue` but you still have to write a schema.

```csharp
public class DateTimeOffsetPropertyValueConverter : PropertyValueConverter<DateTimeOffset>
{
    public override IPropertyDecoder<DateTimeOffset> GetDecoder(
        Type typeToConvert,
        PropertyValueSerializerOptions options) =>
            Decoders.String(str => DateTimeOffset.ParseExact(str,
                "MM/dd/yyyy", CultureInfo.InvariantCulture));

    public override IPropertyEncoder<DateTimeOffset> GetEncoder(
        PropertyValueSerializerOptions options) => 
            Encoders.String(dt => dt.ToString("MM/dd/yyyy", CultureInfo.InvariantCulture))

    public override string GetSchema() {
        "{\"type\": \"string\"}"
    }
}
```

This is a fairly nice API, and for more complex objects its much easier to use than manually checking `PropertyValue` cases. Here's an example put together by the engineer who proposed this style:

```csharp
class UserConverter : PropertyValueConverter<User>
{
    public override IPropertyDecoder<User> Decoder()
    {
        return Decoders.Object(decode => 
            new User {
                Name = decode.Field("name", Decoders.String()),
                Aliases = decode.OptionalField(
                    fieldKey: "aliases",
                    fieldDecoder: Decoders.ImmutableArray(Decoders.String()),
                    defaultValue: () => new ImmutableArray<string>())
            }
        );
    }

    public override IPropertyEncoder<User> Encoder()
    {
        return Encoders.Object<User>(user => 
            new Dictionary<string, Task<PropertyValue>>
            {
                ["name"] = 
                    Encoders.String().Encode(user.Name),
                ["aliases"] =
                    Encoders.Sequence(Encoders.String()).Encode(user.Aliases)
            }
        );
    }
}
```

The longhand code of doing all those type checks is much worse, but we still need deal with the schema manually. There's no way for us to reflect over one of those decoder or encoder objects to see the inner shape. What we want is a way that ties the encode and decode together in such a way that we can also tell what the schema should be.

# Idea 3: Joint converters

Writing the encode and decode logic together looks really easy for the simple types. For example you could have `Converter.String<T>(str => t, t => str)` which builds a converter that always reads a string to T, and writes a T to string. Clearly the schema for this can be inferred just from the converter instance.

You can even do an array value pretty easily with `Converter.Array<T, U>(uConverter, uArray => t, t => uArray)`. The inner converter is known up front and we know this converter is building an array, so again it's pretty simple to infer the schema again.

## Invariant functor

A note on the above with the conversion functions arguments in each converter function. They aren't actually needed on these constructor functions. We can just have `Converter<string> String()`, `Converter<T[]> Array<T>(Converter<T> items)`, and an invmap function to then map to any other value `Converter<T> Invmap<T, U>(Converter<U>, Func<T, U>, Func<U, T>)`. You can apply `invmap` to any `Converter<U>` to turn it into a `Converter<T>`.

`invmap` because this is an instance of an "invariant functor", it's a functor not a bifunctor because it has only one type parameter, but that type parameter is in both an input position (for Write) and an output position (for Read). Normally when we talk about functors we mean covariant functors (where the type parameter is only in an output position). All covariant functors are also invariant functors, but not vice versa, so we can't define `fmap` for our `Converter<T>` type.

## Objects

The hard part of this design is when you get to objects. It's not hard to think up a design that works for a limited number of fields, you just ask for the name of each field and a converter for it and then you can make a converter to a tuple type, which given we know the field names we can build an isomorphism between the tuple and the object type.

```csharp
public static Converter<Tuple<T1, T2>> Object<T1, T2>(
    string field1, Converter<T1> converter1, 
    string field2, Converter<T2> converter2)
```

This again is pretty easy to work with, it's a bit sad we lose the field names in the returned tuple but there's no way to do generic records in C#. It's also pretty clear that this could automatically generate schema, we can ask the two sub-converters for their schemas and combine them together as the properties of an object type for this converters schema.

But what if a user want's to write an object with three fields, or seven, or twenty!? 

## HList

This is where `HList` comes into play. `HList` is a heterogenous list, that is each item in it can have a different type, but the whole list is strongly typed; not just `List<Object>`. You can think of them as variadic tuples. 

A `HList` is made of two values, nil and cons, just like a normal functional style linked list, but with different types.

```csharp
public static class HList {
    public static HNil Nil { get; }
    public static HCons<H, T> Cons<H, T>(H head, T tail) where T : HList<T>
}
```

Note the cons function taking a head and tail like a normal list, but that tail not being constrained to the same type of list as the head, just constrained to being a `HList` type (either `HNil` or `HCons<H, T>`).

Using this we can write two object converter functions that can be used to recursively build up any size object you want.

```csharp
public static class Converter {
    public static ObjectConverter<HNiL> Object();
    public static ObjectConverter<HCons<H, T>> Object<H ,T>(
        string name, 
        Converter<T> converter, 
        ObjectConverter<T> rest) where T : HList<T>
}
```

Note that we need to return `ObjectConverter` here not just `Converter`, but that's only for building up the overall object converter. The exposed interface of `ObjectConvereter` is exactly the same, the framework just needs that strong type so it can do the introspection we need to build the schema.

Using this type isn't too bad, not as simple as just a tuple or a record but it does support any number of fields safely. Here's an example of converting a `DateOnly` into a property value object of three fields "year", "month", "day".

```csharp
public static Converter<DateOnly> Date()
{
    var obj =
        Converter.Object("year", Converter.Number().Invmap(i => i, f => (int)f),
        Converter.Object("month", Converter.String(),
        Converter.Object("day", Converter.Number().Invmap(i => i, f => (int)f),
            Converter.Object())));

    return obj.Invmap(h => {
        string m;
        switch (h.Month)
        {
            case 0: m = "jan"; break;
            case 1: m = "feb"; break;
            // TODO: The other months
            default:
                throw new Exception("invalid month");
        }

        return HList.Cons(h.Year, HList.Cons(m, HList.Cons(h.Day, HList.Nil)));
    },
    v => {
        var year = v.Head;
        var month = v.Tail.Head;
        var day = v.Tail.Tail.Head;

        int m;
        switch(month)
        {
            case "jan": m = 0; break;
            case "feb": m = 1; break;
            // TODO: The other months
            default:
                throw new Exception("invalid month");
        }

        return new DateOnly(year, m, day);
    });
}
```

Note that building the object converter is pretty simple, despite using `HList` here the compiler correctly infers all the generic arguments.
The worst part of this API is having to ensure the order of the two lists used are the same, if the fields are different types the compiler does catch this but if you just had `HCons<int, HCons<int, HNil>>` for "x" and "y" you could accidentally build it as `HCons(x, HCons(y, HNil))` but read it as `var y = h.Head; var x = h.Tail.Head`. But that's the sort of mistake that's very easy to pick up in unit tests.

So we end up with a custom converter API that lets the user write their own logic but in a way that we can safely know the shape of the data and automatically infer Pulumi schema for it.

# Summary

I'm not sure we're going to build and publish a system like this, but I thought it was an interesting demonstration of what "exotic" types can give you.

