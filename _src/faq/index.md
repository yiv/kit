{
    "template": "../inc/page.template.source",
    "subtitle": "Frequently asked questions",
    "toc": true
}
---
# 概要

<<<<<<< HEAD
## Go kit 是什么？
Go kit 是一套可以帮助你构建强健的、可靠的和易于维护的微服务的 Go 软件包（库）的集合。
它最初的构想是作为一个工具箱，可以帮助大型组织（当代企业）采用 Go 作为开发语言。
但是它的目标群体却迅速向下扩散，现在同样适用于小型创业公司和机构。
了解更多 Go kit 源起，请阅读 [Go kit: Go in the modern enterprise](http://peter.bourgon.org/go-kit/)。
=======
## Go kit 是什么?
>>>>>>> e52804fa148d2812b09fd606c87a880ee27bb9bf

## 我为什么应该使用 Go kit?

如果你想把微服务设计模式引入公司，你应该使用 Go kit。
Go kit 可以帮助你组织和构建你的服务，规避常陷阱，保证代码优雅地增长。
Go kit 还可以有助于向工程经理或技术主管等利益相关者证明 Go 作为一种实践语言的可行性。

Go kit 通过提供成熟的设计模式和编程风格可大大降低在使用 Go 语言和微服务实践中的风险，它由一大批经验丰富的贡献者编写和维护，并且已经在生产环境中得到过验证。


## 谁在背后支持 Go kit 的开发

Go kit 最初由 [Peter Bourgon]（https://peter.bourgon.org/about）发起，
但现在由来自不同背景和组织的[大量贡献者]（https://github.com/go-kit/kit/contributors）共同构建和维护。
Go kit 目前是一项全志愿者项目，并且没有商业支持。

## Go kit 是否已经适合生产环境？

**是**  Go kit 正在被多个组织用于生产环境中，包括大型和小型企业。

## 哪些组织正在使用 Go kit？

Watch this space :)

## Go kit 与 Micro 相比如何？

与 Go kit一样，[Micro]（https://micro.mu）将自己描述为 [微服务工具包]（https://github.com/micro/micro/wiki/Architecture）。
但与 Go kit 不同，Micro 也将自己描述为[微服务生态系统]（https://micro.mu/）。
它有一个更广阔的主张，同时满足基础设施和架构平台的期望和想法。
简而言之，我认为 Micro 希望成为一个平台;相比之下，Go kit 希望集成到您的平台中。

# 架构与设计

## 简介 - 了解 Go kit 的关键概念

如果您来自 Symfony（PHP）、Rails（Ruby）、Django（Python），或者任何一个流行的 MVC 风格的框架，你要知道的第一件事是 Go kit 不是 MVC 框架。
相反，Go kit 服务分为三层：

1. Transport layer
2. Endpoint layer
3. Service layer

调用请求从第1层进入服务，向内流转到第3层，请求结果以相反方式返回。

