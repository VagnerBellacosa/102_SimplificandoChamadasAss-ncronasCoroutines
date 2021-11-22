# Introduction to Kotlin Coroutines

Last modified: February 8, 2021

by [baeldung](https://www.baeldung.com/kotlin/author/baeldung)





- [Asynchronous Programming](https://www.baeldung.com/kotlin/category/asynchronous-programming)

- [Coroutines](https://www.baeldung.com/kotlin/tag/coroutines)

If you have a few years of experience with the Kotlin language and server-side development, and you’re interested in sharing that experience with the community, have a look at our [**Contribution Guidelines**](https://www.baeldung.com/contribution-guidelines).

## **1. Overview**

In this article, we’ll be looking at coroutines from the Kotlin language. Simply put, **coroutines allow us to create asynchronous programs in a very fluent way**, and they’re based on the concept of *[Continuation-passing style](https://en.wikipedia.org/wiki/Continuation-passing_style)* programming.

The Kotlin language gives us basic constructs but can get access to more useful coroutines with the *kotlinx-coroutines-core* library. We’ll be looking at this library once we understand the basic building blocks of the Kotlin language.

## **2. Creating a Coroutine With \*BuildSequence\***

Let’s create the first coroutine using the *[buildSequence](https://kotlinlang.org/docs/reference/coroutines/composing-suspending-functions.html)* function.

And let’s implement a Fibonacci sequence generator using this function:

```scala
val fibonacciSeq = buildSequence {
    var a = 0
    var b = 1

    yield(1)

    while (true) {
        yield(a + b)

        val tmp = a + b
        a = b
        b = tmp
    }
}
```

The signature of a *yield* function is:

```scala
public abstract suspend fun yield(value: T)
```

The *suspend* keyword means that this function can be blocking. Such a function can suspend a *buildSequence* coroutine.

**Suspending functions can be created as standard Kotlin functions, but we need to be aware that we can only call them from within a coroutine.** Otherwise, we’ll get a compiler error.

If we’ve suspended the call within the *buildSequence,* that call will be transformed to the dedicated state in the state machine. A coroutine can be passed and assigned to a variable like any other function.

In the *fibonacciSeq* coroutine, we have two suspension points. First, when we’re calling *yield(1)* and second when we’re calling *yield(a+b).*

If that *yield* function results in some blocking call, the current thread will not block on it. It will be able to execute some other code. Once the suspended function finishes its execution, the thread can resume the execution of the *fibonacciSeq* coroutine.

We can test our code by taking some elements from the Fibonacci sequence:

```java
val res = fibonacciSeq
  .take(5)
  .toList()

assertEquals(res, listOf(1, 1, 2, 3, 5))
```

## **3. Adding the Maven Dependency for \*kotlinx-coroutines\***

Let’s look at the *kotlinx-coroutines* library which has useful constructs build on top of basic coroutines.

Let’s add the dependency to the *kotlinx-coroutines-core* library. Note that we also need to add the *jcenter* repository:

```xml
<dependency>
    <groupId>org.jetbrains.kotlinx</groupId>
    <artifactId>kotlinx-coroutines-core</artifactId>
    <version>0.16</version>
</dependency>

<repositories>
    <repository>
        <id>central</id>
        <url>http://jcenter.bintray.com</url>
     </repository>
</repositories>
```

## **4. Asynchronous Programming Using the \*launch() C\*oroutine**

The *kotlinx-coroutines* library adds a lot of useful constructs that allow us to create asynchronous programs. Let’s say that we have an expensive computation function that is appending a *String* to the input list:

```java
suspend fun expensiveComputation(res: MutableList<String>) {
    delay(1000L)
    res.add("word!")
}
```

We can use a *launch* coroutine that will execute that suspend function in a non-blocking way – we need to pass a thread pool as an argument to it.

The *launch* function is returning a *Job* instance on which we can call a *join()* method to wait for the results:

```scala
@Test
fun givenAsyncCoroutine_whenStartIt_thenShouldExecuteItInTheAsyncWay() {
    // given
    val res = mutableListOf<String>()

    // when
    runBlocking<Unit> {
        val promise = launch(CommonPool) { 
          expensiveComputation(res) 
        }
        res.add("Hello,")
        promise.join()
    }

    // then
    assertEquals(res, listOf("Hello,", "word!"))
}
```

To be able to test our code, we pass all logic into the *runBlocking* coroutine – which is a blocking call. Therefore our *assertEquals()* can be executed synchronously after the code inside of the *runBlocking()* method.

Note that in this example, although the *launch()* method is triggered first, it is a delayed computation. The main thread will proceed by appending the *“Hello,” String* to the result list.

After the one-second delay that is introduced in the *expensiveComputation()* function, the *“word!” String* will be appended to the result.

## **5. Coroutines Are Very Lightweight**

Let’s imagine a situation in which we want to perform 100000 operations asynchronously. Spawning such a high number of threads will be very costly and will possibly yield an *OutOfMemoryException.*

Fortunately, when using the coroutines, this is not the case. We can execute as many blocking operations as we want. Under the hood, those operations will be handled by a fixed number of threads without excessive thread creation:

```scala
@Test
fun givenHugeAmountOfCoroutines_whenStartIt_thenShouldExecuteItWithoutOutOfMemory() {
    runBlocking<Unit> {
        // given
        val counter = AtomicInteger(0)
        val numberOfCoroutines = 100_000

        // when
        val jobs = List(numberOfCoroutines) {
            launch(CommonPool) {
                delay(1000L)
                counter.incrementAndGet()
            }
        }
        jobs.forEach { it.join() }

        // then
        assertEquals(counter.get(), numberOfCoroutines)
    }
}
```

Note that we’re executing 100,000 coroutines and each run adds a substantial delay. Nevertheless, there is no need to create too many threads because those operations are executed in an asynchronous way using thread from the *CommonPool.*

## **6. Cancellation and Timeouts**

**Sometimes, after we have triggered some long-running asynchronous computation, we want to cancel it because we’re no longer interested in the result.**

When we start our asynchronous action with the *launch()* coroutine, we can examine the *isActive* flag. This flag is set to false whenever the main thread invokes the *cancel()* method on the instance of the *Job:*

```scala
@Test
fun givenCancellableJob_whenRequestForCancel_thenShouldQuit() {
    runBlocking<Unit> {
        // given
        val job = launch(CommonPool) {
            while (isActive) {
                println("is working")
            }
        }

        delay(1300L)

        // when
        job.cancel()

        // then cancel successfully

    }
}
```

This is a very elegant and **easy way to use the cancellation mechanism**. In the asynchronous action, we only need to check if the *isActive* flag is equal to *false* and cancel our processing.

When we’re requesting some processing and are not sure how much time that computation will take, it’s advisable to set the timeout on such an action. If the processing does not finish within the given timeout, we’ll get an exception, and we can react to it appropriately.

For example, we can retry the action:

```java
@Test(expected = CancellationException::class)
fun givenAsyncAction_whenDeclareTimeout_thenShouldFinishWhenTimedOut() {
    runBlocking<Unit> {
        withTimeout(1300L) {
            repeat(1000) { i ->
                println("Some expensive computation $i ...")
                delay(500L)
            }
        }
    }
}
```

If we do not define a timeout, it’s possible that our thread will be blocked forever because that computation will hang. We cannot handle that case in our code if the timeout is not defined.

## **7. Running Asynchronous Actions Concurrently**

Let’s say that we need to start two asynchronous actions concurrently and wait for their results afterward. If our processing takes one second and we need to execute that processing twice, the runtime of synchronous blocking execution will be two seconds.

It would be better if we could run both those actions in separate threads and wait for those results in the main thread.

**We can leverage the \*async()\* coroutine to achieve this** by starting processing in two separate threads concurrently:

```scala
@Test
fun givenHaveTwoExpensiveAction_whenExecuteThemAsync_thenTheyShouldRunConcurrently() {
    runBlocking<Unit> {
        val delay = 1000L
        val time = measureTimeMillis {
            // given
            val one = async(CommonPool) { 
                someExpensiveComputation(delay) 
            }
            val two = async(CommonPool) { 
                someExpensiveComputation(delay) 
            }

            // when
            runBlocking {
                one.await()
                two.await()
            }
        }

        // then
        assertTrue(time < delay * 2)
    }
}
```

After we submit the two expensive computations, we suspend the coroutine by executing the *runBlocking()* call. Once results *one* and *two* are available, the coroutine will resume, and the results are returned. Executing two tasks in this way should take around one second.

We can pass *CoroutineStart.LAZY* as the second argument to the *async()* method, but this will mean the asynchronous computation will not be started until requested. Because we are requesting computation in the *runBlocking* coroutine, it means the call to *two.await()* will be made only once the *one.await()* has finished:

```java
@Test
fun givenTwoExpensiveAction_whenExecuteThemLazy_thenTheyShouldNotConcurrently() {
    runBlocking<Unit> {
        val delay = 1000L
        val time = measureTimeMillis {
            // given
            val one 
              = async(CommonPool, CoroutineStart.LAZY) {
                someExpensiveComputation(delay) 
              }
            val two 
              = async(CommonPool, CoroutineStart.LAZY) { 
                someExpensiveComputation(delay) 
            }

            // when
            runBlocking {
                one.await()
                two.await()
            }
        }

        // then
        assertTrue(time > delay * 2)
    }
}
```

**The laziness of the execution in this particular example causes our code to run synchronously.** That happens because when we call *await()*, the main thread is blocked and only after task *one* finishes task *two* will be triggered.

We need to be aware of performing asynchronous actions in a lazy way as they may run in a blocking way.

## **8. Conclusion**

In this article, we looked at the basics of Kotlin coroutines.

We saw that *buildSequence* is the main building block of every coroutine. We described how the flow of execution in this Continuation-passing programming style looks.

Finally, we looked at the *kotlinx-coroutines* library that ships a lot of very useful constructs for creating asynchronous programs.

The implementation of all these examples and code snippets can be found in the [GitHub project](https://github.com/Baeldung/kotlin-tutorials/tree/master/core-kotlin-modules/core-kotlin-concurrency).

If you have a few years of experience with the Kotlin language and server-side development, and you’re interested in sharing that experience with the community, have a look at our [**Contribution Guidelines**](https://www.baeldung.com/contribution-guidelines).

