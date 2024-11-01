`org.apache.dubbo.remoting.transport.netty4.NettyChannel.send(Object, boolean)`
`org.apache.dubbo.remoting.transport.AbstractClient.send(Object, boolean)`
`org.apache.dubbo.remoting.transport.AbstractPeer.send(Object)`
`org.apache.dubbo.remoting.exchange.support.header.HeaderExchangeChannel.request(Object, int, ExecutorService)`
`org.apache.dubbo.remoting.exchange.support.header.HeaderExchangeClient.request(Object, int, ExecutorService)`
`org.apache.dubbo.rpc.protocol.dubbo.ReferenceCountExchangeClient.request(Object, int, ExecutorService)`

​	**主要是实现了请求发送与异步调用的支持。**

​	使用 `HeaderExchangeChannel.request` 发送请求。在通道关闭 (`closed`) 时会抛出 `RemotingException` 异常并给出相关错误提示。

​	`HeaderExchangeChannel.request`创建 `Request` 对象并关联 `DefaultFuture` 以支持异步调用。将请求对象 `req` 发送至 `channel`，若发送失败，则会取消该 `future` 并抛出 `RemotingException`，通道连接失败或关闭会导致异常，当前日志记录详细描述了通道状态及失败原因。

​	`	NettyChannel.send` 将消息发送到远程 Netty 服务器，并在有必要时等待发送完成，若当前 `NettyChannel` 没有连接成功或已关闭，`send` 方法会抛出 `RemotingException`，`send` 方法能够在节点不可达的情况下及时反馈错误。



***



`org.apache.dubbo.rpc.protocol.dubbo.DubboInvoker.doInvoke(Invocation)`

​	当前在方法中假设 `clientsProvider.getClients()` 始终返回有效的 `ExchangeClient` 列表。若 `clientsProvider` 提供的客户端列表为空或者客户端不可用，将导致节点不可达的问题，此时调用可能会抛出 `RpcException` 或 `NullPointerException`并被相应的异常处理捕获，异常处理中均提供了详细的错误信息。

​	**`clientsProvider.getClients()` 获取后**，可以添加日志记录 `exchangeClients` 的数量以及每个客户端的连接状态，在节点找不到或不可用时能提供基本的状态信息。



***



`org.apache.dubbo.rpc.protocol.AbstractInvoker.doInvokeAndReturn(RpcInvocation)`

​	在`isDestroyed()`判定节点销毁时，当前已经记录了较为清晰的日志信息（节点地址、消费者地址、Dubbo版本号等），基本上可以满足定位问题的需求。



***



`org.apache.dubbo.rpc.protocol.AbstractInvoker.invoke(Invocation)`

​	`Invoker`实例被销毁，在`invoke`方法执行时会进行检查并记录警告日志，若节点不可用或存在其他错误情况，例如网络异常、序列化异常等，这些异常会被捕获，并在`RpcException`中抛出相应的错误代码。

​	



***



`org.apache.dubbo.rpc.listener.ListenerInvokerWrapper.invoke(Invocation)`

​	在 `Invoker` 的生命周期事件中通知注册的监听器（`InvokerListener`），以便监听器可以在服务引用或销毁时执行相关操作。



***



`org.apache.dubbo.rpc.filter.RpcExceptionFilter.invoke(Invoker, Invocation)`

​	**该过滤器在调用完成后检查 `Result` 对象中是否包含 `RpcException`，并直接抛出该异常。**

​	在此过滤器中，**节点找不到**的问题不会直接出现，因为此过滤器只是在调用完成后检查 `Result`，不会影响调用链的构建或节点查找。但在调用链上游发生的节点找不到问题，例如缺失 `Invoker`，将导致调用链提前中断，最终结果会在 `onError` 中被捕获。



***

`org.apache.dubbo.rpc.cluster.filter.FilterChainBuilder$CopyOfFilterChainNode.invoke(Invocation)`
`org.apache.dubbo.rpc.cluster.filter.FilterChainBuilder$CallbackRegistrationInvoker.invoke(Invocation)`

***



