## 生命周期协议

=================

**版本**: 2.0 (2019-03-20)

生命周期 ASGI 子规范概述了如何在 ASGI 中通信生命周期事件，例如启动和关闭。

生命周期消息允许应用程序在运行事件循环的上下文中进行初始化和关闭。一个例子是创建连接池，然后关闭连接池以释放连接。

生命周期应在将处理请求的每个事件循环中执行一次。
在多进程环境中，每个进程将有生命周期事件；而在多线程环境中，每个线程将有生命周期。
重要的是生命周期和请求在同一个事件循环中运行，以确保诸如数据库连接池之类的对象不会在事件循环之间移动或共享。

以下是一个可能的协议实现：

```python

async def app(scope, receive, send):
if scope['type'] == 'lifespan':
while True:
message = await receive()
if message['type'] == 'lifespan.startup':
... # 在这里执行启动操作！
await send({'type': 'lifespan.startup.complete'})
elif message['type'] == 'lifespan.shutdown':
... # 在这里执行关闭操作！
await send({'type': 'lifespan.shutdown.complete'})
return
else:
pass # 处理其他类型
```

### 范围

生命周期范围在事件循环的持续时间内存在。

`scope` 中传递的范围信息包含基本元数据：

* `type` (*Unicode 字符串*) -- `"lifespan"`。
* `asgi["version"]` (*Unicode 字符串*) -- ASGI 规范的版本。
* `asgi["spec_version"]` (*Unicode 字符串*) -- 使用的规范版本。可选；如果缺少，则默认值为 `"1.0"`。
* `state` 可选(*dict[Unicode 字符串, Any]*) -- 一个空名称空间，应用程序可以在其中持久化状态，以便在处理后续请求时使用。可选；如果缺少，则服务器不支持此功能。

如果在使用 `lifespan.startup` 消息或类型为 `lifespan` 的 `scope` 调用应用程序可调用对象时引发异常，则服务器必须继续运行，但不发送任何生命周期事件。

这允许与不支持生命周期协议的应用程序兼容。如果你想记录启动期间发生的错误并阻止服务器启动，则应发送 `lifespan.startup.failed`。

### 生命周期状态

应用程序通常希望将生命周期周期中的数据持久化到请求/响应处理中。
例如，可以在生命周期周期中建立数据库连接并将其持久化到请求/响应周期。
`scope["state"]` 名称空间提供了一个用于存储这些类型的东西的地方。
服务器将确保将名称空间的 *浅拷贝* 传递到每个后续的应用程序请求/响应调用中。
由于服务器管理应用程序的生命周期，并且通常也管理事件循环，因此这确保了应用程序始终访问对应于正确事件循环和生命周期的数据库连接（或其他存储的对象），而无需使用上下文变量，全局可变状态或担心对过时/关闭连接的引用。

实现此功能的 ASGI 服务器将在 `lifespan` 范围内提供 `state`：

```json
"scope": {
...
"state": {},
}
```

此名称空间完全由 ASGI 应用程序控制，服务器不会与其交互，除了复制它。
但是，应用程序应通过正确命名它们的键进行合作，以防止它们与其他框架或中间件发生冲突。

### 启动 - `receive` 事件

当服务器准备启动并接收连接，但尚未开始接收连接时，将其发送到应用程序。

键：

* `type` (*Unicode 字符串*) -- `"lifespan.startup"`。

### 启动完成 - `send` 事件

应用程序在完成其启动后将其发送。服务器必须等待此消息才能开始处理连接。

键：

* `type` (*Unicode 字符串*) -- `"lifespan.startup.complete"`。


### 启动失败 - `send` 事件

应用程序在启动失败时将其发送。如果服务器看到此消息，它应记录/打印提供的消息，然后退出。

键：

* `type` (*Unicode 字符串*) -- `"lifespan.startup.failed"`。
* `message` (*Unicode 字符串*) -- 可选；如果缺少，则默认为 `""`。


### 关闭 - `receive` 事件

当服务器停止接受连接并关闭所有活动连接时，将其发送到应用程序。

键：

* `type` (*Unicode 字符串*) -- `lifespan.shutdown"`。


### 关闭完成 - `send` 事件

应用程序在完成清理后将其发送。服务器必须等待此消息才能终止。

键：

* `type` (*Unicode 字符串*) -- `"lifespan.shutdown.complete"`。


### 关闭失败 - `send` 事件

应用程序在完成清理失败时将其发送。如果服务器看到此消息，它应记录/打印提供的消息，然后终止。

键：

* `type` (*Unicode 字符串*) -- `"lifespan.shutdown.failed"`。
* `message` (*Unicode 字符串*) -- 可选；如果缺少，则默认为 `""`。

### 版本历史记录

* 2.0 (2019-03-04): 添加了 startup.failed 和 shutdown.failed，明确了启动阶段中的异常处理。
* 1.0 (2018-09-06): 使用生命周期协议更新了 ASGI 规范。

### 版权

本文档已归入公有领域。
