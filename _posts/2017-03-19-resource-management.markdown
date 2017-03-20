---
layout: post
title:  "Resource management in Scala"
date:   2017-03-19 11:34:54 -0500
categories: scala
---
When I was adding a feature to an existing application, I found a method that looked like this:

```scala
def logIn(username: String, password: String) = {
  var conn: Option[Connection] = None

  val user:Try[UserData]  = for {
   c <- getConnection() // returns scala.util.Try
   conn = Some(c)
   u <- findUser(username, c) // returns Try[UserData]
   _ <- checkPassword(u, c, password)
  } yield {
    c.close()
    u
  }

  // close connection in case of failure
  user.recoverWith {
    case ex =>
     conn.foreach(_.close)
     Failure(ex)
  }
}
```

<!--description-->
My first thought was: that `var conn` is really ugly! When I tried to remove it, I struggled a little bit. Why? because of the monadic behaviour of `Try`: if `findUser` method fails, the `yield` block is not executed, thereby leaving the connection opened. At that moment I understood why the author of this code created a variable outside the for comprenhension: to be able to close the connection in case of failure.

Then I started to search online how to deal with resources in a monadic way. What I ended up doing was something like: 

```scala
def logIn(username: String, password: String) = {
  for {
   c <- getConnection() // If this fails it means it couldn't create the connection, so there is no need to close
   u <- findUser(username, c) // The connection is closed inside this method in case of failure
   _ <- checkPassword(u, c, password) // The connection is closed inside this method in case of failure
  } yield {
    c.close()
    u
  }
}
```
Because I wanted to remove that variable that held a reference to the connection, I moved the logic of failure handling inside the methods `findUser` and `logIn` (which are the ones that use the connection). The resulting code was cleaner, but I wasn't happy about the fact that now two methods were polluted with the logic of closing the connection in case something bad happened. One could say that two methods is not a big deal, but what if we have more?

So, I kept searching how can I refactor this code, and I found a library for automatic resource management called [scala-arm](https://github.com/jsuereth/scala-arm/wiki){:target="_blank"}. But, I didn't want to add a whole library to the project for this simple use case. Then, I found a couple of functions that looked like this:

```scala
def managed[A <: { def close() }, B](resource: A)(f: A => B): B =
{
  try {
    f(resource)
  } finally {
    Try{resource.close()} // In case close throws exceptions
  }
}
```
This function was all I needed. I could refactor the code in the following way:

```scala
def logIn(username: String, password: String) = {
  for {
   c <- getConnection()
   u <- managed(c)(logIn(_, username, password))
  } yield u
}

private def logIn(conn: Connection, username: String, password: String): Try[UserData] = 
  for {
   u <- findUser(username, conn) 
   _ <- checkPassword(u, conn, password)
  } yield u

```
As you can see, this code is cleaner and doesn't have all the nasty things that the original version had. And, it's better than the previous attempt because the functions don't have the noise that was introduced with the error handling logic.

