# OpenTelemetry QuickStart

<!-- Re-generate TOC with `markdown-toc --no-first-h1 -i` -->

<!-- toc -->
- [Set up](#set-up)
- [Tracing](#tracing)
  * [Create basic Span](#create-basic-span)
  * [Create nested Spans](#create-nested-spans)
  * [Span Attributes](#span-attributes)
  * [Create Spans with events](#create-spans-with-events)
  * [Create Spans with links](#create-spans-with-links)
  * [Context Propagation](#context-propagation)
- [Metrics](#metrics)
- [Tracing SDK Configuration](#tracing-sdk-configuration)
  * [Sampler](#sampler)
  * [Span Processor](#span-processor)
  * [Exporter](#exporter)
  * [TraceConfig](#traceconfig)
- [Logging And Error Handling](#logging-and-error-handling)
  * [Examples](#examples)
<!-- tocstop -->

OpenTelemetry能被用于intrument code以收集遥测数据（telemetry data）.可以前往[OpenTelemetry Website]了解更多.

**Libraries** that want to export telemetry data using OpenTelemetry MUST only depend on the
`opentelemetry-api` package and should never configure or depend on the OpenTelemetry SDK. The SDK
configuration must be provided by **Applications** which should also depend on the
`opentelemetry-sdk` package, or any other implementation of the OpenTelemetry API. This way,
libraries will obtain a real implementation only if the user application is configured for it. For
more details, check out the [Library Guidelines].

## Set up
第一步是获取`OpenTelemetry`接口实例的handle

如果你是一名应用开发者，你需要尽可能早地在你的应用中配置一个`OpenTelemetrySdk`的实例，这个可以通过使用`OpenTelemetrySdk.builder()`方法开始实现。

例如：

```java
    SdkTracerProvider sdkTracerProvider = SdkTracerProvider.builder().build();
    sdkTracerProvider.addSpanProcessor(BatchSpanProcessor.builder(OtlpGrpcSpanExporter.builder().build()).build());
    
    OpenTelemetry openTelemetry = OpenTelemetrySdk.builder()
      .setTracerProvider(sdkTracerProvider)
      .setPropagators(ContextPropagators.create(W3CTraceContextPropagator.getInstance()))
      .build();
    
    //optionally set this instance as the global instance:
    GlobalOpenTelemetry.set(openTelemetry);
```

此外，如果你正在writing library instrumentation，强烈推荐你提供向instrumentation code中注入`OpenTelemetry`实例的能力。
如果因为某些原因这个无法实现，你可以退回（fall back）去使用`GlobalOpenTelemetry`的实例。注意，你不能强制要求end-users配置the global，
所以对于library instrumentation来说这是最脆弱的（（brittle））选择。

## Tracing

下面，我们来展示如何使用OpenTelemetry API来trace code。
**注意：** 千万不要使用OpenTelemetry SDK的方法。
In the following, we present how to trace code using the OpenTelemetry API. **Note:** Methods of the
OpenTelemetry SDK should never be called.
 
首先，必须要获取 `Tracer`，它负责创建spans以及和[Context](#context-propagation)交互。
使用OpenTelemetry API获取tracer时需要指定[library instrumenting][Instrumentation Library] 或 [instrumented library]
的name和version。更多的信息可见Specification部分 [Obtaining a Tracer]。

```java
Tracer tracer =
    openTelemetry.getTracer("instrumentation-library-name", "1.0.0");
```

重要：tracer的name和optional version完全是informational。
所有被`OpenTelemetry`单例创建的的Tracer都可以interoperate,无论他们的name如何。
Important: the "name" and optional version of the tracer are purely informational. 
All `Tracer`s that are created by a single `OpenTelemetry` instance will interoperate, regardless of name.

### Create basic Span
想要创建一个basic span，你只需要指定span的名字。
OpenTelemetry SDK将会自动设置span的start time和end time。

```java
Span span = tracer.spanBuilder("my span").startSpan();
// put the span into the current Context
try (Scope scope = span.makeCurrent()) {
	// your use case
	...
} catch (Throwable t) {
    span.setStatus(StatusCode.ERROR, "Change it to your error message");
} finally {
    span.end(); // closing the scope does not end the span, this has to be done manually
}
```

### Create nested Spans

大多数情况下，我们想要在嵌套操作中关联span。OpenTelemetry支持在本地不同进程之间，以及远程不同进程之间进行tracing。
关于如何在远程不同进程间共享context请看[Context Propagation](#context-propagation).

For a method `a` calling a method `b`, the spans could be manually linked in the following way:
```java
void a() {
  Span parentSpan = tracer.spanBuilder("a").startSpan();
  try {
    b(parentSpan);
  } finally {
    parentSpan.end();
  }
}

void b(Span parentSpan) {
  Span childSpan = tracer.spanBuilder("b")
        .setParent(Context.current().with(parentSpan))
        .startSpan();
  // do stuff
  childSpan.end();
}
```
OpenTelemetry API提供了一种自动化的方式在当前线程中传递parent span。
```java
void a() {
  Span parentSpan = tracer.spanBuilder("a").startSpan();
  try(Scope scope = parentSpan.makeCurrent()) {
    b();
  } finally {
    parentSpan.end();
  }
}
void b() {
  Span childSpan = tracer.spanBuilder("b")
    // NOTE: setParent(...) is not required; 
    // `Span.current()` is automatically added as the parent
    .startSpan();
  try(Scope scope = childSpan.makeCurrent()) {
    // do stuff
  } finally {
    childSpan.end();
  }
}
``` 
想要link远程进程中的span，将[Remote Context](#context-propagation)设置为parent是个非常有效的方法。

```java
Span childRemoteParent = tracer.spanBuilder("Child").setParent(remoteContext).startSpan();
```

### Span Attributes
在OpenTelemetry中，可以随意创建span，由实现者使用属性（Attributes）来标注他们（span），这些Attributes反映了特定的操作。
Attributes将向span提供附加的Context，以反映span跟踪的特定操作，如results or operation properties.

```java
Span span = tracer.spanBuilder("/resource/path").setSpanKind(Span.Kind.CLIENT).startSpan();
span.setAttribute("http.method", "GET");
span.setAttribute("http.url", url.toString());
```

其中的某些操作代表了常见的协议调用，如HTTP，database等。
对于这些操作，OpenTelemetry要求set 特定的Attributes。完整的Attributes列表可见跨语言Specification：
[Semantic Conventions](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/semantic_conventions/README.md)中

### Create Spans with events

可以使用带有0至多个[Span Attributes](#span-attributes)的命名过的event来命名span。
每个[Span Attributes](#span-attributes)都是一对带有时间戳的key:value。

```java
span.addEvent("Init");
...
span.addEvent("End");
```

```java
Attributes eventAttributes = Attributes.of(
    AttributeKey.stringKey("key"), "value",
    AttributeKey.longKey("result"), 0L);

span.addEvent("End Computation", eventAttributes);
```

### Create Spans with links
一个span可以同0至多个span连接起来，后者互相之间的连接关系没有要求。连接能够用来表示批操作，在这些批操作中，
该span被多个initiating Spans初始化。每一个Link表示批操作中即将被处理的单项。

```java
Span child = tracer.spanBuilder("childWithLink")
        .addLink(parentSpan1.getSpanContext())
        .addLink(parentSpan2.getSpanContext())
        .addLink(parentSpan3.getSpanContext())
        .addLink(remoteSpanContext)
    .startSpan();
```

关于如果读取远程进程的context可见[Context Propagation](#context-propagation).


### Context Propagation

OpenTelemetry提供了一个基于文本的方式——[W3C Trace Context](https://www.w3.org/TR/trace-context/) HTTP headers
来propagate上下文（context）到远程服务。
OpenTelemetry provides a text-based approach to propagate context to remote services using the

下面展示了使用`HttpURLConnection`来传出（发送）HTTP请求的例子：

 
```java
// Tell OpenTelemetry to inject the context in the HTTP headers
TextMapPropagator.Setter<HttpURLConnection> setter =
  new TextMapPropagator.Setter<HttpURLConnection>() {
    @Override
    public void set(HttpURLConnection carrier, String key, String value) {
        // Insert the context as Header
        carrier.setRequestProperty(key, value);
    }
};

URL url = new URL("http://127.0.0.1:8080/resource");
Span outGoing = tracer.spanBuilder("/resource").setSpanKind(Span.Kind.CLIENT).startSpan();
try (Scope scope = outGoing.makeCurrent()) {
  // Semantic Convention.
  // (Note that to set these, Span does not *need* to be the current instance in Context or Scope.)
  outGoing.setAttribute(SemanticAttributes.HTTP_METHOD, "GET");
  outGoing.setAttribute(SemanticAttributes.HTTP_URL, url.toString());
  HttpURLConnection transportLayer = (HttpURLConnection) url.openConnection();
  // Inject the request with the *current*  Context, which contains our current Span.
  openTelemetry.getPropagators().getTextMapPropagator().inject(Context.current(), transportLayer, setter);
  // Make outgoing call
} finally {
  outGoing.end();
}
...
```

类似地，基于文本的方法也能被用于读取到来的请求中的W3C Trace Context。
下面展示了使用[HttpExchange](https://docs.oracle.com/javase/8/docs/jre/api/net/httpserver/spec/com/sun/net/httpserver/HttpExchange.html)处理HTTP请求的例子

```java
TextMapPropagator.Getter<HttpExchange> getter =
  new TextMapPropagator.Getter<HttpExchange>() {
    @Override
    public String get(HttpExchange carrier, String key) {
      if (carrier.getRequestHeaders().containsKey(key)) {
        return carrier.getRequestHeaders().get(key).get(0);
      }
      return null;
    }

   @Override
   public Iterable<String> keys(HttpExchange carrier) {
     return carrier.getRequestHeaders().keySet();
   } 
};
...
public void handle(HttpExchange httpExchange) {
  // Extract the SpanContext and other elements from the request.
  Context extractedContext = openTelemetry.getPropagators().getTextMapPropagator()
        .extract(Context.current(), httpExchange, getter);
  try (Scope scope = extractedContext.makeCurrent()) {
    // Automatically use the extracted SpanContext as parent.
    Span serverSpan = tracer.spanBuilder("GET /resource")
        .setSpanKind(Span.Kind.SERVER)
        .startSpan();
    try {
      // Add the attributes defined in the Semantic Conventions
      serverSpan.setAttribute(SemanticAttributes.HTTP_METHOD, "GET");
      serverSpan.setAttribute(SemanticAttributes.HTTP_SCHEME, "http");
      serverSpan.setAttribute(SemanticAttributes.HTTP_HOST, "localhost:8080");
      serverSpan.setAttribute(SemanticAttributes.HTTP_TARGET, "/resource");
      // Serve the request
      ...
    } finally {
      serverSpan.end();
    }
  }
}
```

## Metrics (alpha only!)

span是一个非常好的方式，通过它你能获取你的应用当前的细节信息，但是如果想获得一个更加综合的视角呢（more aggregated perspective）？
OpenTelemetry提供了metrics，它是一串时序数字，可能代表着CPU utilization, HTTP server的请求数目
或者业务度量（metric）如交易数量（transactions）。

为了access the alpha metrics library，你需要显式地依赖`opentelemetry-api-metrics`和`opentelemetry-sdk-metrics` 模块。
这些模块目前并没有被包含在opentelemetry-bom中，因为它们还不够稳定，也尚未为long-term-support做好准备。

所有的metrics能用labels来标注，labels是用于描述metric所代表的measurements的一部分标识符。

（All metrics can be annotated with labels: additional qualifiers that help describe what
subdivision of the measurements the metric represents.）

首先，你需要get access to a `MeterProvider`。注意，这里的API是变化的，所以这里没有提供example code。
First, you'll need to get access to a `MeterProvider`. Note the APIs for this are in flux, so no
example code is provided here for that.

下面是使用counter的一个例子：

```java
// Gets or creates a named meter instance
Meter meter = meterProvider.getMeter("instrumentation-library-name", "1.0.0");

// Build counter e.g. LongCounter 
LongCounter counter = meter
        .longCounterBuilder("processed_jobs")
        .setDescription("Processed jobs")
        .setUnit("1")
        .build();

// It is recommended that the API user keep a reference to a Bound Counter for the entire time or 
// call unbind when no-longer needed.
BoundLongCounter someWorkCounter = counter.bind(Labels.of("Key", "SomeWork"));

// Record data
someWorkCounter.add(123);

// Alternatively, the user can use the unbounded counter and explicitly
// specify the labels set at call-time:
counter.add(123, Labels.of("Key", "SomeWork"));
```

`Observer` 是一个附加的instrument，支持异步API和每隔一段时间按要求收集metric data。


下面是使用observer的例子：

```java
// Build observer e.g. LongObserver
LongObserver observer = meter
        .observerLongBuilder("cpu_usage")
        .setDescription("CPU Usage")
        .setUnit("ms")
        .build();

observer.setCallback(
        new LongObserver.Callback<LongObserver.ResultLongObserver>() {
          @Override
          public void update(ResultLongObserver result) {
            // long getCpuUsage()
            result.observe(getCpuUsage(), Labels.of("Key", "SomeWork"));
          }
        });
```

## Tracing SDK Configuration

这篇文档中的配置示例仅仅应用于`opentelemetry-sdk`提供的SDK。
API的其他实现可能提供不同的配置机制。

应用必须安装带有exporter的span processor，并且可能customize OpenTelemetry SDK的行为。

举个例子，一个basic配置实例化SDK tracer provider，并且将trace导出到一个日志流中：

```java
    SdkTracerProvider tracerProvider = SdkTracerProvider.builder().build();
    tracerProvider.addSpanProcessor(BatchSpanProcessor.builder(new LoggingSpanExporter()).build());
```

### Sampler

没有必要trace和export应用中每一个用户请求。
为了在observability（可观测性） 和 expenses（开销）中间取得平衡，可以对traces进行采样。

OpenTelemetry SDK提供了四个开箱即用的采样器（samplers）:
 - [AlwaysOnSampler] which samples every trace regardless of upstream sampling decisions.
 - [AlwaysOffSampler] which doesn't sample any trace, regardless of upstream sampling decisions.
 - [ParentBased] which uses the parent span to make sampling decisions, if present.
 - [TraceIdRatioBased] which samples a configurable percentage of traces, and additionally samples any
   trace that was sampled upstream.

可以通过实现`io.opentelemetry.sdk.trace.Sampler`来添加采样器。

```java
TraceConfig alwaysOn = TraceConfig.getDefault().toBuilder().setSampler(
        Sampler.alwaysOn()
).build();
TraceConfig alwaysOff = TraceConfig.getDefault().toBuilder().setSampler(
        Sampler.alwaysOff()
).build();
TraceConfig half = TraceConfig.getDefault().toBuilder().setSampler(
        Sampler.traceIdRatioBased(0.5)
).build();
// Configure the sampler to use
tracerProvider.updateActiveTraceConfig(
    half
);
```

### Span Processor

OpenTelemetry提供了不同的Span processor。
`SimpleSpanProcessor`直接将ended spans同the exporter相连。
`BatchSpanProcessor`将ended spans打包，一起发送出去。
`MultiSpanProcessor`在作了相应配置后，此Processor可以实现Multiple Span processors的同时处理。


```java
    SdkTracerProvider tracerProvider = SdkTracerProvider.builder().build();
    tracerProvider.addSpanProcessor(
      MultiSpanProcessor.create(Arrays.asList(
        SimpleSpanProcessor.builder(new LoggingSpanExporter()).build(),
        BatchSpanProcessor.builder(new LoggingSpanExporter()).build()
      ))
    );
```

### Exporter

processor是和exporter一起初始化的，后者负责将telemetry data发送至特定的后端（backend）。
OpenTelemetry提供了四个开箱即用的Exporter
- In-Memory Exporter: keeps the data in memory, useful for debugging.
- Jaeger Exporter: prepares and sends the collected telemetry data to a Jaeger backend via gRPC.
- Zipkin Exporter: prepares and sends the collected telemetry data to a Zipkin backend via the Zipkin APIs.
- Logging Exporter: saves the telemetry data into log streams.
- OpenTelemetry Exporter: sends the data to the [OpenTelemetry Collector].

其他的exporter可见[OpenTelemetry Registry].

```java
ManagedChannel jaegerChannel =
    ManagedChannelBuilder.forAddress([ip:String], [port:int]).usePlaintext().build();
    JaegerGrpcSpanExporter jaegerExporter = JaegerGrpcSpanExporter.builder()
      .setServiceName("example").setChannel(jaegerChannel).setDeadline(30000)
      .build()

    SdkTracerProvider tracerProvider = SdkTracerProvider.builder().build();
    tracerProvider.addSpanProcessor(BatchSpanProcessor.builder(jaegerExporter).build()));
```

### TraceConfig

`TraceConfig`和它有关：通过system properties（系统属性）或者environment variables（环境变量）和 builder `set*`方法
来更新（update）Tracer SDK。

```java
  // Get TraceConfig Builder
  TraceConfigBuilder builder = TraceConfig.builder();
  
  // Read configuration options from system properties
  builder.readSystemProperties();
  
  // Read configuration options from environment variables
  builder.readEnvironmentVariables()
  
  // Set options via builder.set* methods, e.g.
  builder.setMaxNumberOfLinks(10);
  
  // Use the resulting TraceConfig instance
  SdkTracerProvider tracerProvider = SdkTracerProvider.builder().setTraceConfig(traceConfig).build();
```

支持的system properties 和 environment variables如下：

| System property                  | Environment variable             | Purpose                                                                                             | 
|----------------------------------|----------------------------------|-----------------------------------------------------------------------------------------------------|       
| otel.config.sampler.probability  | OTEL_CONFIG_SAMPLER_PROBABILITY  | Sampler which is used when constructing a new span (default: 1)                                     |                        
| otel.span.attribute.count.limit  | OTEL_SPAN_ATTRIBUTE_COUNT_LIMIT  | Max number of attributes per span, extra will be dropped (default: 1000)                              |                        
| otel.span.event.count.limit      | OTEL_SPAN_EVENT_COUNT_LIMIT      | Max number of Events per span, extra will be dropped (default: 1000)                                 |                        
| otel.span.link.count.limit       | OTEL_SPAN_LINK_COUNT_LIMIT       | Max number of Link entries per span, extra will be dropped (default: 1000)                            |
| otel.config.max.event.attrs      | OTEL_CONFIG_MAX_EVENT_ATTRS      | Max number of attributes per event, extra will be dropped (default: 32)                             |
| otel.config.max.link.attrs       | OTEL_CONFIG_MAX_LINK_ATTRS       | Max number of attributes per link, extra will be dropped  (default: 32)                             |
| otel.config.max.attr.length      | OTEL_CONFIG_MAX_ATTR_LENGTH      | Max length of string attribute value in characters, too long will be truncated (default: unlimited) |

[AlwaysOnSampler]: https://github.com/open-telemetry/opentelemetry-java/blob/master/sdk/tracing/src/main/java/io/opentelemetry/sdk/trace/samplers/Sampler.java#L29
[AlwaysOffSampler]:https://github.com/open-telemetry/opentelemetry-java/blob/master/sdk/tracing/src/main/java/io/opentelemetry/sdk/trace/samplers/Sampler.java#L40
[ParentBased]:https://github.com/open-telemetry/opentelemetry-java/blob/master/sdk/tracing/src/main/java/io/opentelemetry/sdk/trace/samplers/Sampler.java#L54
[TraceIdRatioBased]:https://github.com/open-telemetry/opentelemetry-java/blob/master/sdk/tracing/src/main/java/io/opentelemetry/sdk/trace/samplers/Sampler.java#L78
[Library Guidelines]: https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/library-guidelines.md
[OpenTelemetry Collector]: https://github.com/open-telemetry/opentelemetry-collector
[OpenTelemetry Registry]: https://opentelemetry.io/registry/?s=exporter
[OpenTelemetry Website]: https://opentelemetry.io/
[Obtaining a Tracer]: https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/api.md#get-a-tracer
[Semantic Conventions]: https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/semantic_conventions
[Instrumentation Library]: https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/glossary.md#instrumentation-library
[instrumented library]: https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/glossary.md#instrumented-library

## Logging and Error Handling 

OpenTelemetry使用[java.util.logging](https://docs.oracle.com/javase/7/docs/api/java/util/logging/package-summary.html)来log 自身的信息。
这其中包含了配置错误，或者export data失败引发的errors 和 warnings.


默认情况下，log信息由你的应用（application）的root handler处理。如果你的应用尚未安装custom root handler
那么`INFO`及以上级别（level）的日志将默认显示在console中。

你可能想改变OpenTelemetry的logger的行为，比如：
debuggging的时候，你想降低日志等级（logging level）
想要忽略某个类的error，你想提高日志等级
或者你想安装一个custom handler
或者当OpenTelemetry日志打印出特定信息时运行特定的代码

下面是例子

### Examples

```properties
## Turn off all OpenTelemetry logging 
io.opentelemetry.level = OFF
```

```properties
## Turn off logging for just the BatchSpanProcessor 
io.opentelemetry.sdk.trace.export.BatchSpanProcessor.level = OFF
```

```properties
## Log "FINE" messages for help in debugging 
io.opentelemetry.level = FINE

## Sets the default ConsoleHandler's logger's level 
## Note this impacts the logging outside of OpenTelemetry as well 
java.util.logging.ConsoleHandler.level = FINE

```
更细粒度的控制，特殊case处理，个性话handler和filter，都能使用code来指定


```java
// Custom filter which does not log errors that come from the export
public class IgnoreExportErrorsFilter implements Filter {

 public boolean isLoggable(LogRecord record) {
    return !record.getMessage().contains("Exception thrown by the export");
 }
}
```

```properties
## Registering the custom filter on the BatchSpanProcessor
io.opentelemetry.sdk.trace.export.BatchSpanProcessor = io.opentelemetry.extension.logging.IgnoreExportErrorsFilter
```
