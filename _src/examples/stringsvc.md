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

## Transport logging 传输层记录

任何需要记录的组件都应该像把数据连接当作依赖项那样，同样也应将 logger 视为依赖项。
因此，我们在我们的  `func main` 中构建我们的 logger，并将其传递给需要它的组件。
我们一定不要使用全局范围的 logger。

我们可以将 logger 直接传递给我们的 stringService 实现，但是还有一个更好的方法。

让我们使用 **middleware** ，也称为装饰器。

中间件是一个以 endpoint 作为输入参数，同时以 endpoint 作为输出参数的函数。

```go
type Middleware func(Endpoint) Endpoint
```

> 请注意，中间件类型是由 go-kit 为您提供的。

在这之间可以做任何事情。

您可以在下面看到如何实现基本的日志记录中间件：

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

使用 [go-kit log](https://gokit.io/faq/#logging-mdash-why-is-package-log-so-different) 日志软件包，移出标准库的 [log](https://golang.org/pkg/log/)。

您需要从`main.go`文件的底部删除`log.Fatal`。

```go
import (
 "github.com/go-kit/kit/log"
)
```

并将其连接到我们的每个处理程序中。

请注意，在您遵循定义 loggingMiddleware 的 ** Application Logging ** 部分之前，下面的代码段将 *不会* 编译。

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

事实证明，这种技术不仅仅只用于日志记录，还有很多用处。

许多 Go kit 组件被实现为 endpoint 端点中间件。

## Application logging 应用日志

但是，如果我们想要为我们的应用程序域进行日志记录呢，比如传入的参数？

事实证明，我们可以为我们的服务定义一个中间件，可以获得同样好的组合效果。

由于我们的 StringService 被定义为接口，我们只需要创建一个新的类型，它包装现有的 StringService，并执行额外的日志记录任务。

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

然后把它连接起来。

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
使用端点中间件来解决 transport 传输层问题，例如熔断和速率限制。

使用服务中间件来解决业务领域问题，例如日志记录和度量。

说到度量...

## Application instrumentation 应用度量

在 Go kit 包中，度量意味着使用 **package metrics** 来记录有关服务运行时行为的统计信息。

计算处理的任务数量。

记录请求完成的持续时间，

跟踪  in-flight operations 的数量都将被视为度量。

我们可以使用跟日志记录中间件相同的模式。

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

然后把它连接起来。

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

到目前为止，完整的服务是 [stringsvc2](https://github.com/go-kit/kit/blob/master/examples/stringsvc2).

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

# 调用其他服务

很少有服务会孤立地存在。
通常，你需要调用其他服务。
**这是 Go kit 大展拳脚的地方**。
我们提供传输中间件来解决接下来出现的许多问题。
假设我们希望我们的 string 服务去调用一个 _不同的_ string 服务以满足 Uppercase 方法。
实际就是，将请求代理到另一个服务。
让我们来将代理中间件实现为一个 ServiceMiddleware，与日志记录或度量中间件相同。

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

我们在这需要我们已经了解过的完全相同的端点，但是我们将使用它来调用请求，而不是响应请求。
以这种方式使用时，我们将其称为 _client_ endpoint。
为了能调用 client endpoint，我们只需进行一些简单的转换。

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

现在，为了构建这些代理中间件之一，我们将代理URL字符串转换为端点。
如果我们通过 HTTP 传输 JSON，我们可以在 transport/http 包中使用一个帮助函数。
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

## 服务发现和负载平衡

如果我们只有一个远程服务问题就比较简单。
但在现实中，我们可能会有许多可用的服务实例。
我们希望通过一些服务发现机制来发现它们，并将负载分散到它们所有当中。
如果这些实例中的任何一个开始表现不好，我们希望在不影响我们自己的服务可靠性的情况下处理这个问题。
Go kit 为不同的服务发现系统提供适配器，以获取最新的实例集，并作为单独的端点公开。这些适配器称为 Subscriber。
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

在内部，subscribers 使用提供的工厂函数将每个发现的实例字符串（通常是 host：port ）转换为可用的端点。

```go
type Factory func(instance string) (endpoint.Endpoint, error)
```

到目前为止，我们的工厂函数 makeUppercaseProxy 仅仅是直接调用URL。
但是，将一些安全中间件（如熔断器和限流器）放入工厂函数中也非常重要。

```go
var e endpoint.Endpoint
e = makeUppercaseProxy(instance)
e = circuitbreaker.Gobreaker(gobreaker.NewCircuitBreaker(gobreaker.Settings{}))(e)
e = kitratelimit.NewTokenBucketLimiter(jujuratelimit.NewBucketWithRate(float64(maxQPS), int64(maxQPS)))(e)
```

现在我们已经有了一组 endpoints 端点，我们需要选择一个。
负载均衡器会封装 subscribers，并从许多 endpoint 中选择一个。
Go kit 提供了几个基本的负载均衡器，如果你想要更高级的启发式算法，你可以很容易地编写自己的负载均衡器。

```go
type Balancer interface {
	Endpoint() (endpoint.Endpoint, error)
}
```

现在我们可以根据一些启发式算法选择 endpoint 端点。
我们可以使用它为消费者提供单一的、合理的、健壮的 endpoint 端点。
使用一个重试策略封装负载均衡器，以保证返回一个可用的 endpoint 端点。
重试策略将重试失败的请求，直到达到最大尝试次数或超时。

```go
func Retry(max int, timeout time.Duration, lb Balancer) endpoint.Endpoint
```

让我们把最终的代理中间件连接起来。
为简单起见，我们假设用户会指定多个逗号分隔符和标志来区分 endpoint 实例。

```go
func proxyingMiddleware(instances string, logger log.Logger) ServiceMiddleware {
	// 如果实例为空，不代理
	if instances == "" {
		logger.Log("proxy_to", "none")
		return func(next StringService) StringService { return next }
	}

	// 为我们的客户端设置一些参数。
	var (
		qps         = 100                    // 超出该值我们将返回错误
		maxAttempts = 3                      // 在放弃前，每请求的尝试次数
		maxTime     = 250 * time.Millisecond // 在放弃前，最在等待时间
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

到目前为止，完整的服务是 [stringsvc3](https://github.com/go-kit/kit/blob/master/examples/stringsvc3).

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

# 高级主题 

## Threading a context

上下文对象用于在单个请求的范围内跨概念边界传送信息。
在我们的示例中，我们尚未在我们的业务逻辑当中处理上下文。
但这几乎总是一个好想法。
它允许您在业务逻辑和中间件之间传递请求范围的信息，并且对于更复杂的任务（如分布式精细跟踪标记）是必需的。

具体来说，这意味着您的业务逻辑接口将如下所示：

```go
type MyService interface {
	Foo(context.Context, string, int) (string, error)
	Bar(context.Context, string) error
	Baz(context.Context) (int, error)
}
```

## 分布式追踪

一旦您的基础设施增长超过一定规模，
对跨越多个服务的请求进行追踪就会变得很重要，
这样您就可以识别并排查故障热点。

参考 [package tracing](https://github.com/go-kit/kit/blob/master/tracing) 获取更多信息。

## 创建 client 包

可以使用 Go kit 为您的服务创建客户端代码包，以便能从其他 Go 程序中轻松使用您的服务。
实际上，您的客户端代码包将提供服务接口的实现，它使用特定的传输层调用远程服务实例。
参考 [addsvc/cmd/addcli](https://github.com/go-kit/kit/blob/master/examples/addsvc/cmd/addcli)
 或 [package profilesvc/client](https://github.com/go-kit/kit/blob/master/examples/profilesvc/client)
 作为示例.
