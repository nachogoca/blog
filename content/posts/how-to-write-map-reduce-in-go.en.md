---
title: "How to Write a MapReduce in Go Part 1"
date: 2020-08-06T22:07:05-05:00
draft: false
toc: false
images:
tags:
  - golang
  - engineering
  - distributed systems
---

In this post we'll learn about what is MapReduce and how we can use it.

## Problem

Let's say our job consists of counting words. Yes, in the middle of a pandemic, the world needs
someone to count words. This a daunting task since there are a lot of books. Therefore we decide
to write a program to do it for us. Obviously, in Go, our favorite programming language.

After a few minutes we reach a simple solution. Great! Our program is able to receive as arguments
the filepath(s) of the books to process. It will read the contents of each book sequentially, and 
store the word count of that book.

```go
package main

import (
	"fmt"
	"io/ioutil"
	"log"
	"os"
	"strings"
)

func main() {

	if len(os.Args) < 2 {
		log.Fatal("no arguments were given")
	}

	// filepaths of the books to be processed
	books := os.Args[1:]
	booksCount := map[string]int{}
	for _, book := range books {

		// load book content in memory
		// this could not be the best solution for big files
		b, err := ioutil.ReadFile(book)
		if err != nil {
			log.Fatalf("could not read file: %v", err)
		}

		// separate by words and count
		words := strings.Split(string(b), " ")
		booksCount[book] = len(words)
	}

	// print results
	total := 0
	for book, count := range booksCount {
		fmt.Printf("%d %s\n", count, book)
		total += count
	}
	fmt.Printf("%d total\n", total)
}

```
 Yeah, we might run into memory issues if one of the books is very
large, because we load all the content in memory at once, but we are satisfied with the results in the
list of books we were assigned to word-count.

These are the results:

```
> go run simple.go books/pg-being_ernest.txt books/pg-dorian_gray.txt books/pg-frankenstein.txt books/pg-grimm.txt books/pg-huckleberry_finn.txt books/pg-metamorphosis.txt books/pg-sherlock_holmes.txt books/pg-tom_sawyer.txt
25528 books/pg-metamorphosis.txt
108992 books/pg-sherlock_holmes.txt
77488 books/pg-tom_sawyer.txt
24130 books/pg-being_ernest.txt
83498 books/pg-dorian_gray.txt
78329 books/pg-frankenstein.txt
105203 books/pg-grimm.txt
120780 books/pg-huckleberry_finn.txt
623948 total

```

After seeing the results our client is very satisfied, so satisfied that he want us to count all the words
in the Gutenberg library! Although Go is fast, it's going to take us ages to count all those words if we do the calculations
sequentially. Maybe we could do all that counting faster if we do the work in parallel, using a programming model called MapReduce.

## MapReduce

Around 2004, Google had the same problem as us. Well, not exactly word counting but the need to process large amounts
of raw data, such as crawled documents, web request logs, etc., to compute various kinds of derived data, such an inverted
indices, very useful for their search capabilities. Most of the computations are simple and straightforward.
But, the input data is huge and the computations have to be distributed across hundreds or thousands of machines in order
to finish on time. And a lot of complexity comes from parallelizing the computation, distributing the data and handling failures.
So, in order to hide that complexity from the other engineers, Google researchers designed an abstraction called MapReduce.
You can read the original paper [here](https://static.googleusercontent.com/media/research.google.com/en//archive/mapreduce-osdi04.pdf).

The word *MapReduce* comes from the **map** and **reduce** primitives present in functional languages.
In MapReduce, a *map* operation is applied to each record in our input, that generates a set of intermediate key/value pairs.
Then a *reduce* operation is applied to all the values that share the same key, in order to combine the data appropriately.

Let's build an example of word-counting using MapReduce. Let's say we have a file *words.txt* with 5 words.

```
alpha alpha beta gamma
beta
```

The map process would read this file, and for every word it would generate an intermediate key/value pair:

```
alpha "1"
alpha "1"
beta "1"
gamma "1"
beta "1"
```

It's almost like the map process it's generating a **signal** for every word it's seeing.

Now the input for the reduce process would be each intermediate key and the list of all its values. In the case of *alpha*:

```
alpha ["1", "1"]
```

Reduce will count all occurrences of the values, for each key (word). And generate the desired output:

```
alpha "2"
beta "2"
gamma "1"
```

How would the map and reduce functions look like in Go code?

```go
// The KeyValue struct is used to represent the intermediate key/value
// pairs
type KeyValue struct {
	Key   string
	Value string
}

// The map function is called once for each file of input. The first
// argument is the name of the input file, and the second is the
// file's complete contents. You should ignore the input file name,
// and look only at the contents argument. The return value is a slice
// of key/value pairs.
//
func Map(filename string, contents string) []KeyValue {
	// function to detect word separators.
	ff := func(r rune) bool { return !unicode.IsLetter(r) }

	// split contents into an array of words.
	words := strings.FieldsFunc(contents, ff)

	kva := []KeyValue{}
	for _, w := range words {
		kv := KeyValue{w, "1"}
		kva = append(kva, kv)
	}
	return kva
}

// The reduce function is called once for each key generated by the
// map tasks, with a list of all the values created for that key by
// any map task.
//
func Reduce(key string, values []string) string {
	// return the number of occurrences of this word.
	return strconv.Itoa(len(values))
}
```

Notice how the keys from the intermediate key/value pairs are the input for the Reduce function.

A programmer using a MapReduce library only needs to define the Map and Reduce functions for the problem he's trying to
solve, and the library will take care of the internal intricacies that applying those functions in a distributed manner.
Here, we have solved an easy problem, but MapReduce can be used for many other non-trivial [applications](https://highlyscalable.wordpress.com/2012/02/01/mapreduce-patterns/).

We'll learn how to implement the MapReduce in the part 2.

