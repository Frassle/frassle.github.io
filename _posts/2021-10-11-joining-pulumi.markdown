---
layout: post
title: "Joining Pulumi"
date: 2021-10-11 08:30:00 +0100
categories: update personal
---

Today is my first day at [Pulumi](https://www.pulumi.com/)!

I've been following Pulumi since Joe Duffy [first announced](http://joeduffyblog.com/2017/06/01/an-update-on-me-pulumi/) it on his blog over four years ago! I was already an avid reader of Joe's blog, enjoying the technical aspects of his midori series and influenced by his writing on software leadership and architecture. So while very little was said in that first post it got my interest and I signed up for updates.

## Arm

At the time I was a software engineer at [Arm](https://www.arm.com/) working in the test submission and management team. We were responsible for building the systems that allowed the GPU hardware and software developers to test their work. Imagine lots of FPGAs and custom silicon racked up along with a chunk of x86 compute boxes for doing cross platform builds of the drivers, test applications, and other artefacts.

We had spent the end of 2017 and the first half of 2018 trying to modernise our stack. The company had built a new OpenStack cluster to replace our out-going physical machines and the team was learning all about this new Infrastructure as Code, DevOps, and playbook automation. This meant learning puppet, ansible, and terraform. While this was all new and kind of cool (oh look I can get a whole network and set of virtual machines built from the command line), there was something about it that always seemed lacking. Then at the start of June 2018 I got an email about the Pulumi private beta.

![Pulumi private beta email](/assets/pulumi_private_beta.png)

At the time Pulumi only supported TypeScript as a language, and I think only AWS and Azure as infrastructure targets. But there was enough there to see the core idea and get excited. This had the spark that the other tools were missing, I could write actual software and not a load of configuration files.

A few days later my second child was born and I took eight weeks off for paternity leave. I spent the first couple of weeks of that leave writing [OpenStack support for Pulumi](https://github.com/pulumi/pulumi-openstack/tree/818d30ec785d09579da571a94e278289bfb880d8) (New babies sleep a lot) as that was what we were using at Arm. Pulumi very kindly accepted that contribution into their GitHub org and took over maintenance of it. At some point that summer I played around with re-writing all of the terraform code we had at work into TypeScript to see how it felt and looked. While in raw numbers it wasn't a huge decrease in line count (There wasn't a _huge_ amount to start with) it did show how much stuff had been copy-pasted, and flagged up a few bugs. It was also really easy and a joy to write it. Having full IDE support with type checking and auto-complete was so nice compared to the raw text editor experience we had wrote all the HCL with.

On return from paternity leave I gave a very enthusiastic presentation about Pulumi to the team, but very shortly after that left Arm. My engineering manager did ask on my last day if I was leaving to join Pulumi as I had been so talkative about it! I haven't kept in touch with that team, so no idea if my sales pitch landed, if they stuck with terraform or something else.

## G-Research

On the 3rd of September 2018 I joined [G-Research](https://www.gresearch.co.uk/). There I joined the team responsible for building parts of the system for helping the quants run and manage large sets of jobs to our internal compute farm. Initially this was focused on user facing libraries and programs so I had a break from thinking about services and infrastructure. However the company was starting to [set up Kubernetes clusters](https://www.gresearch.co.uk/article/kubernetes-on-windows-are-we-crazy/) for development teams to move services to, and so new services we had been planning to build would be targeted at that. Pulumi had support for Kubernetes by this point, but my team in G-Research was primarily an F# team with some C# and that was not yet supported by Pulumi. At the end of September I raised a [PR](https://github.com/pulumi/pulumi/pull/2003) to see about getting dotnet languages supported in Pulumi. As I said services weren't a major part of my work, so while I was interested to see this happen it wasn't a priority to push on it. Still I was glad to see Pulumi took the ideas from that PR and added support for dotnet towards the end of 2019. It was around this time that my team were starting to work more with setting up Kubernetes services.

Using Kubernetes brought back a lot of the feelings I'd had at Arm using terraform, except templated YAML is even worse than HCL. We also had a lot of issues with getting sensible observable deployments set up, and even more trouble with trying to allow developers to just deploy their own instances for testing. I knew a lot of these problems _could_ be solved with Pulumi. It would easily allow code sharing between our different applications to set all the common per-team and G-Research required annotations and fixes we needed. It had first class concepts for developers starting up their own stacks and solving the name collision problems that normally entails. It already understood that pipeline deployments need to wait for the kubectl apply to actually be successful (It was sad the amount of early day deployments that were just using kubectl apply and applied gibberish but marked the pipeline as successful). Finally if we convinced the company to get the Pulumi service it would give immediate observability into everyone's deployments, not just the major ones from pipelines but also every developers use of the cluster for their development work.

I had started prototyping moving our Kubernetes YAML into Pulumi via TypeScript during 2019 before the dotnet support landed. Getting and using non-standard software into G-Research was an effort and there was no way I'd be able to use the public Pulumi service from our company network. So these early experiments ran into multiple issues. Firstly was getting an update version of Pulumi and all the associated provider binaries to developers workstations, we couldn't just hit a public http link to download them and Pulumi wasn't packaged into standard packaging repos like apt or aports. Secondly was the lack of the Pulumi service thus us falling back to using the file based state management, which had [bugs with windows network paths](https://github.com/pulumi/pulumi/issues/2693) and [didn't support stacks and projects](https://github.com/pulumi/pulumi/issues/2522) to the same level as the web service. Given the issues and that I was resourcing this in my own time, progress was slow to non-existent but I felt like if we could write all our Kubernetes config in F# using Pulumi instead of YAML it would be a huge win so slowly trudged on. In early 2021 I started looking at seeing if I could get Pulumi [into aports](https://github.com/pulumi/pulumi/pull/6016) which would have given us an easy way to ingress it to the company. Alas come April I had another baby born and that put a massive dampener on getting any programming done in my spare time.

## Pulumi

It was not long after that, driven by some introductions that had been started months before, I interviewed for a position at Pulumi. Clearly that went well and we're finally back to the present day. Here I am joining Pulumi as my engineering manager at Arm suspected all those years ago.

I'm obviously a huge fan of their (our?) product, and having spoken the team both during my contributions over the years and during the interviews I'm looking forward to working with them, they seem to be an incredibly smart and friendly group.

## Why Pulumi?

This applies to all writing on this blog, but just to explicitly caveat that the following is my personal writing.

I think my thoughts on this are going to echo ideas that Joe and others in the company have espoused before. But this is my take on things.

Cloud is important and only going to get more important. If your familiar with [Wardley maps](https://medium.com/wardleymaps) then you will have already heard of the idea of "commoditisation of compute", the idea that aspects of our compute environment become more and more standard and utility like. We've seen the movement of raw compute go from people managing their own racks of physical hardware to now just sending a http request to AWS/Azure and getting back a virtual machine to use. We've seen raw data go from people managing physical racks of hard drives and the logistics of backups being shipped to cold storage to now again just asking for an S3/Blob storage item. These changes are going to continue, and more and more parts of our systems that used to be custom built, or brought products are going to become just a utility that you pay as you use from the cloud.

As these effects continue then tooling to build in this new environment is going to become more important, as is tooling to manage these environments. I hope with Pulumi we can help with wrangling some of this essential complexity to make it easier for developers to build new amazing things using the power that utility like services can provide.

Given how Pulumi works one of the most interesting things I think we can accomplish is improving the ability to share cloud solutions. Pulumi is exposing cloud infrastructure via standard programming languages, this gets us two major advantages. Firstly we can compose cloud solutions in the same way we compose programs, just via function calls and objects. Secondly we get all the ways of sharing code that normal programs benefit from, whether that's just pulling the source from GitHub or getting a package from npm/pypi/nuget/etc. This has the potential for some great network effects where people can build and improve on each other, and as more people build and share solutions with Pulumi it gets easier to build even more. In the same way how git is _the_ way to do version control because of the network effects of GitHub, Pulumi could be _the_ way to do IaC (Hopefully with a better story on UX then git).

## Why me?

So that's all good, but why do I want to work there rather than just using it.

My last two jobs have basically been around providing programs that allow users to express a graph of work to be done. Turns out this is the same core problem that Pulumi is solving, the graph of work in this case being the creation of infrastructure resources. I _like_ these problems, and I feel like I've learnt a lot about how to solve these sorts of problems in my last two positions and am looking forward to applying that Pulumi and learning more.

Pulumi also has some other interesting problems like how to best interface with the various cloud and other infrastructure providers; how to parse other IaC format to generate good Pulumi code; how to define and generate multi-language packages; and in general how to expose everything with great UX.

A lot of these are the sort of compiler-like puzzles that I also enjoy. So I'll probably get to continue yammering insufferably about type systems and functional programming.

All in all I think it is going to be great fun!

See you in the cloud!