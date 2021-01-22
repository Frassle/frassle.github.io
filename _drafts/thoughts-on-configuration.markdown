---
layout: post
title:  "Thoughts on configuration"
date:   2021-01-07 20:34:25 +0000
categories: programming
---

I was looking into configuration languages the other day and ended up in a bit of a typical rabbit hole for software of "everything is awful". Every configuration format seemed to be lacking, and each formats community would lambast the others for having issues but they all seem to have issues.

## Categorization

Anyway a situation summary for those who haven't tried to delve these depths recently. Configuration formats come in a variety of formats, from very simple lines of "key=value" pairs to literal programming languages. I think I've had the joy of dealing with the whole spectrum (although thankfully not every separate format). I've found that you can categorize them in three main ways.

### Bespoke formats

I'm going to group lines of key value pairs under this heading because they're so simple that most of the time you'll just write a bespoke function to line split then string split on '='. There are more complex bespoke formats, the main things about these though is no standard and normally no libraries.

The positives of these are pretty limited, you potentially get a syntax that is "just right" for your configuration domain but unless your domain is really simple (like just flat key value pairs) then you either need to do a lot of work on your one use case configuration parser, or just put up with a sub-standard format for the domain.

I'm not really going to talk any more about these, I don't think their interesting. Lots of programs did bespoke formats back in the day because well there wasn't really any other choice. You'd be sane to pick one of the options below if starting something new today.

### Standard formats

This covers everything from INI to HCL. These are formats that are used by many people and have some sort of specification. The strictness of that can vary, everyone knows what INI looks like even if there isn't really an official specification for it, while JSON is fully defined by [ECMA-404].

I'm going to circle back to discuss these later. There's more of interest to discuss regarding these formats.

### Programming languages

Programming languages can express data so just run a program to get the configuration. It sounds neat but it has quite a few downsides, however there are two conditions that if met make this probably the best choice.

If you can use a language the team are very comfortable with (not necessarily the same the system is written in), and the configuration is complex, then just invoke a program to build it.

Common examples of this are people invoking Python or Lua to build configuration because they're pretty easy to script and you can just eval in the config files.

A good example of this approach is [Pulumi]. The configuration used to build cloud infrastructure is normally set up by the same people using and deploying software to that infrastructure, and they are going to be comfortable in some sort of programming language, and cloud infrastructure gets very complex fast. Being able to build that config up with a proper language rather than with thousands of lines of yaml is great.

However this is not always a great choice. If your configuration isn't complex than the overhead of a full programming rather than a simpler configuration language probably isn't worth it. If the people doing the configuration aren't all comfortable with a set language then the overhead of which ever language picked is probably not going to be worth it. Your not going to make friends telling people they need to learn your favourite niche language X just to configure a system you've made that they want to use.

One honourable idea in this space is [Dhall]. It's a non-turing complete, sandboxed language for configuration. Those two attributes means no worry about infinite loops from user config, and no worry about overly malicious config going and reading files and accessing the network. It's a really nice idea, but even though it's a relatively simple programming language I suspect its still much much harder to lean than say YAML, espeically for a non-programmer. In fact given it's Haskell like syntax I imagine most programmers would find it a bit taxing to learn.

I'm going to cycle back to this idea after discussing the issues with standard formats, I think there's something to be said for making this idea more attainable.

## Standard formats

There's lots of standard formats, I'm not going to try to be exhaustive but I want to comment on the more popular ones.

### INI

Ah good old INI. This has been around for a long time, it's quite simple and has been used for program configuration across both Windows and Linux. It isn't strictly defined however and there's a lot of variation in what programs mean when they say INI. See the list of options in [libconfini](https://github.com/madmurphy/libconfini/wiki/INI-formats) for evidence of this.

```
[Section]
key=value
```

INI isn't so popular these days. I think probably from a combination of not being specified well enough to be implemented mostly equivalently across lots of languages, and due to not being able to express hierarchal data very well.

One thing INI gets right _imo_ is that strings don't need to be quoted. I'll explain why I think this later on.

### JSON

If I was writing this five years ago I would of probably said JSON is the new INI. Times have moved on and I feel that position is now taken by YAML.

```
{ "number": 4.0, "string": "some string", "array": [1, "2", true] }
```

