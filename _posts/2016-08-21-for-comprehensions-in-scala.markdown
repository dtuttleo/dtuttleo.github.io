---
layout: post
title:  "For comprehensions in Scala"
date:   2016-08-21 20:28:53 -0500
categories: scala
---

One of the features that I like the most in Scala, is for comprehensions. This feature helps us to create concise and elegant solutions for a bunch of different problems. In a nutshell, for comprehensions are syntactic sugar for the following methods: `withFilter`, `foreach`, `map`, and `flatMap`.

For comprehensions come in different flavors, and depending on how we write them, they translate to a different thing. Through this post, I'm going to cover different ways you can write a for comprehension, and what kind of problems it will help you solve.

<!--description-->

## Anatomy of a for comprehension

Before we get into the different usages of a for comprehension, I would like to show what a for comprehension looks like:

```scala
for {
  val1 <- gen1  // generator
  x = 2         // value definition
  if booleanExpression // guard
 } yield val1 + x   // yield keyword is optional
```

Every for comprehension should start with what is called a generator. A generator in its simplest form is something that generates values. After the generator you can have other generators, value definitions, and guards. And finally, we have an optional `yield` keyword that affects the behavior of the for comprehension as we will see in the following examples.

Let's begin with the simplest form:

```scala
val languages = Seq("Python", "Java", "Clojure", "Scala")

for (l <- languages) println(l)

// this for comprehension translates to:
languages.foreach(println)
```

This form is like Java's enhanced for-loops statements, we traverse a collection with the sole purpose of performing some side effect action. It gets translated to a `foreach` method call on the collection, and in this example we just print each element of the `languages` sequence. I don't particularly use this form very often because I tend to create code without side effects.

What if we want to apply some operations on each element and return a new collection, rather than performing some side effects? That’s where the next form comes into play.

```scala
for {
  n <- (1 to 4)
} yield n * 2

// it's translated to:
(1 to 4).map(_ * 2)
```

When we add the `yield` keyword, for comprehensions return a new collection of values that are created after applying the expression that is specified after it. So, in this example we are multiplying each value of the sequence by 2, and getting Vector(2, 4, 6, 8) as a result.

Right now, you might be thinking that using map directly on the collection is less verbose than the for comprehension; that's true. But I'm going to show you how it shines when we have a more complex scenario.

Imagine we have a paragraph represented by a `Seq[Seq[String]]`, and we would like to get the decimal representation of each character in a paragraph. We can do something like:

```scala
val paragraph = Seq(Seq("sentence", "one"),
                    Seq("sentence", "two"),
                    Seq("sentence", "three"))
for {
  s <- paragraph
  w <- s
  c <- w
} yield c.toByte

// it gets translated to:
paragraph.flatMap(s => s.flatMap(w => w.map(c => c.toByte)))

// returns: List(115, 101, 110, 116, 101, 110, 99, 101, 111, 110, 101, 115, 101, 110, 116, 101, 
// 110, 99, 101, 116, 119, 111, 115, 101, 110, 116, 101, 110, 99, 101, 116, 104, 114, 101, 101)
```
We traverse each collection and finally call `toByte` method on each character. Here, we can see how the for syntax is more readable that the calls of flatMap and map on these collections. One important point to notice here, is that when we have multiple generators all but the last are translated to a flatMap call; the last one becomes a map call.

## Filtering values

We have seen so far that we use for comprehensions to iterate over each value of a collection. But what if we want to work only with a specific subset of items of the collections? Let's see how we can use guards to filter out some elements.

```scala
for {
 n <- (1 to 10)
 if n > 7
} yield s"$n is bigger than 7"

// this is translated to
(1 to 10).withFilter(_ > 7).map(n => s"$n is bigger than 7")

// both expressions return
// Vector("8 is bigger than 7", "9 is bigger than 7", "10 is bigger than 7")
```

The expression `s"$n is bigger than 7"` only gets evaluated for the values 7, 8, and 9 because of the guard `if n > 7`.

## For comprehensions are not limited to work with collections

Previous examples showed you how to use for comprehensions on collections. But this doesn't mean that we can only use them with Vectors, Lists, Strings, etc. In fact, we can use any datatype that supports the operations `filter`, `map`, `withFilter`, and `flatMap`.

```scala
for {
  p1 <- sys.props.get("p1") // returns Option[String]
  p2 <- sys.props.get("p2")
} yield myFunction(p1, p2)
// if we assume that myFunction return type is Int, 
// then this for expression return type is Option[Int]
```

In the example above, we use for comprehensions with Scala `Option` type. Here, we get some system properties, and if those are defined, we call `myFunction` with the value of the properties as its input parameters. This is a good example of how we can use for comprehensions to validate some optional input before we perform some operation.

We have seen how we can use for comprehensions to create readable solutions to different problems. We used this construct with collections and optional types, but as we saw before, we can use for comprehensions with types that implement the required methods.

What do you think about for comprehensions?

