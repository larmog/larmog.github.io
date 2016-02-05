+++
date = "2016-02-04T20:56:35+01:00"
draft = false
title = "Thank you Java, but it's time to Go"
author = "Lars Mogren"
tags = [ "Development", "Java", "Golang" ]
categories = [ "Development" ]
+++

I've been a dedicated Java developer since 1998 and it's been a great tool.
But recently there's a couple of things that's been nagging me. I always been
a slow typer (never learnt how use a keyboard correctly). When programming
that's not always a bad thing.
But Java is very verbose and all those `private`, `pubic`, `;`, `getXXX`,
`setXXX` kind of gets in the way.
<!--more-->
```bash
package main

import "fmt"

type Vertex struct {
	X int
	Y int
}

func main() {
	v := Vertex{1, 2}
	v.X = 4
	fmt.Println(v.X)
}
```

Another thing that turns out to be complicated is coding standards. It's all
well when your on a greenfield project and the team can decide their own rules.
But have you ever worked with legacy code that's been abused for several years
buy developers who rather wish they were on that super cool project? I know you
can't blame `Java` for this but it's still a problem. Coding standards are like
fashion and every developer has an opinion on where to put the `{}` and don't
mention line endings when your in a mixed Windows and \*nix environment.

```shell
# Gofmt is a tool that automatically formats Go source code
$go fmt
```
When Java was released in 1995 the promise _"write once, run anywhere"_ sounded
very tempting but has shown not to be altogether true. There have been, and
will always be, bugs and small differences in the JVM which means that your
program will always be dependent on the version of the JVM running your byte
code. That means you have at least two artifacts to consider and
one of them is out of your control.

```shell
# Static compiled go binary that can be used in a Docker image.
# The binary is compiled using golang Docker image.
$CGO_ENABLED=0 GOOS=linux go build -ldflags "-s" -a -installsuffix cgo -o main
```

If your like me feeling young and haven't aged a day since 1995, the truth is
time goes by. Even if you do your best to keep young, it's hard to change. Java
has done a good job: HotSpot, Regular expressions, NIO, Generics, Annotations,
primitive wrapper classes with autoboxing, Enumerations, Varargs and Lambda
Expressions. Just take a look at the
[Java version history](https://en.wikipedia.org/wiki/Java_version_history). But
all this adds up and gains weight. Java has become hard to learn and understand
and there are many pitfalls and compromises. Some times the right decision is
to start all over and make something new based on what you've learnt.

```shell
# OSX
$open http://tour.golang.org/welcome/1
```

I'm keeping Java in my toolbox but I've started using
[The Go Programming Language](https://golang.org/) as my new tool and
it's great fun.
