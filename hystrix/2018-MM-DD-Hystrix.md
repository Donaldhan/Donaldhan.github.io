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




# What does it do?
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
* Shedding load and failing fast instead of queueing.
分流负载，快速失败取代排队。
* Providing fallbacks wherever feasible to protect users from failure.
* Using isolation techniques (such as bulkhead, swimlane, and circuit breaker patterns) to limit the impact of any one dependency.
使用隔离技术（隔离墙，泳道，熔断器）限制任一依赖的影响
* Optimizing for time-to-discovery through near real-time metrics, monitoring, and alerting
* Optimizing for time-to-recovery by means of low latency propagation of configuration changes and support for dynamic property changes in most aspects of Hystrix, which allows you to make real-time operational modifications with low latency feedback loops.
* Protecting against failures in the entire dependency client execution, not just in the network traffic.



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

* The application is fully protected from run away client libraries. The pool for a given dependency library can fill up without impacting the rest of the application.
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


# HystrixCollapser请求合并
* Collapser Properties：用来控制命令合并相关的行为。
* maxRequestsInBatch：该参数用来设置一次请求合并批处理中允许的最大请求数。
* timerDelayInMilliseconds：用来设置批处理过程中每个命令延迟的时间，单位为毫秒，默认值为10。
* requestCache.enabled：设置批处理过程中是否开启请求缓存。

使用场景：对负载要求较高，对延迟要求不高的高。请求高延时

## Concept

Request Collapsing enables many concurrent requests, on an external service, to be batched together into one request.

For example if a user wanted to load bookmarks for 300 video objects, rather than performing 300 network calls against a web service, these requests could be batched into one request.

Requests can be Collapsed around, the batch size and elapsed time since the batch started.

## Advantages/Disadvantages

### Advantages

Reduces the number of threads and network connections needed to perform requests
Reduces the load on the external service

### Disadvantages

An increased latency before the actual command is executed. The maximum cost is the size of the batch window
Example

The following example is taken from: Hystrix - Request Collapsing. With a slight modification, the batch size window has been increased to 2 seconds.

The example demonstrates four concurrent calls being performed, and shows how the four concurrent requests are batched together.

The following visualization shows just the first two calls, to keep the example concise.

具体时序图，可以参考:
https://design.codelytics.io/hystrix/request-collapsing


![collapser](/image/Hystrix/collapser-1280.png)


## Global Context (Across All Tomcat Threads)
The ideal type of collapsing is done at the global application level, so that requests from any user on any Tomcat thread can be collapsed together.

For example, if you configure a HystrixCommand to support batching for any user on requests to a dependency that retrieves movie ratings, then when any user thread in the same JVM makes such a request, Hystrix will add its request along with any others into the same collapsed network call.

Note that the collapser will pass a single HystrixRequestContext object to the collapsed network call, so downstream systems must need to handle this case for this to be an effective option.

## What Is the Cost of Request Collapsing?
The determination of whether this cost is worth it depends on the command being executed. A high-latency command won’t suffer as much from a small amount of additional average latency. Also, the amount of concurrency on a given command is key: There is no point in paying the penalty if there are rarely more than 1 or 2 requests to be batched together. In fact, in a single-threaded sequential iteration collapsing would be a major performance bottleneck as each iteration will wait the 10ms batch window time.

If, however, a particular command is heavily utilized concurrently and can batch dozens or even hundreds of calls together, then the cost is typically far outweighed by the increased throughput achieved as Hystrix reduces the number of threads it requires and the number of network connections to dependencies.

## Collapser Flow
![collapser-flow](/image/Hystrix/collapser-flow-1280.png)

## Request Caching ???
HystrixCommand and HystrixObservableCommand implementations can define a cache key which is then used to de-dupe calls within a request context in a concurrent-aware manner.

The Hystrix RequestCache will execute the underlying run() method once and only once, and both threads executing the HystrixCommand will receive the same data despite having instantiated different instances.

* Data retrieval is consistent throughout a request.
Instead of potentially returning a different value (or fallback) each time the command is executed, the first response is cached and returned for all subsequent calls within the same request.

* Eliminates duplicate thread executions.
Since the request cache sits in front of the construct() or run() method invocation, Hystrix can de-dupe calls before they result in thread execution.

If Hystrix didn’t implement the request cache functionality then each command would need to implement it themselves inside the construct or run method, which would put it after a thread is queued and executed.

