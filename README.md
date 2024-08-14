# 方案设计

## websocket 通信

此需求的难点之一在于如何选取合适的通信方式, 传统的http请求只支持客户端主动向服务端发起通信, 这里需要服务端向客户端通信的能力。

这里选用应用最广泛且拓展性最强WebSocket做为服务端向客户端发数据技术方案。

### 使用案例

> websocket会话和用户绑定细节

- 引入`pom.xml`: 使用`springboot`官方提供的`start`

```xml
 <!--websocket-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

- 添加一个 `websocket` 配置类

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.config.annotation.EnableWebSocket;
import org.springframework.web.socket.server.standard.ServerEndpointExporter;

/**
 * websocket的配置类
 */
@Configuration
@EnableWebSocket
public class WebSocketConfiguration {
    /**
     * 这个配置类的作用是要注入ServerEndpointExporter，会自动注册使用了@ServerEndpoint注解声明的Websocket endpoint
     * 如果采用tomcat容器进行部署启动，而不是直接使用springboot的内置容器
     * 就不要注入ServerEndpointExporter，因为它将由容器自己提供和管理。
     */
    @Bean
    public ServerEndpointExporter serverEndpointExporter() {
        return new ServerEndpointExporter();
    }
}
```

- 创建`websocket`核心处理类，用于处理连接和消息的相关操作

```java
package com.haiskynology.mall.worldstreet.server.websocket;

import lombok.NonNull;
import lombok.SneakyThrows;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import javax.websocket.*;
import javax.websocket.server.PathParam;
import javax.websocket.server.ServerEndpoint;
import java.io.IOException;
import java.util.concurrent.ConcurrentHashMap;

/**
 * websocket处理创建、推送、接受、关闭类
 * ServerEndpoint 定义websocket的监听连接地址
 */

@Slf4j
@Component
@ServerEndpoint("/ws/{appId}")
public class WebSocketServer {
    /**
     * 用来存放每个客户端对应的 Session 对象, session对象存储着连接信息
     */
    private static final ConcurrentHashMap<String, Session> webSocketMap = new ConcurrentHashMap<>();

    /**
     * 创建连接
     */
    @SneakyThrows
    @OnOpen
    public void onOpen(Session session, @PathParam("appId") String appId) {
        webSocketMap.put(session.getId(), session);
        sendMessage(session, session.getId(), String.format("连接成功 (appId=%s), id: %s", appId, session.getId()));
        log.info(String.format("【%s】连接成功！(appId=%s)", session.getId(), appId));
    }

    /**
     * 根据消息体内容发送消息
     */
    @SneakyThrows
    private void sendMessage(@NonNull  Session session, @NonNull String targetId, @NonNull String message){
        Session targetSession = webSocketMap.get(targetId);
        if(targetSession == null){
            session.getBasicRemote().sendText(String.format("目标id(%s)不在线", targetId));
            return;
        }

        targetSession.getBasicRemote().sendText(message);
    }

    /**
     * 接受消息
     */
    @OnMessage
    public void onMessage(Session session, String message) {
        log.info(String.format("用户【%s】发送信息: %s", session.getId(), message));

        // 消息格式: [对方id],[发送消息]
        String[] split = message.split(",");
        if(split.length == 2){
            String targetId = split[0];
            String targetMsg = split[1];
            sendMessage(session, targetId, targetMsg);
        } else {
            sendMessage(session, session.getId(), "消息格式错误, 正确格式: [对方id],[发送消息]");
        }
    }

    /**
     * 关闭连接
     */
    @OnClose
    public void onClose(Session session) {
        try {
            webSocketMap.remove(session.getId()).close();
            log.info(String.format("用户【%s】关闭连接成功！", session.getId()));
        } catch (IOException e) {
            log.error(String.format("用户【%s】关闭连接失败！", session.getId()));
        }
    }

    /**
     * 发生错误
     */
    @OnError
    public void onError(Throwable error, Session session) {
        log.info(String.format("用户【%s】发送错误！", session.getId()));
        error.printStackTrace();
    }
}
```

### 多节点通信

