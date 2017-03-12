---
layout: post
title:  "Singletons can't be mocked"
date:   2016-10-15 15:57:54 -0500
categories: ['scala', 'unit tests']
---
The other day, I was making some changes to one of the projects that I'm currently working on. I realized that there weren't unit test for the code that I was changing. I wanted to create some unit tests so I could be sure that I wasn't breaking anything with the modifications that I was doing.

Some of the methods that I wanted to test consume a web service. My plan was to mock these HTTP calls because I was interested in testing the logic around the data returned by the web service and not in the HTTP calls. Then, I found that the way the code was structured didn't let me mock the web service response. Let me explain why.

<!--description-->

We had a singleton object that was in charge of consuming the web service. The role of this object was to act as a container for a bunch of methods that consume web services.

```scala
object TokenServiceClient {
 // this method calls a web service that validates the given token.
 def callValidationService(token: Token): ServiceResponse = ...
}
```

Then the method that I wanted to test was:

```scala
object TokenValidator {

 def validateToken(token: Token): TokenValidation = {
    val response = TokenServiceClient.callValidationService(token)
    // some business logic ...
  }
}
```
The first thing you notice here is that both are defined as `object`. It looks to me that we were writing Scala the Java way because in Java is pretty common to create static methods for computations that don't deal with any instance variables and return a result based on its input parameters. And the Scala way to create "static" methods is to create them in a singleton object.

When I wanted to test the `validateToken` method, I found two problems:

1. There were no way to inject a mocked version of `TokenServiceClient`.
2. `TokenServiceClient` couldn't be mocked because is a singleton.

I'm going to describe two ways of refactoring this code that will make it more flexible and easy to test.

## Using Traits

The first thing we need to do is to solve problem 1. To do that, we need to change TokenValidator from an object to a class because Scala's objects cannot have constructor parameters.

```scala
class TokenValidator(serviceClient: TokenServiceClient) {

   def validateToken(token: Token): TokenValidation = {
    val response = serviceClient.callValidationService(token)
    // some business logic ...
  }
}
```

Now, we need to solve problem 2 because we cannot mock TokenServiceClient due to the fact it is a singleton. In this case, I'm going to change TokenServiceClient so it can be a `trait`.

```scala
trait TokenServiceClient {
 def callValidationService(token: Token): ServiceResponse = {...}
}
```

This way we can inject any implementation of `TokenServiceClient`.

Finally, we can write our unit test:

```scala
class TokenValidatorTest {
  // anonymous classes
  val mockedService = new TokenServiceClient { 
    override def callValidationService(token: Token): ServiceResponse = { // return mocked response
    }
  }

  test("my test"){
    val token = ...
    val sut = TokenValidator(mockedService)

    sut.validateToken(token)
    // rest of the test
  }

}
```

With these changes we can mock the web service response and test only the logic that we are interested in.

## Using functions

Another possible way to change the code so it can be easier to test is to use functions. Let’s see how we can do that:

Original code:

```scala
object TokenValidator {

 def validateToken(token: Token): TokenValidation = {
    val response = TokenServiceClient.callValidationService(token)
    // some business logic ...
  }
}
```
We can modify this method so it can receive a function that receives a `Token` and returns a `ServiceResponse`:

```scala
object TokenValidator {

 def validateToken(token: Token)(f: Token => ServiceResponse): TokenValidation = {
    val response = f.(token)
    // some business logic ...
  }
}
```

Now, to use this method we only need to pass one of the defined methods in the object `TokenServiceClient`:

```scala
val myToken = Token("123")

TokenValidator.validate(myToken)(TokenServiceClient.callValidationService)
```

Finally, we can write our unit test:

```scala
class TokenValidatorTest {
  test("my test"){
    TokenValidator.validate(myToken)(t => mockedResponse)
    // rest of the test
  }
}
```
Using functions we can mock the web service response just by passing a function that returns the response that we want.

## Conclusions

Both approaches made this code more flexible and easier to test. We can see the first approach as the OOP way to solve our problem; we defined a `Trait` that allowed us to inject different implementations to our `TokenValidator` class. Since we don't have multiple implementations of the trait, one could argue, is it worth to create a trait just for testing purposes?

The second one is a more functional approach; we used methods/functions to define our business logic. It can be tedious to pass a function every time you want to call a method on `TokenValidator`, but you could use implicits to ease the pain.

How would you refactor this code?