`org.apache.dubbo.rpc.protocol.ReferenceCountInvokerWrapper.invoke(Invocation)`
`org.apache.dubbo.rpc.cluster.support.AbstractClusterInvoker.invokeWithContext(Invoker, Invocation)`
`org.apache.dubbo.rpc.cluster.support.FailoverClusterInvoker.doInvoke(Invocation, List, LoadBalance)`
`org.apache.dubbo.rpc.cluster.support.AbstractClusterInvoker.invoke(Invocation)`
`org.apache.dubbo.rpc.cluster.router.RouterSnapshotFilter.invoke(Invoker, Invocation)`

​	**用于控制集群调用的行为，并在服务调用链中执行如路由快照、重试逻辑和引用计数管理。**

​	`**RouterSnapshotFilter**`过滤器检查开关状态并设置路由快照标记。由于过滤器本身没有节点选择逻辑，因此不会导致节点不可用问题。

​	**`AbstractClusterInvoker.checkInvokers`**是节点不可用异常的关键检查点，若调用`list(invocation)`后获取到的`invokers`列表为空，则会在`checkInvokers`方法中抛出`RpcException`并给出异常信息，明确提示没有可用节点。异常信息记录了调用方法、服务接口名称、消费者地址等关键信息，为节点不可用问题的排查提供详尽的上下文，不需要额外日志。

​	`FailoverClusterInvoker`在重试过程中对已失败的提供者进行记录并在日志中输出，追踪所有尝试的提供者及其地址。若最后一次调用仍然失败，会在异常中包含所有失败的提供者信息，记录重试次数和具体的节点，已有详细的日志记录。

​	`ReferenceCountInvokerWrapper`主要负责管理调用器的销毁状态。通过读写锁确保在销毁后不再接受调用请求，若在销毁后调用会记录警告日志，并抛出异常，提示调用器已销毁。



***

`org.apache.dubbo.rpc.cluster.filter.FilterChainBuilder$CopyOfFilterChainNode.invoke(Invocation)`

***



`org.apache.dubbo.monitor.support.MonitorFilter.invoke(Invoker, Invocation)`

​	**服务提供方的监控过滤器，主要用于跟踪和记录服务调用的统计信息。**

​	过滤器本身只负责监控信息的收集和发布，因此在节点不可用时不会直接导致调用失败。在 `collect` 方法出现异常时记录警告日志，提示监控信息发送失败，包含服务的 URL 和异常信息。



***

`org.apache.dubbo.rpc.cluster.filter.FilterChainBuilder$CopyOfFilterChainNode.invoke(Invocation)`

***



`org.apache.dubbo.rpc.cluster.filter.support.MetricsClusterFilter.invoke(Invoker, Invocation)`

​	**消费端的过滤器，用于在服务调用过程中收集并发布度量信息。**

​	主要用于收集调用度量信息，不直接参与节点的路由或负载均衡，因此在节点不可用的情况下不会直接导致调用失败。

​	当前代码中没有显式的日志输出，通过 `MetricsEventBus` 发布事件。可以在 `handleMethodException` 中添加简要日志输出，以记录在发生 `Forbidden` 类型的 `RpcException` 异常时`appName` 和 `invocation` 的基本信息：

~~~java
if (e.isForbidden()) {
    logger.warn("Forbidden request detected for app: {appName}, method: {MethodName} with code: {errorCode}");
}
~~~





***

`org.apache.dubbo.rpc.cluster.filter.FilterChainBuilder$CopyOfFilterChainNode.invoke(Invocation)`

***



`org.apache.dubbo.rpc.protocol.dubbo.filter.FutureFilter.invoke(Invoker, Invocation)`

​	**消费端的过滤器，用于异步调用场景下的回调处理。**

​	由于 `FutureFilter` 依赖于 `AsyncMethodInfo` 来判断是否配置了回调方法，但并不直接负责节点的查找或路由选择，因此该过滤器自身不会因为节点不可用而引发调用失败。

​	代码中已包含了必要的错误日志。`fireThrowCallback` 方法中已经通过 `logger.error` 捕获并记录了回调执行失败时的详细信息，包括回调方法名和对应的 URL。



***

`org.apache.dubbo.rpc.cluster.filter.FilterChainBuilder$CopyOfFilterChainNode.invoke(Invocation)`

***



`org.apache.dubbo.metrics.filter.MetricsFilter.invoke(Invoker, Invocation, boolean)`
`org.apache.dubbo.rpc.cluster.filter.support.MetricsConsumerFilter.invoke(Invoker, Invocation)`

​	**为消费者端的调用收集监控指标，帮助捕获请求、响应和错误事件，以便通过 `MetricsDispatcher` 和 `MetricsEventBus` 分发事件。**

