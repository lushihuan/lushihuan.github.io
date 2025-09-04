---
title: Gateway获取multipart/form-data请求类型参数的可用方法
date: 2023-04-01 00:00:00
tags: Java
---

# 前言

前阵子有个网关项目选型了Spring Cloud Gateway，由于需要进行API签名校验，项目组里又有很多老接口都是multipart/form-data请求类型的，所以就要咋子Spring Cloud Gateway中获取multipart/form-data请求类型的参数用于API签名。这其中走了不少坑，顺便给要做网关项目的人一个忠告......

# 先看看有没有现成的方法可以调用

相信看到这篇文章的人都对Spring Cloud Gateway有所了解，这里就不过多介绍了。过滤器的过滤方法的参数如下所示：

```java
Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain);
```

ServerWebExchange类里有这么一个方法：

```java
/**
 * Return the parts of a multipart request if the Content-Type is
 * {@code "multipart/form-data"} or an empty map otherwise.
 * <p><strong>Note:</strong> calling this method causes the request body to
 * be read and parsed in full and the resulting {@code MultiValueMap} is
 * cached so that this method is safe to call more than once.
 * <p><strong>Note:</strong>the {@linkplain Part#content() contents} of each
 * part is not cached, and can only be read once.
 */
Mono<MultiValueMap<String, Part>> getMultipartData();
```

这不是得来全不费工夫？没怎么简单，由于网关只是一个中间层，参数是要向下传递的，而请求的body只能被读取一次（这一点在注释中也有体现），因此如果使用了这个方法将导致body丢失导致报错。

# 看看度娘有没有好方法

