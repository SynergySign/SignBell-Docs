## 트러블슈팅: WebSocket 방 퇴장 시 중복 처리 및 예외 발생 문제

* **작성자:** [강관주](https://github.com/Kanggwanju)

-----

### 1. 문제 현상 (Problem)

> 사용자가 퇴장 버튼을 누르면 Controller와 Listener에서 중복으로 퇴장 처리를 수행하여 불필요한 예외가 100% 발생

* **문제 1**: 사용자가 퇴장 버튼 클릭 시, `GameRoomLeaveController`에서 퇴장 처리 후 `SessionDisconnectEvent`가 발생하여 `WebSocketEventsListener`에서 다시 퇴장 처리를 시도함
* **문제 2**: 이미 DB에서 참가자가 삭제된 상태에서 재처리를 시도하여 `PARTICIPANT_NOT_IN_ROOM` 예외가 매번 발생 (예외 발생률 100%)
* **문제 3**: Controller와 Listener가 동일한 퇴장 로직을 수행하여 코드 중복 및 불필요한 DB 쿼리(5회) 발생

<br>

**[문제 상황 흐름도]**

```
1. 퇴장 버튼 클릭
   ↓
2. STOMP 메시지 전송: /app/room/{id}/leave
   ↓
3. GameRoomLeaveController에서 퇴장 처리 (DB에서 참가자 삭제)
   ↓
4. WebSocket 연결 종료
   ↓
5. SessionDisconnectEvent 발생
   ↓
6. WebSocketEventsListener에서 다시 퇴장 처리 시도
   ↓
7. ❌ BusinessException: PARTICIPANT_NOT_IN_ROOM (이미 삭제됨)
```

-----

### 2. 원인 분석 (Analysis)

> WebSocket의 Stateful 연결 특성을 제대로 이해하지 못해 Controller와 Listener가 중복된 역할을 수행

* **원인 1: WebSocket 생명주기에 대한 잘못된 이해**

  * **잘못된 이해**: "퇴장 메시지를 명시적으로 보낸 후 연결을 끊어야 한다"
  * **올바른 이해**: "WebSocket 연결 종료 자체가 퇴장을 의미한다"
  * WebSocket은 HTTP와 달리 Stateful한 지속 연결이므로, 연결 종료 = 세션 종료
  * Spring WebSocket은 연결이 종료되면 `SessionDisconnectEvent`를 자동으로 발생시킴
  * 따라서 별도의 퇴장 메시지(`/app/room/{id}/leave`)를 보낼 필요가 없음

* **원인 2: 컴포넌트 간 역할 중복으로 인한 이중 처리**

  * `GameRoomLeaveController`: STOMP 메시지(`/app/room/{id}/leave`)를 받아 퇴장 처리
  * `WebSocketEventsListener`: `SessionDisconnectEvent`를 감지하여 퇴장 처리
  * 두 컴포넌트가 동일한 `GameRoomLeaveService.leaveCurrentRoomByUser()` 메서드를 호출
  * 첫 번째 호출에서 참가자가 DB에서 삭제되고, 두 번째 호출에서 참가자를 찾지 못해 예외 발생
  * 단일 책임 원칙(SRP) 위반: 하나의 책임(퇴장 처리)을 두 컴포넌트가 나눠 가짐

-----

### 3. 해결 방안 (Solution)

> GameRoomLeaveController를 제거하고 WebSocket 연결 종료만으로 퇴장 처리를 완료하도록 구조 단순화

* **해결 방안 1: GameRoomLeaveController 완전 제거**

  * STOMP 메시지 엔드포인트(`/app/room/{id}/leave`) 제거
  * 퇴장 처리는 오직 `WebSocketEventsListener`에서만 수행
  * WebSocket 연결 종료 = 방 퇴장으로 개념 통일
  * 프론트엔드에서는 `stompClient.deactivate()`만 호출하면 자동으로 퇴장 처리됨

    ```javascript
    // ✅ 변경 후: 연결만 종료
    function handleLeaveRoom() {
        stompClient.deactivate();
        // WebSocketEventsListener가 자동으로 모든 처리 수행
    }
    ```

* **해결 방안 2: 단일 책임 원칙에 따른 명확한 계층 구조 설계**

  * **WebSocketEventsListener**: 이벤트 감지 및 사용자 정보 추출 (진입점)
  * **WebSocketSessionService**: 세션 생명주기 관리 및 검증
  * **GameRoomLeaveService**: 비즈니스 로직 처리 (방 퇴장, 방장 위임 등)

    ```
    [변경 후 구조]
    WebSocketEventsListener (진입점)
        ↓
    WebSocketSessionService (세션 관리)
        ↓
    GameRoomLeaveService (비즈니스 로직)
    ```

    ```java
    // WebSocketEventsListener: 이벤트 감지
    @EventListener
    public void onDisconnect(SessionDisconnectEvent event) {
        DisconnectInfo info = extractDisconnectInfo(event);
        if (info == null) return;
        
        sessionManager.handleSessionDisconnect(info.userId(), info.sessionId());
    }
    
    // WebSocketSessionService: 세션 검증 및 관리
    public boolean handleSessionDisconnect(Long userId, String sessionId) {
        try {
            if (!userSessionRegistry.isActive(userId, sessionId)) {
                log.info("비활성 세션 무시");
                return false;
            }
            handleRoomLeave(userId);
            return true;
        } finally {
            userSessionRegistry.unbind(userId, sessionId);
        }
    }
    ```

* **해결 방안 3: 단일 조회 쿼리로 최적화 및 방장 처리 로직 개선**

  * 기존 5회 쿼리를 3회로 감소 (40% 개선)
  * `findByParticipant_Id()` 단일 조회로 참가자와 방 정보 동시 획득
  * 방장 여부에 따라 처리 분기: 방장이면 위임 또는 방 삭제, 일반 참가자면 퇴장 처리

    ```java
    @Transactional
    public ParticipantEventResponse leaveCurrentRoomByUser(Long userId) {
        GameParticipant participant = participantRepository
            .findByParticipant_Id(userId)
            .orElseThrow(() -> new BusinessException(ErrorCode.PARTICIPANT_NOT_IN_ROOM));
        
        return leaveRoom(participant);
    }
    
    private ParticipantEventResponse leaveRoom(GameParticipant participant) {
        boolean isHost = participant.isHost();
        
        if (isHost) {
            return handleHostLeave(room, participantResponse);
        } else {
            return handleParticipantLeave(participant, room, participantResponse);
        }
    }
    ```

-----

### 4. 교훈 (Lessons Learned)

> WebSocket은 Stateful 연결이므로 연결 종료 자체가 세션 종료를 의미하며, 별도의 명시적 메시지가 불필요함

* **교훈 1: 프로토콜의 근본적 특성을 이해하고 설계하기**
  * WebSocket은 HTTP와 달리 Stateful 지속 연결이므로 연결 종료 = 세션 종료
  * Spring WebSocket의 `SessionDisconnectEvent`가 자동으로 제공되므로 이를 활용
  * "명시적 퇴장 메시지"는 불필요한 복잡도를 추가할 뿐이며, 연결 종료만으로 충분
  * 개선 결과: 코드 33% 감소, DB 쿼리 40% 감소, 예외 발생률 100% → 0%

* **교훈 2: 단일 책임 원칙(SRP)을 준수하여 중복 제거**
  * 하나의 책임(퇴장 처리)을 하나의 진입점(Listener)에서만 처리
  * Controller와 Listener가 동일한 로직을 수행하는 것은 명백한 중복
  * 각 계층의 역할을 명확히 분리: Listener(이벤트 감지) → SessionService(세션 관리) → LeaveService(비즈니스 로직)

* **교훈 3: 단순한 설계가 최선의 설계**
  * "퇴장 메시지 전송 + 연결 종료" 보다 "연결 종료만"이 더 직관적이고 안전
  * 복잡한 설계는 버그와 예외의 온상이 됨
  * 프로토콜의 자연스러운 흐름을 따르는 것이 가장 이해하기 쉽고 유지보수하기 좋은 코드