​	此过滤器的逻辑主要是监控事件的分发和记录，并不直接参与节点的查找或选择，不会因节点不可用而直接引发调用失败问题。



***

`org.apache.dubbo.rpc.cluster.filter.FilterChainBuilder$CopyOfFilterChainNode.invoke(Invocation)`

***



`org.apache.dubbo.rpc.cluster.filter.support.ConsumerClassLoaderFilter.invoke(Invoker, Invocation)`

​	`Dubbo` 消费者端的过滤器，将当前线程的 `ClassLoader` 暂时切换为服务模型 (`ServiceModel`) 中定义的 `ClassLoader`，确保服务调用时使用正确的类加载上下文。

​	不涉及服务节点的查找与调用路径的控制，因此不会直接导致节点不可用问题。



***

`org.apache.dubbo.rpc.cluster.filter.FilterChainBuilder$CopyOfFilterChainNode.invoke(Invocation)`

***



`org.apache.dubbo.tracing.filter.ObservationSenderFilter.invoke(Invoker, Invocation)`

​	**消费者侧的过滤器，用于 Dubbo 客户端调用的观察与监控。**

​	仅负责调用的监控与记录，不涉及服务节点的查找与选择，所以不会直接因节点不可用导致调用失败。如果调用过程本身发生异常，`Observation` 会捕获错误，且会在 `onError` 或 `onResponse` 的异常处理中生成相关记录。

​	 `ObservationSenderFilter` 已较好地记录了调用中的关键信息，通过 `observation.error()` 等记录异常；日志信息在 `ObservationRegistry` 中统一处理，不需额外的日志补充。



***

`org.apache.dubbo.rpc.cluster.filter.FilterChainBuilder$CopyOfFilterChainNode.invoke(Invocation)`

***



`org.apache.dubbo.rpc.cluster.filter.support.ConsumerContextFilter.invoke(Invoker, Invocation)`

​	**在消费端调用时将当前的 `RpcContext` 配置为包含必要的上下文信息，如 `invoker`、`invocation`、本地和远程地址等，以便执行线程访问。**

​	该类不涉及对节点（服务提供者）的选择或负载均衡。主要职责是传递调用过程中的附加上下文信息，并不影响节点的可用性。该方法未添加额外日志，主要依赖 `RpcException` 的返回来传递错误信息。



***



`org.apache.dubbo.rpc.cluster.filter.FilterChainBuilder$CopyOfFilterChainNode.invoke(Invocation)`

​	**调用过滤器链中的单个过滤器，并确保结果是 `AsyncRpcResult` 类型，同时处理调用过程中的异常和监听器事件。**

​	此方法主要是确保过滤器链的调用流转和正确性，不直接涉及节点选择或服务地址定位，不会因节点缺失导致调用失败，也不涉及对节点的可用性判断。

​	当返回结果类型不符合预期时，会记录错误日志，详细描述问题，并抛出异常。对于调用过程中发生的异常，在 `onError` 处理过程中，已经使用日志记录了详细的异常信息。



***



`org.apache.dubbo.rpc.cluster.filter.FilterChainBuilder$CallbackRegistrationInvoker.invoke(Invocation)`

​	`invoke` 方法在执行 `filterInvoker.invoke(invocation)` 后，为返回的异步 `Result` 对象注册了一个 `whenCompleteWithContext` 回调。此回调的作用是依次处理 `filters` 列表中的监听器 (`Filter.Listener`)，在调用过程中捕获并处理异常。

​	方法主要用于监听回调，不会直接导致节点找不到的失败问题。当监听器调用出现异常时记录错误日志，并附带过滤器索引和名称，且在 `DEBUG` 级别提供详细的过滤器列表信息。



***

`org.apache.dubbo.rpc.cluster.support.wrapper.AbstractCluster$ClusterFilterInvoker.invoke(Invocation)`

***



`org.apache.dubbo.rpc.cluster.support.wrapper.MockClusterInvoker.invoke(Invocation)`

​	**通过 URL 配置检查是否启用了 `mock` 功能，按不同的 mock 类型提供降级机制。**

