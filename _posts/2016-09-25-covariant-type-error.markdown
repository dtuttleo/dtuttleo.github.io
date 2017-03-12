---
layout: post
title:  "Understanding \"covariant type T occurs in contravariant position\" error"
date:   2016-09-25 15:04:20 -0500
categories: scala
---
A couple of weeks ago, I was studying functional programming. I was working on my own implementation of the Option type. The type hierarchy that I defined was something like:

```scala
trait MyOption[+T]
case class MySome[T](value: T) extends MyOption[T]
case object MyNone extends MyOption[Nothing]
```
I wanted `MyOption` to be covariant since that allowed me to assign `MyNone` object to any variable that has the type `MyOption`; because `Nothing` is a subtype of everything, I can do something like `val x : MyOption[String] = MyNone` for example. When I tried to add the method `getOrElse` to `MyOption` trait, I got the error:

```scala
trait MyOption[+T] {
  def getOrElse(default: T): T 
  // error: covariant type T occurs in contravariant position in type T of value default
}
```
<!--description-->

After doing some research, I found that the Scala compiler performs some validations when it comes to variance in type parameters. These rules are defined in section 4.5 of the Scala Language Specification. The rule that I was breaking with my definition of `getOrElse` was:

> The variance position of a method parameter is the opposite of the variance position of the enclosing parameter clause.

That's why the compiler was complaining about my method definition. So, in order to fix the definition of `getOrElse` method all I had to do was to adjust the argument of the method so it could be in a contravariant position:

```scala
trait MyOption[+T] {
  def getOrElse[S >: T](default: S): S
}
```

This solved my problem, and I could continue with the implementation of `MyOption` type.

While I was reading about type variance in Scala, I remembered that Java has a design problem due to the fact that arrays are covariant. Let me show you what I mean in the following example:

```java
String[] s = {"one", "two"};
Object[] o = s;
o[0] = 3; // we get a runtime exception: java.lang.ArrayStoreException: java.lang.Integer
```
This piece of code compiles. But when we execute this program, we get a `java.lang.ArrayStoreException` because we cannot assign an `Integer` value to a `String` array. What we can learn from this example is that arrays should not be defined as covariant; that's something Scala designers knew. If you go to the Scala API documentation, you will find that the `Array` class is defined as invariant.

Then I started to wonder: what was the need to define such a rule that in this case forced me to change the definition of the method `getOrElse`? My classes were immutable, so I didn't have the risk to get runtime exceptions like in the Java example.

After looking for some answers, I found that one of the reasons Scala doesn't let me define a method that takes a covariant parameter in a covariant class is that it violates the [Liskov substitution principle](https://en.wikipedia.org/wiki/Liskov_substitution_principle){:target="_blank"}. In a few words, this principle says that if I have a type `S` which is a subtype of `T`, then I should be able to replace each occurrence of `T` with an instance of `S` without altering the behavior of the program. Let's see how this is violated:

```scala
trait MyOption[+T] {
  def getOrElse(default: T): T  // suppose this line compiles
}

trait C
case class B extends C
case class A extends C

val b = B
val op1: MyOption[C] = ...
op1.getOrElse(b) 

val op2: MyOption[A] = ...
op2.getOrElse(b) // this doesn't compile because B is not a subtype of A!!
```

This example defines three classes: A, B, and C: where A and B are subclasses of C. Now, let's assume that we can define the method `getOrElse` with a covariant parameter. According to the LSP, we could substitute the value op1 which is of type `MyOption[C]` with the subtype `MyOption[A]` without affecting the behavior of the program. But, as we see in the code, the program no longer compiles because B is not a subtype of A. Hence, `MyOption[A]` can't be a subtype of `MyOption[C]`.

Working with type parameters can be tricky, specially with variance. I recommend you to read section 4.5 of the Scala specification if you want to know more about how variance annotations work in this language.

Have you struggled with this kind of error?
