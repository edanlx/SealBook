gateWay接收到请求以后，从注册中心获取相应服务器进行转发。基于spring WebFlux和netty。在WebFlux中所有的请求都会到org.springframework.web.reactive.DispatcherHandler#handle。

## 4.org.springframework.web.reactive.DispatcherHandler#handle
```java
public Mono<Void> handle(ServerWebExchange exchange) {
	if (this.handlerMappings == null) {
		return createNotFoundError();
	}
	return Flux.fromIterable(this.handlerMappings)
			// 类似springMVC的确定handler
			.concatMap(mapping -> mapping.getHandler(exchange))
			.next()
			.switchIfEmpty(createNotFoundError())
			// 相当于找handlerAdapter
			.flatMap(handler -> invokeHandler(exchange, handler))
			.flatMap(result -> handleResult(exchange, result));
}
```
### 4.1org.springframework.web.reactive.handler.AbstractHandlerMapping#getHandler
```java
public Mono<Object> getHandler(ServerWebExchange exchange) {
	// org.springframework.cloud.gateway.handler.RoutePredicateHandlerMapping#getHandlerInternal
	return getHandlerInternal(exchange).map(handler -> {
		if (logger.isDebugEnabled()) {
			logger.debug(exchange.getLogPrefix() + "Mapped to " + handler);
		}
		ServerHttpRequest request = exchange.getRequest();
		if (hasCorsConfigurationSource(handler) || CorsUtils.isPreFlightRequest(request)) {
			CorsConfiguration config = (this.corsConfigurationSource != null ? this.corsConfigurationSource.getCorsConfiguration(exchange) : null);
			CorsConfiguration handlerConfig = getCorsConfiguration(handler, exchange);
			config = (config != null ? config.combine(handlerConfig) : handlerConfig);
			if (!this.corsProcessor.process(config, exchange) || CorsUtils.isPreFlightRequest(request)) {
				return REQUEST_HANDLED_HANDLER;
			}
		}
		return handler;
	});
}
```
#### 4.1.1org.springframework.cloud.gateway.handler.RoutePredicateHandlerMapping#getHandlerInternal
```java
protected Mono<?> getHandlerInternal(ServerWebExchange exchange) {
	// don't handle requests on management port if set and different than server port
	if (this.managementPortType == DIFFERENT && this.managementPort != null
			&& exchange.getRequest().getURI().getPort() == this.managementPort) {
		return Mono.empty();
	}
	// handlerMapping
	exchange.getAttributes().put(GATEWAY_HANDLER_MAPPER_ATTR, getSimpleName());
	// 查找路由并匹配
	return lookupRoute(exchange)
			// .log("route-predicate-handler-mapping", Level.FINER) //name this
			.flatMap((Function<Route, Mono<?>>) r -> {
				exchange.getAttributes().remove(GATEWAY_PREDICATE_ROUTE_ATTR);
				if (logger.isDebugEnabled()) {
					logger.debug(
							"Mapping [" + getExchangeDesc(exchange) + "] to " + r);
				}
				// 绑定路由信息
				exchange.getAttributes().put(GATEWAY_ROUTE_ATTR, r);
				return Mono.just(webHandler);
			}).switchIfEmpty(Mono.empty().then(Mono.fromRunnable(() -> {
				exchange.getAttributes().remove(GATEWAY_PREDICATE_ROUTE_ATTR);
				if (logger.isTraceEnabled()) {
					logger.trace("No RouteDefinition found for ["
							+ getExchangeDesc(exchange) + "]");
				}
			})));
}
```
### 4.2org.springframework.web.reactive.DispatcherHandler#invokeHandler
```java
private Mono<HandlerResult> invokeHandler(ServerWebExchange exchange, Object handler) {
	if (this.handlerAdapters != null) {
		for (HandlerAdapter handlerAdapter : this.handlerAdapters) {
			if (handlerAdapter.supports(handler)) {
				return handlerAdapter.handle(exchange, handler);
			}
		}
	}
	return Mono.error(new IllegalStateException("No HandlerAdapter: " + handler));
}
```
#### 4.2.1org.springframework.web.reactive.result.SimpleHandlerAdapter#handle
```java
public Mono<HandlerResult> handle(ServerWebExchange exchange, Object handler) {
	WebHandler webHandler = (WebHandler) handler;
	Mono<Void> mono = webHandler.handle(exchange);
	return mono.then(Mono.empty());
}
```
##### 4.2.1.1org.springframework.cloud.gateway.handler.FilteringWebHandler#handle
```java
public Mono<Void> handle(ServerWebExchange exchange) {
	Route route = exchange.getRequiredAttribute(GATEWAY_ROUTE_ATTR);
	// 添加过滤器
	List<GatewayFilter> gatewayFilters = route.getFilters();
	// 添加全局过滤器GatewayFilterAdapter进行转换
	List<GatewayFilter> combined = new ArrayList<>(this.globalFilters);
	combined.addAll(gatewayFilters);
	// TODO: needed or cached?
	AnnotationAwareOrderComparator.sort(combined);

	if (logger.isDebugEnabled()) {
		logger.debug("Sorted gatewayFilterFactories: " + combined);
	}

	return new DefaultGatewayFilterChain(combined).filter(exchange);
}
```
- org.springframework.cloud.gateway.filter.LoadBalancerClientFilter#filter|负载均衡ribbon整合
```java
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
	URI url = exchange.getAttribute(GATEWAY_REQUEST_URL_ATTR);
	String schemePrefix = exchange.getAttribute(GATEWAY_SCHEME_PREFIX_ATTR);
	if (url == null
			|| (!"lb".equals(url.getScheme()) && !"lb".equals(schemePrefix))) {
		return chain.filter(exchange);
	}
	// preserve the original url
	addOriginalRequestUrl(exchange, url);

	if (log.isTraceEnabled()) {
		log.trace("LoadBalancerClientFilter url before: " + url);
	}
	// 整合ribbon负载恒河
	final ServiceInstance instance = choose(exchange);

	if (instance == null) {
		throw NotFoundException.create(properties.isUse404(),
				"Unable to find instance for " + url.getHost());
	}

	URI uri = exchange.getRequest().getURI();

	// if the `lb:<scheme>` mechanism was used, use `<scheme>` as the default,
	// if the loadbalancer doesn't provide one.
	String overrideScheme = instance.isSecure() ? "https" : "http";
	if (schemePrefix != null) {
		overrideScheme = url.getScheme();
	}

	URI requestUrl = loadBalancer.reconstructURI(
			new DelegatingServiceInstance(instance, overrideScheme), uri);

	if (log.isTraceEnabled()) {
		log.trace("LoadBalancerClientFilter url chosen: " + requestUrl);
	}

	exchange.getAttributes().put(GATEWAY_REQUEST_URL_ATTR, requestUrl);
	return chain.filter(exchange);
}
```
- org.springframework.cloud.gateway.filter.NettyRoutingFilter#filter|通过netty发送请求
```java
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
	URI requestUrl = exchange.getRequiredAttribute(GATEWAY_REQUEST_URL_ATTR);

	String scheme = requestUrl.getScheme();
	if (isAlreadyRouted(exchange)
			|| (!"http".equals(scheme) && !"https".equals(scheme))) {
		return chain.filter(exchange);
	}
	setAlreadyRouted(exchange);

	ServerHttpRequest request = exchange.getRequest();

	final HttpMethod method = HttpMethod.valueOf(request.getMethodValue());
	final String url = requestUrl.toASCIIString();

	HttpHeaders filtered = filterRequest(getHeadersFilters(), exchange);

	final DefaultHttpHeaders httpHeaders = new DefaultHttpHeaders();
	filtered.forEach(httpHeaders::set);

	boolean preserveHost = exchange
			.getAttributeOrDefault(PRESERVE_HOST_HEADER_ATTRIBUTE, false);
	Route route = exchange.getAttribute(GATEWAY_ROUTE_ATTR);
	Flux<HttpClientResponse> responseFlux = getHttpClient(route, exchange)
			.headers(headers -> {
				headers.add(httpHeaders);
				// Will either be set below, or later by Netty
				headers.remove(HttpHeaders.HOST);
				if (preserveHost) {
					String host = request.getHeaders().getFirst(HttpHeaders.HOST);
					headers.add(HttpHeaders.HOST, host);
				}
			}).request(method).uri(url).send((req, nettyOutbound) -> {
				if (log.isTraceEnabled()) {
					nettyOutbound
							.withConnection(connection -> log.trace("outbound route: "
									+ connection.channel().id().asShortText()
									+ ", inbound: " + exchange.getLogPrefix()));
				}
				// 使用HttpClient进行send，直接内存
				return nettyOutbound.send(request.getBody().map(this::getByteBuf));
			}).responseConnection((res, connection) -> {
				// 处理响应信息
				// Defer committing the response until all route filters have run
				// Put client response as ServerWebExchange attribute and write
				// response later NettyWriteResponseFilter
				exchange.getAttributes().put(CLIENT_RESPONSE_ATTR, res);
				exchange.getAttributes().put(CLIENT_RESPONSE_CONN_ATTR, connection);

				ServerHttpResponse response = exchange.getResponse();
				// put headers and status so filters can modify the response
				HttpHeaders headers = new HttpHeaders();

				res.responseHeaders().forEach(
						entry -> headers.add(entry.getKey(), entry.getValue()));

				String contentTypeValue = headers.getFirst(HttpHeaders.CONTENT_TYPE);
				if (StringUtils.hasLength(contentTypeValue)) {
					exchange.getAttributes().put(ORIGINAL_RESPONSE_CONTENT_TYPE_ATTR,
							contentTypeValue);
				}

				setResponseStatus(res, response);

				// make sure headers filters run after setting status so it is
				// available in response
				HttpHeaders filteredResponseHeaders = HttpHeadersFilter.filter(
						getHeadersFilters(), headers, exchange, Type.RESPONSE);

				if (!filteredResponseHeaders
						.containsKey(HttpHeaders.TRANSFER_ENCODING)
						&& filteredResponseHeaders
								.containsKey(HttpHeaders.CONTENT_LENGTH)) {
					// It is not valid to have both the transfer-encoding header and
					// the content-length header.
					// Remove the transfer-encoding header in the response if the
					// content-length header is present.
					response.getHeaders().remove(HttpHeaders.TRANSFER_ENCODING);
				}

				exchange.getAttributes().put(CLIENT_RESPONSE_HEADER_NAMES,
						filteredResponseHeaders.keySet());

				response.getHeaders().putAll(filteredResponseHeaders);
				// 返回值
				return Mono.just(res);
			});

	Duration responseTimeout = getResponseTimeout(route);
	if (responseTimeout != null) {
		responseFlux = responseFlux
				.timeout(responseTimeout, Mono.error(new TimeoutException(
						"Response took longer than timeout: " + responseTimeout)))
				.onErrorMap(TimeoutException.class,
						th -> new ResponseStatusException(HttpStatus.GATEWAY_TIMEOUT,
								th.getMessage(), th));
	}
	return responseFlux.then(chain.filter(exchange));
}
```