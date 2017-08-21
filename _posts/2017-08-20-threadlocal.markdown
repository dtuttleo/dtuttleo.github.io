---
layout: post
title:  "Playing with ThreadLocal"
date:   2017-08-20 15:26:36 -0500
categories: concurrency
---
When you work on enterprise applications, `TheadLocal` variables are something you don't see very often. You see them once in a while, but you never have to deal with them directly. So, in this short post I want to illustrate how `ThreadLocal` variables work. 

<!--description-->
As the name suggests, this variables are local to the thread. So for example if you set some value in thread1 and then you ask for the value while running in thread2 you will get `null` (assuming that no one set some value in this thread before) because that value is only visible inside thread1.

Let's see how this works in code!

```scala
import java.util.concurrent.{Executor, Executors}


object ThreadLocalTest {

  class TestClass(name: String){

    private[this] val tl : ThreadLocal[String] = new ThreadLocal()

    def setLocal(s: String): Unit = {
      println("Setting value '" + s + "' in " + Thread.currentThread().getName)
      tl.set(s)
    }

    def getLocal: String = {
      val s = tl.get()
      println("Getting value '" + s + "' in " + Thread.currentThread().getName)
      s
    }

  }

  def main(args: Array[String]): Unit = {
    val testObject = new TestClass("TestClass 1")

    val t1 = Executors.newSingleThreadExecutor()
    val t2 = Executors.newSingleThreadExecutor()

    execute{testObject.setLocal("th1!!")}(t1)
    Thread.sleep(1000) // sleep main thread to make sure the previous execution has run
    execute{testObject.setLocal("th2!!")}(t2)
    Thread.sleep(1000)
    execute{testObject.getLocal}(t1)
    Thread.sleep(1000)
    execute{testObject.getLocal}(t2)
    Thread.sleep(1000)
    println("In Main " + testObject.getLocal)

  }

  /**
    * Executes the given task in the given executor
    */
  private def execute(f: => Unit)( e: Executor) = {
    e.execute(new Runnable {
      override def run(): Unit = f
    })
  }
}
```

Once we run the program, we get:

```
Setting value 'th1!!' in pool-1-thread-1
Setting value 'th2!!' in pool-2-thread-1
Getting value 'th1!!' in pool-1-thread-1
Getting value 'th2!!' in pool-2-thread-1
Getting value 'null' in main
In Main null
```

As you can see, the value that is set in a specific thread is only visible in that thread, and because we never call `setlocal` in the main thread we get `null` when we call `getLocal`.

