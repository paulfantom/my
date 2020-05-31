---
authors:
- name: "Bartek Płotka"
date: 2020-05-30
linktitle: Require a Go Tool for Your Project? That's a Bingo! 
type:
- post 
- posts
title: Require a Go Tool for Your Project? That's a Bingo! 
weight: 1
categories:
- go
featuredImage: "/images/blog/bingo/demo.gif"
---
_See bigger version of the above demo as .gif [here](https://raw.githubusercontent.com/bwplotka/bingo/master/examples/bingo-demo.gif)_

In this blog post I would like to introduce [`bingo`](https://github.com/bwplotka/bingo). Simple and efficient CLI I wrote for managing versioning of Go binaries that are required for your project development.

**TL;DR: `bingo`is built on top of native [Go Modules dependency management](https://github.com/golang/go/wiki/Modules) and in my option it
 is a must-have for all repositories requiring dev tools written in Go. It's also already integrated into production grade projects like [Thanos](https://thanos.io). Check it out and contribute! 🤗**

# Automate all the things!

![This is NOT  what you want to do in your project...](/images/blog/bingo/automation.gif)

> Automation is **amazing**. 

Especially nowadays, all open or closed projects use at least bunch of tools to automate some tasks. For example in Go community
you can see following popular tools enabling automation:

* Running tests:  `go test`, `go test -race`
* Benchmarks: `go bench`, [`benchcmp`](https://pkg.go.dev/golang.org/x/tools/cmd/benchcmp?tab=doc), [`benchstat`](https://godoc.org/golang.org/x/perf/cmd/benchstat), [`funcbench`](https://github.com/prometheus/test-infra/tree/master/funcbench)
* Auto formatters: `go fmt`, [`goimports`](https://godoc.org/golang.org/x/tools/cmd/goimports)
* Static analysis: `go vet`, [`lint`](https://github.com/golang/lint), [`golangci-lint`](https://github.com/golangci/golangci-lint), [`faillint`](https://github.com/fatih/faillint)
* Documentation Generators: [`embedmd`](https://github.com/campoy/embedmd), [docs auto generators](https://github.com/thanos-io/thanos/tree/master/scripts/cfggen)
* Build tools: [`goreleaser`](https://github.com/goreleaser/goreleaser), [`promu`](https://github.com/prometheus/promu)
* Other projects (!) you want to run integrations tests against.
* ...and millions of other tools: [website builder](https://gohugo.io/), [link check](https://github.com/raviqqe/liche), [copyright checker](https://github.com/thanos-io/thanos/tree/master/scripts/copyright), protobuf + gRPC generation, jsonnet generators and bundlers, [YAML tools](https://github.com/brancz/gojsontoyaml), etc... 

I could list even more, and I am so amazed with the amount of things we have **for free**! They help maintaining very good quality of the
project, and saves an enormous amount of time. 

# Problem that [`bingo`](https://github.com/bwplotka/bingo) solves

So what projects do? They obviously tend to use gazillion of those tools, reusing an amazing ideas and solid code that someone else in open source wrote
and maintains every day with love.

That's great, but how many times you have cloned the project, want to contribute run `make` and see things like:

```shell
$ make 
zsh: command not found: clang
zsh: command not found: goimports
zsh: command not found: goreleaser
zsh: command not found: hugo
```

And that's even a nice error! Usually you get confusing error that does not make any sense like: 

```shell 
$ make 
dafg;lkghd;lsfjgpdsjgp 
zsh: exit 1
```

...because you have the required tool installed but **WRONG version** of it!

Now, if you want to be a nice project to work with, you give a hand and either print meaningful errors and tells how
to install a dependant tool and what version of it, in case of users not having a tool or have wrong version installed. This is definitely 
a step in a good direction, but in my opinion **we can do much better.**

## Installing Tools for User
 
In all projects I maintain, we tried hard to be even more explicit and helpful by ensuring our scripts will install all dependencies
needed manually. We were largely laverage good, old school [`Makefile`](https://www.gnu.org/software/make/manual/make.html) for tasks likes this.
It is as easy as:

```makefile
DEPENDENCY_TOOL = $(GOBIN)/tool 

task: $(DEPENDENCY_TOOL)
	@do task using $(DEPENDENCY_TOOL)

# Will be built only if $(GOBIN)/tool file is missing.
$(DEPENDENCY_TOOL):
	@install tool
```     

## Always Pin the Version

The above example works great, but obviously will not always work if you require a **certain** version of the tool.
In fact, if you would remember only one thing from this blog post, remember this:

> You really want to pin version of ALL the tools your project depends on!

Why? Well, quick answer is that versioning is hard and things are constantly moving. Tools are being improved and changed every day.

Trust me, the last things you want to do is to investigate why CI constantly fails or have spam of bugs and issues, about someone, somewhere
being unable to build your project because of obsolete, or too new tool that was required.

We can easily extend our above `Makefile` example to improve resiliency and development experience by pinning tools' versions
and maintaining immutable binary files.

Immutable binaries safe guards us from users installing manually wrong version of tool. Without immutability, we would have to 
checksum and verify all builds to ensure the correct version which is doable, just bit more complex to maintain. (BTW, how nice it would
be to have CLI `man` equivalent for printing `version`! (: ). We can achieve this as follows:   

```makefile
DEPENDENCY_TOOL_VERSION = v1.124.3
DEPENDENCY_TOOL        = $(GOBIN)/tool-$(DEPENDENCY_TOOL_VERSION)

task: $(DEPENDENCY_TOOL)
	@do task using $(DEPENDENCY_TOOL) 

# Will be built only if $(GOBIN)/tool-v1.124.3 file is missing.
$(DEPENDENCY_TOOL):
	@install tool@$(DEPENDENCY_TOOL_VERSION)
	@cp tool $(DEPENDENCY_TOOL)
```     

## Cross-Platform Installation of the Correct Version of Tools

Pinning version was easy, right? Now fun stuff, how can we reliably install those tools in required version? 
Let's look on possible solutions:

### Commit built, dependant binaries straight to your VCS (e.g git) 😱

This is usually **a bad idea**. While it might work for you, other users would probably want to run your software on different
machines with different architectures and different operating systems etc. And even if you would host all permutations of
each required binary, each can "weight" from 10-100MB which is most likely too much for common VCSs (e.g `git`). This
will make it extremely slow, if not impossible to clone your project. Plus there is no merge conflict resolution that
works well for binary files.

Hope I made this clear: Please don't do this. (:  

### Use Package Manager

Now this is what's usually recommended, and it looks innocent.

![Use Package Manager they said...](/images/blog/bingo/pkg.jpg)

The idea would be to use the package manager available on user's OS e.g. `apt` `yum` etc 

I will stop here, because in practice this is literally impossible. Let's look on following reasons:

* While there are standard package managers for major operating systems, some people might not use it (you can disable it).
On top of there are 99% chances that the amazing tool you need is not packaged there or the version available is extremely old.
* There are other custom pkg managers like `snap`, `brew`, `npm`, `pacman` `NuGet` `pip` but not everybody has them 
preinstalled, so it's chicken-egg problem.
* Most of the tools work only with tools written in certain programming language that is NOT Go (:

### Curl Released Binaries

You can probably get quite far with just automatic `curl` of pre-built binary against certain operating system, released on
GitHub by authors. Unfortunately, again, in practice, not many projects maintain that. Especially, for a small tool it's an overkill.  

### Solution: Pin Certain version of Source Code Instead! 

Have you spotted a certain pattern among all the tools I mentioned in [Automate all the things!](#automate-all-the-things) section?

**Yes! They all are written in [Go](https://golang.org/).** The Go community really believes in automation, so the amount of tools that was 
produced is **impressing**. And this is NOT only useful for Go projects, but actually any project you can think of. Tools written
in Go tends to be extremely reliable and very easy to maintain and fix. You should try writing your own tooling in Go as well, check [this 
amazing video](https://www.youtube.com/watch?v=oxc8B2fjDvY) from our friend [Fatih](https://twitter.com/fatih) ❤️.

So, if we assume all our tools will be written in Go, does this simplify our life? The answer is: Yes!

#### Rant About Go Modules
 
![Calling Go Modules for help!](/images/blog/bingo/gomod.jpg)

Few years ago Go Team released their sophisticated answer for dependency problem in Go Community: [Go Modules](https://github.com/golang/go/wiki/Modules).
It (usually) works and it's quite amazing for many reasons: 

* It is decentralised. Anyone can publish their code, anywhere.
* Supports using multiple different (major) versions of the same module in the same project. 
* It is ~secure. HTTPS and SSH is the default and `go.sum` exists to verify checksums.
* Supports caching proxies / mirroring.
* It is an official tool, which means finally a single standard.

But let me say this loud and clear: **It's also far from being perfect.** Consider following reasons:

* It assumes everyone uses **semantic versioning** properly. 

> Halo Go Team, can I have your attention for a second? 🤗 

**99.(9)% of Go projects does NOT use semantic versioning properly and NEVER will be!** 

It's because maintainers either have no time, don't care or they simply semantically version their APIs or part of packages only. See [detailed 
discussion on this matter in Prometheus (30k stars' project) mailing list.](https://groups.google.com/forum/#!searchin/prometheus-developers/Go$20Modules%7Csort:date/prometheus-developers/F1Vp0rLk3TQ/TyF2WxlkBgAJ)

This actually leads to most of the problems users experience. You can't escape from a huge amount of `replace` hacks.
Plus, instead of making it easier for such use cases, Go blames others. ):

This is also why `bingo` was needed and why I built it.

* We build and import packages, but version modules. If you add an overhead of maintaining modules and releasing it (see the above issue),
it's clear that maintaining multiple modules is not a good answer. That's why [Duco's](Helcaraxan) amazing [`modularise`](https://github.com/modularise/modularise)
project was born. It would be better if we have good out of box solution instead.

* Managing Major versions is painful. Rewriting path for everything to include this `v2` is very nasty and tedious, but the only
way of doing major release and still support multi-version of the same dependency. Hope it will get redesigned some day.

* Vendoring is allowed and sometimes even (😱) recommended. This is opinionated and controversial, but **WHY THE HECK you use `git` if you want
version things in sub directories, are we in 2005 again?** If for cache purposes, then use cache instead. (: Probably a
good topic for next blog post. I would love to see better and easier ways to use proxies instead and this feature less used in the future.
For now, you can read some details [here](https://groups.google.com/d/msg/prometheus-developers/WLQQd_uNRlw/zKyTD9C4BQAJ) 

> I was complaining a bit, but keep in mind that it's not the easiest task to build solution that meet all the requirement
from tiny projects to madly huge monorepos in both close and open source.

Overall Go Team is doing amazing job and Go Modules is the best what we can have now in the Go Community.
It's also improving every day and it's fairly easy to extend as you will read later on. 

#### World Without Bingo

Adapting Go Modules to pin buildable dependencies as dev tools for your project is actually a common thing nowadays. There are common
patterns that were created recently:

* tools file
* tools file with separate, single Go Module for tools.  

TBD

## Introducing Bingo


## The end
