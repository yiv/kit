{
    "template": "../inc/page.template.source",
    "subtitle": "The stringsvc tutorial",
    "toc": true
}
---
# 第一原则

我们来创建一个最小化的 Go kit 服务。目前，我们仅使用一个 `main.go` 文件来实现。

## 你的业务逻辑

你的服务始于你的业务逻辑。在 Go kit 中，我们将一个服务建模为一个**interface**。

```go
// StringService provides operations on strings.
import "context"

type StringService interface {
	Uppercase(string) (string, error)
	Count(string) int
}
```

这个 interface 会有一个如下的实施实例。

```go
import (
	"context"
	"errors"
	"strings"
)

type stringService struct{}

func (stringService) Uppercase(s string) (string, error) {
	if s == "" {
		return "", ErrEmpty
	}
	return strings.ToUpper(s), nil
}

func (stringService) Count(s string) int {
	return len(s)
}

// ErrEmpty is returned when input string is empty
var ErrEmpty = errors.New("Empty string")
```

## 请求和回复

在 Go kit 中，主要的消息通信模式是 RPC。
因此， 我们的 interface 的每个方法都会被建模为一个远程过程调用（RPC）。
我们为每一个方法定义 **request and response** 结构，用来相应地捕捉所有的输入和输出参数。

```go
type uppercaseRequest struct {
	S string `json:"s"`
}

type uppercaseResponse struct {
	V   string `json:"v"`
	Err string `json:"err,omitempty"` // errors don't JSON-marshal, so we use a string
}

type countRequest struct {
	S string `json:"s"`
}

type countResponse struct {
	V int `json:"v"`
}
```

## Endpoints 端点
Go kit 通过抽像的概念 **endpoint** 端点来提供它的功能。

