## 트러블슈팅: WebSocket에서 예외가 클라이언트에 전달되지 않는 문제

* **작성자:** [강관주](https://github.com/Kanggwanju)

---

### 1. 문제 현상 (Problem)

> WebSocket을 통한 게임 방 입장 기능에서 비즈니스 예외 발생 시, 클라이언트가 에러 메시지를 받지 못하고 무한 대기 상태에 빠지는 문제

* **문제 1**: 이미 가득 찬 게임 방(4명)에 입장 시도 시, 서버 로그에는 `ROOM_FULL` 예외가 정상적으로 발생하지만 클라이언트는 아무런 응답을 받지 못함
* **문제 2**: REST API와 동일한 방식으로 `throw new BusinessException()`을 사용했으나, `@ControllerAdvice`가 예외를 처리하지 못함
* **문제 3**: 클라이언트가 에러 응답을 받지 못해 사용자에게 적절한 피드백을 제공할 수 없음

<br>

**[문제 상황 로그]**

> 서버 로그에는 예외가 기록되지만, 클라이언트는 아무것도 수신하지 못함

```bash
# 서버 로그
2024-10-15 14:32:11 WARN  c.e.controller.GameRoomJoinController - Business exception: ROOM_FULL
# 클라이언트는 응답 없음 - 무한 대기 상태
```

**[기대 동작 vs 실제 동작]**

| 구분 | 기대 동작 | 실제 동작 |
|------|----------|----------|
| 서버 | 예외 발생 → 에러 응답 전송 | 예외 발생 → 로그만 출력 |
| 클라이언트 | 에러 메시지 수신 → 알림 표시 | ❌ 아무것도 받지 못함 |

---

### 2. 원인 분석 (Analysis)

> REST API의 예외 처리 메커니즘이 WebSocket에서는 작동하지 않음

* **원인 1: @ControllerAdvice가 WebSocket 메시지 핸들러에서 작동하지 않음**

    * REST API에서는 `@ControllerAdvice`가 자동으로 예외를 잡아 HTTP 응답으로 변환
    * WebSocket의 `@MessageMapping`에서는 `@ControllerAdvice`가 예외를 처리하지 못함
    * 예외가 발생해도 클라이언트로 전달되지 않고 서버 로그에만 남음

    ```java
    // ❌ 잘못된 방식 (REST API 스타일)
    @MessageMapping("/room/{gameRoomId}/join")
    public void joinRoom(...) {
        try {
            gameRoomJoinService.joinRoom(userId, gameRoomId);
        } catch (BusinessException e) {
            throw e;  // 클라이언트에게 전달되지 않음!
        }
    }
    ```

* **원인 2: 프로토콜 차이로 인한 응답 메커니즘의 근본적 차이**

    * **REST API**: HTTP 요청-응답 모델 (1:1 매칭, 응답 대상이 명확함)
    * **WebSocket**: 양방향 메시지 스트림 (N:M 관계, 응답 대상이 불명확함)
    * WebSocket에서는 에러를 누구에게 보낼지(개인/그룹/전체), 어떤 destination으로 보낼지 개발자가 명시적으로 지정해야 함
    * Spring의 `@ControllerAdvice`는 HTTP 요청-응답 모델에 최적화되어 설계되어 WebSocket의 메시지 라우팅 모델과 호환되지 않음

---

### 3. 해결 방안 (Solution)

> WebSocket에서는 예외를 직접 catch하고, SimpMessagingTemplate을 사용하여 명시적으로 에러 메시지를 전송해야 함

* **해결 방안 1: try-catch로 모든 예외를 명시적으로 처리**

    * 모든 비즈니스 로직을 try-catch 블록으로 감싸기
    * `BusinessException`, `NumberFormatException`, `Exception` 등 예상 가능한 모든 예외를 개별적으로 catch
    * 각 예외 타입에 맞는 적절한 `ErrorCode`를 사용하여 에러 응답 생성

    ```java
    @MessageMapping("/room/{gameRoomId}/join")
    public void joinRoom(...) {
        try {
            Long userId = Long.valueOf(subject);
            JoinRoomResponse response = gameRoomJoinService.joinRoom(userId, gameRoomId);
            // 성공 응답 전송
            messagingTemplate.convertAndSendToUser(
                subject, "/queue/room", 
                ApiResponse.success("방에 입장했습니다.", response)
            );
        } catch (BusinessException e) {
            log.warn("Business exception: {}", e.getErrorCode().getCode());
            sendError(subject, e.getErrorCode());
        } catch (NumberFormatException e) {
            log.error("Invalid user ID format");
            sendError(subject, ErrorCode.INVALID_INPUT);
        } catch (Exception e) {
            log.error("Unexpected error", e);
            sendError(subject, ErrorCode.INTERNAL_SERVER_ERROR);
        }
    }
    ```

* **해결 방안 2: SimpMessagingTemplate을 사용한 에러 메시지 직접 전송**

    * `SimpMessagingTemplate`을 주입받아 에러 메시지를 직접 전송하는 `sendError()` 메서드 구현
    * `ErrorResponse` 객체를 생성하여 timestamp, status, error code, detail, path 정보 포함
    * `convertAndSendToUser()`를 사용하여 특정 사용자의 `/queue/errors` destination으로 전송

    ```java
    private void sendError(String userId, ErrorCode errorCode) {
        ErrorResponse error = ErrorResponse.builder()
            .timestamp(LocalDateTime.now())
            .status(errorCode.getStatus())
            .error(errorCode.getCode())
            .detail(errorCode.getMessage())
            .path("/app/room/join")
            .build();
        
        messagingTemplate.convertAndSendToUser(
            userId, "/queue/errors", error
        );
        
        log.warn("Error sent to user {} - code: {}", userId, errorCode.getCode());
    }
    ```

* **해결 방안 3: 클라이언트에서 에러 메시지 구독 및 처리**

    * 클라이언트에서 `/user/queue/errors` destination을 구독하여 에러 메시지 수신
    * 에러 코드별로 적절한 사용자 피드백 제공 (알림, 페이지 이동 등)
    * 에러 로그를 남겨 디버깅 및 모니터링 가능하도록 구현

    ```javascript
    // 에러 메시지 구독
    client.subscribe('/user/queue/errors', (message) => {
        const error = JSON.parse(message.body);
        console.error('에러 발생:', error);
        alert(`[${error.error}] ${error.detail}`);
        
        // 에러 코드별 처리
        switch (error.error) {
            case 'ROOM_FULL':
                navigate('/quiz/rooms');
                break;
            case 'ROOM_ALREADY_STARTED':
                alert('이미 시작된 방입니다.');
                break;
            case 'PARTICIPANT_ALREADY_IN_ROOM':
                alert('이미 다른 방에 참여 중입니다.');
                break;
        }
    });
    ```

---

### 4. 교훈 (Lessons Learned)

> WebSocket과 REST API는 근본적으로 다른 통신 모델을 사용하므로, 예외 처리 방식도 달라야 함

* **교훈 1: 프로토콜의 차이를 이해하고 적절한 패턴 적용하기**
    * REST API의 패턴을 다른 프로토콜에 그대로 적용하면 안 됨
    * WebSocket은 양방향 메시지 스트림이므로 메시지 라우팅을 명시적으로 제어해야 함
    * 각 프로토콜의 특성에 맞는 예외 처리 전략을 수립하고 문서화할 필요가 있음

* **교훈 2: WebSocket에서는 @ControllerAdvice 대신 명시적 예외 처리 필수**
    * `@MessageMapping` 핸들러에서는 모든 예외를 try-catch로 직접 처리
    * `SimpMessagingTemplate`을 사용하여 에러 메시지를 명시적으로 전송
    * 응답 대상(개인/그룹/전체)과 destination을 명확히 지정해야 함

* **교훈 3: 클라이언트-서버 간 에러 처리 프로토콜 정의의 중요성**
    * 에러 메시지 포맷(`ErrorResponse`)과 destination(`/queue/errors`) 사전 정의
    * 각 에러 코드별 클라이언트 동작 시나리오를 문서화
    * 통합 테스트를 통해 에러 플로우가 정상 작동하는지 검증 필요