​	若 `invoker` 为 `null` 或调用失败，会触发异常并进入降级逻辑，通过 `doMockInvoke` 模拟返回结果，从而避免直接失败，且针对节点不可用导致的问题，日志输出了 `force-mock` 和 `fail-mock` 的状态信息，并记录了 URL、方法名和异常原因，便于在服务节点不可用时进行快速排查。



***



`org.apache.dubbo.rpc.cluster.support.wrapper.ScopeClusterInvoker.invoke(Invocation)`

​	**根据不同条件选择相应的调用方式**

​	当前代码已在各个调用分支（广播调用、点对点直接调用、本地 `JVM` 调用，默认的远程调用）中输出了调试日志，以帮助定位每个调用类型的执行情况。

~~~java
if (isBroadcast()) {
    if (logger.isDebugEnabled()) {
        logger.debug("Performing broadcast call for method: " + RpcUtils.getMethodName(invocation)
                + " of service: " + getUrl().getServiceKey());
    }
    return invoker.invoke(invocation);
}
……
~~~

​	若 `injvmInvoker` 或 `invoker` 为空，将可能导致调用失败。

***



`org.apache.dubbo.registry.client.migration.MigrationInvoker.invoke(Invocation)`

​	**根据不同的迁移步骤（`APPLICATION_FIRST`, `FORCE_APPLICATION`, `FORCE_INTERFACE`）选择合适的 `Invoker`，用于在服务调用中实现灵活的负载与故障切换机制。**

​	在 `serviceDiscoveryInvoker` 不可用时回退至 `invoker`。如果 `decideInvoker` 方法在检查 `serviceDiscoveryInvoker` 可用性时无法通过 `checkInvokerAvailable` 方法确认其可用性，可能导致失败。此外，若 `currentAvailableInvoker` 为空，可能表明未正确设置可用的 `Invoker`，进而导致调用失败。

​	在 `decideInvoker` 中记录 `serviceDiscoveryInvoker` 可用性检查的结果：

~~~java
logger.debug("ServiceDiscoveryInvoker availability {info}");
~~~

​	若 `currentAvailableInvoker` 最终为空且无法调用时，记录错误日志以协助排查：

~~~java
if (currentAvailableInvoker == null) {
    logger.error("No available invoker for invocation: {error info}");
}
~~~



---



`org.apache.dubbo.rpc.proxy.InvocationUtil.invoke(Invoker, RpcInvocation)`

​	**在执行远程调用时管理上下文数据，并使用 `Profiler` 记录调用的性能信息**

​	在超时检测中添加了warn日志。

```java
logger.warn(
            PROXY_TIMEOUT_REQUEST,
            "",
            "",
            String.format(
                    "[Dubbo-Consumer] execute service %s#%s cost %d.%06d ms, this invocation almost (maybe already) timeout. Timeout: %dms\n"
                            + "invocation context:\n%s" + "thread info: \n%s",
                    rpcInvocation.getProtocolServiceKey(),
                    rpcInvocation.getMethodName(),
                    usage / 1000_000,
                    usage % 1000_000,
                    timeout,
                    attachment,
                    Profiler.buildDetail(bizProfiler)));
```



------

`org.apache.dubbo.rpc.proxy.InvokerInvocationHandler.invoke(Object, Method, Object[])`

​	**通过 `InvocationHandler` 接口代理 `Invoker<?>`，使客户端能够透明地调用服务方法，并在必要时通过 `RpcInvocation` 创建远程调用请求**

​	无日志。`InvokerInvocationHandler`主要依赖 `invoker` 和 `serviceModel` 的信息来构建 `RpcInvocation`，若`invoker.getUrl()` 返回的 URL 中缺少 `ServiceModel`，导致 `serviceModel` 为空，`InvocationUtil.invoke(invoker, rpcInvocation)` 可能在节点不可达或服务未注册时发生调用失败。

```java
if (serviceModel == null) {
    logger.warn("warn {info}");
}
```

​	可在 `RpcInvocation` 创建后，但在实际调用之前记录方法名称、参数信息及 `RpcInvocation` 的关键属性。

```java
logger.debug("debug {info}");
```



---

`org.apache.dubbo.config.spring.util.LazyTargetInvocationHandler.invoke(Object, Method, Object[])`

​	无日志。在调用 `lazyTargetSource.getTarget()` 时，如果 `lazyTargetSource` 无法获取目标对象实例（例如目标对象未配置或服务节点不可用），可能导致 `target` 为 `null`。可在target为null时记录错误日志。
