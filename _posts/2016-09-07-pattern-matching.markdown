---
layout: post
title:  "Pattern matching"
date:   2016-09-07 13:41:53 -0500
categories: scala
---
This post is part of a series of posts where I give more details about a particular feature of Scala that I like. Now, the turn is for pattern matching. You can see the list of features that I like [here]({% post_url 2016-08-14-things-i-like-about-scala %}).

Coming from Java or any C-like language, the first thing you will think of when you see a match expression in Scala is that pattern matching is like switch statements in any of these languages. That's probably an unfair comparison since pattern matching is more powerful and has a wider range of use cases. In this post, I'm going to show you the usages of this feature that I find the most interesting.

<!--description-->

## Basic example

Let's start with a simple example that behaves similarly to switch statements in other languages:

```scala
def bar(x: Int) = {
  x match {
    case 1 => "that was the number one"
    case 2 => "that was the number two"
    case other => s"that was the number $other"
  }
}
bar(1) // returns: that was the number one 
bar(2) // returns: that was the number two
bar(5) // returns: that was the number 5
```
Here, we define a function `bar` that takes an int as an argument and matches it against three patterns. If x is equal to 1 the function returns "that was the number one", and if it's equal to 2 then it returns "that was the number two". The last case is a variable definition. Here, the variable `other` will hold the value of x if none of the previous patterns matched. So, in a few words, this works as the default case.

This looks a lot like a switch statement, but there are some differences that we should be aware of. First, notice the lack of the break keyword. In Scala there is no need for such a thing since only zero or one patterns can match, and the first match always wins. Another difference is that if none of the patterns match, a `scala.MatchError` is thrown. Let's see what happens if we remove the last case expression in the previous example:

```scala
def bar(x: Int) = {
  x match {
    case 1 => "that was the number one"
    case 2 => "that was the number two"
  }
}
bar(1) // returns: that was the number one 
bar(2) // returns: that was the number two
bar(5) // boom!! scala.MatchError: 5 (of class java.lang.Integer)
```

And finally, unlike switch statements, matches are expressions.

## Types

We saw in the previous example that we can define some values and variables in case clauses. We can provide the variable's type as well:

```scala
for (el <- Seq("text", 2, 33.7)){
  val result = el match {
    case s: String => s"we found a string!: $s"
    case i: Int => s"if we add five we get: ${i + 5}"
    case _ => s"we didn't expect: $el"
  }
  println(result)
}
//the following gets printed:
//we found a string!: text
//if we add five we get: 7
//we didn't expect: 33.7
```
This is how we specify the types that we would like to work with. In this case, we only care about ints and strings, and since we don't want to get a `MatchError` if we get a value of a different type, we have defined a default case too.

From what we have seen so far, we can conclude: if the term that we specify in the case clause starts with a lowercase letter, the compiler interprets it as the name of a variable that will hold the matched value. And if it starts with uppercase it is assumed to be the name of a type.

## Tuples

Tuples are like containers that help us to store some values that we can pass around. With pattern matching, we can access the elements of a tuple in a simple way. Let's see how we can do this:

```scala
def playAlbum(t: (String, String, Int)) = {
  t match {
    case ("Blake", _, _) => println("I'm not in the mood for Country music right now")
    case (band, album, year) => println(s"playing $album ($year) by: $band")
  }
}
```
We are not limited to case clauses when we want to extract the elements of a tuple. Imagine that we have a function that returns a tuple with the information of a person. We can access the person's data in the following way:

```scala
// findSmartPerson is a function that returns a
// tuple with the name, age, and degree of a person
val (name, age, _) = findSmartPerson()

performCalculation(name, age)
```

We can see here how easy it is to get the elements out of a tuple. Since we are only interested in the name and age of the person, we use the placeholder `_` for the degree value to indicate that we don't care about it.

## Sequences

Matching on sequences is one of the use cases that I really like. With pattern matching we can split a list in its head and tail:

```scala
Seq(1, 2, 3, 4, 5) match {
  case h +: t => println(s"head: $h tail: $t")
}
// head: 1 tail: List(2, 3, 4, 5)
```

And even better, we can get the second element of the list:

```scala
Seq(1, 2, 3, 4, 5) match {
  case e1 +: e2 +: t => println(s"second element: $e2")
}
// second element: 2
```

Pattern matching on lists is a common idiom while writing recursive functions. Let's write a function that sums all the elements of a list of integers:

