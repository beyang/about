---
title: "GopherCon 2019 - Tracking inter-process dependencies using static analysis"
description: "Using a service-oriented architecture instead of a monolithic architecture adds flexibility but also increases complexity. How will we keep track of which parts of our system depend on each other? To help solve this problem, I've been working on a project to analyze Go code (using Go!) to detect inter-process dependencies in our system. In this talk, I'll give a brief introduction to the static analysis libraries I used and share a few tips about how to use them effectively. I'll discuss obstacles I encountered along the way and my attempts to overcome them."
author: Beyang Liu for the GopherCon 2019 Liveblog
publishDate: 2019-07-26T00:00-14:55
tags: [
  gophercon
]
slug: gophercon-2019-tracking-inter-process-dependencies-using-static-analysis
heroImage: https://about.sourcegraph.com/gophercon2019.png
published: true
---

Presenter: Mike Seplowitz

Liveblogger: [Beyang Liu](https://twitter.com/beyang)

## Overview

Using a service-oriented architecture instead of a monolithic architecture adds flexibility but also
increases complexity. How will we keep track of which parts of our system depend on each other? To
help solve this problem, I've been working on a project to analyze Go code (using Go!) to detect
inter-process dependencies in our system. In this talk, I'll give a brief introduction to the static
analysis libraries I used and share a few tips about how to use them effectively. I'll discuss
obstacles I encountered along the way and my attempts to overcome them.

---

Mike works at Bloomberg on the Deployment Solutions team. Here's a link to his slides: 

Outline:

* Introduction
* Important questions
* Exploration
* Primer
* Attempt 1
* Attempt 2
* Takeaways

When Mike joined the Deployment Solutions team 2 years ago, they made the decision to start using Go
and also move to a microservices architecture. Their microservice only had a single endpoint, so
perhaps it was more a "nanoservice".

He embarked a project to accomplish the following:

1. Determine the dependencies of a single service
   * Highlight effects of a code change

2. See inter-service dependencies across the entire system
   * Visualize system graph
   * Detect cycles
   * Overlay traffic data on static graph

Here was his initial plan:

1. Find “interesting” function calls (that would indicate a dependency on an external service)
2. Call “destinations” = dependencies of the service
3. Service dependencies = edges of the system graph

### Important questions

What are some example "interesting" calls?

* Network: `net.Dial, net/http.Get, (*net/http.Client).Get`
* Higher level:
  ```
  database/sql.OpenDB
  google.golang.org/grpc.Dial
  github.com/go-redis/redis.NewClient
  github.com/streadway/amqp.Dial (RabbitMQ)
  github.com/Shopify/sarama.NewAsyncProducer (Kafka)
  ```
* Processes: `(*os/exec.Cmd).Run, os/exec.FindProcess, os.Pipe`
* Filesystem: `os.Create, os.Open, os.Mkdir`

Who (which services) are we calling?

```
http.Get("https:#/myservice.example.com/path/to/resource")
http.Get(myserviceURL)
url #: os.Getenv("MYSERVICE_URL")
...
http.Get(url)
http.Get(app.Config.MyService.URL)
```

So which call do we "mark" (as indicating an external process dependency)?

You can have a direct call (super straightforward):

![image](https://user-images.githubusercontent.com/1646931/61969125-9f0a9b00-af8e-11e9-9d0f-8decc230cb60.png)

You can have a specific helper (i.e., 1 level removed):

![image](https://user-images.githubusercontent.com/1646931/61969150-ad58b700-af8e-11e9-9d95-641fe384525b.png)

You can also have generic helpers:

![image](https://user-images.githubusercontent.com/1646931/61969191-bba6d300-af8e-11e9-95b2-f30a9d7dca95.png)

So you are looking at the call chain:

![image](https://user-images.githubusercontent.com/1646931/61969382-45ef3700-af8f-11e9-9114-6096172f43bf.png)

The question is where in the call chain do you place the marker.

You can look at where the last place where the service URL appears. And you mark that with a comment:

![image](https://user-images.githubusercontent.com/1646931/61969401-4e477200-af8f-11e9-8c32-a570ec805778.png)

To take a more complex example, you want to look at the call graph and determine where to place the
markers. If you mark `func A` in the example below, you've covered all the ingoing paths to `func A`,
and you can disregard all outgoing paths, as well. Then you need to mark other functions that haven't yet been "covered":

![marked paths](https://user-images.githubusercontent.com/1646931/61969413-52738f80-af8f-11e9-88ce-0290d186fe0f.png)


Source diving... He did a quick search of GoDoc for "call graph" and found the following packages:

```
golang.org/x/tools/go/callgraph
golang.org/x/tools/go/callgraph/cha
golang.org/x/tools/go/callgraph/rta
golang.org/x/tools/go/callgraph/static
```

Here are the key functions in each package, all of which take a `*ssa.Program` pointer:

![image](https://user-images.githubusercontent.com/1646931/61969596-c877f680-af8f-11e9-8a9b-37d40035bc0d.png)


A lot of packages in `golang.org/x/tools` that are prefixed with `analysis/`:

![image](https://user-images.githubusercontent.com/1646931/61969626-d7f73f80-af8f-11e9-88cc-506c8105c72e.png)

"The analysis package defines the interface between a modular static analysis and an analysis driver
program."

A "Pass" is running an Analyzer on a single package. There were a few such reusable passes:

analysis/passes/inspect -> `*inspector.Inspector`
analysis/passes/ctrlflow -> `*cfg.CFG (basically)`
analysis/passes/buildssa -> `*ssa.Package, []*ssa.Function`

The Plan: For each package Pass,
1. Grab the SSA result from the buildssa Pass.
2. Build the call graph using rta.Analyze.
3. Traverse the call graph, marking paths that lead to “interesting” calls.
4. Report unmarked paths as linter errors. 
5. Report destination markers as dependency data.



### Primer

This section is a quick primer on static analysis packages in the Go compiler and standard library.

Relevant packages:

```
go/
  token
  ast
  parser
golang.org/x/tools/go/
  packages
  ssa
  callgraph
  analysis
```

Here's the rough control flow in the Go compiler:

![image](https://user-images.githubusercontent.com/1646931/61969635-dd548a00-af8f-11e9-985b-c7b0f36d8982.png)

Here's the data structure that represents locations in code:

![image](https://user-images.githubusercontent.com/1646931/61969639-de85b700-af8f-11e9-9f7e-033c691b317c.png)

ASTs and parsing:

![image](https://user-images.githubusercontent.com/1646931/61969738-17be2700-af90-11e9-9aa2-b82d5588423e.png)

Here's an example AST for a simple code snippet:

![image](https://user-images.githubusercontent.com/1646931/61969739-1987ea80-af90-11e9-99d0-f733a4ee4c0c.png)

Here are the key structures in the `go/ast` package:

![image](https://user-images.githubusercontent.com/1646931/61969740-1ab91780-af90-11e9-8d66-af5c27434910.png)

`go/parser` parses Go expressions and returns an AST.

![image](https://user-images.githubusercontent.com/1646931/61969743-1bea4480-af90-11e9-82c9-e532e490a1d2.png)

`golang.org/x/tools/go/packages` loads packages (this replaces the old `golang/x/tools/go/loader` package)

![image](https://user-images.githubusercontent.com/1646931/61969747-1d1b7180-af90-11e9-8315-ad554b3b6a32.png)

SSA is the intermediate code representation:

![ssa](https://user-images.githubusercontent.com/1646931/61969750-1e4c9e80-af90-11e9-9e08-6ddce468fc20.png)

The `callgraph` package constructs call graph data structures from SSA structures.

![callgraph](https://user-images.githubusercontent.com/1646931/61969758-20aef880-af90-11e9-920e-9dea619ab2dc.png)

The analysis package defines the interface between a modular static analysis and an analysis driver
program:

![analysis](https://user-images.githubusercontent.com/1646931/61969762-2278bc00-af90-11e9-828e-403a094829fa.png)




