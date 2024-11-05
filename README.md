# Ziplinee

The `ziplinee-foundation` library is part of the Ziplinee CI system documented at https://ziplinee.io.

Please file any issues related to Ziplinee CI at https://github.com/ziplineeci/ziplinee-ci-central/issues

## Ziplinee-foundation

This library provides building blocks for creating

This library has contracts for requests / responses between various components of the Ziplinee CI system.

## Development

To start development run

```bash
git clone git@github.com:ziplineeci/ziplinee-foundation.git
cd ziplinee-foundation
```

Before committing your changes run

```bash
go test ./...
go mod tidy
```

## Usage

To add this module to your golang application run

```bash
go get github.com/ziplineeci/ziplinee-foundation
```

### Initialize logging

```go
import "github.com/ziplineeci/ziplinee-foundation"

foundation.InitLogging(app, version, branch, revision, buildDate)
```

### Initialize Prometheus metrics endpoint

```go
import "github.com/ziplineeci/ziplinee-foundation"

foundation.InitMetrics()
```

### Handle graceful shutdown

```go
import "github.com/ziplineeci/ziplinee-foundation"

gracefulShutdown, waitGroup := foundation.InitGracefulShutdownHandling()

// your core application logic, making use of the waitGroup for critical sections

foundation.HandleGracefulShutdown(gracefulShutdown, waitGroup)
```


### Watch mounted folder for changes

```go
import "github.com/ziplineeci/ziplinee-foundation"

foundation.WatchForFileChanges("/path/to/mounted/secret/or/configmap", func(event fsnotify.Event) {
  // reinitialize parts making use of the mounted data
})
```

### Apply jitter to a number to introduce randomness

Inspired by http://highscalability.com/blog/2012/4/17/youtube-strategy-adding-jitter-isnt-a-bug.html you want to add jitter to a lot of parts of your platform, like cache durations, polling intervals, etc.

```go
import "github.com/ziplineeci/ziplinee-foundation"

sleepTime := foundation.ApplyJitter(30)
time.Sleep(time.Duration(sleepTime) * time.Second)
```

### Retry

In order to retry a function you can use the `Retry` function to which you can pass a retryable function with signature `func() error`:

```go
import "github.com/ziplineeci/ziplinee-foundation"

foundation.Retry(func() error { do something that can fail })
```

Without passing any additional options it will by default try 3 times, with exponential backoff with jitter applied to the interval for any error returned by the retryable function.

In order to override the defaults you can pass them in with the following options:

```go
import "github.com/ziplineeci/ziplinee-foundation"

foundation.Retry(func() error { do something that can fail }, Attempts(5), DelayMillisecond(10), Fixed())
```

The following options can be passed in:

| Option   | Config property | Description |
| -------- | --------------- | ----------- |
| Attempts | Attempts        | Sets the number of attempts the retryable function will be attempted before returning the error |
| DelayMillisecond | DelayMillisecond | Sets the base number of milliseconds between the retries or to base the exponential backoff delay on |
| ExponentialJitterBackoff | DelayType |
| ExponentialBackoff | DelayType |
| Fixed | DelayType |
| AnyError | IsRetryableError |

#### Custom options

You can also override any of the config properties by passing in a custom option with signature `func(*RetryConfig)`, which could look like:

```go
import "github.com/ziplineeci/ziplinee-foundation"

isRetryableErrorCustomOption := func(c *foundation.RetryConfig) {
  c.IsRetryableError = func(err error) bool {
    switch e := err.(type) {
      case *googleapi.Error:
        return e.Code == 429 || (e.Code >= 500 && e.Code < 600)
      default:
        return false
    }
  }
}

foundation.Retry(func() error { do something that can fail }, isRetryableErrorCustomOption)
```

### Limit concurrency with a semaphore

To run code in a loop concurrently with a maximum of simultanuous running goroutines do the following:

```go
import "github.com/ziplineeci/ziplinee-foundation"

// limit concurrency using a semaphore
maxConcurrency := 5
semaphore := foundation.NewSemaphore(maxConcurrency)

for _, i := range items {
  // try to acquire a lock, which only succeeds if there's less than maxConcurrency active goroutines
  semaphore.Acquire()

  go func(i Item) {
    // release the lock when done with the slow task
    defer semaphore.Release()

    // do some slow work
  }(i)
}

// wait until all concurrent goroutines are done
semaphore.Wait()
```

If you want to run Acquire within a `select` statement do so as follows:

```go
import "github.com/ziplineeci/ziplinee-foundation"

// limit concurrency using a semaphore
maxConcurrency := 5
semaphore := foundation.NewSemaphore(maxConcurrency)

for _, i := range items {
  select {
  case semaphore.GetAcquireChannel() <- struct{}{}:
    // try to acquire a lock, which only succeeds if there's less than maxConcurrency active goroutines
    semaphore.Acquire()

    go func(i Item) {
      // release the lock when done with the slow task
      defer semaphore.Release()

      // do some slow work
    }(i)

  case <-time.After(1 * time.Second):
    // running the goroutines took to long, exit instead
    return
  }
}

// wait until all concurrent goroutines are done
semaphore.Wait()
```