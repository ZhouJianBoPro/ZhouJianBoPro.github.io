---
layout: post
title: Springboot集成Websocket
date: 2024-08-29
tags: [other]
---

### 什么是Websocket
Websocket是一种在单个tcp连接上进行双向通信的协议。即浏览器端与服务端通过一次握手后建立长连接，
实现客户端和服务端的数据交换。服务端可以主动向客户端发送消息。

### 连接建立过程
![连接建立过程](/images/websocket-link.png)
1. 客户端发起握手请求：发送一个特殊的http请求，请求头中包含需要升级为websocket协议。
2. 服务端握手拦截器：用于websocket在建立连接前的预处理。如日志处理、身份验证和自定义参数处理等。
3. 服务端握手响应：拦截器同意建立连接时返回101状态码，同时响应头包含websocket升级协议。
4. 连接建立成功后处理：如将用户与连接信息绑定，给客户端发送连接成功消息

### 握手拦截器
```java
public class DispatchHandshakeInterceptor implements HandshakeInterceptor {

    @Override
    public boolean beforeHandshake(ServerHttpRequest request, ServerHttpResponse response, WebSocketHandler wsHandler, Map<String, Object> attributes) throws Exception {

        ServletServerHttpRequest servletServerHttpRequest = (ServletServerHttpRequest) request;
        HttpServletRequest httpServletRequest = servletServerHttpRequest.getServletRequest();

        String param = httpServletRequest.getParameter("token");
        attributes.put("token", param);
        return true;
    }

    @Override
    public void afterHandshake(ServerHttpRequest request, ServerHttpResponse response, WebSocketHandler wsHandler, Exception exception) {

    }
}
```
该拦截器用于获取客户端传入的token，每个token绑定了一个WebSocketSession，目的是为了实现用户与连接绑定

### 心跳机制
1. 空闲连接默认在60s内没进行数据交换，会自动断开连接
2. 网络等原因也可能会中断连接，比如服务端重启
3. 心跳机制用于检测连接是否正常，当发现断开时，可以尝试重新建立连接
4. 可以由客户端定时发送心跳请求（不超过60s）, 服务端收到请求后返回响应。该请求不会触发任何业务逻辑

### websocket处理器
```java
@Slf4j
@Component
public class ProduceDispatchHandler implements WebSocketHandler {

    private static final Map<String, WebSocketSession> sessionPool = new ConcurrentHashMap<>();

    /**
     * 连接建立后处理
     * @param session
     */
    @Override
    public void afterConnectionEstablished(WebSocketSession session) {

        Map<String, Object> attributes = session.getAttributes();
        String token = attributes.get("token").toString();

        sessionPool.put(token, session);
        try {
            session.sendMessage(new TextMessage("连接成功"));
        } catch (IOException e) {
            log.error("websocket消息推送失败", e);
        }
    }

    /**
     * 处理客户端消息
     * @param session
     * @param message
     */
    @Override
    public void handleMessage(WebSocketSession session, WebSocketMessage<?> message) {

        try {
            session.sendMessage(new TextMessage("心跳发送成功"));
        } catch (IOException e) {
            log.error("websocket消息推送失败", e);
        }
    }

    @Override
    public void handleTransportError(WebSocketSession session, Throwable exception) {}

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus closeStatus) {
        Map<String, Object> attributes = session.getAttributes();
        String token = attributes.get("token").toString();
        sessionPool.remove(token);
    }

    @Override
    public boolean supportsPartialMessages() {
        return false;
    }

    /**
     * 给客户端发送消息
     * @param message
     */
    public void sendAllMessage(String message) {

        sessionPool.forEach((s, webSocketSession) -> {
            try {
                if(webSocketSession.isOpen()) {
                    webSocketSession.sendMessage(new TextMessage(message));
                }
            } catch (IOException e) {
                log.error("websocket消息推送失败", e);
            }
        });

    }
}
```

### 注册websocket处理器
```java
@Configuration
@EnableWebSocket
public class WebSocketConfiguration implements WebSocketConfigurer {
    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(new ProduceDispatchHandler(), "/dispatchWebsocket")
                .addInterceptors(new DispatchHandshakeInterceptor())
                .setAllowedOrigins("*");
    }
}
```
1. 指定websocket处理器，并配置前端访问路径
2. 为websocket处理器添加拦截器，用于握手阶段建立连接前预处理
3. 设置允许跨域访问路径

### 服务多节点部署问题及解决方案
1. 问题：当服务端需要将消息广播给所有建立连接的客户端时，当前节点只能发送给与该节点建立连接的客户端，无法发送给其他节点的连接客户端
2. 解决方案：使用redis发布订阅或消息队列，将消息广播给所有节点，然后由每个节点根据自身连接情况，选择是否将消息发送给客户端
```java
/**
 * 使用redis发布订阅功能发布websocket消息
 * @param message
 */
private void publishWebsocketMessage(String message) {
    RTopic rTopic = redissonClient.getTopic(RedisKeyConstant.TOPIC_WEBSOCKET);
    rTopic.publish(message);
}

@Component
public class MyApplicationRunner implements ApplicationRunner {

    @Resource
    private RedissonClient redissonClient;

    @Resource
    private ProduceDispatchHandler produceDispatchHandler;

    @Override
    public void run(ApplicationArguments args) throws Exception {

        // 订阅topic
        RTopic topic = redissonClient.getTopic(RedisKeyConstant.TOPIC_WEBSOCKET);

        // 监听topic消息，发送给客户端
        topic.addListener(String.class, (channel, message) -> {
            produceDispatchHandler.sendAllMessage(message);
        });
    }
}
```