请求缓存在run()和construce()执行之前生效，所以可以有效减少不必要的线程开销。你可以通过实现getCachekey()方法来开启请求缓存。

注解 	描述 	属性
@CacheResult 	改注解用来标记请求命令返回的结果应该被缓存，它必须与@HystrixCommand注解结合使用 	cacheKeyMethod
@CacheRemove 	该注解用来让请求命令的缓存失效，失效的缓存根据定义的key决定 	commandKey,cacheKeyMethod
@CacheKey 	改注解用来在请求命令的参数上标记，使其作为缓存的Key值，如果没有标注则会使用所有参数。如果同时还使用了@CacheResult和@CacheRemove注解的cacheKeyMethod方法指定缓存Key的生成，那么该注解将不会起作用   value





# How-To-Use

## Fallback
You can support graceful degradation in a Hystrix command by adding a fallback method that Hystrix will call to obtain a default value or values in case the main command fails. You will want to implement a fallback for most Hystrix commands that might conceivably fail, with a couple of exceptions:

* a command that performs a write operation
If your Hystrix command is designed to do a write operation rather than to return a value (such a command might normally return a void in the case of a HystrixCommand or an empty Observable in the case of a HystrixObservableCommand), there isn’t much point in implementing a fallback. If the write fails, you probably want the failure to propagate back to the caller.
* batch systems/offline compute
If your Hystrix command is filling up a cache, or generating a report, or doing any sort of offline computation, it’s usually more appropriate to propagate the error back to the caller who can then retry the command later, rather than to send the caller a silently-degraded response.
Whether or not your command has a fallback, all of the usual Hystrix state and circuit-breaker state/metrics are updated to indicate the command failure.


## Error Propagation
All exceptions thrown from the run() method except for HystrixBadRequestException count as failures and trigger getFallback() and circuit-breaker logic.

You can wrap the exception that you would like to throw in HystrixBadRequestException and retrieve it via getCause(). The HystrixBadRequestException is intended for use cases such as reporting illegal arguments or non-system failures that should not count against the failure metrics and should not trigger fallback logic.



# 如何配置
The pool should be sized large enough to handle normal healthy traffic but small enough that it will constrain concurrent execution if backend calls become latent.

 For more information see the Github Wiki: https://github.com/Netflix/Hystrix/wiki/Configuration#wiki-ThreadPool and https://github.com/Netflix/Hystrix/wiki/How-it-Works#wiki-Isolation

## HystrixThreadPool
 ``` java
 /**
    * Whether the queue will allow adding an item to it.
    * <p>
    * This allows dynamic control of the max queueSize versus whatever the actual max queueSize is so that dynamic changes can be done via property changes rather than needing an app
    * restart to adjust when commands should be rejected from queuing up.
    *
    * @return boolean whether there is space on the queue
    */
   public boolean isQueueSpaceAvailable();
   protected static class SingleThreadedPoolWithQueue implements HystrixThreadPool {

        final LinkedBlockingQueue<Runnable> queue;
        final ThreadPoolExecutor pool;
        private final int rejectionQueueSizeThreshold;

        public SingleThreadedPoolWithQueue(int queueSize) {
            this(queueSize, 100);
        }

        public SingleThreadedPoolWithQueue(int queueSize, int rejectionQueueSizeThreshold) {
            queue = new LinkedBlockingQueue<Runnable>(queueSize);
            pool = new ThreadPoolExecutor(1, 1, 1, TimeUnit.MINUTES, queue);
            this.rejectionQueueSizeThreshold = rejectionQueueSizeThreshold;
        }

        @Override
        public ThreadPoolExecutor getExecutor() {
            return pool;
        }

        @Override
        public Scheduler getScheduler() {
            return new HystrixContextScheduler(HystrixPlugins.getInstance().getConcurrencyStrategy(), this);
        }

        @Override
        public Scheduler getScheduler(Func0<Boolean> shouldInterruptThread) {
            return new HystrixContextScheduler(HystrixPlugins.getInstance().getConcurrencyStrategy(), this, shouldInterruptThread);
        }

        @Override
        public void markThreadExecution() {
            // not used for this test
        }

        @Override
        public void markThreadCompletion() {
            // not used for this test
        }

        @Override
        public void markThreadRejection() {
            // not used for this test
        }

        @Override
        public boolean isQueueSpaceAvailable() {
            return queue.size() < rejectionQueueSizeThreshold;
        }

}
 ```