这对你可能是一个比较大的转变，但一旦你理解了这些概念，你会看到 Go kit 非常适合现代软件设计：包括微服务和称之为 
[elegant](https://martinfowler.com/bliki/MonolithFirst.html)、
[monoliths](https://inconshreveable.com/10-07-2015/the-neomonolith/) 的软件设计。

## Transports - 什么是 transports?

传输域会与 HTTP 或 gRPC 等具体的传输协议绑定。
同一个微服务能同时支持一个或多个传输协议，是非常强大的功能； 您可以支持传统的 HTTP API 和更新的 RPC 服务，
这些都可以在一个微服务实现。


实现 REST-ish HTTP API 时，您的路由是在HTTP传输中定义的。
最常见的在 HTTP 路由器函数中定义的路由如下：

```go
r.Methods("POST").Path("/profiles/").Handler(httptransport.NewServer(
		e.PostProfileEndpoint,
		decodePostProfileRequest,
		encodeResponse,
		options...,
))
```

## Endpoints - Go kit endpoints 是什么？

一个 endpoint 端点就像 MVC 中的控制器上的动作或处理程序; 它是开发具有安全和强壮逻辑的程序的生命所在。
如果你要实现两个传输协议（ HTTP 和 gRPC ），则可能两种方法都可以将请求发送到同一个 endpoint。

## Services - Go kit service 是什么？

Services 是实现所有业务逻辑的地方。
Services 通常将多个 endpoint 端点集成在一起。
在 Go kit 中，服务通常被建模为接口，
这些接口的具体实现包含实际业务逻辑。
Go kit service 应努力遵守
也就是说，业务逻辑不用了解 endpoint 或特别是 transports 的概念：
你的服务不应该知道有关 HTTP 头部或 gRPC 错误代码的任何信息。


## Middlewares - 在 Go kit 中 middlewares 是什么?

Go kit 试图通过使用中间件（或装饰器）模式来强制严格分离关注点。
中间件可以封装 endpoint 或 service 来添加更多功能，例如日志记录，速率限制，负载均衡或分布式追踪。
在 endpoint 或 service 外层链接多个中间件是很常见的。


## Design - Go kit 微服务是如何建模的？

<img src="onion.png" height=355 width=355 alt="Go kit service diagram" style="float:right;" />

Putting all these concepts together, we see that Go kit microservices are modeled like an onion, with many layers.
The layers can be grouped into our three domains.
The innermost **service** domain is where everything is based on your specific service definition,
 and where all of the business logic is implemented.
The middle **endpoint** domain is where each method of your service is abstracted to the generic
 [endpoint.Endpoint](https://godoc.org/github.com/go-kit/kit/endpoint#Endpoint),
 and where safety and antifragile logic is implemented.
Finally, the outermost **transport** domain is where endpoints are bound
 to concrete transports like HTTP or gRPC.

You implement the core business logic by defining an interface for your service and providing a concrete implementation.
Then, you write service middlewares to provide additional functionality,
 like logging, analytics, instrumentation &mdash; anything that needs knowledge of your business domain.

Go kit provides endpoint and transport domain middlewares,
 for functionality like rate limiting, circuit breaking, load balancing,
 and distributed tracing &mdash; all of which are generally agnostic to your business domain.

In short, Go kit tries to enforce strict **separation of concerns**
 through studious use of the **middleware** (or decorator) pattern.

## Dependency Injection &mdash; Why is func main always so big?

Go kit encourages you to design your services as multiple interacting components,
 including several single-purpose middlewares.
Experience has taught me that the most comprehensible, maintainable, and expressive method
 of defining and wiring up the component graph in a microservice
 is through an explicit and declarative composition in a large func main.

Inversion of control is a common feature of other frameworks,
 implemented via Dependency Injection or Service Locator patterns.
But in Go kit, you should wire up your entire component graph in your func main.
This style reinforces two important virtues.
By strictly keeping component lifecycles in main,
 you avoid leaning on global state as a shortcut,
 which is critical for testability.
And if components are scoped to main,
 the only way to provide them as dependencies to other components
 is to pass them explicitly as parameters to constructors.
This keeps dependencies explicit, which stops a lot of technical debt before it starts.

As an example, let's say we have the following components:

- Logger
- TodoService, implementing the Service interface
- LoggingMiddleware, implementing the Service interface, requiring Logger and concrete TodoService
- Endpoints, requiring a Service interface
- HTTP (transport), requiring Endpoints

Your func main should be wired up as follows:

```go
logger := log.NewLogger(...)

var service todo.Service    // interface
service = todo.NewService() // concrete struct
service = todo.NewLoggingMiddleware(logger)(service)

endpoints := todo.NewEndpoints(service)
transport := todo.NewHTTPTransport(endpoints)
```

At the cost of having a potentially large func main, composition is explicit and declarative.
For more general Go design tips, see
 [Go best practices, six years in](https://peter.bourgon.org/go-best-practices-2016/).

## Deployment &mdash; How should I deploy Go kit services?

It's totally up to you.
You can build a static binary, scp it to your server,
 and use a supervisor like [runit](http://smarden.org/runit/).
Or you can use a tool like [Packer](https://www.packer.io/) to create an AMI,
 and deploy it into an EC2 [autoscaling group](http://docs.aws.amazon.com/autoscaling/latest/userguide/AutoScalingGroup.html).
Or you can package your service up into a container, ship it to a registry,
 and deploy it onto a cloud-native platform like [Kubernetes](http://kubernetes.io).

Go kit is mostly concerned with good software engineering within your service;
 it tries to integrate well with any kind of platform or infrastructure.

## Errors &mdash; How should I encode errors?

Your service methods will probably return errors.
You have two options for encoding them in your endpoints.
You can include an error field in your response struct, and return your business domain errors there.
Or, you can choose to return your business domain errors in the endpoint error return value.

Both methods can be made to work.
But errors returned directly by endpoints are recognized by middlewares that check for failures,
 like circuit breakers.
It's unlikely that a business-domain error from your service
 should cause a circuit breaker to trip in a client.
So, it's likely that you want to encode errors in your response struct.

[addsvc](http://github.com/go-kit/kit/tree/master/examples/addsvc)
 contains examples of both methods.

# More specific topics

## Transports &mdash; Which transports are supported?

Go kit ships with support for HTTP,
 [gRPC](http://www.grpc.io),
 [Thrift](https://thrift.apache.org), and
 [net/rpc](https://golang.org/pkg/net/rpc/).
It's straightforward to add support for new transports;
 just [file an issue](https://github.com/go-kit/kit/issues/new)
 if you need something that isn't already offered.

## Service Discovery &mdash; Which service discovery systems are supported?

Go kit ships with support for
 [Consul](https://consul.io),
 [etcd](https://coreos.com/etcd/),
 [ZooKeeper](https://zookeeper.apache.org/),
 and DNS SRV records.

## Service Discovery &mdash; Do I even need to use package sd?

It depends on your infrastructure.

Some platforms, like Kubernetes, take care of registering services instances
 and making them discoverable automatically, via
 [platform-specific concepts](http://kubernetes.io/docs/user-guide/services).
So, if you run on Kubernetes, you probably don't need to use package sd.

But if you're putting together your own infrastructure or platform with open-source components,
 then your services will likely need to register themselves with the service registry.
Or if you have reached a scale where internal load balancers become a bottleneck,
 you may need to have your services subscribe to the system of record directly,
 and maintain their own connection pools.
(This is the [client-side discovery](http://microservices.io/patterns/client-side-discovery.html) pattern.)
In these situations, package sd will be useful.

## Observability &mdash; Which monitoring systems are supported?

Go kit ships with support for modern monitoring systems like
 [Prometheus](https://prometheus.io) and [InfluxDB](https://influxdata.com/),
 as well as more traditional systems like
 [statsd](https://github.com/etsy/statsd),
 [Graphite](http://graphite.wikidot.com/), and
 [expvar](https://golang.org/pkg/expvar),
 and hosted systems like
 Datadog via [DogStatsD](http://docs.datadoghq.com/guides/dogstatsd/)
 and [Circonus](http://www.circonus.com/).

## Observability &mdash; Which monitoring system should I use?

[Prometheus](https://prometheus.io).

## Logging &mdash; Why is package log so different?

Experience has taught us that a good logging package should be based on a minimal interface
 and should enforce so-called structured logging.
Based on these invariants, Go kit's package log has evolved through
 many design iterations, extensive benchmarking, and plenty of real-world use
 to arrive at its current state.

With a well-defined core contract,
 ancillary concerns like levels, colorized output, and synchronization
 can be easily bolted on using the familiar decorator pattern.
It may feel a little unfamiliar at first,
 but we believe package log strikes the ideal balance between
 usability, maintainability, and performance.

For more details on the evolution of package log, see issues and PRs
 [63](https://github.com/go-kit/kit/issues/63),
 [76](https://github.com/go-kit/kit/pull/76),
 [131](https://github.com/go-kit/kit/issues/131),
 [157](https://github.com/go-kit/kit/pull/157), and
 [252](https://github.com/go-kit/kit/pull/252).
For more on logging philosophy, see
 [The Hunt for a Logger Interface](http://go-talks.appspot.com/github.com/ChrisHines/talks/structured-logging/structured-logging.slide),
 [Let's talk about logging](http://dave.cheney.net/2015/11/05/lets-talk-about-logging), and
 [Logging v. instrumentation](https://peter.bourgon.org/blog/2016/02/07/logging-v-instrumentation.html).

## Logging &mdash; How should I aggregate my logs?

Collecting, shipping, and aggregating logs is the responsibility of the platform, not individual services.
So, just make sure you're writing logs to stdout/stderr, and let another component handle the rest.

## Panics &mdash; How should my service handle panics?

Panics indicate programmer error and signal corrputed program state.
They shouldn't be treated as errors, or ersatz exceptions.
In general, you shouldn't explicitly recover from panics:
 you should allow them to crash your program or handler goroutine,
 and allow your service to return a broken response to the calling client.
Your observability stack should alert you to these problems as they occur,
 and you should fix them as soon as possible.

With that said, if you have the need to handle exceptions, the best strategy
 is probably to wrap the concrete transport with a transport-specific middleware
 that performs a recover.
For example, with HTTP:

```go
var h http.Handler
h = httptransport.NewServer(...)
h = newRecoveringMiddleware(h, ...)
// use h normally
```

## Persistence &mdash; How should I work with databases and data stores?

Accessing databases is typically part of the core business logic.
Therefore, it probably makes sense to include an e.g. *sql.DB pointer in the concrete implementation of your service.

```go
type MyService struct {
	db     *sql.DB
	value  string
	logger log.Logger
}

func NewService(db *sql.DB, value string, logger log.Logger) *MyService {
	return &MyService{
		db:     db,
		value:  value,
		logger: logger,
	}
}
```

Even better: consider defining an interface to model your persistence operations.
The interface will deal in business domain objects, and have an implementation that wraps the database handle.
For example, consider a simple persistence layer for user profiles.

```go
type Store interface {
	Insert(Profile) error
	Select(id string) (Profile, error)
	Delete(id string) error
}

type databaseStore struct{ db *sql.DB }

func (s *databaseStore) Insert(p Profile) error            { /* ... */ }
func (s *databaseStore) Select(id string) (Profile, error) { /* ... */ }
func (s *databaseStore) Delete(id string) error            { /* ... */ }
```

In this case, you'd include a Store, rather than a *sql.DB, in your concrete implementation.