在百度一番搜索中，最终找到一个Github项目：[https://github.com/xurui8691413/sping-cloud-gateway-read-multipart-filter](https://github.com/xurui8691413/sping-cloud-gateway-read-multipart-filter) 。其中关键代码已经被原作者删除了，可以自行查看提交记录。代码是可以成功获取到multipart/form-data请求类型参数的，但是会有内存泄漏问题而且调用到太多WebFlux的底层API了，不太优雅。
这里简单提下为何会有内存泄漏问题，以下是部分关键代码：

```java
@Override
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    return exchange.getRequest().getBody().collectList().flatMap(dataBuffers -> {
        byte[] totalBytes = dataBuffers.stream().map(dataBuffer -> {
            try {
                byte[] bytes = IOUtils.toByteArray(dataBuffer.asInputStream());
                logger.info(new String(bytes));
                return bytes;
            } catch (IOException e) {
                throw new RuntimeException(e);
            }
        }).reduce(this::addBytes).get();
        ServerHttpRequestDecorator decorator = new ServerHttpRequestDecorator(exchange.getRequest()) {
            @Override
            public Flux<DataBuffer> getBody() {
                return Flux.just(buffer(totalBytes));
            }
        };
        Mono<MultiValueMap<String, Part>> multiValueMapMono = repackageMultipartData(decorator, codecConfigurer);
        return multiValueMapMono.flatMap(part->{
            System.out.println(((FormFieldPart) part.getFirst("param")).value());
            return chain.filter(exchange.mutate().request(decorator).build());
        });
    });

}
```

这里可以看见他为了可以重复读取请求body重写了exchange类下的request对象的getBody()方法，这样每次读取body都会构造一份新的DataBuffer对象，问题就出在这个DataBuffer是使用堆外内存创建的，在用后需要release掉，而框架会在请求的最后去release请求的body，如果你调用过多次这个重写过的getBody()方法，框架只会帮你release掉最后一次调用的DataBuffer对象。

# 看看官方文档

在过滤器篇章中，我们可以看见一个用于缓存请求body的过滤器：
[https://docs.spring.io/spring-cloud-gateway/docs/3.1.4/reference/html/#the-cacherequestbody-gatewayfilter-factory](https://docs.spring.io/spring-cloud-gateway/docs/3.1.4/reference/html/#the-cacherequestbody-gatewayfilter-factory)
很可惜，他只能缓存简单的类型，而且只能指定一种类型，这对我们的网关而言，是远远不够的，因为我们支持多种的请求类型。但是请别急，我们看看他的实现。其中的关键代码：

```java
return ServerWebExchangeUtils.cacheRequestBodyAndRequest(exchange, (serverHttpRequest) -> {
    final ServerRequest serverRequest = ServerRequest
            .create(exchange.mutate().request(serverHttpRequest).build(), messageReaders);
    return serverRequest.bodyToMono((config.getBodyClass())).doOnNext(objectValue -> {
        exchange.getAttributes().put(ServerWebExchangeUtils.CACHED_REQUEST_BODY_ATTR, objectValue);
    }).then(Mono.defer(() -> {
        ServerHttpRequest cachedRequest = exchange
                .getAttribute(CACHED_SERVER_HTTP_REQUEST_DECORATOR_ATTR);
        Assert.notNull(cachedRequest, "cache request shouldn't be null");
        exchange.getAttributes().remove(CACHED_SERVER_HTTP_REQUEST_DECORATOR_ATTR);
        return chain.filter(exchange.mutate().request(cachedRequest).build());
    }));
});
```

ServerRequest的bodyToMono()方法是支持复杂类型的入参的，只要我们稍微改造一下就可以成功获取multipart/form-data请求类型参数。代码如下：

```java
package com.lucien.gateway.filter;

import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.support.ServerWebExchangeUtils;
import org.springframework.core.ParameterizedTypeReference;
import org.springframework.http.codec.HttpMessageReader;
import org.springframework.http.codec.multipart.Part;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.stereotype.Component;
import org.springframework.util.Assert;
import org.springframework.util.MultiValueMap;
import org.springframework.web.reactive.function.server.HandlerStrategies;
import org.springframework.web.reactive.function.server.ServerRequest;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.net.URI;
import java.util.List;

import static org.springframework.cloud.gateway.support.ServerWebExchangeUtils.CACHED_SERVER_HTTP_REQUEST_DECORATOR_ATTR;

@Component
public class ReadRequestBodyFilter implements GatewayFilter {

    private final List<HttpMessageReader<?>> defaultMessageReaders;

    private ReadRequestBodyFilter() {
        // 获取默认的messageReaders
        defaultMessageReaders = HandlerStrategies.withDefaults().messageReaders();
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        URI requestUri = request.getURI();
        String scheme = requestUri.getScheme();

        // Record only http requests (including https)
        if ((!"http".equals(scheme) && !"https".equals(scheme))) {
            return chain.filter(exchange);
        }

        Object cachedBody = exchange.getAttribute(ServerWebExchangeUtils.CACHED_REQUEST_BODY_ATTR);
        if (cachedBody != null) {
            return chain.filter(exchange);
        }

        return ServerWebExchangeUtils.cacheRequestBodyAndRequest(exchange, (serverHttpRequest) -> {
            final ServerRequest serverRequest = ServerRequest
                    .create(exchange.mutate().request(serverHttpRequest).build(), defaultMessageReaders);
            return serverRequest.bodyToMono(new ParameterizedTypeReference<MultiValueMap<String, Part>>() {
            }).doOnNext(objectValue -> {
                // 在这里做你想做的事情
                exchange.getAttributes().put(ServerWebExchangeUtils.CACHED_REQUEST_BODY_ATTR, objectValue);
            }).then(Mono.defer(() -> {
                ServerHttpRequest cachedRequest = exchange
                        .getAttribute(CACHED_SERVER_HTTP_REQUEST_DECORATOR_ATTR);
                Assert.notNull(cachedRequest, "cache request shouldn't be null");
                exchange.getAttributes().remove(CACHED_SERVER_HTTP_REQUEST_DECORATOR_ATTR);
                return chain.filter(exchange.mutate().request(cachedRequest).build());
            }));
        });
    }

}
```

# 奇怪的线程挂起问题

当你以为大功告成时，在一定条件下下（可能是body大小大于一定数值）压测时发现压测一段时间后压测线程的请求都会hand住处于类似死锁的假死状态。关闭压测线程重新启动又可以正常请求然后一段时间后又进入假死状态，排查了很久无果后，只在Spring WebFlux的Github Issue里看到可能的原因：[https://github.com/spring-projects/spring-framework/issues/28302](https://github.com/spring-projects/spring-framework/issues/28302) 。虽然我并没有开启问题里提到的stream mode，但是感觉八九不离十是这个DefaultPartHttpMessageReader的问题，这个Reader就是专门解析multipart/form-data请求类型的，而且是后期Spring为了完全响应式而替换了原本的非响应式Reader的（如果你使用的版本没有使用DefaultPartHttpMessageReader则无需往下看了）。因此参试替换会原本的Reader进行压测，故障排除。最终代码：

由于DefaultPartHttpMessageReader替换了SynchronossPartHttpMessageReader，官方已经没有把依赖进行打包。需要添加以下依赖：

```xml
<dependency>
    <groupId>org.synchronoss.cloud</groupId>
    <artifactId>nio-multipart-parser</artifactId>
    <version>1.1.0</version>
</dependency>
```

```java
package com.lucien.gateway.filter;

import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.support.ServerWebExchangeUtils;
import org.springframework.core.ParameterizedTypeReference;
import org.springframework.http.codec.HttpMessageReader;
import org.springframework.http.codec.multipart.MultipartHttpMessageReader;
import org.springframework.http.codec.multipart.Part;
import org.springframework.http.codec.multipart.SynchronossPartHttpMessageReader;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.stereotype.Component;
import org.springframework.util.Assert;
import org.springframework.util.MultiValueMap;
import org.springframework.web.reactive.function.server.ServerRequest;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.net.URI;
import java.util.Collections;
import java.util.List;

import static org.springframework.cloud.gateway.support.ServerWebExchangeUtils.CACHED_SERVER_HTTP_REQUEST_DECORATOR_ATTR;

@Component
public class ReadRequestBodyFilter implements GatewayFilter {

    private final List<HttpMessageReader<?>> multipartMessageReaders;

    private ReadRequestBodyFilter() {
        SynchronossPartHttpMessageReader synchronossPartHttpMessageReader = new SynchronossPartHttpMessageReader();
        // 这里需要设置为-1
        synchronossPartHttpMessageReader.setMaxInMemorySize(-1);
        multipartMessageReaders = Collections.singletonList(new MultipartHttpMessageReader(synchronossPartHttpMessageReader));
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        URI requestUri = request.getURI();
        String scheme = requestUri.getScheme();

        // Record only http requests (including https)
        if ((!"http".equals(scheme) && !"https".equals(scheme))) {
            return chain.filter(exchange);
        }

        Object cachedBody = exchange.getAttribute(ServerWebExchangeUtils.CACHED_REQUEST_BODY_ATTR);
        if (cachedBody != null) {
            return chain.filter(exchange);
        }

        return ServerWebExchangeUtils.cacheRequestBodyAndRequest(exchange, (serverHttpRequest) -> {
            final ServerRequest serverRequest = ServerRequest
                    .create(exchange.mutate().request(serverHttpRequest).build(), multipartMessageReaders);
            return serverRequest.bodyToMono(new ParameterizedTypeReference<MultiValueMap<String, Part>>() {
            }).doOnNext(objectValue -> {
                // 在这里做你想做的事情
                exchange.getAttributes().put(ServerWebExchangeUtils.CACHED_REQUEST_BODY_ATTR, objectValue);
            }).then(Mono.defer(() -> {
                ServerHttpRequest cachedRequest = exchange
                        .getAttribute(CACHED_SERVER_HTTP_REQUEST_DECORATOR_ATTR);
                Assert.notNull(cachedRequest, "cache request shouldn't be null");
                exchange.getAttributes().remove(CACHED_SERVER_HTTP_REQUEST_DECORATOR_ATTR);
                return chain.filter(exchange.mutate().request(cachedRequest).build());
            }));
        });
    }

}
```

# 一点忠告

如果不是专门做网关的部门，没有太多的时间对Spring Cloud Gateway进行学习，**建议不要使用Spring Cloud Gateway**。一开始选型时，没有考虑到Spring WebFlux响应式编程的复杂性，之前在掘金看见字节一篇介绍Zuul网关的文章里说道：*“至于 Zuul1 和 Zuul2 的性能比对，Netflix 给出了一个比较模糊的数据，大致 Zuul2 的性能比 Zuul1 好20% 左右，这里的性能主要指每节点每秒处理的请求数。为什么说模糊呢？因为这个数据受实际测试环境，流量场景模式等众多因素影响，你很难复现这个测试数据。即便这个 20% 的性能提升是确实的，其实这个性能提升也并不大，和异步引入的复杂性相比，这 20% 的提升是否值得是个问题。”* 这篇文章写得很好，感兴趣的可以去看看：[https://juejin.cn/post/7140223037646831647](https://juejin.cn/post/7140223037646831647) 。总而言之，更好的选择应该是**Zuul1**（强如字节都选Zuul1），至少他是常见Servlet、Filter和阻塞模型，排查问题和开发上都会更加方便。

# 后续

今天闲来无事翻看Spring Framework的Github，居然发现上面提到的线程挂起问题似乎被修复了，只需要把Spring Web的版本升级到对应版本即可。Issue地址：[https://github.com/spring-projects/spring-framework/issues/28963](https://github.com/spring-projects/spring-framework/issues/28963) 。

Spring Cloud Gateway也修复了几个相关的内存泄漏问题，需要注意一下对业务有无影响，Issue地址：[https://github.com/spring-cloud/spring-cloud-gateway/pull/2842](https://github.com/spring-cloud/spring-cloud-gateway/pull/2842) ，[https://github.com/spring-cloud/spring-cloud-gateway/pull/2838](https://github.com/spring-cloud/spring-cloud-gateway/pull/2838)。