```scala
def sum(xs: Seq[Int]): Int = {
  xs match {
    case Nil => 0 // return 0 if the list is empty
    case h +: tail => h + sum(tail)
  }
}

sum(Seq(4,3,2,5)) // it returns 14
```

## Regular expressions

Pattern matching allows us to extract the groups in a regular expression in a concise way:

```scala
val extractPropertyRegEx = "(.*)=(.*)".r

val extractPropertyRegEx(key, value) = "myKey=myValue"
//key: String = myKey
//value: String = myValue
```
There is one caveat: if the regular expression doesn't match we get a `scala.MatchError`.

We can also use multiple regular expressions to match against a String using a match expression:

```scala
val workEmail    = "sender=.*@mywork.com subject=(.*)".r
val twitterEmail = "sender=.*@twitter.com subject=(.*)".r

def myNotify(email: String) = {
  email match {
    case workEmail(subject) => println(s"Email from work: $subject")
    case twitterEmail(subject) => println(s"Something happened in twitter: $subject")
    case _ => println("this can wait")
  }
}

myNotify("sender=boss@mywork.com subject=Critical bug in production")
//Email from work: Critical bug in production
myNotify("sender=no-reply@twitter.com subject=you lost 10 followers")
//Something happened in twitter: you lost 10 followers
```

These examples showed us how simple and concise it is to match a regular expression to a given String, and how we can get the values of the groups that we define.

## Case classes

Finally, I'm going to show you how you can use pattern matching against case classes. This is probably the most common use case for this feature; I use it a lot. But before we get into the example, let's see why case classes can be used in pattern matching.

When we define a case class, the compiler generates a companion object for the class and a bunch of methods. Among all of the generated methods, the method `unapply` is the one that allows case classes to be used in pattern matching. This method takes as an argument an instance of the class and returns a tuple that contains the attributes of the instance. This is the opposite of the `apply` method that constructs an instance from the given attributes. So, when you define a case clause like the ones you'll see next, the method `unapply` is invoked.

```scala
case class Album(name: String, year: Int)
case class Band(name: String, albums: Seq[Album])

def play(band: Band) = {
  band match {
    case Band("AnyBand", Album(name, year) +: t) => 
      println(s"playing AnyBand's first album: $name released in $year")
    case Band(name, albums) => 
     println(s"playing random album: ${random(albums)} by: $name")
  }
}

def random(albums: Seq[Album]) : Album = {
  import scala.util.Random
  Random.shuffle(albums).head
}
```

There's a lot of things in this example, so let's start with the first pattern. In this case, we are only interested in the first album of the band "AnyBand" (it's the only one we like). To get the first album, we split the list into its head and tail, and since the head is of type `Album`, we can use pattern matching to have access to its attributes: name and year.

The second pattern acts as the default case. We randomly play an album of the other bands.

From this example, we can see how useful pattern matching is when we need to extract data from some entities. But, there are cases when we want a reference to the whole entity and not just its attributes; the following piece of code is going to show you how to achieve that.

```scala
trait Operation
case class Addition(x: Int, y: Int) extends Operation
case class Subtraction(x: Int, y: Int) extends Operation
case class Multiplication(x: Int, y: Int) extends Operation
case class Division(x: Int, y: Int) extends Operation {
  def isValid: Boolean = y != 0
}

def apply(op: Operation) = {
  op match {
    case Addition(x, y) => println(s"$x + $y = ${x + y}")
    case Subtraction(x, y) => println(s"$x - $y = ${x - y}")
    case Multiplication(x, y) => println(s"$x * $y = ${x * y}")
    case div @ Division(x, y) if div.isValid => println(s"$x / $y = ${x / y}")
    case _ => println("we cannot divide by zero")
  }
```
There is nothing new in Addition, Subtraction, and Multiplication cases. However, in the Division case, we have two new things: @ symbol and an `if` expression. When we want to extract some values and assign the whole object to a variable, we use @ after the variable and before the type part. In this example, we've created a variable named div with the purpose of calling the method `isValid`, which allows us to validate the divisor before we proceed with the division.

I almost forgot to talk about guards!! That if expression in the third case clause is what's called a guard. They allow us to place some conditions that must be true in order to get a match in the defined pattern. In this case, we want `div.isValid` to be true since we don't want to perform a division by zero.

## Conclusion

We've seen different usages of pattern matching. From regular expressions to case classes, this powerful tool gives us a concise way to extract data from different kinds of data structures. And, despite the fact that it can look similar to a switch statement in other languages, we have shown through the examples that with pattern matching we can do a lot more.

And remember, don't forget to define a default case when your case clauses are not exhaustive!!

