# OpenTelemetry
![Continuous Build](https://github.com/open-telemetry/opentelemetry-java/workflows/Continuous%20Build/badge.svg)
[![Coverage Status][codecov-image]][codecov-url]
[![Maven Central][maven-image]][maven-url]

我们定期举行会议，细节可见[社区页面](https://github.com/open-telemetry/community#java-sdk).


我们使用 [GitHub Discussions](https://github.com/open-telemetry/opentelemetry-java/discussions)
来获取支持或者讨论问题.请不要有任何负担，欢迎参与讨论.

## Overview

OpenTelemetry是由OpenCensus和OpenTracing合二为一的项目.

项目包含了以下几个顶层模块（top level components）:

* [OpenTelemetry API](api/):
  * [stable apis](api/all/src/main/java/io/opentelemetry/api/all/) including `Tracer`, `Span`, `SpanContext`, and `Baggage`
  * [semantic conventions](semconv/) Generated code for the OpenTelemetry semantic conventions.
  * [context api](api/context/src/main/java/io/opentelemetry/context/) The OpenTelmetry Context implementation.
  * [metrics api](api/metrics/src/main/java/io/opentelemetry/api/metrics/) alpha code for the metrics API.
* [extensions](extensions/) define additional API extensions, which are not part of the core API.
* [sdk](sdk/) defines the reference implementation complying to the OpenTelemetry API.
* [sdk-extensions](sdk-extensions/) define additional SDK extensions, which are not part of the core SDK.
* [OpenTracing shim](opentracing-shim/) defines a bridge layer from OpenTracing to the OpenTelemetry API.
* [examples](examples/) on how to use the APIs, SDK, and standard exporters.

我们非常乐于看到社区壮大，并从中听取反馈：请积极地提供反馈和建议.

## Requirements

除非特殊说明之外，所有发布的artifacts支持Java 8及以上。查看[CONTRIBUTING.md](./CONTRIBUTING.md)
获取关于开发过程中构建本项目的指导。

### Note about extensions

API和SDK extensions构成了多样的额外组件，这些组件被排除在core artifacts之外，以防止后者增长过大.
但我们仍然致力于提供和核心组件相同的质量保证，所以如果你发现他们有用，请放心使用他们。

## Project setup and contribute

请参考[contribution guide](CONTRIBUTING.md)来了解如何setup和contribute!

## Quick Start

请参考[quick start guide](QUICKSTART.md) 来了解如何使用OpenTelemetry API.

## Published Releases

已发布的Releases版本可在maven中央仓库（maven central）获取.

### Maven

```xml
  <dependencies>
    <dependency>
      <groupId>io.opentelemetry</groupId>
      <artifactId>opentelemetry-api</artifactId>
      <version>0.13.1</version>
    </dependency>
  </dependencies>
```

### Gradle

```groovy
dependencies {
	implementation('io.opentelemetry:opentelemetry-api:0.13.1')
}
```

## Snapshots

基于`master` 分支的Snapshots版本也可以在下面地址中获取，此版本提供了`opentelemetry-api`, `opentelemetry-sdk` 和剩下的artifacts:

### Maven

```xml
  <repositories>
    <repository>
      <id>oss.sonatype.org-snapshot</id>
      <url>https://oss.jfrog.org/artifactory/oss-snapshot-local</url>
    </repository>
  </repositories>

  <dependencies>
    <dependency>
      <groupId>io.opentelemetry</groupId>
      <artifactId>opentelemetry-api</artifactId>
      <version>0.14.0-SNAPSHOT</version>
    </dependency>
  </dependencies>
```

### Gradle

```groovy
repositories {
	maven { url 'https://oss.jfrog.org/artifactory/oss-snapshot-local' }
}

dependencies {
	implementation('io.opentelemetry:opentelemetry-api:0.14.0-SNAPSHOT')
}
```

Libraries一般只需要`opentelemetry-api`, 但是应用（applications）可能需要使用`opentelemetry-sdk`.

## Releases


OpenTelemetry Java仍然在开发中. 发布的版本（Releases）并不保证基于特定的规范（specfications）实现. 
未来的releases版本将不会保持对之前版本的向后兼容性。

核对信息（check information）请参考 [latest release](https://github.com/open-telemetry/opentelemetry-java/releases).

这是 **当前** feature 状态列表:

| Component                   | Version |
| --------------------------- | ------- |
| Tracing API                 | v<!--VERSION_STABLE-->0.13.1<!--/VERSION_STABLE-->  |
| Tracing SDK                 | v<!--VERSION_STABLE-->0.13.1<!--/VERSION_STABLE-->  |
| Metrics API                 | v<!--VERSION_STABLE-->0.13.1<!--/VERSION_STABLE-->  |
| Metrics SDK                 | v<!--VERSION_STABLE-->0.13.1<!--/VERSION_STABLE-->  |
| OTLP Exporter               | v<!--VERSION_STABLE-->0.13.1<!--/VERSION_STABLE-->  |
| Jaeger Trace Exporter       | v<!--VERSION_STABLE-->0.13.1<!--/VERSION_STABLE-->  |
| Zipkin Trace Exporter       | v<!--VERSION_STABLE-->0.13.1<!--/VERSION_STABLE-->  |
| Prometheus Metrics Exporter | v<!--VERSION_STABLE-->0.13.1<!--/VERSION_STABLE-->  |
| Context Propagation         | v<!--VERSION_STABLE-->0.13.1<!--/VERSION_STABLE-->  |
| OpenTracing Bridge          | v<!--VERSION_STABLE-->0.13.1<!--/VERSION_STABLE-->  |
| OpenCensus Bridge           | N/A     |

可参考项目 [milestones](https://github.com/open-telemetry/opentelemetry-java/milestones)
获取即将到来的（upcoming）版本细节.在issues和milestones中描述的dates和features是基于目前
情况的估计，可能会有所变化。

### Summary

我们计划将项目合二为一，为未来建立一个统一社区铺平道路（pave the path），这个社区服务
tracing vendors，用户和library authors，帮助他们更好的管理应用。我们欢迎所有人提供反馈和建议!

## Contributing

参见 [CONTRIBUTING.md](CONTRIBUTING.md)

Approvers ([@open-telemetry/java-approvers](https://github.com/orgs/open-telemetry/teams/java-approvers)):

- [Armin Ruech](https://github.com/arminru), Dynatrace
- [Pavol Loffay](https://github.com/pavolloffay), Traceable.ai
- [Tyler Benson](https://github.com/tylerbenson), DataDog
- [Giovanni Liva](https://github.com/thisthat), Dynatrace
- [Christian Neumüller](https://github.com/Oberon00), Dynatrace
- [Carlos Alberto](https://github.com/carlosalberto), LightStep

*更多有关approver的信息可见 [community repository](https://github.com/open-telemetry/community/blob/master/community-membership.md#approver).*

Maintainers ([@open-telemetry/java-maintainers](https://github.com/orgs/open-telemetry/teams/java-maintainers)):

- [Bogdan Drutu](https://github.com/BogdanDrutu), Splunk
- [John Watson](https://github.com/jkwatson), Splunk
- [Anuraag Agrawal](https://github.com/anuraaga), AWS

* 更多有关maintainer的信息可见 [community repository](https://github.com/open-telemetry/community/blob/master/community-membership.md#maintainer).*

### 感谢所有参与贡献（have contributed）的人

[![contributors](https://contributors-img.web.app/image?repo=open-telemetry/opentelemetry-java)](https://github.com/open-telemetry/opentelemetry-java/graphs/contributors)

[circleci-image]: https://circleci.com/gh/open-telemetry/opentelemetry-java.svg?style=svg 
[circleci-url]: https://circleci.com/gh/open-telemetry/opentelemetry-java
[codecov-image]: https://codecov.io/gh/open-telemetry/opentelemetry-java/branch/master/graph/badge.svg
[codecov-url]: https://codecov.io/gh/open-telemetry/opentelemetry-java/branch/master/
[maven-image]: https://maven-badges.herokuapp.com/maven-central/io.opentelemetry/opentelemetry-api/badge.svg
[maven-url]: https://maven-badges.herokuapp.com/maven-central/io.opentelemetry/opentelemetry-api
