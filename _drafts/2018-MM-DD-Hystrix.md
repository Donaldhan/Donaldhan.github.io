---
layout: page
title: my blog
subtitle: sub title
date: 2018-11-04 15:17:19
author: donaldhan
catalog: true
category: spring-framework
categories:
    - spring-framework
tags:
    - spring-context
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

信号量配置的使用，在什么情况下触发？

Semaphores are used instead of threads for dependency executions known to not perform network calls (such as those only doing in-memory cache lookups) since the overhead of a separate thread is too high for these types of operations.



We also use semaphores to protect against non-trusted fallbacks. Each DependencyCommand is able to define a fallback function (discussed more below) which is performed on the calling user thread and should not perform network calls. Instead of trusting that all implementations will correctly abide to this contract, it too is protected by a semaphore so that if an implementation is done that involves a network call and becomes latent, the fallback itself won’t be able to take down the entire app as it will be limited in how many threads it will be able to block.



The tripping of circuits kicks in when a DependencyCommand has passed a certain threshold of error (such as 50% error rate in a 10 second period) and will then reject all requests until health checks succeed.


This is used primarily to release the pressure on underlying systems (i.e. shed load) when they are having issues and reduce the user request latency by failing fast (or returning a fallback) when we know it is likely to fail instead of making every user request wait for the timeout to occur.

How do we respond to a user request when failure occurs?
In each of the options described above a timeout, thread-pool or semaphore rejection, or short-circuit will result in a request not retrieving the optimal response for our customers.

An immediate failure (“fail fast”) throws an exception which causes the app to shed load until the dependency returns to health. This is preferable to requests “piling up” as it keeps Tomcat request threads available to serve requests from healthy dependencies and enables rapid recovery once failed dependencies recover.

However, there are often several preferable options for providing responses in a “fallback mode” to reduce impact of failure on users. Regardless of what causes a failure and how it is intercepted (timeout, rejection, short-circuited etc) the request will always pass through the fallback logic (step 8 in flow chart above) before returning to the user to give a DependencyCommand the opportunity to do something other than “fail fast”.



```java
```


###

```java
```


###

```java
```


## 总结
