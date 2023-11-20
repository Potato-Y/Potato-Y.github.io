---
title: "Spring boot + WebSocket 의존성 주입하기"

toc: true
toc_sticky: true

categories:
    - SpringBoot
tags:
    - Java
    - SpringBoot
    - WebSocket
---

# Spring Boot + WebSocket 사용하기
## 기존 문제 파악

작성일 기준 Spring boot (BackEnd) + Android Kotlin (Client) 조합의 프로젝트를 진행하고 있다. 기존에 애너테이션을 활용한 WebSocket 사용이 정상적으로 동작하는 것을 확인하고, 개발을 시작했는데 의존성 주입 문제가 생겼다. 기존 코드는 다음과 같다.

```java
import java.util.Collections;
import java.util.HashSet;
import java.util.Set;

import org.springframework.stereotype.Service;

import jakarta.websocket.OnClose;
import jakarta.websocket.OnMessage;
import jakarta.websocket.OnOpen;
import jakarta.websocket.Session;
import jakarta.websocket.server.ServerEndpoint;

@Service
@ServerEndpoint(value = "/chatt")
public class SocketTestController {

    private static Set<Session> CLIENTS = Collections.synchronizedSet(new HashSet<>());

    @OnOpen
    public void onOpen(Session session) {
        System.out.println(session.toString());

        if (CLIENTS.contains(session)) {
            System.out.println("이미 연결된 세션입니다. :" + session);
        } else {
            CLIENTS.add(session);
            System.out.println("새로운 세션 :" + session);
        }
    }

    @OnClose
    public void onClose(Session session) throws Exception {
        CLIENTS.remove(session);
        System.out.println("세션을 닫습니다. :" + session);
    }

    @OnMessage
    public void onMessage(String message, Session session) throws Exception {
        System.out.println("입력된 메시지입니다. : " + message);

        for (Session client : CLIENTS) {
            System.out.println("메시지를 전달합니다. : " + message);
            client.getBasicRemote().sendText(message);
        }
    }
}
```

의존성 주입을 하려고 ChatGPT에게도 물어보고, 구글링도 많이 해보았지만 딱히 도움 되는 글은 찾지 못하였다.

우선 정확하지 않지만 `jakarta.websocket`과 `Service`의 혼용이 문제였던 것으로 파악하고 있다.

### 대안 방법 모색
표준 WebSocket을 사용함으로써 클라이언트 개발 팀원의 부담을 줄이고 싶었으나 방법이 없다 생각하고 `SockJS` 혹은 `STOMP`로 방향을 변경하려 했다. 다만 `STOMP`는 어째서인지 테스트 툴이 사망을 하였고, 기존 Android 라이브러리도 사망하고 다른 라이브러리가 등장하였다. `SockJS`도 쓸만한 라이브러리는 보이지 않았다. 캡스톤 디자인을 하며 공식 라이브러리를 사용하지 않으면서 발생하는 헬게이트를 경험했기에 다시 `WebSocket`을 사용하는 방법을 찾았고, 좋은 글을 발견했다.

## WebSocket 사용
> 발견한 좋은 게시글은 [여기](https://www.pubnub.com/blog/java-websocket-programming-with-android-and-spring-boot/)를 클릭한다.

하단으로 내려가면 다음의 두 클래스가 있다.
- `WebSocketHandler`
- `WebSocketConfiguration`

이 두 클래스를 활용하면 기존의 문제를 해결할 수 있다.

### 완성된 예시 코드
필자는 `WebSocketHandler`에서 `UserService`를 사용하는 것이 목표다.

`WebSocketHandler`를 다음과 같이 작성한다.
```java
import com.github.dadogk.user.UserService;
import java.io.IOException;
import java.util.Collections;
import java.util.HashSet;
import java.util.Set;
import lombok.extern.log4j.Log4j2;
import org.springframework.web.socket.TextMessage;
import org.springframework.web.socket.WebSocketSession;
import org.springframework.web.socket.handler.AbstractWebSocketHandler;

@Log4j2
public class WebSocketHandler extends AbstractWebSocketHandler {
    private static Set<WebSocketSession> CLIENTS = Collections.synchronizedSet(new HashSet<>());

    private final UserService userService;

    public WebSocketHandler(UserService userService) {
        this.userService = userService;
    }

    @Override
    public void afterConnectionEstablished(WebSocketSession session) throws Exception {
        if (CLIENTS.contains(session)) {
            log.info("이미 연결된 세션입니다. session={}", session);
            return;
        } else {

        }
        CLIENTS.add(session);
        log.info("새로운 세션: session={}", session);
    }

    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) throws IOException {
        String msg = message.getPayload();
        log.info("입력된 메시지: {}", message);

        for (WebSocketSession client :
                CLIENTS) {
            log.info("메시지 전달: {}", msg);
            client.sendMessage(new TextMessage(msg));
        }
    }

}
```
클래스가 생성될 때 `UserService`를 주입받도록 한다.

이제 핸들러를 `WebSocketConfigure`에서 불러온다.
```java
import com.github.dadogk.user.UserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.config.annotation.EnableWebSocket;
import org.springframework.web.socket.config.annotation.WebSocketConfigurer;
import org.springframework.web.socket.config.annotation.WebSocketHandlerRegistry;

@Configuration
@EnableWebSocket
public class WebSocketConfiguration implements WebSocketConfigurer {

    private final UserService userService;

    @Autowired
    public WebSocketConfiguration(UserService userService) {
        this.userService = userService;
    }

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(new WebSocketHandler(userService), "/websocket");
    }
}
```

이제 `WebSocketHandler`에서 `UserService`를 사용할 수 있다.