rejectionQueueSizeThreshold属性，用于控制线程池任务队列为多少时，具体接受任务。


*请求缓存和请求合并，具体功能要自己实现。*

如果关注负载，没有网络的请求但是关注延迟可以使用信号量执行策略：
```Java
public CommandUsingSemaphoreIsolation(int id) {
       super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"))
               // since we're doing an in-memory cache lookup we choose SEMAPHORE isolation
               .andCommandPropertiesDefaults(HystrixCommandProperties.Setter()
                       .withExecutionIsolationStrategy(ExecutionIsolationStrategy.SEMAPHORE)));
       this.id = id;
   }

```



```java
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
```


# 配置
Global default from code<Dynamic global default property<Instance default from code<Dynamic instance property

优先级依次递增：
全局默认值< 动态配置全局默认属性< 代码层实例配置< 动态实例配置


Command属性：
Command Properties
  Execution
    execution.isolation.strategy
    THREAD，SEMAPHORE ，默认为线程隔离策略，强烈建议使用线程隔离策略，线程策略可以避免网络超时带来的请求延时。针对没有网络请求和每秒有大量请求，并且每个请求线程的负载较高的情况下，可以使用信号量策略。信号量执行策略没有超时检查。
    execution.isolation.thread.timeoutInMilliseconds
    隔离线程执行超时时间，默认为1000ms。超时则执行fallback。
    execution.timeout.enabled
    是否执行线程执行超时检查，默认为true
    execution.isolation.thread.interruptOnTimeout
    线程执行超时，是否中断执行run
    execution.isolation.thread.interruptOnCancel
    线程执行请求取消是，是否中断线程，默认为false。
    execution.isolation.semaphore.maxConcurrentRequests
    信号量执行策略下，允许的并发请求数，默认为10/s,理论上讲，应该是为容器线程池的线程数，从隔离的原则上来说，建议为线程池数量的小部分。官方给出的参考为，内存级的度量数据请求，2个信号量，可以处理5000rps
  Fallback
  此配置使用线程和信号量两种隔离策略
    fallback.isolation.semaphore.maxConcurrentRequests
    同时降级的最大并发量，默认为10
    fallback.enabled
    执行失败，或拒绝执行发生时，是否调用降级方法，默认为true，false，则抛出Hystrix运行时异常
  Circuit Breaker
    circuitBreaker.enabled
    针对请求处理失败，是否短路，追踪应用健康状况
    circuitBreaker.requestVolumeThreshold
    在一个滑动窗口内，开启熔断器检查统计条件，默认为20/10s。需要注意，如果在十秒内，有19个请求，都出现错误，也不会，触发熔断。
    circuitBreaker.sleepWindowInMilliseconds
    触发熔断器后，等待关闭熔断器的时间，默认为5000s。
    circuitBreaker.errorThresholdPercentage
    在一个统计窗口内，达到错误百分比时，触发熔断，默认为50%，主要看我们对错误民不敏感，如果敏感可以设置10%。
    circuitBreaker.forceOpen
    强制打开熔断器，拒绝所有请求，默认为false
    circuitBreaker.forceClosed
    强制关闭熔断器，忽略错误百分比，当期circuitBreaker.forceOpen配置为true，此配置忽略
  Metrics
      metrics.rollingStats.timeInMilliseconds
      一个滑动度量窗口的时间，默认为10s
      metrics.rollingStats.numBuckets
      一个滑动窗口内，统计桶的数量，默认为10；这样做的目的是将滑动窗口分为多个小的统计小窗口，防止统计数据的丢失。需要注意metrics.rollingStats.timeInMilliseconds % metrics.rollingStats.numBuckets == 0，否则将会抛出异常。
      metrics.rollingPercentile.enabled
      请求延时统计(mean, percentiles)，默认为true，没有则为-1
      metrics.rollingPercentile.timeInMilliseconds
      延时统计滑动窗口时间，默认为60s
      metrics.rollingPercentile.numBuckets
      延时统计窗口内的桶数，默认为6, 需要注意metrics.rollingPercentile.timeInMilliseconds % metrics.rollingPercentile.numBuckets == 0，否则将会抛出异常。
      metrics.rollingPercentile.bucketSize
      每个桶内的请求数，默认为100，如果在一个延时统计窗口的桶内，数量大于桶size，则将开一个新的桶。
      metrics.healthSnapshot.intervalInMilliseconds
      度量间隔，用户统计错误，成功错误请求的百分比，统计结果将用于是否触发熔断（错误百分比），默认为500ms
  Request Context
    requestCache.enabled
    是否使用请求缓存key，默认true
    requestLog.enabled
    是否开启命令执行，时间日志，默认为true