一个 endpoint 端点的定义如下(you don't have to put it anywhere in the code, it is provided by `go-kit`)：

```go
type Endpoint func(ctx context.Context, request interface{}) (response interface{}, err error)
```

它代表一个单独的 RPC。即我们服务 interface 一个方法。
我们会写一个简单的适配函数，将服务的每个方法转化成一个 endpoint 端点。每个适配函数以 StringService 作为输入参数，以 endpoint 作为返回参数，对应服务的方法。


```go
import (
	"context"
	"github.com/go-kit/kit/endpoint"
)

func makeUppercaseEndpoint(svc StringService) endpoint.Endpoint {
	return func(_ context.Context, request interface{}) (interface{}, error) {
		req := request.(uppercaseRequest)
		v, err := svc.Uppercase(req.S)
		if err != nil {
			return uppercaseResponse{v, err.Error()}, nil
		}
		return uppercaseResponse{v, ""}, nil
	}
}

func makeCountEndpoint(svc StringService) endpoint.Endpoint {
	return func(_ context.Context, request interface{}) (interface{}, error) {
		req := request.(countRequest)
		v := svc.Count(req.S)
		return countResponse{v}, nil
	}
}
```

## Transports 传输层

现在我们需要向外公开我们的服务，以便被外部调用。你们公司可能对于服务间如何通信已经有了自己的选项。
或是 Thrift，或是基于 HTTP 传输的自定义 JSON 协议。
Go kit 支持多种 **transports** 协议的开箱即用。

对于这个最小化的示例服务，让我们使用基于 HTTP 传输的 JSON。
Go kit 在 transport/http 包里提供了一个辅助 struct。



```go
import (
	"context"
	"encoding/json"
	"log"
	"net/http"

	httptransport "github.com/go-kit/kit/transport/http"
)

func main() {
	svc := stringService{}

	uppercaseHandler := httptransport.NewServer(
		makeUppercaseEndpoint(svc),
		decodeUppercaseRequest,
		encodeResponse,
	)

	countHandler := httptransport.NewServer(
		makeCountEndpoint(svc),
		decodeCountRequest,
		encodeResponse,
	)

	http.Handle("/uppercase", uppercaseHandler)
	http.Handle("/count", countHandler)
	log.Fatal(http.ListenAndServe(":8080", nil))
}

func decodeUppercaseRequest(_ context.Context, r *http.Request) (interface{}, error) {
	var request uppercaseRequest
	if err := json.NewDecoder(r.Body).Decode(&request); err != nil {
		return nil, err
	}
	return request, nil
}

func decodeCountRequest(_ context.Context, r *http.Request) (interface{}, error) {
	var request countRequest
	if err := json.NewDecoder(r.Body).Decode(&request); err != nil {
		return nil, err
	}
	return request, nil
}

func encodeResponse(_ context.Context, w http.ResponseWriter, response interface{}) error {
	return json.NewEncoder(w).Encode(response)
}
```

## stringsvc1

目前为止完整的服务是 [stringsvc1](https://github.com/go-kit/kit/blob/master/examples/stringsvc1).

```
$ go get github.com/go-kit/kit/examples/stringsvc1
$ stringsvc1
```

```
$ curl -XPOST -d'{"s":"hello, world"}' localhost:8080/uppercase
{"v":"HELLO, WORLD","err":null}
$ curl -XPOST -d'{"s":"hello, world"}' localhost:8080/count
{"v":12}
```

# Middlewares 中间件

没有彻底的日志记录和度量，任何服务都不能被视为生产就绪。

## 关注点分离

随着服务 endpoints 端点数量的增长， 将调用序列的每一层分成单独的文件，可以更轻松地阅读和理解 go-kit 工程代码。
我们的第一个示例 [stringsvc1](https://github.com/go-kit/kit/blob/master/examples/stringsvc1) 在一个 main 文件中包含了这些所有层。
在我们添加更多复杂性之前，让我们将代码分成以下文件，并将所有剩余代码留在 main.go 中。 

将你的 **services** 放一个 service.go 文件中，并包含如下的函数和类型。


```
type StringService
type stringService
var ErrEmpty
```

Place your **transports** into a `transport.go` file with the following functions and types.

```
func makeUppercaseEndpoint
func makeCountEndpoint
func decodeUppercaseRequest
func decodeCountRequest
func encodeResponse
type uppercaseRequest
type uppercaseResponse
type countRequest
type countResponse
```

## Transport logging 传输层日志记录

任何需要记录的组件都应该像把数据连接当作依赖项那样，同样也应将 logger 视为依赖项。
因此，我们在我们的  `func main` 中构建我们的 logger，并将其传递给需要它的组件。
我们一定不要使用全局范围的 logger。

我们可以将 logger 直接传递给我们的 stringService 实现，但是还有一个更好的方法。

让我们使用 **middleware** ，也称为装饰器。

中间件是一个以 endpoint 作为输入参数，同时以 endpoint 作为输出参数的函数。

```go
type Middleware func(Endpoint) Endpoint
```

> Note, that the Middleware type is provided for you by go-kit.

In between, it can do anything.
Below you can see how a basic logging middleware could be implemented (you don't need to copy/paste this code anywhere):

```go
func loggingMiddleware(logger log.Logger) Middleware {
	return func(next endpoint.Endpoint) endpoint.Endpoint {
		return func(_ context.Context, request interface{}) (interface{}, error) {
			logger.Log("msg", "calling endpoint")
			defer logger.Log("msg", "called endpoint")
			return next(request)
		}
	}
}
```

Use the [go-kit log](https://gokit.io/faq/#logging-mdash-why-is-package-log-so-different) package and remove the standard libraries [log](https://golang.org/pkg/log/).
You will need to remove `log.Fatal` from the bottom of the `main.go` file.

```go
import (
 "github.com/go-kit/kit/log"
)
```

And wire it into each of our handlers.
Note that the next code section will *not* compile until you follow the **Application Logging** section, which defines loggingMiddleware.

```go
logger := log.NewLogfmtLogger(os.Stderr)

svc := stringService{}

var uppercase endpoint.Endpoint
uppercase = makeUppercaseEndpoint(svc)
uppercase = loggingMiddleware(log.With(logger, "method", "uppercase"))(uppercase)

var count endpoint.Endpoint
count = makeCountEndpoint(svc)
count = loggingMiddleware(log.With(logger, "method", "count"))(count)

uppercaseHandler := httptransport.NewServer(
	// ...
	uppercase,
	// ...
)

countHandler := httptransport.NewServer(
	// ...
	count,
	// ...
)
```

It turns out that this technique is useful for a lot more than just logging.
Many Go kit components are implemented as endpoint middlewares.

## Application logging

But what if we want to log in our application domain, like the parameters that are passed in?
It turns out that we can define a middleware for our service, and get the same nice and composable effects.
Since our StringService is defined as an interface, we just need to make a new type
 which wraps an existing StringService, and performs the extra logging duties.

```go
type loggingMiddleware struct {
	logger log.Logger
	next   StringService
}

func (mw loggingMiddleware) Uppercase(s string) (output string, err error) {
	defer func(begin time.Time) {
		mw.logger.Log(
			"method", "uppercase",
			"input", s,
			"output", output,
			"err", err,
			"took", time.Since(begin),
		)
	}(time.Now())

	output, err = mw.next.Uppercase(s)
	return
}

func (mw loggingMiddleware) Count(s string) (n int) {
	defer func(begin time.Time) {
		mw.logger.Log(
			"method", "count",
			"input", s,
			"n", n,
			"took", time.Since(begin),
		)
	}(time.Now())

	n = mw.next.Count(s)
	return
}
```

And wire it in.

```go
import (
	"os"

	"github.com/go-kit/kit/log"
	httptransport "github.com/go-kit/kit/transport/http"
)

func main() {
	logger := log.NewLogfmtLogger(os.Stderr)

	var svc StringService
	svc = stringService{}
	svc = loggingMiddleware{logger, svc}

	// ...

	uppercaseHandler := httptransport.NewServer(
		// ...
		makeUppercaseEndpoint(svc),
		// ...
	)

	countHandler := httptransport.NewServer(
		// ...
		makeCountEndpoint(svc),
		// ...
	)
}
```

Use endpoint middlewares for transport-domain concerns, like circuit breaking and rate limiting.
Use service middlewares for business-domain concerns, like logging and instrumentation.
Speaking of instrumentation...

## Application instrumentation

In Go kit, instrumentation means using **package metrics** to record statistics about your service's runtime behavior.
Counting the number of jobs processed,
 recording the duration of requests after they've finished,
 and tracking the number of in-flight operations would all be considered instrumentation.

We can use the same middleware pattern that we used for logging.

```go
type instrumentingMiddleware struct {
	requestCount   metrics.Counter
	requestLatency metrics.Histogram
	countResult    metrics.Histogram
	next           StringService
}

func (mw instrumentingMiddleware) Uppercase(s string) (output string, err error) {
	defer func(begin time.Time) {
		lvs := []string{"method", "uppercase", "error", fmt.Sprint(err != nil)}
		mw.requestCount.With(lvs...).Add(1)
		mw.requestLatency.With(lvs...).Observe(time.Since(begin).Seconds())
	}(time.Now())

	output, err = mw.next.Uppercase(s)
	return
}

func (mw instrumentingMiddleware) Count(s string) (n int) {
	defer func(begin time.Time) {
		lvs := []string{"method", "count", "error", "false"}
		mw.requestCount.With(lvs...).Add(1)
		mw.requestLatency.With(lvs...).Observe(time.Since(begin).Seconds())
		mw.countResult.Observe(float64(n))
	}(time.Now())

	n = mw.next.Count(s)
	return
}
```

And wire it into our service.

```go
import (
	stdprometheus "github.com/prometheus/client_golang/prometheus"
	kitprometheus "github.com/go-kit/kit/metrics/prometheus"
	"github.com/go-kit/kit/metrics"
)

func main() {
	logger := log.NewLogfmtLogger(os.Stderr)

	fieldKeys := []string{"method", "error"}
	requestCount := kitprometheus.NewCounterFrom(stdprometheus.CounterOpts{
		Namespace: "my_group",
		Subsystem: "string_service",
		Name:      "request_count",
		Help:      "Number of requests received.",
	}, fieldKeys)
	requestLatency := kitprometheus.NewSummaryFrom(stdprometheus.SummaryOpts{
		Namespace: "my_group",
		Subsystem: "string_service",
		Name:      "request_latency_microseconds",
		Help:      "Total duration of requests in microseconds.",
	}, fieldKeys)
	countResult := kitprometheus.NewSummaryFrom(stdprometheus.SummaryOpts{
		Namespace: "my_group",
		Subsystem: "string_service",
		Name:      "count_result",
		Help:      "The result of each count method.",
	}, []string{}) // no fields here

	var svc StringService
	svc = stringService{}
	svc = loggingMiddleware{logger, svc}
	svc = instrumentingMiddleware{requestCount, requestLatency, countResult, svc}

	uppercaseHandler := httptransport.NewServer(
		makeUppercaseEndpoint(svc),
		decodeUppercaseRequest,
		encodeResponse,
	)

	countHandler := httptransport.NewServer(
		makeCountEndpoint(svc),
		decodeCountRequest,
		encodeResponse,
	)

	http.Handle("/uppercase", uppercaseHandler)
	http.Handle("/count", countHandler)
	http.Handle("/metrics", promhttp.Handler())
	logger.Log("msg", "HTTP", "addr", ":8080")
	logger.Log("err", http.ListenAndServe(":8080", nil))
}
```

## stringsvc2

The complete service so far is [stringsvc2](https://github.com/go-kit/kit/blob/master/examples/stringsvc2).

```
$ go get github.com/go-kit/kit/examples/stringsvc2
$ stringsvc2
msg=HTTP addr=:8080
```

```
$ curl -XPOST -d'{"s":"hello, world"}' localhost:8080/uppercase
{"v":"HELLO, WORLD","err":null}
$ curl -XPOST -d'{"s":"hello, world"}' localhost:8080/count
{"v":12}
```

```
method=uppercase input="hello, world" output="HELLO, WORLD" err=null took=2.455µs
method=count input="hello, world" n=12 took=743ns
```

# Calling other services

It's rare that a service exists in a vacuum.
Often, you need to call other services.
**This is where Go kit shines**.
We provide transport middlewares to solve many of the problems that come up.

Let's say that we want to have our string service call out to a _different_ string service
 to satisfy the Uppercase method.
In effect, proxying the request to another service.
Let's implement the proxying middleware as a ServiceMiddleware, same as a logging or instrumenting middleware.

```go
// proxymw implements StringService, forwarding Uppercase requests to the
// provided endpoint, and serving all other (i.e. Count) requests via the
// next StringService.
type proxymw struct {
	next      StringService     // Serve most requests via this service...
	uppercase endpoint.Endpoint // ...except Uppercase, which gets served by this endpoint
}
```

## Client-side endpoints

We've got exactly the same endpoint we already know about, but we'll use it to invoke, rather than serve, a request.
When used this way, we call it a _client_ endpoint.
And to invoke the client endpoint, we just do some simple conversions.

```go
func (mw proxymw) Uppercase(s string) (string, error) {
	response, err := mw.uppercase(uppercaseRequest{S: s})
	if err != nil {
		return "", err
	}
	resp := response.(uppercaseResponse)
	if resp.Err != "" {
		return resp.V, errors.New(resp.Err)
	}
	return resp.V, nil
}
```

Now, to construct one of these proxying middlewares, we convert a proxy URL string to an endpoint.
If we assume JSON over HTTP, we can use a helper in the transport/http package.

```go
import (
	httptransport "github.com/go-kit/kit/transport/http"
)

func proxyingMiddleware(proxyURL string) ServiceMiddleware {
	return func(next StringService) StringService {
		return proxymw{next, makeUppercaseProxy(proxyURL)}
	}
}

func makeUppercaseProxy(proxyURL string) endpoint.Endpoint {
	return httptransport.NewClient(
		"GET",
		mustParseURL(proxyURL),
		encodeUppercaseRequest,
		decodeUppercaseResponse,
	).Endpoint()
}
```

## Service discovery and load balancing

That's fine if we only have a single remote service.
But in reality, we'll probably have many service instances available to us.
We want to discover them through some service discovery mechanism, and spread our load across all of them.
And if any of those instances start to behave badly, we want to deal with that, without affecting our own service's reliability.

Go kit offers adapters to different service discovery systems, to get up-to-date sets of instances, exposed as individual endpoints.
Those adapters are called subscribers.

```go
type Subscriber interface {
	Endpoints() ([]endpoint.Endpoint, error)
}
```

Internally, subscribers use a provided factory function to convert each discovered instance string (typically host:port) to a usable endpoint.

```go
type Factory func(instance string) (endpoint.Endpoint, error)
```

So far, our factory function, makeUppercaseProxy, just calls the URL directly.
But it's important to put some safety middleware, like circuit breakers and rate limiters, into your factory, too.

```go
var e endpoint.Endpoint
e = makeUppercaseProxy(instance)
e = circuitbreaker.Gobreaker(gobreaker.NewCircuitBreaker(gobreaker.Settings{}))(e)
e = kitratelimit.NewTokenBucketLimiter(jujuratelimit.NewBucketWithRate(float64(maxQPS), int64(maxQPS)))(e)
```

Now that we've got a set of endpoints, we need to choose one.
Load balancers wrap subscribers, and select one endpoint from many.
Go kit provides a couple of basic load balancers, and it's easy to write your own if you want more advanced heuristics.

```go
type Balancer interface {
	Endpoint() (endpoint.Endpoint, error)
}
```

Now we have the ability to choose endpoints according to some heuristic.
We can use that to provide a single, logical, robust endpoint to consumers.
A retry strategy wraps a load balancer, and returns a usable endpoint.
The retry strategy will retry failed requests until either the max attempts or timeout has been reached.

```go
func Retry(max int, timeout time.Duration, lb Balancer) endpoint.Endpoint
```

Let's wire up our final proxying middleware.
For simplicity, we'll assume the user will specify multiple comma-separate instance endpoints with a flag.

```go
func proxyingMiddleware(instances string, logger log.Logger) ServiceMiddleware {
	// If instances is empty, don't proxy.
	if instances == "" {
		logger.Log("proxy_to", "none")
		return func(next StringService) StringService { return next }
	}

	// Set some parameters for our client.
	var (
		qps         = 100                    // beyond which we will return an error
		maxAttempts = 3                      // per request, before giving up
		maxTime     = 250 * time.Millisecond // wallclock time, before giving up
	)

	// Otherwise, construct an endpoint for each instance in the list, and add
	// it to a fixed set of endpoints. In a real service, rather than doing this
	// by hand, you'd probably use package sd's support for your service
	// discovery system.
	var (
		instanceList = split(instances)
		subscriber   sd.FixedSubscriber
	)
	logger.Log("proxy_to", fmt.Sprint(instanceList))
	for _, instance := range instanceList {
		var e endpoint.Endpoint
		e = makeUppercaseProxy(instance)
		e = circuitbreaker.Gobreaker(gobreaker.NewCircuitBreaker(gobreaker.Settings{}))(e)
		e = kitratelimit.NewTokenBucketLimiter(jujuratelimit.NewBucketWithRate(float64(qps), int64(qps)))(e)
		subscriber = append(subscriber, e)
	}

	// Now, build a single, retrying, load-balancing endpoint out of all of
	// those individual endpoints.
	balancer := lb.NewRoundRobin(subscriber)
	retry := lb.Retry(maxAttempts, maxTime, balancer)

	// And finally, return the ServiceMiddleware, implemented by proxymw.
	return func(next StringService) StringService {
		return proxymw{next, retry}
	}
}
```

## stringsvc3

The complete service so far is [stringsvc3](https://github.com/go-kit/kit/blob/master/examples/stringsvc3).

```
$ go get github.com/go-kit/kit/examples/stringsvc3
$ stringsvc3 -listen=:8001 &
listen=:8001 caller=proxying.go:25 proxy_to=none
listen=:8001 caller=main.go:72 msg=HTTP addr=:8001
$ stringsvc3 -listen=:8002 &
listen=:8002 caller=proxying.go:25 proxy_to=none
listen=:8002 caller=main.go:72 msg=HTTP addr=:8002
$ stringsvc3 -listen=:8003 &
listen=:8003 caller=proxying.go:25 proxy_to=none
listen=:8003 caller=main.go:72 msg=HTTP addr=:8003
$ stringsvc3 -listen=:8080 -proxy=localhost:8001,localhost:8002,localhost:8003
listen=:8080 caller=proxying.go:29 proxy_to="[localhost:8001 localhost:8002 localhost:8003]"
listen=:8080 caller=main.go:72 msg=HTTP addr=:8080
```

```
$ for s in foo bar baz ; do curl -d"{\"s\":\"$s\"}" localhost:8080/uppercase ; done
{"v":"FOO","err":null}
{"v":"BAR","err":null}
{"v":"BAZ","err":null}
```

```
listen=:8001 caller=logging.go:28 method=uppercase input=foo output=FOO err=null took=5.168µs
listen=:8080 caller=logging.go:28 method=uppercase input=foo output=FOO err=null took=4.39012ms
listen=:8002 caller=logging.go:28 method=uppercase input=bar output=BAR err=null took=5.445µs
listen=:8080 caller=logging.go:28 method=uppercase input=bar output=BAR err=null took=2.04831ms
listen=:8003 caller=logging.go:28 method=uppercase input=baz output=BAZ err=null took=3.285µs
listen=:8080 caller=logging.go:28 method=uppercase input=baz output=BAZ err=null took=1.388155ms
```

# Advanced topics

## Threading a context

The context object is used to carry information across conceptual boundaries in the scope of a single request.
In our example, we haven't yet threaded the context through our business logic.
But that's almost always a good idea.
It allows you to pass request-scoped information between business logic and middlewares,
 and is necessary for more sophisticated tasks like granular distributed tracing annotations.

Concretely, this means your business logic interfaces will look like

```go
type MyService interface {
	Foo(context.Context, string, int) (string, error)
	Bar(context.Context, string) error
	Baz(context.Context) (int, error)
}
```

## Distributed tracing

Once your infrastructure grows beyond a certain size,
 it becomes important to trace requests through multiple services,
 so you can identify and troubleshoot hotspots.
See [package tracing](https://github.com/go-kit/kit/blob/master/tracing) for more information.

## Creating a client package

It's possible to use Go kit to create a client package to your service,
 to make consuming your service easier from other Go programs.
Effectively, your client package will provide an implementation of your service interface,
 which invokes a remote service instance using a specific transport.
See [addsvc/cmd/addcli](https://github.com/go-kit/kit/blob/master/examples/addsvc/cmd/addcli)
 or [package profilesvc/client](https://github.com/go-kit/kit/blob/master/examples/profilesvc/client)
 for examples.
