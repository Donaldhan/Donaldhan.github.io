---
layout: page
title: my blog
subtitle: sub title
date: 2018-11-04 15:17:19
author: donaldhan
catalog: true
category: Hystrix
categories:
    - Hystrix
tags:
    - Hystrix
---

# 引言



# 目录
* [](#)
    * [](#)
    * [](#)
* [总结](#总结)

##




What does it do?
1. Latency and Fault Tolerance
Stop cascading failures. Fallbacks and graceful degradation. Fail fast and rapid recovery.

Thread and semaphore isolation with circuit breakers.

2. Realtime Operations
Realtime monitoring and configuration changes. Watch service and property changes take effect immediately as they spread across a fleet.

Be alerted, make decisions, affect change and see results in seconds.

3. Concurrency
Parallel execution. Concurrency aware request caching. Automated batching through request collapsing.

This demo simulates 4 different HystrixCommand implementations with failures, latency, timeouts and duplicate calls in a multi-threaded environment.

It logs the results of HystrixRequestLog and metrics from HystrixCommandMetrics.

# Keep the Streams Flowing

Our implementation goes a little further than the basic CircuitBreaker pattern in that fallbacks can be triggered in a few ways:

* A request to the remote service times out
* The thread pool and bounded task queue used to interact with a service dependency are at 100% capacity
* The client library used to interact with a service dependency throws an exception

These buckets of failures factor into a service’s overall error rate and when the error rate exceeds a defined threshold then we “trip” the circuit for that service and immediately serve fallbacks without even attempting to communicate with the remote service.

Each service that’s wrapped by a circuit breaker implements a fallback using one of the following three approaches:

* Custom fallback — in some cases a service’s client library provides a fallback method we can invoke, or in other cases we can use locally available data on an API server (eg, a cookie or local JVM cache) to generate a fallback response
* Fail silent — in this case the fallback method simply returns a null value, which is useful if the data provided by the service being invoked is optional for the response that will be sent back to the requesting client
* Fail fast — used in cases where the data is required or there’s no good fallback and results in a client getting a 5xx response. This can negatively affect the device UX, which is not ideal, but it keeps API servers healthy and allows the system to recover quickly when the failing service becomes available again.


# Real-time Stats Drive Software and Diagnostics


# Netflix DependencyCommand Implementation
The service-oriented architecture at Netflix allows each team freedom to choose the best transport protocols and formats (XML, JSON, Thrift, Protocol Buffers, etc) for their needs so these approaches may vary across services.

In most cases the team providing a service also distributes a Java client library.

Because of this, applications such as API in effect treat the underlying dependencies as 3rd party client libraries whose implementations are “black boxes”. This in turn affects how fault tolerance is achieved.

In light of the above architectural considerations we chose to implement a solution that uses a combination of fault tolerance approaches:

* network timeouts and retries
* separate threads on per-dependency thread pools
* semaphores (via a tryAcquire, not a blocking call)
* circuit breakers

he Netflix DependencyCommand implementation wraps a network-bound dependency call with a preference towards executing in a separate thread and defines fallback logic which gets executed (step 8 in flow chart below) for any failure or rejection (steps 3, 4, 5a, 6b below) regardless of which type of fault tolerance (network or thread timeout, thread pool or semaphore rejection, circuit breaker) triggered it.

![Netflix-DependencyCommand-Implementation](/image/Hystrix/Netflix-DependencyCommand-Implementation.png)   

信号量配置的使用，在什么情况下触发？

![Semaphores-trigger-saturated-by-latent-network-connections](/image/Hystrix/Semaphores-trigger-saturated-by-latent-network-connections.png)   

Semaphores are used instead of threads for dependency executions known to not perform network calls (such as those only doing in-memory cache lookups) since the overhead of a separate thread is too high for these types of operations.



We also use semaphores to protect against non-trusted fallbacks. Each DependencyCommand is able to define a fallback function (discussed more below) which is performed on the calling user thread and should not perform network calls. Instead of trusting that all implementations will correctly abide to this contract, it too is protected by a semaphore so that if an implementation is done that involves a network call and becomes latent, the fallback itself won’t be able to take down the entire app as it will be limited in how many threads it will be able to block.



The tripping of circuits kicks in when a DependencyCommand has passed a certain threshold of error (such as 50% error rate in a 10 second period) and will then reject all requests until health checks succeed.


This is used primarily to release the pressure on underlying systems (i.e. shed load) when they are having issues and reduce the user request latency by failing fast (or returning a fallback) when we know it is likely to fail instead of making every user request wait for the timeout to occur.

How do we respond to a user request when failure occurs?
In each of the options described above a timeout, thread-pool or semaphore rejection, or short-circuit will result in a request not retrieving the optimal response for our customers.

An immediate failure (“fail fast”) throws an exception which causes the app to shed load until the dependency returns to health. This is preferable to requests “piling up” as it keeps Tomcat request threads available to serve requests from healthy dependencies and enables rapid recovery once failed dependencies recover.

However, there are often several preferable options for providing responses in a “fallback mode” to reduce impact of failure on users. Regardless of what causes a failure and how it is intercepted (timeout, rejection, short-circuited etc) the request will always pass through the fallback logic (step 8 in flow chart above) before returning to the user to give a DependencyCommand the opportunity to do something other than “fail fast”.

# How do we respond to a user request when failure occurs?

An immediate failure (“fail fast”) throws an exception which causes the app to shed load until the dependency returns to health. This is preferable to requests “piling up” as it keeps Tomcat request threads available to serve requests from healthy dependencies and enables rapid recovery once failed dependencies recover.

However, there are often several preferable options for providing responses in a “fallback mode” to reduce impact of failure on users. Regardless of what causes a failure and how it is intercepted (timeout, rejection, short-circuited etc) the request will always pass through the fallback logic (step 8 in flow chart above) before returning to the user to give a DependencyCommand the opportunity to do something other than “fail fast”.

Some approaches to fallbacks we use are, in order of their impact on the user experience:

* Cache: Retrieve data from local or remote caches if the realtime dependency is unavailable, even if the data ends up being stale
* Eventual Consistency: Queue writes (such as in SQS) to be persisted once the dependency is available again
* Stubbed Data: Revert to default values when personalized options can’t be retrieved
* Empty Response (“Fail Silent”): Return a null or empty list which UIs can then ignore

# Example Use Case

![Example-Use-Case](/image/Hystrix/Example-Use-Case.png)  

The above diagram shows an example configuration where the dependency has no reason to hit the 99.5th percentile and thus cuts it short at the network timeout layer and immediately retries with the expectation to get median latency most of the time, and accomplish this all within the 300ms thread timeout.

If the dependency has legitimate reasons to sometimes hit the 99.5th percentile (i.e. cache miss with lazy generation) then the network timeout will be set higher than it, such as at 325ms with 0 or 1 retries and the thread timeout set higher (350ms+).

The threadpool is sized at 10 to handle a burst of 99th percentile requests, but when everything is healthy this threadpool will typically only have 1 or 2 threads active at any given time to serve mostly 40ms median calls.

When configured correctly a timeout at the DependencyCommand layer should be rare, but the protection is there in case something other than network latency affects the time, or the combination of connect+read+retry+connect+read in a worst case scenario still exceeds the configured overall timeout.

The aggressiveness of configurations and tradeoffs in each direction are different for each dependency.

Configurations can be changed in realtime as needed as performance characteristics change or when problems are found all without risking the taking down of the entire app if problems or misconfigurations occur.

# Isolating functionality rather than the transport layer
Isolating functionality rather than the transport layer is valuable as it not only extends the bulkhead beyond network failures and latency, but also those caused by client code. Examples include request validation logic, conditional routing to different or multiple backends, request serialization, response deserialization, response validation, and decoration. Network responses can be latent, corrupted, or incompatibly changed at any time, which in turn can result in unexpected failures in this application logic.

Bulkheading around all of this—transport layer, network clients and client code—permits reliable protection against changing behavior, misconfigurations, transitive dependencies performing unexpected network activity, response handling failures, and overall latency, regardless of where it comes from.

Applying bulkheads at the functional level also enables addition of business logic for fallback behavior to allow graceful degradation when failure occurs. Failure may come via network or client code exceptions, timeouts, short-circuiting, or concurrent request throttling. All of them, however, can now be handled with the same failure handler to provide fallback responses. Some functionality may not be able to gracefully degrade, and will consequently fail fast and shed load until recovery, but many others can return stale data, use secondary systems, use defaults or other such patterns.

Operations and insight into what is going on is equally important to the actual isolation techniques. The key aspects of this are low latency metrics, low latency configuration changes, and common insight into all service relationships regardless of how the network transport is being implemented.

![Operations-insight](/image/Hystrix/Operations-insight.png)  


# Conclusion
The approaches discussed in this post have had a dramatic effect on our ability to tolerate and be resilient to system, infrastructure and application level failures without impacting (or limiting impact to) user experience.

Despite the success of this new DependencyCommand resiliency system over the past 8 months, there is still a lot for us to do in improving our fault tolerance strategies and performance, especially as we continue to add functionality, devices, customers and international markets.

If these kinds of challenges interest you, the API team is actively hiring:


# What Is Hystrix For?
Hystrix is designed to do the following:

* Give protection from and control over latency and failure from dependencies accessed (typically over the network) via third-party client libraries.
* Stop cascading failures in a complex distributed system.
* Fail fast and rapidly recover.
* Fallback and gracefully degrade when possible.
* Enable near real-time monitoring, alerting, and operational control.

## What Design Principles Underlie Hystrix?
Hystrix works by:

* Preventing any single dependency from using up all container (such as Tomcat) user threads.
阻止任何单依赖耗光所有的容器的线程。
* Shedding load and failing fast instead of queueing.
分流负载，快速失败代理排队。
* Providing fallbacks wherever feasible to protect users from failure.
* Using isolation techniques (such as bulkhead, swimlane, and circuit breaker patterns) to limit the impact of any one dependency.
使用隔离技术（隔离墙，泳道，熔断器）限制任一依赖的影响
* Optimizing for time-to-discovery through near real-time metrics, monitoring, and alerting
* Optimizing for time-to-recovery by means of low latency propagation of configuration changes and support for dynamic property changes in most aspects of Hystrix, which allows you to make real-time operational modifications with low latency feedback loops.
* Protecting against failures in the entire dependency client execution, not just in the network traffic.
防止真个客户端以来执行的失败，而不仅仅是网络拥挤。


# How Does Hystrix Accomplish Its Goals?
Hystrix does this by:

* Wrapping all calls to external systems (or “dependencies”) in a HystrixCommand or HystrixObservableCommand object which typically executes within a separate thread (this is an example of the command pattern).
* Timing-out calls that take longer than thresholds you define. There is a default, but for most dependencies you custom-set these timeouts by means of “properties” so that they are slightly higher than the measured 99.5th percentile performance for each dependency.
* Maintaining a small thread-pool (or semaphore) for each dependency; if it becomes full, requests destined for that dependency will be immediately rejected instead of queued up.
* Measuring successes, failures (exceptions thrown by client), timeouts, and thread rejections.
* Tripping a circuit-breaker to stop all requests to a particular service for a period of time, either manually or automatically if the error percentage for the service passes a threshold.
* Performing fallback logic when a request fails, is rejected, times-out, or short-circuits.
* Monitoring metrics and configuration changes in near real-time.


# How it Works

![hystrix-command-flow-chart](/image/Hystrix/hystrix-command-flow-chart.png)  


The following sections will explain this flow in greater detail:

1. Construct a HystrixCommand or HystrixObservableCommand Object
2. Execute the Command
3. Is the Response Cached?
4. Is the Circuit Open?
5. Is the Thread Pool/Queue/Semaphore Full?
6. HystrixObservableCommand.construct() or HystrixCommand.run()
7. Calculate Circuit Health
8. Get the Fallback
9. Return the Successful Response


# Return the Successful Response
If the Hystrix command succeeds, it will return the response or responses to the caller in the form of an Observable. Depending on how you have invoked the command in step 2, above, this Observable may be transformed before it is returned to you:

![hystrix-return-flow](/image/Hystrix/hystrix-return-flow.png)  

* execute() — obtains a Future in the same manner as does .queue() and then calls get() on this Future to obtain the single value emitted by the Observable
* queue() — converts the Observable into a BlockingObservable so that it can be converted into a Future, then returns this Future
* observe() — subscribes to the Observable immediately and begins the flow that executes the command; returns an Observable that, when you subscribe to it, replays the emissions and notifications
* toObservable() — returns the Observable unchanged; you must subscribe to it in order to actually begin the flow that leads to the execution of the command

# Circuit Breaker
The following diagram shows how a HystrixCommand or HystrixObservableCommand interacts with a HystrixCircuitBreaker and its flow of logic and decision-making, including how the counters behave in the circuit breaker.

![circuit-breaker-1280](/image/Hystrix/circuit-breaker-1280.png)


The precise way that the circuit opening and closing occurs is as follows:

1. Assuming the volume across a circuit meets a certain threshold (HystrixCommandProperties.circuitBreakerRequestVolumeThreshold())...

在熔断器统计时间窗口（默认10s）内，达到请求阈值默认（20个）

```Java
// default => statisticalWindow: 10000 = 10 seconds (and default of 10 buckets so each bucket is 1 second)
static final Integer default_metricsRollingStatisticalWindow = 10000;
// default => statisticalWindowBuckets: 10 = 10 buckets in a 10 second window so each bucket is 1 second
private static final Integer default_metricsRollingStatisticalWindowBuckets = 10;
// default => statisticalWindowVolumeThreshold: 20 requests in 10 seconds must occur before statistics matter
private static final Integer default_circuitBreakerRequestVolumeThreshold = 20;
/**
  * Duration of statistical rolling window in milliseconds. This is passed into {@link HystrixRollingNumber} inside {@link HystrixCommandMetrics}.
  *
  * @return {@code HystrixProperty<Integer>}
  */
public HystrixProperty<Integer> metricsRollingStatisticalWindowInMilliseconds() {
     return metricsRollingStatisticalWindowInMilliseconds;
}
/**
  * Minimum number of requests in the {@link #metricsRollingStatisticalWindowInMilliseconds()} that must exist before the {@link HystrixCircuitBreaker} will trip.
  * <p>
  * If below this number the circuit will not trip regardless of error percentage.
  *
  * @return {@code HystrixProperty<Integer>}
  */
public HystrixProperty<Integer> circuitBreakerRequestVolumeThreshold() {
     return circuitBreakerRequestVolumeThreshold;
}
```
2. And assuming that the error percentage exceeds the threshold error percentage (HystrixCommandProperties.circuitBreakerErrorThresholdPercentage())...

在统计窗口时间之前，错误率达到错误阈值，默认50%
```java
/**
     * Error percentage threshold (as whole number such as 50) at which point the circuit breaker will trip open and reject requests.
     * <p>
     * It will stay tripped for the duration defined in {@link #circuitBreakerSleepWindowInMilliseconds()};
     * <p>
     * The error percentage this is compared against comes from {@link HystrixCommandMetrics#getHealthCounts()}.
     *
     * @return {@code HystrixProperty<Integer>}
     */
    public HystrixProperty<Integer> circuitBreakerErrorThresholdPercentage() {
        return circuitBreakerErrorThresholdPercentage;
  }
  // default => sleepWindow: 5000 = 5 seconds that we will sleep before trying again after tripping the circuit
  private static final Integer default_circuitBreakerSleepWindowInMilliseconds = 5000;
  // default => errorThresholdPercentage = 50 = if 50%+ of requests in 10 seconds are failures or latent then we will trip the circuit
  private static final Integer default_circuitBreakerErrorThresholdPercentage = 50;
```

3. Then the circuit-breaker transitions from CLOSED to OPEN.

如果达到1,2条件，则将会触发熔断，并睡眠一定的时间，默认为5s。
4. While it is open, it short-circuits all requests made against that circuit-breaker.
5. After some amount of time (HystrixCommandProperties.circuitBreakerSleepWindowInMilliseconds()), the next single request is let through (this is the HALF-OPEN state). If the request fails, the circuit-breaker returns to the OPEN state for the duration of the sleep window. If the request succeeds, the circuit-breaker transitions to CLOSED and the logic in 1. takes over again.

如果熔断触发，睡眠给定的时间后，熔断器关闭，新进的请求，将会通过，如果请求失败，则重新触发熔断，并随眠给定的时间。

如果以上五步都没有问题的情况下，再去检查线程池和信号量的配置。

```java
/**
    * @param form
    * @return
    */
   @RequestMapping(value = "queryBookPage", method = { RequestMethod.POST })
   @ResponseBody
   @MvcControllerMethodLog
   @HystrixCommand(fallbackMethod = "queryBookPageFailBack",
           threadPoolProperties = {
                   @HystrixProperty(name = "coreSize", value = "100"),
                   @HystrixProperty(name = "maximumSize", value = "300"),
                   @HystrixProperty(name = "maxQueueSize", value = "1000"),
                   @HystrixProperty(name = "queueSizeRejectionThreshold", value = "1000"),
                   @HystrixProperty(name = "allowMaximumSizeToDivergeFromCoreSize", value = "true") // 默认false, 如果不设置该参数maximumSize=coreSize
           },
           commandProperties = {
                   @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = "50"),   //（出错百分比阈值，当达到此阈值后，开始短路。默认50%）
                   @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "200"),      // 在统计数据之前，必须在10秒内发出3个请求。  默认是20
                   @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "4000"), //（短路多久以后开始尝试是否恢复，默认5s）
                   @HystrixProperty(name = "execution.timeout.enabled", value = "true"),               //该方法不做超时校验
                   @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "1000"), // 响应超时时间，默1s
                   @HystrixProperty(name = "fallback.isolation.semaphore.maxConcurrentRequests", value = "1000")    //执行fallback方法的semaphore数量
           })
   public WebResponse<CommonPageResultVO<BookInfoVO>> queryBookPage(@Valid  @RequestBody BookPageForm form) {
     ...
  }
```

# Isolation
Hystrix employs the bulkhead pattern to isolate dependencies from each other and to limit concurrent access to any one of them.

# Threads & Thread Pools
Clients (libraries, network calls, etc) execute on separate threads. This isolates them from the calling thread (Tomcat thread pool) so that the caller may “walk away” from a dependency call that is taking too long.

Hystrix uses separate, per-dependency thread pools as a way of constraining any given dependency so latency on the underlying executions will saturate the available threads only in that pool.

It is possible for you to protect against failure without the use of thread pools, but this requires the client being trusted to fail very quickly (network connect/read timeouts and retry configuration) and to always behave well.

Netflix, in its design of Hystrix, chose the use of threads and thread-pools to achieve isolation for many reasons including:

* Many applications execute dozens (and sometimes well over 100) different back-end service calls against dozens of different services developed by as many different teams.
* Each service provides its own client library.
* Client libraries are changing all the time.
* Client library logic can change to add new network calls.
* Client libraries can contain logic such as retries, data parsing, caching (in-memory or across network), and other such behavior.
* Client libraries tend to be “black boxes” — opaque to their users about implementation details, network access patterns, configuration defaults, etc.
* In several real-world production outages the determination was “oh, something changed and properties should be adjusted” or “the client library changed its behavior.”
* Even if a client itself doesn’t change, the service itself can change, which can then impact performance characteristics which can then cause the client configuration to be invalid.
* Transitive dependencies can pull in other client libraries that are not expected and perhaps not correctly configured.
* Most network access is performed synchronously.
* Failure and latency can occur in the client-side code as well, not just in the network call.

![isolation-options](/image/Hystrix/isolation-options-1280.png)


# Benefits of Thread Pools
The benefits of isolation via threads in their own thread pools are:

* The application is fully protected from runaway client libraries. The pool for a given dependency library can fill up without impacting the rest of the application.
* The application can accept new client libraries with far lower risk. If an issue occurs, it is isolated to the library and doesn’t affect everything else.
* When a failed client becomes healthy again, the thread pool will clear up and the application immediately resumes healthy performance, as opposed to a long recovery when the entire Tomcat container is overwhelmed.
* If a client library is misconfigured, the health of a thread pool will quickly demonstrate this (via increased errors, latency, timeouts, rejections, etc.) and you can handle it (typically in real-time via dynamic properties) without affecting application functionality.
* If a client service changes performance characteristics (which happens often enough to be an issue) which in turn cause a need to tune properties (increasing/decreasing timeouts, changing retries, etc.) this again becomes visible through thread pool metrics (errors, latency, timeouts, rejections) and can be handled without impacting other clients, requests, or users.
* Beyond the isolation benefits, having dedicated thread pools provides built-in concurrency which can be leveraged to build asynchronous facades on top of synchronous client libraries (similar to how the Netflix API built a reactive, fully-asynchronous Java API on top of Hystrix commands).
* In short, the isolation provided by thread pools allows for the always-changing and dynamic combination of client libraries and subsystem performance characteristics to be handled gracefully without causing outages.

*Note*: Despite the isolation a separate thread provides, your underlying client code should also have timeouts and/or respond to Thread interrupts so it can not block indefinitely and saturate the Hystrix thread pool.


建议使用配置线程池方式，可以隔离调用失败对整个容器和应用的影响。


# Drawbacks of Thread Pools
The primary drawback of thread pools is that they add computational overhead. Each command execution involves the queueing, scheduling, and context switching involved in running a command on a separate thread.

Netflix, in designing this system, decided to accept the cost of this overhead in exchange for the benefits it provides and deemed it minor enough to not have major cost or performance impact.

使用线程池的劣势，因为需要维护请求队列，调度，和上线文切换，及使用独立的线程，处理请求，会增加负载，但Netflix相信，消耗负载带来的益处远大于带来的负载弊端。同时
Netflix最小化负载的消耗，对主性能和负载没有影响。


# Semaphores
You can use semaphores (or counters) to limit the number of concurrent calls to any given dependency, instead of using thread pool/queue sizes. This allows Hystrix to shed load without using thread pools but it does not allow for timing out and walking away. If you trust the client and you only want load shedding, you could use this approach.

HystrixCommand and HystrixObservableCommand support semaphores in 2 places:

* Fallback: When Hystrix retrieves fallbacks it always does so on the calling Tomcat thread.
* Execution: If you set the property execution.isolation.strategy to SEMAPHORE then Hystrix will use semaphores instead of threads to limit the number of concurrent parent threads that invoke the command.
You can configure both of these uses of semaphores by means of dynamic properties that define how many concurrent threads can execute. You should size them by using similar calculations as you use when sizing a threadpool (an in-memory call that returns in sub-millisecond times can perform well over 5000rps with a semaphore of only 1 or 2 … but the default is 10).

Note: if a dependency is isolated with a semaphore and then becomes latent, the parent threads will remain blocked until the underlying network calls timeout.

Semaphore rejection will start once the limit is hit but the threads filling the semaphore can not walk away.

信号量也可以控制并发，但是没有超时的熔断条件。如果你相信依赖的应用或更在乎负载，则可以使用信号量控制并发。



```java
//fallback默认最大并发
private static final Integer default_fallbackIsolationSemaphoreMaxConcurrentRequests = 10;
//请求最大并发量
private static final Integer default_executionIsolationSemaphoreMaxConcurrentRequests = 10;
```




###

```java
```


###

```java
```


## 总结
