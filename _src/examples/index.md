{
    "template": "../inc/page.template.source",
    "subtitle": "Examples"
}
---
用户们一致发现学习 Go kit 最高效的方式是通过研究和学习示例服务。
在这里，你可以找到一些示例，可以帮助你适应 Go kit 设计风格、设计模式和最佳实践。


- **[stringsvc](stringsvc.html)** 是一个通过从最初原则指导你编写服务的教程示例。
它可以帮助你理解 Go kit 设计中的取舍。

- **[addsvc](https://github.com/go-kit/kit/blob/master/examples/addsvc)**
  是示例服务的原型。
  它展示了一组在所有支持的传输层上可进行的操作。
  这些操作是有关于完全日志记录、测量和如何使用分布式追踪。
  它演示了如何创建和使用客户端代码包。
  它演示了 Go kit 几乎所有的特性。

- **[profilesvc](https://github.com/go-kit/kit/blob/master/examples/profilesvc)**
  演示如何使用 Go kit 编写具有 REST-ish API 的微服务。
  它使用了标准库的 net/http 和 卓越的 Gorilla Web 开发工具包。

- **[shipping](https://github.com/go-kit/kit/blob/master/examples/shipping)**
  是由多个基于领域驱动设计模型的微服务整合的完整的、“仿现实”的应用。

- **[apigateway](https://github.com/go-kit/kit/blob/master/examples/apigateway)**
  演示了如何实施[API 网关设计模式](http://microservices.io/patterns/apigateway.html)，
  使用 Consul 完成服务发现。