Collapser Properties
    maxRequestsInBatch
    允许批量处理的数量，默认为Integer.MAX_VALUE
    timerDelayInMilliseconds
    创建批请求后，执行批请求的等待时间，默认为10ms
    requestCache.enabled
    是否开启请求缓存
Thread Pool Properties
配置的属性与线程池的属性基本相同。

    coreSize
    线程池核心线程池大小，默认为10
    maximumSize
    最大线程池，此参数生效，必须allowMaximumSizeToDivergeFromCoreSize为true，默认为10
    maxQueueSize
    阻塞任务队列的大小，配置为正数是LinkedBlockingQueue，-1为SynchronousQueue。注意：此于队列实现不能重新调整大小，参数只用于任务的初始化，针对线程池执行器不能重新初始化的，也不支持重新调整大小。如果克服上线的限制，可以使用queueSizeRejectionThreshold。默认为-1.
    queueSizeRejectionThreshold
    队列拒绝阈值，在未到达队列容量时，任务数量达到阈值，则拒绝执行，默认为5；之所以设置这个属性的原因，队列容量一旦设置，不可修改。如果maxQueueSize == -1，此配置无效。
    keepAliveTimeMinutes
    当前，coresize<maximumSize时，默认为1；空闲线程的释放等待的时间。
    allowMaximumSizeToDivergeFromCoreSize
    此配置允许，maximumSize>coreSize, 默认为false
    metrics.rollingStats.timeInMilliseconds
    线程次metric，滑动窗口，默认为10s
    metrics.rollingStats.numBuckets
    滑动窗口内的桶数量，默认为10。需要注意metrics.rollingStats.timeInMilliseconds % metrics.rollingStats.numBuckets == 0，否则将会抛出异常。



大部分情况下，默认10个线程变现已经很不错，实际上可以更小。如果线程池需要调大，则可以根据以下公式：
（峰值每秒的请求数*健康状况下99%的响应时间+缓冲空间）
一般原则，线程越小越好，因为线程池，承担着负载分流和控制延迟阻塞资源的消耗。


https://blog.bramp.net/post/2015/12/17/the-importance-of-tuning-your-thread-pools/
队列理论，利特尔法则;
![littlelaw.png](/image/Hystrix/littlelaw.png)




![thread-configuration](/image/Hystrix/thread-configuration-1280.png)

10个线程处理99%的请求，在监控健康的情况下，一两个活跃的线程可以服务中位数的请求。
当我们很难准确的配置处理超时时间，可以保护处网络延迟之外的时间影响场景。
如果我们降级掉最后一层网络的timeout的请求，大部分情况下，可以达到中位数的响应时间，如果同时可以在300ms内完成所有请求。

我们合理的timeout设置应该为300ms，也就是99.5%的访问延时，计算方法是，因为判断每次访问延时最多在250ms（TP99如果是200ms的话），再加一次重试时间50ms，就是300ms，感觉也应该足够了

不同的场景和应用，有不同的tradeoffs方案。

如果在真是环境中，如果配置失效，可以动态的调整配置参数。

ThreadPoolExecutor
当前任务提交时，如果线程没有达到核心size，即使有其他空闲线程，则创建一个线程。对于工作线程大于coresize，但小于maximumPoolSize，首先将任务添加到队列，直到任务队列满时，才创建线程；keepAliveTime运行大于coreSize的空闲线程，保活时间。

即当线程池中的线程数大于corePoolSize时，如果一个线程空闲的时间达到keepAliveTime，则会终止，直到线程池中的线程数不超过corePoolSize。
allowCoreThreadTimeOut默认是false，即核心线程不会超时关闭，除非手动关闭线程池才会销毁线程。

SynchronousQueue
每一插入操作，必须对应一个remove操作。
消费者没拿走当前的产品，生产者是不能再给产品的。应该是为了保证消费者和生产者的节奏一致吧，其它的队列实际上有缓存的意思，比如说消费者在高峰期消费不了那么多，那么队列会缓存一部分产品，这样就不至于影响生产者的速度。

