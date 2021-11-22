# The Beginner’s Guide to Kotlin Coroutine Internals

 November 9, 2021 16 Minute Read Backend [24](https://doordash.engineering/2021/11/09/the-beginners-guide-to-kotlin-coroutine-internals/#)

**[![img](https://i0.wp.com/doordash.engineering/wp-content/uploads/2021/11/Screen-Shot-2021-11-05-at-5.05.43-PM.png?fit=96%2C96&ssl=1)Sonic Wang](https://doordash.engineering/2021/11/09/the-beginners-guide-to-kotlin-coroutine-internals/#)** 1

When moving from a [monolith to a microservices architecture](https://doordash.engineering/2020/12/02/how-doordash-transitioned-from-a-monolith-to-microservices/), engineering teams often need to master a new programming paradigm. During our own migration, we opted to base our new backend architecture on [Kotlin](https://kotlinlang.org/), shifting away from [Python](https://www.python.org/). Kotlin coroutines were an area our engineering teams needed to quickly understand.

While Kotlin’s benefits include [a rich ecosystem, interoperability with Java, and developer friendliness](https://doordash.engineering/2021/05/04/migrating-from-python-to-kotlin-for-our-backend-services/), many engineers are unfamiliar with using Kotlin in backend development. It's typically considered a frontend language, but it also allows implementing robust and high performance backend services. 

We created this guide to help engineers understand how coroutines work in Kotlin, as compared to Java threading. We will discuss what coroutines are, get an overview of language, library and scheduling components of Kotlin coroutines, and then deep dive into these components. The guide also helps illustrate programming with coroutines in general. 

# What are coroutines? 

Let’s start with the basics. Coroutines, [according to Wikipedia](https://en.wikipedia.org/wiki/Coroutine), are “computer program components that generalize subroutines for non-preemptive multitasking, by allowing execution to be suspended and resumed.“ Kotlin’s native language and runtime support for coroutines allows easy asynchronous programming when building high performance applications. Compared to a Java thread, a Kotlin coroutine has a smaller memory footprint and lower overhead in context-switching.

The example below comes from Kotlin’s official [tutorial](https://kotlinlang.org/docs/coroutines-basics.html#coroutines-are-light-weight). This example created 100K coroutines with a few lines of code. The highlighted code will be executed in coroutines. Each coroutine prints a dot character after five seconds. The language and standard libraries make it super convenient to create coroutines, and the Kotlin runtime scheduling hides the complexities of managing concurrent coroutines.

```kotlin
import kotlinx.coroutines.*

//sampleStart
fun main() = runBlocking {
    repeat(100_000) { // launch a lot of coroutines
        launch {
            delay(5000L)
            print(".")
        }
    }
}
//sampleEnd
```

Copy

![img](https://doordash.engineering/wp-content/uploads/2021/11/coroutine-11-1-1024x484.jpg)Figure 1: When coroutines are waiting for an external response, the Kotlin scheduler puts them in a suspended state and continues to process other coroutines.

Figure 1 above shows how a coroutine’s body (actual code written by a developer to achieve a certain purpose) is executed by a thread. Coroutines are also suspendable. meaning that when a coroutine is waiting for an external response (such as a network response or a device I/O) it becomes suspended and the Kotlin scheduler moves this coroutine off the thread. The same thread can then pick up other coroutines waiting to be executed. This way coroutines appear to always be executing concurrently (but may not be simultaneously). In fact, replacing “suspended” with “blocked” and “coroutine” with “thread” points to an obvious analogy in that coroutines and threads both enter waiting states and can resume executing after transitioning out of waiting states. 

In addition to the code blocks in the above example, a function can be defined as suspendable using the keyword, “suspend”:

```kotlin
suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) // pretend we are doing something useful here
    return 13
} 
```

Copy

In the above example, a coroutine running doSomethingUsefulOne will suspend upon executing the delay call, which makes the coroutine wait for one second. When the delay is finished, the Kotlin scheduler puts the suspended coroutine back into the thread for execution, returns 13 and finishes this function call.

By design, it is convenient to write asynchronous programming code without nested callbacks. Asynchronous calls, such as the delay code above, simply become suspend points and developers can focus on business logic.

# Overview of the language, library, and scheduling components

Approximately three components, the Kotlin language, the standard library, and scheduling, combine to make coroutines work in Kotlin.

- **Language**: The Kotlin language compiles directly to JVM bytecode. The runtime context and control flow states of a coroutine are managed through a mechanism called [continuation passing](https://en.wikipedia.org/wiki/Continuation-passing_style). First, a method is called, and local variables are created on the stack. Then, upon suspension, the runtime saves all the local variables used so far, saves where the code has executed so far in this method, and returns the state. Finally, upon resume, the runtime restores all the local variables from the saved context, jumps to where the code was suspended, and resumes execution.

- Standard library

  : All the support for coroutines is written in Kotlin’s core libraries. A coroutine internally uses a

   

  Continuation

   

  class to capture the contexts for its execution. Then the dynamic aspect is modeled as a

   

  Job

   

  class. The use of async usually creates a

   

  Deferred

   

  job, which is a subclass of the Job class. The

   

  CoroutineContext

   

  type is required for a coroutine to execute. To summarize:

  - “Continuation” is concerned with how Kotlin passes around blocks of code in a higher order functional way.
  - “Job” is concerned with how Kotlin manages the lifecycle of a coroutine (cancelled, completed, active, etc.).
  - “CoroutineContext” is concerned with the runtime of a coroutine or a group of coroutines, such as how it’s dispatched or existing context data. One of the important related concepts is [CoroutineScope](https://elizarov.medium.com/coroutine-context-and-scope-c8b255d59055) which defines an interface to wrap the coroutine context and facilitates context propagation among coroutines.

- Scheduling

  : This is actually also part of the standard library. With language support we have a coroutine natively represented as some mechanisms in JVM bytecode. With standard libraries we are able to write code to create coroutines. The last step is to make the JVM runtime execute coroutines.

  - There are many ways to schedule a coroutine. Some common ways include default scheduling, which uses the [CoroutineScheduler](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/jvm/src/scheduling/CoroutineScheduler.kt), and [ExecutorCoroutineDispatcher](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/jvm/src/Executors.kt), which wraps around a Java SDK Executor thread pool.
  - When a Job is ready to execute, it will be executed as a [DispatchedTask](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/common/src/internal/DispatchedTask.kt). The code block (coroutine in byte code) will be run and the returned state will further be dispatched through a [dispatcher](https://kotlinlang.org/docs/coroutine-context-and-dispatchers.html).

# A closer look at the Kotlin language for coroutines through compiled codes

Let’s see in a code sample what a coroutine is. We will use a suspend function, shown in Figure 2, below, as the running example. In this example we used the [IntelliJ](https://www.jetbrains.com/idea/) IDE for Kotlin-to-bytecode inspection and JVM bytecode decompilation.

The IntelliJ IDE supports inspecting a Kotlin function through compiled JVM bytecodes. This means we can compare side by side a snippet of Kotlin code and the corresponding JVM bytecodes the same Kotlin code compiles to. This feature is located in the “Tools” option in the IntelliJ IDE. By opening the Tools option list we can further locate the “Kotlin” option to see the “Show Kotlin Bytecode” option. 

The figure below is an example that shows what we would expect to see the side by side comparison between a suspend function in Kotlin and its corresponding bytecodes on the right. 

![img](https://doordash.engineering/wp-content/uploads/2021/11/image5-1024x691.png)Figure 2: The left side is the Kotlin code and the right side is the compiled JVM bytecode. For anyone who is really curious, this is the JVM bytecode [spec](https://en.wikipedia.org/wiki/Java_bytecode_instruction_listings).

After seeing the JVM bytecodes of a Kotlin function we can rely on IntelliJ to de-compile the bytecodes back to Java. With the bytecodes in the above example (Figure 2, right hand side), the de-compiled Java code looks like below (figure 3):

![img](https://doordash.engineering/wp-content/uploads/2021/11/image10-1024x695.png)Figure 3: The above screenshot is the Java-equivalent code from decompiling the bytecode. It shows that a suspend function is a **static method.**

Ignore the details of variable names (they are programmatic and hard to read, because they were generated in a programmatic way), and for now let’s focus on the following aspects of the aforementioned code snippet:

1. **The code uses labels**: Java supports [goto](https://en.wikipedia.org/wiki/Goto) statements through the use of labels. We will soon see that scopes and labels are used extensively as the way to implement coroutines. Now we know a Kotlin suspend function (the static aspect of a coroutine) will be compiled into a static JVM method.
2. **One label is defined as an int variable**: Using an int variable label is very important because it serves as the key to a switch statement for a coroutine to resume back to where it left off.

![img](https://doordash.engineering/wp-content/uploads/2021/11/image4-924x1024.png)Figure 4: shows the beginning of the control flow switch code block. The Label variable will be passed around as the “next code block pointer” to resume. The Kotlin compiler at compilation time will figure out all the suspend call sites and reorganize the code (in a generic code generation way) so that each number will map to a suspend call site.

From the above example we are making several observations regarding how the code is written and structured:

- The smallest execution unit is the code between two suspend calls within a method. If we have a method where we call 50 suspend functions, logically Kotlin considers this method to have up to 50 units of execution scheduling (as the upper bound).
- Case 0 is special. It corresponds to the beginning of a method up to the first call site of a suspend function. This implementation detail perfectly explains the [documentation](https://kotlinlang.org/docs/reference/coroutines/coroutine-context-and-dispatchers.html#unconfined-vs-confined-dispatcher) on the Kotlin website: “*... The Dispatchers.Unconfined coroutine dispatcher starts a coroutine in the caller thread, but only until the first suspension point…*”
- Notice this variable: *Object var8 = IntrinsicsKt.getCOROUTINE_SUSPENDED()*. We can see that a suspend function can either return *IntrinsicsKt.getCOROUTINE_SUSPENDED****()\*** or the actual result of the function. If the return value is the former case, we know the suspend function we are calling is not ready, and we need to suspend the current function and return.
- Notice the variables: L$0, L$1, L$2…. These are the states to save for local variables. When the coroutine resumes, these variables will be restored with the saved values. Also, Kotlin captures all necessary local variables statically at compile time and makes them member objects of the continuation class.

Now let’s see another more complex example:

![img](https://doordash.engineering/wp-content/uploads/2021/11/image2-946x1024.png)Figure 5: an example of a complex control flow switch code block with nested labels.

Notice the break labelxx statements and similar try blocks. The innermost scope has a switch statement, which is the control flow statement we discussed before as the jump table for suspend code blocks. The Kotlin compiler programmatically generates numbered label scopes for suspend code blocks. In the above example, if label == 1, it breaks off to run the try block immediately following the switch block. If label == 2, it breaks off to run the try block immediately following label477’s block, and so on.

Now we can see that with the control flow switch statement, local variable capturing, and labeled code scopes, Kotlin has implemented the control and data flows of a coroutine in JVM.

Before concluding this section, let’s examine the following code snippet:

![img](https://doordash.engineering/wp-content/uploads/2021/11/image9-1024x222.png)

![img](https://doordash.engineering/wp-content/uploads/2021/11/image1-1024x379.png)Figure 6: The upper half is the decompiled Java code for a coroutine’s suspend function call. The lower half is the bytecode for the if-return block. The above logic is as follows: if the call to the suspend function returns suspended, then we return from this coroutine.

This [file](https://github.com/JetBrains/kotlin/blob/ee1e05fedd9b80a4524ae0d4117148012d40a41a/compiler/backend/src/org/jetbrains/kotlin/codegen/coroutines/CoroutineTransformerMethodVisitor.kt#L902-L904) in Kotlin’s compiler generates the above bytecode:

![img](https://doordash.engineering/wp-content/uploads/2021/11/image7-1024x378.png)Figure 7: The Kotlin compiler implementation that generates code in figure 6

# Introducing Kotlin’s standard library

Kotlin provides a set of out-of-the-box classes and utilities to make app development easier. Kotlin’s classes and utilities are wrapped inside the Kotlin standard library. In this section we will examine a few classes to illustrate how they facilitate development with coroutines.

We will use the following code (which is a Kotlin *async* code block) to see what we could find out:

![img](https://doordash.engineering/wp-content/uploads/2021/11/image8-1024x169.png)Figure 8: An example of creating a Kotlin coroutine through an *async* code block.

Then it leads us to the [Builders.common.kt](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/common/src/Builders.common.kt#L82) file.

![img](https://doordash.engineering/wp-content/uploads/2021/11/image3-1024x293.png)Figure 9: The implementation of function *async* in the standard library.

There are a whole bunch of classes that we don’t know about yet in the above figures. Let’s do some more digging to understand the functionalities of each class in the above example:

- [CoroutineContext](https://github.com/JetBrains/kotlin/blob/master/libraries/stdlib/coroutines-experimental/src/kotlin/coroutines/experimental/CoroutineContext.kt): We see this almost everywhere throughout the code bases. It’s used for passing some context and data as key-value pairs. What’s interesting is the `public operator fun plus(context: CoroutineContext)`: CoroutineContext method. Now we know why we sometimes do “context + something” when we launch new coroutines. The “+” is just an “operator overloading” defined for CoroutineContext.
- [CoroutineStart](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/common/src/CoroutineStart.kt): This determines how and when a coroutine should start. Ignore other cases (we only occasionally or rarely do), and focus on DEFAULT. Later the file defined the “invoke” operator overloading. In Java/JVM “invoke” means “calling”. This means CoroutineStart essentially is a “[Functional Interface](https://www.baeldung.com/java-8-functional-interfaces)”. It in turn will call `startCoroutineCancellable` of the provided block, which is of type suspend () -> T. Recall the Language section where we use the same trick to see what Kotlin does with it. Below is the result:

```kotlin
/** A function that takes 1 argument. */
public interface Function1<in P1, out R> : Function<R> {
    /** Invokes the function with the specified argument. */
    public operator fun invoke(p1: P1): R
}
```

Copy

It’s a functional interface with **one** input parameter. Of course, every suspend function under the cover will have one extra parameter, which is the Continuation object.

- [Deferred](https://github.com/Kotlin/kotlinx.coroutines/blob/1b34e1c7dd6207d7683c307bae0b934a3dc18d09/kotlinx-coroutines-core/common/src/Deferred.kt): From tracing the source code we know it’s a subclass of [Job](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html). And this “Deferred” class has a result. From this observation we now know that an “async” call does a bunch of bookkeeping and finally creates a coroutine and dispatches a Deferred Job with a result.

- Job

  : This file is rather big and complicated, but it’s well documented. It defines the state machine of a job. The description of this state machine is “.

  .. 

  A background job. Conceptually, a job is a cancellable thing with a life-cycle that culminates in its completion.” A few things worth noticing:

  - A job only completes when all children jobs are completed.
  - A child job’s cancelation will cancel the parent job.
  - Cancellation is such an important concept that Kotlin models it natively and weaves this concept throughout its implementation for coroutines.

We have looked at a bunch of Kotlin classes for async and coroutines. Here are a few takeaways:

- Whenever in doubt, we could just look at the implementation given that the core classes are not too many and it’s fairly straightforward to understand what Kotlin does under the covers.
- Coroutines have their code represented as static Java method/classes in bytecode in compilation time. Coroutines have their runtime data structures represented as Jobs along with contexts, which are just a bunch of JVM objects which are [object oriented pointers](https://www.baeldung.com/java-memory-layout). Coroutines are indeed cheap (compared to creating tons of threads which in turn create the same number of Linux threads).
- Coroutines, at the end of the day, are just a bunch of jobs getting scheduled and executed at some point in JVM. As long as we keep that in mind, from a programming model perspective it’s very similar to thread programming.

# Scheduling coroutines

So far we figured out how JVM understands a coroutine so it can run a coroutine, suspend a coroutine, and resume a coroutine. We also figured out how a coroutine is represented with core classes in the Kotlin standard library. Now let’s look at how a coroutine (or a Job) is actually getting run in JVM.

There are quite a number of ways to run coroutines. The official [documentation](https://kotlinlang.org/docs/reference/coroutines/coroutine-context-and-dispatchers.html) has a good high-level overview. We will take an implementation perspective. Here we examine three types (in fact there could be more, but this article is not intended to be exhaustive):

- [Event loop](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/common/src/EventLoop.common.kt#L252)-based: A thread runs a while true loop looks for jobs to execute in its queue. Such an event loop also takes care of scheduling delayed jobs (e.g. scheduling to run in the future given a delay time). Usually when we do runBlocking or launch(Dispatchers.Unconfined) we end up submitting a coroutine to an event-based dispatch for execution.
- [CoroutineScheduler](https://github.com/Kotlin/kotlinx.coroutines/blob/72238976ac6c06f7b1f6a72d51a41b0573bbc225/kotlinx-coroutines-core/jvm/src/scheduling/CoroutineScheduler.kt): This [work stealing](https://en.wikipedia.org/wiki/Work_stealing) scheduler is used as the default dispatcher. The dispatcher spawns a pool of worker threads with each thread attached to a designated CPU core. Each worker maintains a local job queue and in addition there is a global queue. Jobs (coroutines) submitted by a worker will be added to the worker’s local queue. And workers can steal other workers' jobs in their local queues to work on so that, on average, if there are P processors and N jobs the workload per processor is expected to be N/P. Golang’s scheduler is similar to this design, which is commonly referred to as m:n scheduling.
- [ExecutorCoroutineDispatcher](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/jvm/src/Executors.kt): This is the most used thread type throughout DoorDash’s codebases. Essentially it wraps around a plain [Java Executor](https://www.baeldung.com/java-executor-service-tutorial). Each submitted coroutine is also wrapped as a Runnable, a type expected by the [Java Development Kit](https://en.wikipedia.org/wiki/Java_Development_Kit). The per-thread affinity is not preserved, because when a coroutine returns due to suspension, the next scheduling will be added to the thread pool’s job queue. And every thread has a fair chance to grab it (most common thread pool implementations).

Whenever a coroutine is up for execution by one of the dispatcher types, the following [code](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/common/src/internal/DispatchedTask.kt) will be run (in normal use cases):

1. A DispatchedTask will be used to represent a coroutine job in the context of a Dispatcher. This is a subclass of kotlinx.coroutines.scheduling.Task, which in turn is a subclass of Java’s Runnable.

2. The DispatchedTask’s overriding

    

   run()

    

   method does all the magic for running a coroutine, including:

   1. Bookkeeping to prepare for coroutine state to pass into the continuation
   2. Running the coroutine code block (recall from the Language section how that’s done) with the states and contexts to pass to the code block
   3. Checking return results. And, if there is more work, dispatch (e.g. submit to the worker queue) again the next call for the coroutine with the newly obtained context from the return result. This way at some point in the future the dispatcher’s thread(s) will pick up the new work from the queue and run it.

Before concluding this section, a few points about scheduling:

1. Event loop-based dispatchers will be frequently referred to in the mobile app development (e.g. main thread). We need to be careful not to schedule anything CPU-intensive (or blocking the event loop thread).
2. The default dispatcher is a general purpose dispatcher which is optimized for CPU-intensive computation with no-blocking. The reason is that default dispatch only has a small set of worker threads (number of cores) blocking any of these would significantly hurt the throughput.
3. Executor dispatchers are more tolerant to misuses. But in that case the general wisdom of Java concurrently/threading/performance will apply. Properly sizing the thread pool is always a good approach toward more predictable Kotlin service performances.

# Conclusion

We have examined Kotlin coroutines from multiple angles through a combination of source code reading and bytecode/Kotlin compilation/decompilation exercises. Now we know that coroutines are resumable tasks that are lightweight to create and schedule. We also know when and how a coroutine suspends during its execution. Finally we have a better understanding of what mechanism is used for scheduling the execution of coroutines. These deeper understandings of coroutines should guide us writing high performance code, or simply, better code in Kotlin.