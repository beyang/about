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
published: false
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
















"Interesting" calls:

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



Who are we calling?

```
http.Get("https:#/myservice.example.com/path/to/resource")
http.Get(myserviceURL)
url #: os.Getenv("MYSERVICE_URL")
...
http.Get(url)
http.Get(app.Config.MyService.URL)
```