LinkedBlockingQueue 可以扩展，但是我们可以使用queueSizeRejectionThreshold来动态控制队列大小。




线程池配置策略:

一般说来，大家认为线程池的大小经验值应该这样设置：（其中N为CPU的个数）
    如果是CPU密集型应用，则线程池大小设置为N+1
    如果是IO密集型应用，则线程池大小设置为2N+1（因为io读数据或者缓存的时候，线程等待，此时如果多开线程，能有效提高cpu利用率）
如果一台服务器上只部署这一个应用并且只有这一个线程池，那么这种估算或许合理，具体还需自行测试验证。
但是，IO优化中，这样的估算公式可能更适合：

最佳线程数目 = （（线程等待时间+线程CPU时间）/线程CPU时间 ）* CPU数目

因为很显然，*线程等待时间所占比例越高，需要越多线程。线程CPU时间所占比例越高，需要越少线程*。

平均每个线程CPU运行时间为0.5s，而线程等待时间（非CPU运行时间，比如IO）为1.5s，CPU核心数为8，那么根据上面这个公式估算得到：((0.5+1.5)/0.5)\*8=32。这个公式进一步转化为：

最佳线程数目 = （线程等待时间与线程CPU时间之比 + 1）* CPU数目

如何来设置

    需要根据几个值来决定
        tasks ：每秒的任务数，假设为500~1000
        taskcost：每个任务花费时间，假设为0.1s
        responsetime：系统允许容忍的最大响应时间，假设为1s
    做几个计算
        corePoolSize = 每秒需要多少个线程处理？
            threadcount = tasks/(1/taskcost) =tasks*taskcout =  (500~1000)\*0.1 = 50~100 个线程。corePoolSize设置应该大于50
            根据8020原则，如果80%的每秒任务数小于800，那么corePoolSize设置为80即可
        queueCapacity = (coreSizePool/taskcost)\*responsetime
            计算可得 queueCapacity = 80/0.1*1 = 80。意思是队列里的线程可以等待1s，超过了的需要新开线程来执行
            切记不能设置为Integer.MAX_VALUE，这样队列会很大，线程数只会保持在corePoolSize大小，当任务陡增时，不能新开线程来执行，响应时间会随之陡增。
        maxPoolSize = (max(tasks)- queueCapacity)/(1/taskcost)
            计算可得 maxPoolSize = (1000-80)/10 = 92
            （最大任务数-队列容量）/每个线程每秒处理能力 = 最大线程数

假设每秒rps为6000, 响应时间为0.2s，则需要线程数如下61000
            6000 * 0.2 = 1200 + buffer = 1300，
            一个服务器内一个线程池给1300个线程有点夸张，我们分，20台机
            1300 / 20 = 65线程/每台。
            如果你一个依赖服务占据的线程数量太多的话，会导致其他的依赖服务对应的线程池里没有资源可以用了。      
当前核心线程数与每秒并发数相同。

*线程队列大小*
acceptable time range / time to complete a task.

Concrete Example : if your client service expects that the task it submits would have to be completed in less than 100 seconds, and knowing that every task takes 1-2 seconds, you should limit the queue to 50-100 tasks because once you have 100 tasks waiting in the queue, you're pretty sure that the next one won't be completed in less than 100 seconds, thus rejecting the task to prevent the service from waiting for nothing.


ArrayBlockingQueue helps prevent resource exhaustion when
used with finite maximumPoolSizes, but can be more difficult to
tune and control.  Queue sizes and maximum pool sizes may be traded
off for each other: Using large queues and small pools minimizes
CPU usage, OS resources, and context-switching overhead, but can
lead to artificially low throughput.  If tasks frequently block (for
example if they are I/O bound), a system may be able to schedule
time for more threads than you otherwise allow. Use of small queues
generally requires larger pool sizes, which keeps CPUs busier but
may encounter unacceptable scheduling overhead, which also
decreases throughput.  



限制线程池队列的大小，可以防止资源被耗尽，但是很对控制和调整。队列大小和最大线程池size两者
可以取一个traded off，使用大队列，小线程池可以最小化CPU的使用，及资源消耗，上线文切换负载，将导致吞吐量降低。
如果任务是IO密集型的，系统也许可以调度比设置的线程更多的线程。一般情况下，使用容量小的队列，需要一个
更大的线程池size，这样可以保证CPU处于忙碌状态，但是会遇到不可接受的调度负载，也会降低吞度量。


