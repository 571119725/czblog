## IM中的websocket设计

在实现提供 WebSocket 服务的项目中，一般有如下几种解决方案：

- 方案一 [Spring WebSocket](https://docs.spring.io/spring-framework/docs/5.0.0.BUILD-SNAPSHOT/spring-framework-reference/html/websocket.html)
- 方案二 [Tomcat WebSocket](https://www.cnblogs.com/xdp-gacl/p/5193279.html)
- 方案三 [Netty WebSocket](https://netty.io/news/2012/11/15/websocket-enhancement.html)

## 1. 使用tomcat websocket

>  参考文献：https://blog.csdn.net/weixin_42073629/article/details/105607802

### 消息设计

所有消息都需要实现Message接口作为标记接口，其中不会定义任何操作

```java
public interface Message{}
public class AuthRequest implements Message {
  // 静态属性，用于配合不同的消息处理器
  public static final String TYPE = "AUTH_REQUEST";
  private String accessToken;
}
public class SendToOneRequest implements Message {
  public static final String TYPE = "SEND_TO_ONE_REQUEST";
  private String toUser;
  //由客户端生成的唯一id，request和response的id要互相对应
  private String msgId;
}
```

### 消息处理器

通过泛型的设计完成接口MessageHandler对于Message类型信息的处理

```java
public interface MessageHandler<T extends Message> {
  // 对于不同的消息类型进行不同的处理
  void execute(Session session, T message);
  // 消息类型，对应Message实现类的TYPE字段
  String getType();
}
// 实现实例
@Component
public class AuthMessageHandler implements MessageHandler<AuthRequest> {
  @Override
  public void execute(Session session, AuthRequest message) {
    // 通过accessToken进行鉴权
    // 将连接保存到map集合中
    WebSocketUtil.addSession(session, userId);
    // 回复登陆成功，通过TYPE来调用不同的Message处理器
    WebSocketUtil.send(session, AuthResponse.TYPE, new AuthResponse().setCode(0));
  }
  @Override 
  public String getType() {
    return AuthResponse.TYPE;
  }
}
```

### 连接管理

WebSocketUtil 连接管理，包括保存与用户的长连接，连接的添加和删除以及通过session发送消息

```java
public class WebSocketUtil {
  private static final Map<Session, Integer> SESSION_USER_MAP = new ConcurrentHashMap<>();
  private static final Map<Integer, Session> USER_SESSION_MAP = new ConcurrentHashMap<>();
  public static void addSession(Session session, Integer user){}
  public static void removeSession(Session session) {
    Integer user = SESSION_USER_MAP.remove(session);
    if(user != null) {
      USER_SESSION_MAP.remove(user);
    }
  }
  public static <T extends Message> boolean send(int user, String type, T message){}
}
```

### 完善消息处理

```java
// WebSocketConfiguration.java
 
@Configuration
// @EnableWebSocket // 无需添加该注解，因为我们并不是使用 Spring WebSocket
public class WebSocketConfiguration {
 		// 该Bean的作用是扫描添加有@ServerEndpoint注解的Bean
    @Bean
    public ServerEndpointExporter serverEndpointExporter() {
        return new ServerEndpointExporter();
    }
 
}

@Controller
@ServerEndpoint("/")
public class WebSocketEndpoint implements InitializingBean {
  /**
   * 消息类型与 MessageHandler 的映射
   *
   * 注意，这里设置成静态变量。虽然说 WebsocketServerEndpoint 是单例，但是 Spring Boot 还是会为每个 WebSocket 创建一个 WebsocketServerEndpoint Bean 。
   */
  private static final Map<String, MessageHandler> HANDLERS = new HashMap<>();
  // BeanFactory的一个子类，可以获取Bean，并查看Bean的某些信息（是否存在，是否是单例等等），可以将其视为Bean的容器
  @Autowired 
  private ApplicationContext applicationContext; 
  @Override
  public void OnOpen(Session session, EndpointConfig config){}
  // 避免消息处理器调整时的手动配置
  @Override
  public void afterPropertiesSet() throw Exception {
    applicationContext.getBeansOfType(MessageHandler.class).value()
      .forEach(messageHandler -> HANDLERS.put(messageHandler.getType(), messageHandler));
  }
  @OnOpen
  public void onOpen(Session session, @PathParam("token") String token) {}
}
```

## 2. 使用Spring WebSocket

因为 Tomcat WebSocket 使用的是 Session 作为会话，而 Spring WebSocket 使用的是 WebSocketSession 作为会话，导致我们需要略微修改下 WebSocketUtil 工具类。改动非常略微。 主要有两点：

将所有使用 Session 类的地方，调整成 WebSocketSession 类。
将发送消息，从 Session 修改成 WebSocketSession 。

### WebSocket握手拦截器

WebSocketSession无法获得ws地址上的请求参数，所以需要通过拦截器获取accessToken参数并设置到attribute中

```java
public class DemoWebSocketShakeInterceptor extends HttpSessionHandshakeInterceptor {
    @Override
    public boolean beforeHandshake(ServerHttpRequest request, ServerHttpResponse response, WebSocketHandler wsHandler, Map<String, Object> attributes) throws Exception {
        if(request instanceof ServletServerHttpRequest) {
            ServletServerHttpRequest serverRequest = (ServletServerHttpRequest) request;
            attributes.put("accessToken", serverRequest.getServletRequest().getParameter("accessToken"));
        }
        return super.beforeHandshake(request, response, wsHandler, attributes);
    }
}
```

### 消息处理器

```java
public class DemoWebSocketHandler extends TextWebSocketHandler implements InitializingBean {
    // Spring Websocket 是单例的形式，所以不需要添加static关键字
    private final Map<String, MessageHandler> HANDLERS = new HashMap<>();
    @Autowired
    private ApplicationContext applicationContext;

    @Override
    public void afterConnectionEstablished(WebSocketSession session) throws Exception {
    }

    @Override
    public void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {
    }
    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) throws Exception {
    }
    @Override
    public void afterPropertiesSet() throws Exception {

    }
}
```

### 配置类

Bean的装配以及拦截器的注册

```java
// WebSocketConfiguration.java
 
@Configuration
@EnableWebSocket // 开启 Spring WebSocket
public class WebSocketConfiguration implements WebSocketConfigurer {
 
    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(this.webSocketHandler(), "/") // 配置处理器
                .addInterceptors(new DemoWebSocketShakeInterceptor()) // 配置拦截器
                .setAllowedOrigins("*"); // 解决跨域问题
    }
 
    @Bean
    public DemoWebSocketHandler webSocketHandler() {
        return new DemoWebSocketHandler();
    }
 
    @Bean
    public DemoWebSocketShakeInterceptor webSocketShakeInterceptor() {
        return new DemoWebSocketShakeInterceptor();
    }
 
}
```

