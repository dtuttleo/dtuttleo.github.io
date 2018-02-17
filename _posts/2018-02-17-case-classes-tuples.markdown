---
layout: post
title:  "From Case Classes to Tuples, and Vice Versa"
date:   2018-02-17 10:26:36 -0500
categories: scala
---
When you work with Slick, and you want [to map a table to a case class](http://slick.lightbend.com/doc/3.2.1/schemas.html#mapped-tables){:target="_blank"}, you need to provide a couple of functions:

One that takes a tuple and returns an instance of your case class. And another that takes an instance of your case class and returns a tuple wrapped in an `Option`.

If the types of your case class are the same than the ones you have in your table definition, you only need to provide the `tupled` and `unapply` methods (as you can see in the Slick docs).

But, what if you need to perform some transformation because the types of your table definition don't match the ones in your case class?
<!--description-->
Well, you have to implement these two functions. Let's see an example:


```scala
case class User(id: Int, name: String, isAdmin: Boolean)

class Users(tag: Tag) extends Table[User](tag, "USERS") {
  def id = column[Int]("ID", O.PrimaryKey, O.AutoInc)

  def name = column[String]("NAME")

  // varchar column with two possible values:
  // 'Y': means yes
  // 'N': means no
  def isAdmin = column[String]("ISADMIN")

  def * = (id, name, isAdmin) <> (fromTuple, toTuple)
```
Here, we have a type mismatch in the `isAdmin` field. We have a `Boolean` in our case class and a `String` in our table definition. The way I've been implementing these functions is as follows:

```scala
def toTuple(u: User): Option[(Int, String, String)] = {
  Some(
    (u.id, u.name, if(u.isAdmin) "Y" else "N")
  )
}

def fromTuple(t:(Int, String, String)): User = {
  User(t._1, t._2, t._3 == "Y")
}
```
The implementation is pretty straightforward. But, what if our case class have around 10 fields? Well, this approach could become a tedious task, especially if we only need to transform one field.

So, after using the above pattern for some time, I found a more concise way to implement them:

```scala
def toTuple(u: User): Option[(Int, String, String)] = {
  User.unapply(u)
    .map(t => t.copy(_3 = if(u.isAdmin) "Y" else "N"))
}
  
// here we only need to deal with the attribute we need to change
def fromTuple(t:(Int, String, String)): User = {
  User.tupled(t.copy(_3 = t._3 == "Y"))
}

```
We don't see a big difference here because the `User` class only has three attributes. But, we will save a lot of keystrokes when the number of attributes starts to grow.
