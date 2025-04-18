## IM中的websocket设计，使用tomcat websocket

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
}
```