当前配置方案，最大队列和最大线程数相同。
核心线程与最大线程数之比为8:10(8:2)
最大队列与核心线程池相同，队列塞满，所有线程处于忙碌状态；
线程也不是越多越好，如果到达最大线程，核心线程是不会销毁的。

错误比例以压测为准：

fallback.isolation.semaphore.maxConcurrentRequests，可以最大线程数*错误比率熔断率为准。



# 挖宝活动
2018-08-28 09:00:00	2018-08-28 12:00:00

```Java
@ResponseBody
   @RequestMapping("crit")
   @HystrixCommand(fallbackMethod = "critFallback",
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
                   @HystrixProperty(name = "fallback.isolation.semaphore.maxConcurrentRequests", value = "1000")    //执行fallback方法的semaphore数量
           })
```
平均RT：20.15ms，最慢1002,1338,489
并发93：93/20=5
(rps):4854147/2/60=40451/20=2000*0.02=40；

超时时间设为：500ms

# 幸运大抽奖
A:2018-05-09 10:00:00	2018-05-16 22:00:00

S:2018-05-09 10:00:00	2018-05-09 22:00:00

```Java
@RequestMapping(value = "", method = {RequestMethod.POST})
    @ResponseBody
    @HystrixCommand(fallbackMethod = "lotteryFailBack",
            threadPoolProperties = {
                    @HystrixProperty(name = "coreSize", value = "10"),
                    @HystrixProperty(name = "maximumSize", value = "30"),
                    @HystrixProperty(name = "maxQueueSize", value = "30"),
                    @HystrixProperty(name = "allowMaximumSizeToDivergeFromCoreSize", value = "true") // 默认false, 如果不设置该参数maximumSize=coreSize
            },
            commandProperties = {
                    @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = "20"),   //（出错百分比阈值，当达到此阈值后，开始短路。默认50%）
                    @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "40"),      // 在统计数据之前，必须在10秒内发出3个请求。  默认是20
                    @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "8000"), //（短路多久以后开始尝试是否恢复，默认5s）
                    @HystrixProperty(name = "execution.timeout.enabled", value = "false"),               //该方法不做超时校验
                    @HystrixProperty(name = "fallback.isolation.semaphore.maxConcurrentRequests", value = "100")    //执行fallback方法的semaphore数量
            })
```
平均：80.65ms, 最慢的4s。            
并发：23/20=2
(rps):324010/2/60=2700/20=140*0.08=11.2

超时时间设为350ms。


# 世界杯竞猜
2018-06-24 06:00:00	2018-06-27 21:00:00
```java
@HystrixCommand(fallbackMethod = "wagerFailBack", threadPoolProperties = {
          @HystrixProperty(name = "coreSize", value = "10"), @HystrixProperty(name = "maximumSize", value = "30"),
          @HystrixProperty(name = "maxQueueSize", value = "30"),
          @HystrixProperty(name = "allowMaximumSizeToDivergeFromCoreSize", value = "true") // 默认false,
                                                                                           // 如果不设置该参数maximumSize=coreSize
  }, commandProperties = { @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = "20"), // （出错百分比阈值，当达到此阈值后，开始短路。默认50%）
          @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "40"), // 在统计数据之前，必须在10秒内发出3个请求。
                                                                                          // 默认是20
          @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "8000"), // （短路多久以后开始尝试是否恢复，默认5s）
          @HystrixProperty(name = "execution.timeout.enabled", value = "false"), // 该方法不做超时校验
          @HystrixProperty(name = "fallback.isolation.semaphore.maxConcurrentRequests", value = "100") // 执行fallback方法的semaphore数量
  })
```

平均：107.54
并发：4
(rps):46586/2/60=388/20=19*0.1=19



并发数，配置调试。

-XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=256m -Xms512m -Xmx2048m -XX:+UseConcMarkSweepGC -XX:+UseParNewGC -XX:+CMSClassUnloadingEnabled -XX:+ExplicitGCInvokesConcurrent -XX:+ExplicitGCInvokesConcurrentAndUnloadsClasses -XX:+HeapDumpOnOutOfMemoryError


###

```java
```



## 总结