Everyone knows JSON and everyone knows how so many things are configured by JSON files. It's well specified has implementations in many languages and isn't very complex. It can be very painful to maintain by hand however. It's a very good serialisation format, it's not such a nice configuration format.

### YAML

JSON made "friendly" to humans. An easier for humans format of JSON isn't a terrible idea, but a number of questionable decisions (e.g. [the norway problem](https://hitchdev.com/strictyaml/why/implicit-typing-removed/)) have lead to a strange format that seems human friendly and then blows your foot off.

```
number: 4.0
string: some string
array:
    - 1
    - "2"
    - true
```

[I *do not* like YAML](https://noyaml.com/). This feeling has probably grown due to the vast amount of dev tools that decided YAML was the future. Devs know programming languages and a lot of the tools had complex configuration and so fit into my comment above about _just use a programming language!_ (cf. k8s with Pulumi vs k8s with helm)

### TOML

TOML is a sort of modern INI. It's reasonably well known because it's used by [Rust] for cargo files.

```
[Section]
number = 4.0
string = "some string"
array = [1, "2", true]
```

TOML is what started by descent into configuration languages. I'd heard generally good things about it from Rust so did some more searching and very quickly come across various comments and articles lamenting that it wasn't actually very good \[[1](https://hitchdev.com/strictyaml/why-not/toml/)\] \[[2](https://github.com/madmurphy/libconfini/wiki/An-INI-critique-of-TOML)\].

### [UCL](https://github.com/vstakhov/libucl) / [HCL](https://github.com/hashicorp/hcl)

UCL is an attempt at trying to standardise the config style used by nginx. HCL is hashicorps implementation of the same idea, the syntax of both is very similar.

```
key = value;
section {
    flag = true;
    number = 10k;
    time = 0.2s;
    string = "something";
    subsection {
        host = {
            host = "hostname";
            port = 900;
        }
        host = {
            host = "hostname";
            port = 901;
        }
    }
}
```

It follows YAML in that it's quite feature rich, so people feel they can push it quite far. I've seen some *huge* HCL files. If kept in check for complexity its probably my favourite of the list, because it can handle hierarchies and arrays pretty well, doesn't have as many footguns as YAML, and is more pleasant for humans to write than JSON.

## What's the point of configuration files

Most of the above I feel fall over due to one design choice they've made. They are not type driven, they all consider a configuration file to be much the same as a JSON file. In fact most of them provide an easy mapping to and from JSON (at least semantically, if not textually).

I think this is the wrong way of thinking about configuration. It does not need to be a general serialisation format and the two conflict with each other on various points.

If your writing a configuration file its to define values for a given application, that application can define what the _type_ of the configuration is. Once you have a type you can drive your parser from that type. YAMLs auto casting of "no" to the boolean value false is fine if the application wants a boolean for that key, and it's catastrophic if the application wants a string. But with type driven parsing you know this so you can have human friendly features like "bools are expressible as yes/no, true/false, on/off" without causing issues.






Serialisation != Configuration

Types driven by application, not the file

Compare to command line arguments, if --foo is a string you just write `--foo string` not `--foo \"string\"` as long as it parses to the right type all is good.


"Be conservative in what you do, be liberal in what you accept from others. — Postel's law"
Eh this always feels helpful but often leads to mess over the long term

yamls auto casting of "no" to boolean false is fine IFF the application wants a boolean in that location.

Reading a yaml or similar config file you shouldn't get back types that the parser thinks things are, the parser should just recall that there is a value there and what it's raw text was.
E.g. the yaml:
key: no

In something like python would currently give you back something like {"key": false}, instead it should give back {"key": YamlNode("no") } where the application then has to call .AsBoolean or .AsString on that YamlNode to invoke the relevant parser.

Likewise in a more staticly type langauge like C# where you'd get a YamlNode with a Kind or Type property of Boolean, just get rid of that property keep hold of the raw text and invoke the parser when the application tells you what the type should be. This idea is fine even for configurations where the value could be a string or an integer, you can just try the more restrictive parse first and if that fails try the other one.

Fancy ideas could even take a type that describes the whole configruation file and try to parse the whole file into that type, C# could use generics and reflection to control the parser.



[ECMA-404]: http://www.ecma-international.org/publications/files/ECMA-ST/ECMA-404.pdf
[Pulumi]: https://pulumi.com
[Dhall]: https://dhall-lang.org/
[Rust]: https://www.rust-lang.org/