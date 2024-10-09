---
title: "Rate Limiter"
tags: [algorithms, system design]
published: 2024-10-10
---

I did some research into rate limiter algorithms so here I present what I learnt.

Minimally, a rate limiter has to be defined with these 2 parameters:
- Limit of permitted requests
- Time window for that limit of permitted requests

Optionally, some rate limit algorithms allow specifying a burst limit where requests are permitted without rate limit as a form of buffer before the actual rate limit kicks in.

If you are looking to decide which algorithm to use, here is a summary in my opinion:
- Leaky bucket as meter: basic rate limiter, most popular choice, allows for bursts, same as token bucket
- Leaky bucket as queue: allows queuing/delaying of request
- Generic cell rate algorithm: same as leaky bucket as meter plus retry timing information
- Sliding window log: for low frequency limiter in units of minutes and above
- Sliding window counter: performant in exchange for slight inaccuracy
- Fixed window counter: simplest to understand but broken in practice, don't use

## Token bucket

In token bucket, for a request to be permitted, it would consume a token in the bucket. The bucket is constantly filled with the specified limit of token on every window. Below is a naive implementation.

```go
const limit = 5 // aka refill rate, should be < capacity
const window = time.Second
const capacity = 10 // max tokens in bucket, ie the burst limit
tokens := capacity  // how many tokens are in bucket
go func() {
  for {
    tokens = int(math.Min(float64(capacity), float64(tokens+limit)))
    time.Sleep(window)
  }
}()
isPermitted := func() bool {
  if tokens > 0 {
    tokens--
    return true
  }
  return false
}
```
[Try it out](https://go.dev/play/p/Szts2LW_CDa)

This implementation has the issue of refilling in bursts at every window, wrongly blocking requests when the bucket should have been partially refilled.An alternative implementation without refilling asynchronously is to count how many tokens would have been refilled since the last time the rate limiter was requested.

```go
const limit = 5 // aka refill rate, should be < capacity
const window = time.Second
const capacity = 10         // max tokens in bucket, ie the burst limit
tokens := float64(capacity) // how many tokens are in bucket
lastChecked := time.Now()   // when isPermitted was last called
isPermitted := func() bool {
  sinceLastChecked := time.Since(lastChecked)
  tokens = math.Min(capacity, tokens+limit*float64(sinceLastChecked)/float64(window))
  lastChecked = time.Now()
  if tokens >= 1 {
    tokens--
    return true
  }
  return false
}
```
[Try it out](https://go.dev/play/p/aWBTb909EM4)

Here we use float to count the tokens since we do floating point calculation on `sinceLastChecked`. Alternatively, we can use a finer time unit like ms or ns and scale all the variables for that time unit and use integer. This fixes the burst refill issue.

## Leaky bucket as a meter

This algorithm is basically the same concept as token bucket but instead of removing tokens when requesting, we add tokens into the bucket. They are effectively the same algorithm implemented differently. As described on [wikipedia](https://en.wikipedia.org/wiki/Leaky_bucket#Comparison_with_the_token_bucket_algorithm):

> So, is an implementation that adds tokens for a conforming packet and removes them at a fixed rate an implementation of the leaky bucket or of the token bucket? Similarly, which algorithm is used in an implementation that removes water for a conforming packet and adds water at a fixed rate? In fact both are effectively the same, i.e. implementations of both the leaky bucket and token bucket, as these are the same basic algorithm described differently.

```go
const limit = 5 // aka leak rate, should be < capacity
const window = time.Second
const capacity = 10       // max tokens in bucket, ie the burst limit
tokens := 0.0             // how many tokens are in bucket
lastChecked := time.Now() // when isPermitted was last called
isPermitted := func() bool {
  sinceLastChecked := time.Since(lastChecked)
  tokens = math.Max(0, tokens-limit*float64(sinceLastChecked)/float64(window))
  lastChecked = time.Now()
  if tokens+1 <= capacity {
    tokens++
    return true
  }
  return false
}
```
[Try it out](https://go.dev/play/p/Zh6nHhcrGVY)

This algorithm seems to be the most popular choice among the other rate limiter algorithms due to its simplicity and accuracy.

It is used by [Nginx](https://blog.nginx.org/blog/rate-limiting-nginx) for the [ngx_http_limit_req_module](http://nginx.org/en/docs/http/ngx_http_limit_req_module.html). Out of curiosity, I found its [source code](https://github.com/nginx/nginx/blob/e24f7cc/src/http/modules/ngx_http_limit_req_module.c#L445-L474) where this algorithm is implemented.

### Generic Cell Rate Algorithm (GCRA)

GCRA is effectively another alternative implementation of the leaky bucket as a meter. Logically, this algorithm works the same as the leaky bucket as a meter but it calculates the timing instead of counting tokens. This algorithm originated as a scheduling algorithm designed for the now defunct Asynchronous Transfer Mode (ATM) network protocol. It tracks the "theoretical arrival time" aka TAT for when the next request would be permitted if all requests are timed equally apart at the emission interval (ie the duration to leak 1 request). Think of TAT as the time at which the bucket would be empty. One benefit of this algorithm is its ability to return additional timing information such as the duration until the next retry would pass and duration until rate limiter is reset (ie until the TAT).

```go
const limit = 5
const window = time.Second
const burst = 10
var tat time.Time
emissionInterval := window / time.Duration(limit)
burstOffset := time.Duration(burst) * emissionInterval // aka delay variation tolerance
type result struct {
  isPermitted  bool
  remaining  int           // how many tokens can be accepted before bucket is full
  retryAfter time.Duration // how long to wait until the next retry will pass
  resetAfter time.Duration // how long to wait until the bucket is empty
}
isPermitted := func() result {
  now := time.Now()
  var newTat time.Time
  if now.After(tat) {
    newTat = now.Add(emissionInterval)
  } else {
    newTat = tat.Add(emissionInterval)
  }
  allowAt := newTat.Add(-burstOffset)
  diff := now.Sub(allowAt)
  // remaining = (now - newTat + burstOffset) / emissionInterval
  remaining := int(math.Floor(float64(diff) / float64(emissionInterval)))
  if remaining >= 0 {
    tat = newTat
    return result{
      isPermitted: true,
      remaining:   remaining,
      retryAfter:  -1,
      resetAfter:  newTat.Sub(now),
    }
  }
  return result{
    isPermitted: false,
    remaining:   0,
    retryAfter:  time.Duration(-diff),
    resetAfter:  tat.Sub(now),
  }
}
```
[Try it out](https://go.dev/play/p/tzecPyMY5tK)

I discovered this algorithm from the [script in redis_rate](https://github.com/go-redis/redis_rate/blob/v10/lua.go) library which was copied from the [script in redis-gcra](https://github.com/rwz/redis-gcra/blob/master/vendor/perform_gcra_ratelimit.lua) library. redis-gcra is [inspired](https://github.com/rwz/redis-gcra/tree/master?tab=readme-ov-file#inspiration) by a [blog post](https://brandur.org/rate-limiting#gcra) by Brandur whose colleague also implemented it in [throttled](https://github.com/throttled/throttled/blob/master/rate.go#L183) library.

## Leaky bucket as a queue

The other commonly known implementation of leaky bucket is to store a queue of requests as the bucket instead of counting tokens. These requests are then processed (ie leaked) asynchronously. Since this is an asynchronous implementation, the function will return before the request is actually executed. This also means that the requests are always processed in fixed intervals and not allowed to be bursty unlike the token bucket or the leaky bucket as a meter.

```go
const limit = 5 // aka leak rate
const window = time.Second
const capacity = 10                  // max requests in queue
requestQueue := make([]time.Time, 0) // we use timestamp to illustrate the request
ticker := time.NewTicker(window / limit)
// leak requests asynchronously
go func() {
  for {
    <-ticker.C
    if len(requestQueue) > 0 {
      // dequeue request
      request := requestQueue[0]
      requestQueue = requestQueue[1:]
      // process request without blocking the ticker
      go func() {
        fmt.Printf("processing request from %v\n", request)
        time.Sleep(1 * time.Second)
      }()
    }
  }
}()
isPermitted := func() bool {
  if len(requestQueue) < capacity {
    request := time.Now()
    requestQueue = append(requestQueue, request)
    return true
  }
  return false
}
```
[Try it out](https://go.dev/play/p/52Zr1DsR2_x)

## Fixed window counter

This is the simplest algorithm to understand and implement. When asked to write a rate limiter without prior knowledge, I would have written it this way. Since we always describe a rate limit in terms of the number of allowed requests per unit time, the naive implementation idea would be to count the number of requests at every interval, block requests when the count will exceed the limit, and reset the count every interval.

```go
const limit = 5
const window = time.Second
currCount := 0
go func() {
  for {
    time.Sleep(window)
    currCount = 0
  }
}()
isPermitted := func() bool {
  if currCount < limit {
    currCount += 1
    return true
  }
  return false
}
```
[Try it out](https://go.dev/play/p/GoQPcpRju93)

However, this is a broken implementation since it will exceed the specified rate limit at the boundaries between intervals. Using the above implementation for 5 requests per second, let's say 5 requests arrive at time t=0.9s and they are all permitted. Then another 5 requests arrive at time t=1s, they too will be incorrectly permitted since the count has reset at time t=1s. This means that we have allowed a burst of 10 requests within a span of a second, double of what we defined the rate limiter to be.

## Sliding window log

This algorithm tracks the permitted requests' timestamps and uses them to decide whether the next request should be permitted. When receiving an incoming request, count the past permitted requests' timestamps within the window until now, permit only if the count is below the limit. I find this algorithm quite similar to the leaky bucket as meter but we log instead of count the requests. However, unlike the token/leaky bucket, this does not continuously refill/leak tokens and instead refill in bursts every window causing starvation for requests arriving late in every window.

```go
const limit = 5
const window = time.Second
requestLog := make([]time.Time, 0)
isPermitted := func() bool {
  now := time.Now()
  // trim requests older than window
  i := 0
  for i < len(requestLog) && now.Sub(requestLog[i]) >= window {
    i++
  }
  requestLog = requestLog[i:]

  if len(requestLog) < limit {
    requestLog = append(requestLog, now)
    return true
  }
  return false
}
```
[Try it out](https://go.dev/play/p/qzpoRF2ZyKt)

This algorithm is better suited for rate limit with very low frequency requirement such as in units of minutes or days. As such, the `requestLog` is usually persisted externally in database/cache, so filtering them may not be necessary and we can just query for the count of requests within the window.

```go
const limit = 5
const window = time.Second
requestLog := make([]time.Time, 0) // treat this as stored externally
isPermitted := func() bool {
  now := time.Now()
  windowStart := now.Add(-window)
  // query from storage for request count within window
  count := len(requestLog)
  for i := len(requestLog) - 1; i >= 0; i-- {
    if requestLog[i].Before(windowStart) {
      count = len(requestLog) - 1 - i
      break
    }
  }

  if count < limit {
    // save to storage
    requestLog = append(requestLog, now)
    return true
  }
  return false
}
```
[Try it out](https://go.dev/play/p/y5l1J6ez4nE)

Grab implemented this to [limit their marketing communications](https://engineering.grab.com/frequency-capping).

## Sliding window counter

Sliding window counter approximates the current rate by calculating the weighted sum of requests in the previous fixed window and the current window. This works under the assumption that the requests are distributed evenly. In exchange for the lower accuracy, this algorithm seems to be the most performant one when implemented for high throughput distributed load balancer.

```go
const limit = 5
const window = time.Second
prevCount := 0
currCount := 0
var currWindow int64 = 0
isPermitted := func() bool {
  nowNano := time.Duration(time.Now().UnixNano())
  windowIndex := int64(nowNano / window)
  if windowIndex != currWindow {
    prevCount = currCount
    currCount = 0
    currWindow = windowIndex
  }
  prevWeight := 1 - float64((nowNano%window))/float64(window)
  rate := float64(prevCount)*prevWeight + float64(currCount)
  if rate < limit {
    currCount++
    return true
  }
  return false
}
```
[Try it out](https://go.dev/play/p/HF0R47ZUdqN)

Alternatively, we can use a cache system like redis to cache the request count using the window as key with TTL of 2 time windows. This will minimize the operations required when checking the rate limit.

```go
const limit = 5
const window = time.Second
requestCounter := make(map[int64]int64) // map of window to request count, to illustrate the cache
isPermitted := func() bool {
  nowNano := time.Duration(time.Now().UnixNano())
  windowIndex := int64(nowNano / window)
  prevWeight := 1 - float64((nowNano%window))/float64(window)
  rate := float64(requestCounter[windowIndex-1])*prevWeight + float64(requestCounter[windowIndex])
  if rate < limit {
    requestCounter[windowIndex]++
    return true
  }
  return false
}
```
[Try it out](https://go.dev/play/p/Fa81OhNag01)

Calling an external cache will take milliseconds, so for very high throughput rate limiter, say in the scale of millions of qps, we can locally cache the increments in memory then periodically updates and syncs with redis to achieve eventual consistency. This would however, sacrifice more accuracy and you will need to ensure the system clocks are synchronized within very tight tolerance. For such high throughput, I reckon it's simpler and likely as effective to just setup rate limiting per load balancer instance to divide the throughput.

From my testing, the actual observed rate seems to be < 1qps above the configured limit, eg I got 5.3qps when I tested sending 10qps through the above code. This is due to the `rate < limit` check. If we want to enforce to never reach above the configured limit, we can do `rate+1 <= limit` instead which would reduce the actual observed rate to be below the configured limit by < 1qps.

According to [Cloudflare's analysis over 400 million requests](https://blog.cloudflare.com/counting-things-a-lot-of-different-things/), they observed that this algorithm resulted in only "0.003% of requests have been wrongly allowed or rate limited" and "None of the mitigated sources was below the threshold (false positives)". I suspect that their limit check is likely the relaxed form I mentioned.

[Figma](https://www.figma.com/blog/an-alternative-approach-to-rate-limiting/) and [Kong](https://konghq.com/blog/engineering/how-to-design-a-scalable-rate-limiting-algorithm) implements this as well.

## Considerations when implementing

### Rate limiting per client

Sometimes we want to rate limit by client instead of a system wide rate limiter to ensure fairness among clients. This can easily be solved by storing the rate limiter state per client in a key value store with the client identifier as the key.

### Distributed rate limiting

If the service that requires the rate limiting is deployed in multiple instances, we can either rate limit within the service instance individually and tolerate the inaccuracy, or we can have an external shared rate limiter such as in redis. Using a single shared rate limiter would also prevent the clock drift issue since all rate limiter algorithms require reading the system time. On the other hand, we would need to prevent race condition by doing atomic updates or locking, as well as consider the network latency and replication lag if the shared store runs in replicas as well.

### Variable request cost

Historically, the rate limiter concept stems from traffic shaping/policing in networking. We can think of each packet incurring a cost, number of bytes, on a bandwidth limit. All my example code snippets above assumes all requests incur the same cost. All the counter based algorithms should be trivial to convert to handle variable request cost. The rest would probably need to add a counter to handle this.

### Multiple layer rate limit

In some cases, there may be requirement to rate limit by different window sizes, eg 60 queries per minute but allow up to 10 queries per second. For this, we would need to modify the algorithms such that the state updates only after all rate limiters permit the request.

## Main references

https://medium.com/@patrikkaura/the-fundamentals-of-rate-limiting-how-it-works-and-why-you-need-it-fd86d39e358d  
https://blog.algomaster.io/p/rate-limiting-algorithms-explained-with-code  
https://www.thealgorists.com/SystemDesign/RateLimiter  
https://en.wikipedia.org/wiki/Leaky_bucket  
https://blog.nginx.org/blog/rate-limiting-nginx  
https://github.com/nginx/nginx/blob/e24f7cc/src/http/modules/ngx_http_limit_req_module.c#L445-L474  
https://github.com/rwz/redis-gcra/blob/master/vendor/perform_gcra_ratelimit.lua  
https://brandur.org/rate-limiting#gcra  
https://en.wikipedia.org/wiki/Generic_cell_rate_algorithm  
https://engineering.grab.com/frequency-capping  
https://blog.cloudflare.com/counting-things-a-lot-of-different-things/

