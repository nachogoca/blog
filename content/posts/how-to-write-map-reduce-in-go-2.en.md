---
title: "How to Write a MapReduce in Go Part 2"
date: 2020-09-17T15:42:40-05:00
draft: true
toc: false
images:
tags:
  - golang
  - engineering
  - distributed systems
---


## Implementation

In the Part 1, we learned how would a client use MapReduce through a simple and powerful interface that enables automatic
parallelization and distribution. It's almost like magic, right? Well, we'll now learn how to implement that magic.

We'll need a way to distribute the work, ensure that maps and reduces know how to communicate the intermediate key/value
pairs between each other, and some kind of mechanism to ensure the re-execution of work in case of a failure, 
to ensure a **fault tolerant** system.

{{< figure src="/img/02-how-to-write-map-reduce/diagram.png" title="MapReduce diagram" >}} 