> 主要参考文献  
> - [https://gitee.com/searonhe/websocket-redis-demo](https://gitee.com/searonhe/websocket-redis-demo)

#### 解决思路

对于使用案例里的例子,存在一个问题: **只能单节点内通信, 不可以实现多节点通信**。而我们的应用场景里又恰恰需要多节点通信。

我首先想到的办法就是将websocket会话序列化存在缓存里,这样每个节点都能访问了,但是很可惜`javax.websocket.Session`不能序列化,细想过后我发现这种思路有很多地方都行不通,和某一个节点建立的长连接怎么可能转移另一个节点上面呢?

于是我只能转变思路,查询了大量的资料过后,我了解到目前比较常用的解决方案就是使用消息队列或者redis发布订阅模式,虽然实现方式不同,但思路是一样的——使用**发布订阅模式**:所有节点订阅一个主题,有新消息时往这个主题里推消息,所有节点都去消费这条消息,但只有自己节点内有目标websocket连接的节点才能成功消费消息。

这里我选择采用redis发布订阅模式, 思路如下:

- ~~把"websocket是否为多节点"作为一个可配置项，配置在~~`~~application.yml~~`~~或者~~`~~mysql~~`~~某一个静态表中，当发送消息时，判断是否开启了多节点:~~

- ~~如果没有开启多节点: 直接在当前节点内发消息即可。~~
- ~~如果开启了多节点:不会直接发送消息，而是使用Redis的订阅发布者模式，向指定的通道推送一条消息，所有的开启了多节点模式的websocket应用都去监听这个主题，当节点收到消息的时候，就去找自己这里有没有对应的消息接收者的Session,如果有则发送消息,如果没有不做任何的操作~~

- 发送消息时,向指定的通道推送一条消息, 所有的开启了多节点模式的websocket应用都去监听这个主题
- 当节点收到消息的时候，就去找自己这里有没有对应的消息接收者的Session

- 如果有则发送消息
- 如果没有不做任何的操作

这样就可以确保这条消息一定会被消费

##### 流程

- 推送服务需要发消息时往目标主题里推新消息
- 消费服务是订阅了这个主题的服务

![](http://www.kdocs.cn/api/v3/office/copy/NlZIM0RLYzRxK0U3ck5aWS81SXZMVEF2WmxUaXpjbHFaSDF5a3VtTXJlR0dPNnBCcGJuM2hRTVN1UTFsVzFYVGs5VkltSHVKdktUNm1lbU1XQ010WjFLek13RVZBN1JhVnlDM1R4cW9ZeTltcGltL0RNcTYyYjJPSTJ5VUl1aUhTRUZuZkprMFB1S0k3YjdnUG14am1Fdm5JQ3Ztc0crRmRFRGJNaFU5UHJnaVlJelBkTEhjM3k3VndheEFnenpyclErRGYyZmFOZ25CR1lpdXNZb281U1dvV1lVZEJYa0ZwdDFmU0psb0hBWkg5NEcwZCt5RkpaazY2YjZYTk5nZitpMDZ2azZ3SU5RPQ==/attach/object/NDR7MVQ2ABADS? "po_bhccfifjjhfgja")

#### 实现案例

redis引入依赖项和配置略过, 用springboot那一套

- 推送消息时: 使用redis发布消息

```java
    /**
     * 接受消息
     */
    @OnMessage
    public void onMessage(String message) {
        stringRedisTemplate.convertAndSend(RedisConfig.REDIS_CHANNEL, message);
    }
```

- redis配置

```java
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.listener.PatternTopic;
import org.springframework.data.redis.listener.RedisMessageListenerContainer;
import org.springframework.data.redis.listener.adapter.MessageListenerAdapter;

/**
 * redis配置
 */
@Configuration
@EnableCaching
public class RedisConfig {
    /**
     * 定义信道名称
     */
    public static final String REDIS_CHANNEL = "wsMessage";

    @Bean
    RedisMessageListenerContainer container(RedisConnectionFactory connectionFactory, MessageListenerAdapter listenerAdapter) {
        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);
        // 订阅消息频道
        container.addMessageListener(listenerAdapter, new PatternTopic(REDIS_CHANNEL));
        return container;
    }

    @Bean
    MessageListenerAdapter listenerAdapter(RedisReceiver receiver) {
        // 消息监听适配器
        return new MessageListenerAdapter(receiver, "onMessage");
    }

    @Bean
    StringRedisTemplate template(RedisConnectionFactory connectionFactory) {
        return new StringRedisTemplate(connectionFactory);
    }
}
```

- 信道消息监听器(订阅消息)

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.connection.Message;
import org.springframework.data.redis.connection.MessageListener;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;

/**
 * 消息监听对象，接收订阅消息
 */
@Component
@Slf4j
public class RedisReceiver implements MessageListener {

    @Resource
    private WebSocketServer webSocketServer; // 之前定义的websocket服务节点

    /**
     * 处理接收到的订阅消息
     */
    @Override
    public void onMessage(Message message, byte[] pattern) {
        // 订阅的频道名称
        String channel = new String(message.getChannel());
        String msg = "";
        try {
            msg = new String(message.getBody());
            if (!StringUtils.isEmpty(msg)) {
                if (RedisConfig.REDIS_CHANNEL.endsWith(channel)) {
                    webSocketServer.sendMessage(msg);
                } else {
                    // todo 处理其他订阅的消息
                }
            } else {
                log.info("消息内容为空，不处理。");
            }
        } catch (Exception e) {
            log.error("处理消息异常：" + e.toString());
            e.printStackTrace();
        }
    }
}
```

### 连接时机

> [https://juejin.cn/post/7248918622693048379](https://juejin.cn/post/7248918622693048379)

- 初次建立连接: 进入app时建立连接
- 断开连接: 退出app时断开连接
- 重连: 当意外断开连接时, 尝试重连 , 严格来说是每次网络连接后都重新连接websocket

### 连接设备限制

连接设备与登录设备保存一致, 登录那里做了设备限制, 这里连接只需要限制一台设备只有一个连接即可。

### 通信协议

#### 客户端 -> 服务端

```js
{
    "authorization": token, // token, 根据这个解析用户信息
    "path": "connect/hello", // 根据这个来执行不同的业务逻辑
    "body": { // 参数
        "param1": 1,
        "param2": "hello world"
    }
}
```

- token为用户登录时获取的token, 这里解析token沿用http那一套

- 如果token过期, 需要通知客户端

- path是具体的业务逻辑区分字段, 类似于请求路径
- body 是执行业务函数时传的参数, 类似于请求参数

#### 服务端 -> 客户端

```js
{
    "path": "group/message", // 根据这个来执行不同的业务逻辑
    "body": { // 参数
        "param1": 1,
        "param2": "hello world"
    }
}
```

- path是具体的业务逻辑区分字段, 类似于请求路径
- body 是执行业务函数时传的参数, 类似于请求参数

## 会话聊天室

> [https://github.com/zongzibinbin/MallChat/blob/main/docs/mallchat.sql](https://github.com/zongzibinbin/MallChat/blob/main/docs/mallchat.sql)

> 把私聊看成特殊的群聊, 加个字段区分

### 成员类型

将会话看做聊天室, 一共有两种成员类型, 店铺 和 普通用户, 使用字段`type`区分

- 店铺(`shop`): 该店铺中的所有管理员均可访问该聊天室
- 普通用户(`user`): 该用户可访问该聊天室

两个成员之间可以私聊,对于私聊, 直接将成员id做为字段存在聊天室表里面

```sql
create table private_room(
    ...
    member_type1 varchar(32) not null COMMENT '成员1类型, user-用户, shop-店铺', 
    member_id1 bigint unsigned not null COMMENT '成员1id',
    member_type2 varchar(32) not null COMMENT '成员2类型, user-用户, shop-店铺', 
    member_id2 bigint unsigned not null COMMENT '成员2id',
    ...
```

可以发起群聊,对于群聊,限制最大人数(500),将聊天室里面的成员单独放在另一个表里面

```sql
create table room_member(
    ...
    room_id bigint unsigned not null, -- 所属聊天室id
    member_type varchar(32) not null, -- 成员类型
    member_id bigint unsigned not null, -- 成员id
    ...
```

### 查询规则

- 查询的时候有未读消息的聊天室排在最前面, 按照时间排序

- 可能会存在返回数据规模过大的问题, 需要分页,前端下拉懒加载

## 消息

### 消息类型

- 一条消息对应一种消息类型, 用字段 `type` 区分, 字段 `content` 存具体内容

```js
// 文本
  {
    "type": "text",
    "content": {
      "text": "123"
    }
  }

// 图片
  {
    "type": "image",
    "content": {
        "normal": "http://xxx/hello.png",
        "thumbnail": "http://xxx/hello1.png",
        "original": "http://xxx/hello2.png"
    }
  }
```

### 引用消息

使用一个字段`ref_id`指定引用的消息id, 如果没有引用消息则`ref_id = null`

### 新消息

有两张表会记录新消息: **消息表**和**未读消息表**, 消息表记录所有消息,未读消息表记录未读消息(涉及到已读未读的逻辑在后面),如果某一条消息被读了,就删除对应的未读消息数据。有点类似于MySQL的binlog和redo-log, binlog记录全局日志, redo-log只记录待同步的数据日志, 不同点在于redo-log未处理时是`prepare`状态, 处理后是`commit`状态, 而这里的未读消息已读后就直接删除了。

### 已读未读

> [http://www.52im.net/thread-3054-1-1.html](http://www.52im.net/thread-3054-1-1.html)

主要策略是: 有新消息时构建全员(除了发送者)的未读消息, 如果某个用户已读, 删除对应的未读消息。

因为一个店铺对应多个管理员, 所以一个这里的全员是包括店铺的所有管理员和买家,例如, 现在有个店铺叫做"小明的店铺", 其中有管理员"小明","小红","小亮",那么这个聊天室的全员就是["小明","小红","小亮","买家"]

流程如下:

- 买家发送一条信息时, 构建全员(除了发送者)的未读消息,插入未读消息表
- 小明读到某条消息时发送这条消息的消息已读请求,这时删除对应的消息未读表

![](http://www.kdocs.cn/api/v3/office/copy/NlZIM0RLYzRxK0U3ck5aWS81SXZMVEF2WmxUaXpjbHFaSDF5a3VtTXJlR0dPNnBCcGJuM2hRTVN1UTFsVzFYVGs5VkltSHVKdktUNm1lbU1XQ010WjFLek13RVZBN1JhVnlDM1R4cW9ZeTltcGltL0RNcTYyYjJPSTJ5VUl1aUhTRUZuZkprMFB1S0k3YjdnUG14am1Fdm5JQ3Ztc0crRmRFRGJNaFU5UHJnaVlJelBkTEhjM3k3VndheEFnenpyclErRGYyZmFOZ25CR1lpdXNZb281U1dvV1lVZEJYa0ZwdDFmU0psb0hBWkg5NEcwZCt5RkpaazY2YjZYTk5nZitpMDZ2azZ3SU5RPQ==/attach/object/W7IVYZQ2ACQAO? "po_bhcciedbffifea")

### 撤回消息

将撤回消息看做一种特殊的新消息,有撤回消息时删除撤回的那条消息内容, 更新状态, 生成一条新消息来表示用户撤回了一条消息,给撤回消息专门定一个消息类型。

撤回消息类型有以下几个特点:

- 不可撤回
- 客户端展示时展示为这种形式

![](http://www.kdocs.cn/api/v3/office/copy/NlZIM0RLYzRxK0U3ck5aWS81SXZMVEF2WmxUaXpjbHFaSDF5a3VtTXJlR0dPNnBCcGJuM2hRTVN1UTFsVzFYVGs5VkltSHVKdktUNm1lbU1XQ010WjFLek13RVZBN1JhVnlDM1R4cW9ZeTltcGltL0RNcTYyYjJPSTJ5VUl1aUhTRUZuZkprMFB1S0k3YjdnUG14am1Fdm5JQ3Ztc0crRmRFRGJNaFU5UHJnaVlJelBkTEhjM3k3VndheEFnenpyclErRGYyZmFOZ25CR1lpdXNZb281U1dvV1lVZEJYa0ZwdDFmU0psb0hBWkg5NEcwZCt5RkpaazY2YjZYTk5nZitpMDZ2azZ3SU5RPQ==/attach/object/FTUKWZQ2AAAEE?)

> 对方还没收到就撤回了不显示

![](http://www.kdocs.cn/api/v3/office/copy/NlZIM0RLYzRxK0U3ck5aWS81SXZMVEF2WmxUaXpjbHFaSDF5a3VtTXJlR0dPNnBCcGJuM2hRTVN1UTFsVzFYVGs5VkltSHVKdktUNm1lbU1XQ010WjFLek13RVZBN1JhVnlDM1R4cW9ZeTltcGltL0RNcTYyYjJPSTJ5VUl1aUhTRUZuZkprMFB1S0k3YjdnUG14am1Fdm5JQ3Ztc0crRmRFRGJNaFU5UHJnaVlJelBkTEhjM3k3VndheEFnenpyclErRGYyZmFOZ25CR1lpdXNZb281U1dvV1lVZEJYa0ZwdDFmU0psb0hBWkg5NEcwZCt5RkpaazY2YjZYTk5nZitpMDZ2azZ3SU5RPQ==/attach/object/TBP2UZQ2ABQHE?)

流程图如下:

![](http://www.kdocs.cn/api/v3/office/copy/NlZIM0RLYzRxK0U3ck5aWS81SXZMVEF2WmxUaXpjbHFaSDF5a3VtTXJlR0dPNnBCcGJuM2hRTVN1UTFsVzFYVGs5VkltSHVKdktUNm1lbU1XQ010WjFLek13RVZBN1JhVnlDM1R4cW9ZeTltcGltL0RNcTYyYjJPSTJ5VUl1aUhTRUZuZkprMFB1S0k3YjdnUG14am1Fdm5JQ3Ztc0crRmRFRGJNaFU5UHJnaVlJelBkTEhjM3k3VndheEFnenpyclErRGYyZmFOZ25CR1lpdXNZb281U1dvV1lVZEJYa0ZwdDFmU0psb0hBWkg5NEcwZCt5RkpaazY2YjZYTk5nZitpMDZ2azZ3SU5RPQ==/attach/object/DYME62Y2AAACY? "po_bhcciedbffifea")

撤回的那条消息只是消息表删除对应的消息内容, 未读消息表不删除撤回的那条消息,理由如下:

我们考虑两种情况:

1. 撤回的那条信息客户端已经收到了: 此时在撤回的那条信息那里显示为"对方撤回了一条消息"即可
2. 撤回的那条消息客户端没收到: 如果删除未读消息表里撤回的那条消息, 按照前面的思路就行不通了, 因为用户都还没收到撤回的那条消息, 根本就不知道它的位置, 怎么在那个位置标记"对方撤回了一条信息", 所以未读消息表不删除撤回的那条消息。

### 查询未读消息

当用户初次进入app时, 需要查询自上一次退出app后的所有未读消息, 主要查询的表是未读消息表(unread_message), 其中涉及的字段有:

- room_id: 聊天室id
- message_id: 消息id
- uid: 用户id
- create_time: 创建时间

查询未读消息的流程如下:

> 只能查到未读的

- 首先查询该用户有未读消息的聊天室及其消息个数,按照时间排序, 大致sql参考下面:

- 可能会存在返回数据规模过大的问题, 需要分页,前端下拉懒加载

```java
SELECT 
    room_id,
    COUNT(message_id) AS unread_count, -- 消息个数
    MAX(create_time) AS last_message_time -- 最后一条消息创建时间
FROM 
    unread_message
WHERE 
    uid = ? -- 这里的 ? 需要被实际的用户ID替换
GROUP BY 
    room_id
ORDER BY 
    last_message_time DESC
; -- 根据最后一条消息的创建时间排序
```

- 对于上方查询到的聊天室,还要查询其最后一条消息

- 这里看起来可以在未读消息表里查,也可以去消息表里查, 都是去查创建时间最晚的那条消息, 但是考虑到有的聊天室没有未读消息, 所以还是要去消息表里查

> 对于撤回消息这种,查询的不是撤回的那条消息,是撤回时新生成的那条专门表示撤回的消息(细节见"撤回消息")

- 当用户点进某个聊天室的时候, 查询该聊天室的所有未读消息

- 可能会存在返回数据规模过大的问题, 需要分页,前端上拉懒加载

流程图如下:

![](http://www.kdocs.cn/api/v3/office/copy/NlZIM0RLYzRxK0U3ck5aWS81SXZMVEF2WmxUaXpjbHFaSDF5a3VtTXJlR0dPNnBCcGJuM2hRTVN1UTFsVzFYVGs5VkltSHVKdktUNm1lbU1XQ010WjFLek13RVZBN1JhVnlDM1R4cW9ZeTltcGltL0RNcTYyYjJPSTJ5VUl1aUhTRUZuZkprMFB1S0k3YjdnUG14am1Fdm5JQ3Ztc0crRmRFRGJNaFU5UHJnaVlJelBkTEhjM3k3VndheEFnenpyclErRGYyZmFOZ25CR1lpdXNZb281U1dvV1lVZEJYa0ZwdDFmU0psb0hBWkg5NEcwZCt5RkpaazY2YjZYTk5nZitpMDZ2azZ3SU5RPQ==/attach/object/CXKWG2Y2ABQG6? "po_bhcciehceagaca")

### 消息通知

当用户在app内(严格来说是websocket已连接未断开)时, 此用户有新消息时需要服务端推送消息通知给用户。

对于新消息, 客户端需要做以下处理:

- 将消息同步在本地

- 新消息这种直接加数据
- 撤回消息需要删除撤回的那一条消息内容

- 将新消息属于的聊天室顺序移到列表第一个
- 如果当前页面就是在某个聊天室里, 且此聊天室就是消息通知的聊天室, 直接把消息标记为已读

### 上拉刷新

一次最多显示100条消息, 当用户上拉时, 优先查询本地存的消息记录(细节见"缓存方案"), 如果本地没有了, 再去服务端查询。

> 优化：记录服务端和本地数据是否同步

> 正常情况本地消息记录和远程是同步的, 因为本地消息记录都是一次次查询未读消息和未读消息通知的合集。  
> 特殊场景比如用户换手机了, 本地没有数据, 此时只能去服务端查数据了。

## 缓存方案

使用sqlite存本地数据, 结构如下:

- 聊天记录表: 用于记录每一条聊天内容的信息，包括发送者、接收者、内容及时间戳等。
- 聊天会话表: 用于管理用户之间的会话信息，存储参与会话的用户及最后一条消息的引用, 未读消息个数等。

客户端表结构与后端类似, 以下是他们之间的一些区别

1. **发送失败:** 对于发送失败的消息, 客户端记录在聊天记录表里, 消息前面显示一个红色感叹号([❗](https://emojipedia.org/zh/%E7%BA%A2%E8%89%B2%E6%84%9F%E5%8F%B9%E5%8F%B7)),点击后可以重新发送。
2. **已读未读:**

1. 客户端对于未读消息直接记录在聊天记录表里, 用某一个字段区分, 不需要像后端那样单独分个表处理。
2. 本地收到了消息存在本地但是用户未读的话, 后端仍然是未读状态, 下次去查询未读消息还是可以查到这条消息。需要用户阅读后再通知后端, 这时后端消息才记录为已读。这里有两点需要注意:

1. 对于未读信息, 因为如果用户未读某条信息就退出了app, 下次再进入还是会查询到这条信息。
2. 多次查询时, 未读信息必须要从后端查, 不能查本地保存的未读信息, 考虑用户换设备或者多个设备的情况, 每个设备的未读信息都必须依赖服务端而不是本地。

> 优化  
> 历史消息: 根据本地最早的消息的创建时间create_time, 服务端查询 < create_time 的消息

1. **撤回消息:** 自己撤回的消息本地不会删除(可以重新编辑)
2. **重新编辑**: 消息框内的消息没发出去后，用PrefUtils进行缓存，方便用户退出会话、退出App后再次编辑

> 分页查询重复数据, 消息时序问题  
> 点进聊天室, 不用查询未读消息列表

![](http://www.kdocs.cn/api/v3/office/copy/NlZIM0RLYzRxK0U3ck5aWS81SXZMVEF2WmxUaXpjbHFaSDF5a3VtTXJlR0dPNnBCcGJuM2hRTVN1UTFsVzFYVGs5VkltSHVKdktUNm1lbU1XQ010WjFLek13RVZBN1JhVnlDM1R4cW9ZeTltcGltL0RNcTYyYjJPSTJ5VUl1aUhTRUZuZkprMFB1S0k3YjdnUG14am1Fdm5JQ3Ztc0crRmRFRGJNaFU5UHJnaVlJelBkTEhjM3k3VndheEFnenpyclErRGYyZmFOZ25CR1lpdXNZb281U1dvV1lVZEJYa0ZwdDFmU0psb0hBWkg5NEcwZCt5RkpaazY2YjZYTk5nZitpMDZ2azZ3SU5RPQ==/attach/object/PDJV64A2ADQEQ?)

# 数据库

- 私聊聊天室

```
create table private_room(
    id bigint unsigned NOT NULL AUTO_INCREMENT COMMENT '自增主键',
    code varchar(80) NOT NULL COMMENT '唯一编码, 区分私聊和群聊',
    member_type1 varchar(32) not null COMMENT '成员1类型, user-用户, shop-店铺', 
    member_id1 bigint unsigned not null COMMENT '成员1id',
    member_type2 varchar(32) not null COMMENT '成员2类型, user-用户, shop-店铺', 
    member_id2 bigint unsigned not null COMMENT '成员2id',
    create_time datetime(3) DEFAULT CURRENT_TIMESTAMP(3) COMMENT '创建时间',
    update_time datetime(3) DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '更新时间',
    primary key (`id`),
    KEY `idx_code` (`code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='私聊聊天室';
```

- 群聊聊天室

```
create table group_room(
    id bigint unsigned NOT NULL AUTO_INCREMENT COMMENT '自增主键',
    code varchar(80) NOT NULL COMMENT '唯一编码, 区分私聊和群聊',
    limit_num int default 500 not null comment '群聊限制人数',
    create_time datetime(3) DEFAULT CURRENT_TIMESTAMP(3) COMMENT '创建时间',
    update_time datetime(3) DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '更新时间',
    primary key (`id`),
    KEY `idx_code` (`code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='群聊聊天室';
```

- 群聊聊天室成员

```
create table room_member(
    id bigint unsigned NOT NULL AUTO_INCREMENT COMMENT '自增主键',
    room_id bigint unsigned not null, -- 所属聊天室id
    member_type varchar(32) not null, -- 成员类型
    member_id bigint unsigned not null, -- 成员id
    create_time datetime(3) DEFAULT CURRENT_TIMESTAMP(3) COMMENT '创建时间',
    update_time datetime(3) DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '更新时间',
    primary key (`id`)  
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='群聊聊天室成员';
```

- 消息记录

```
create table im_message(
    id bigint unsigned NOT NULL AUTO_INCREMENT COMMENT '自增主键',
    room_code bigint unsigned NOT NULL COMMENT '聊天室code',
    sender_id bigint unsigned NOT NULL COMMENT '发送者id',
    type varchar(32) not null comment '消息类型',
    content varchar(500) not null comment '消息内容',
    ref_id bigint unsigned default null comment '引用的消息',
    state int not null default 0 comment '0-正常, 1-撤回, 2-删除',
    create_time datetime(3) DEFAULT CURRENT_TIMESTAMP(3) COMMENT '创建时间',
    update_time datetime(3) DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '更新时间',
    primary key (`id`)  
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='消息记录';
```

- 未读消息

```
create table unread_message(
    id bigint unsigned NOT NULL AUTO_INCREMENT COMMENT '自增主键',
    message_id bigint unsigned NOT NULL COMMENT '消息id',
    to_id bigint unsigned NOT NULL COMMENT '发送目标人id',
    create_time datetime(3) DEFAULT CURRENT_TIMESTAMP(3) COMMENT '创建时间',
    update_time datetime(3) DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '更新时间',
    primary key (`id`)  
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='未读消息';
```

# 接口

对于客户端给服务端发送信息, 因为需要响应机制, 还是采用http, 服务端给客户端采用websocket。

## websocket

- 连接路径

```
ws/chat
```

因为websocket是服务端和客户端可以互发信息, 所以这里需要区分一下`客户端给服务端发消息`和`服务端给客户端发消息`。

### 客户端 -> 服务端

客户端给服务端发信息, 也就是服务端需要提供的websocket接口, 前面提到了客户端给服务端发信息采用http, 所以这里只需简单提供建立连接和断开连接的接口。

#### 信息格式

- token为用户登录时获取的token, 这里解析token沿用http那一套

- 如果token过期, 需要通知客户端

- path是具体的业务逻辑区分字段, 类似于请求路径
- body 是执行业务函数时传的参数, 类似于请求参数

```
{
    "authorization": token, // token, 根据这个解析用户信息
    "path": "connect/hello", // 根据这个来执行不同的业务逻辑
    "body": { // 参数
        "param1": 1,
        "param2": "hello world"
    }
}
```

#### 接口

|        |            |      |                               |
| ------ | ---------- | ---- | ----------------------------- |
| 功能     | path       | body | 说明                            |
| 建立连接   | connect    | null | - 进入app时尝试建立连接<br>- 网络连接后建立连接 |
| 手动断开连接 | disconnect | null | - 退出app时手动断开连接                |

### 服务端 -> 客户端

服务端给客户端发信息, 也就是客户端需要提供的websocket接口。

#### 信息格式

```
{
    "path": "group/message", // 根据这个来执行不同的业务逻辑
    "body": { // 参数
        "param1": 1,
        "param2": "hello world"
    }
}
```

- path是具体的业务逻辑区分字段, 类似于请求路径
- body 是执行业务函数时传的参数, 类似于请求参数

#### 接口

|   |   |   |   |
|---|---|---|---|
|功能|path|body|说明|
|有新私聊聊天室|```<br>privateRoom/new<br>```|```<br>{<br>  "id": 1,<br>  "code": "unique_chat_code",<br>  "member_type1": "user",<br>  "member_id1": 12345,<br>  "member_type2": "shop",<br>  "member_id2": 67890,<br>  "create_time": "2024-08-06T06:49:29.744676+00:00",<br>  "update_time": "2024-08-06T06:49:29.744676+00:00"<br>}<br>```|- 如果用户发消息前这个聊天室还不存在, 先创建这个聊天室<br>- 返回的是 private_room 表里的一行数据, 字段含义参考"数据库"|
|有新消息|```<br>message/new<br>```|```<br>{<br>    "id": 1,<br>    "room_code": "123456789",<br>    "sender_id": 987654321,<br>    "type": "text",<br>    "content": {"text": "123"},<br>    "ref_id": null,<br>    "state": 0,<br>    "create_time": "2024-08-06T06:40:40.288Z",<br>    "update_time": "2024-08-06T06:40:40.288Z"<br>}  <br>```|- 返回的是 im_message 表里的一行数据, 字段含义参考"数据库"<br>- content里的内容与type有关, 细节参考"消息类型"|
|对方撤回了消息|```<br>message/retract<br>```|```<br>{<br>    "id": 123,<br>    "room_code": "123456789",<br>    "sender_id": 987654321,<br>    "type": "retract",<br>    "content": {<br>        "id": 1<br>    },<br>    "ref_id": null,<br>    "state": 0,<br>    "create_time": "2024-08-06T06:40:40.288Z",<br>    "update_time": "2024-08-06T06:40:40.288Z"<br>}    <br>```|- 返回的是 im_message 表里的一行数据, 字段含义参考"数据库"<br>- content.id为撤回的那条消息id, 客户端需要在那条消息的位置显示"对方撤回了一条消息"或者"你撤回了一条消息, 重新编辑"|
|某一条消息已读|```<br>message/read<br>```|```<br>{<br>    "id": 123,<br>    "room_code": "123456789"<br>}    <br>```|- 返回的是 im_message 表里的一行数据, 字段含义参考"数据库"<br><br>> 避免使用数据库|

## http

|   |   |   |   |   |
|---|---|---|---|---|
|功能|接口|请求参数|响应|说明|
|查询聊天室列表|```<br>/service/im/queryRoomPage<br>```|```<br>page: 1 // 当前页<br>limit: 100 // 每页个数<br>```|```<br>{<br>    page: 1,<br>    size: 100,<br>    total: 1,<br>    data: [<br>        {<br>          "id": 1,<br>          "code": "unique_chat_code",<br>          "member_type1": "user",<br>          "member_id1": 12345,<br>          "member_type2": "shop",<br>          "member_id2": 67890,<br>          "create_time": "2024-08-06T06:49:29.744676+00:00",<br>          "update_time": "2024-08-06T06:49:29.744676+00:00"<br>        }<br>    ]<br>}<br>```|- 有未读消息的聊天室排在最前面|
|新建私聊聊天室|```<br>/service/im/newRoom<br>```|```<br>{<br>    "member_type1": "user",<br>    "member_id1": 12345,<br>    "member_type2": "shop",<br>    "member_id2": 67890<br>  }<br>```||- 请求参数字段含义参考 private_room 表|
|查询聊天室里的未读信息|```<br>/service/im/queryUnreadMessageOfRoom/{roomId}<br>```|```<br>路径/{roomId}<br>```<br><br>```<br>page: 1 // 当前页<br>limit: 100 // 每页个数<br>```|```<br>{<br>    page: 1,<br>    size: 100,<br>    total: 1,<br>    data: [<br>        {<br>            "id": 1,<br>            "room_code": "123456789",<br>            "sender_id": 987654321,<br>            "type": "text",<br>            "content": {"text": "123"},<br>            "ref_id": null,<br>            "state": 0,<br>            "create_time": "2024-08-06T06:40:40.288Z",<br>            "update_time": "2024-08-06T06:40:40.288Z"<br>        }<br>    ]<br>}<br>```||
|查询聊天室里的信息列表|```<br>/service/im/queryMessageOfRoom/{roomId}<br>```|```<br>请求路径/{roomId}<br>```<br><br>```<br>page: 1 // 当前页<br>limit: 100 // 每页个数<br>```|```<br>{<br>    page: 1,<br>    size: 100,<br>    total: 1,<br>    data: [<br>        {<br>            "id": 1,<br>            "room_code": "123456789",<br>            "sender_id": 987654321,<br>            "type": "text",<br>            "content": {"text": "123"},<br>            "ref_id": null,<br>            "state": 0,<br>            "create_time": "2024-08-06T06:40:40.288Z",<br>            "update_time": "2024-08-06T06:40:40.288Z"<br>        }<br>    ]<br>}<br>```|- 一次最多显示500条消息, 当用户上拉时, 优先查询本地存的消息记录, 如果本地没有了, 再去服务端查询。(细节见"上拉刷新")<br>- 因为这里是显示最新的消息, 所以分页不是正序的, 是按照时间分页, 第一页是时间最近的。|
|发送消息|```<br>/service/im/sendMessage<br>```|```<br>{<br>    "room_code": "123456789",<br>    "type": "text",<br>    "content": {"text": "123"},<br>    "ref_id": null<br>}<br>```||- 参数为 im_message 表里的一行数据, 字段含义参考"数据库"|
|撤回消息|```<br>/service/im/retractMessage<br>```|```<br>message_id: 1<br>```|||
|消息已读|```<br>/service/im/readMessage<br>```|```<br>message_ids: [1, 2, 3]<br>```|||
|清除未读|```<br>/service/im/clearUnReadMessage<br>```||||

# 估时

|   |   |   |
|---|---|---|
|功能|人-日|备注|
|websocket建立连接|1-0.5|- 参考"websocket通信"|
|websocket断开连接|1-0.5|- 参考"websocket通信"|
|查询聊天室列表|1-0.5|- 参考"查询未读消息"|
|查询聊天室里的未读信息|1-0.5|- 参考"查询未读消息"|
|发送信息|1-0.5|- 参考"新消息"|
|撤回信息|1-0.5|- 参考"撤回消息"|
|发送消息通知|1-0.5|- 参考"websocket通信"|
|撤回消息通知|1-0.5|- 参考"websocket通信"|
|消息已读|1-1|- 参考"已读未读"|
|新建聊天室|1-0.5|- 用户第一次点进某个聊天室时创建|
|新建聊天室通知|1-0.5|- 参考"websocket通信"|
